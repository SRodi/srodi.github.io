---
title: "Deep Dive into Retina Open-Source Kubernetes Network Observability"
date: 2025-05-19 07:00:00 +0100
categories: [kubernetes, ebpf]
tags: [kubernetes, ebpf, prometheus, grafana, containers, go, c] ## always lowercase !!
image:
  path: /retina-logo-poster.webp
  alt: "eBPF distributed networking observability tool for Kubernetes."
---

## Introduction
Microsoft Retina is an open-source, cloud-agnostic network observability platform for Kubernetes clusters. It was designed by the Azure Container Networking team to help DevOps and SecOps teams visualize, debug, and analyze network traffic in containerized applications across diverse environments.

## Project Overview and Goals
Retina’s goal is to provide a centralized hub for monitoring application connectivity, network health, and security posture in Kubernetes – regardless of underlying cloud, on-prem infrastructure, operating system, or CNI (Container Network Interface) plugin. In other words, it aims to “democratize” network observability by making deep network insights available on any cluster, not just those using specialized networking solutions.

Key objectives of the Retina project include:

* **Unified Network Visibility** – Leverage kernel-level instrumentation (eBPF on Linux, analogous mechanisms on Windows) to capture rich telemetry about pod-to-pod traffic, service connectivity, DNS queries, packet drops, and more, all without requiring application changes or sidecar proxies. This gives actionable insights to cluster operators and developers about how microservices communicate and where issues might lie.
* **Environment Agnosticism** – Ensure the solution works with any Kubernetes platform: on Azure, AWS, Google Cloud, on-premises, or hybrid clouds. Retina is CNI-agnostic and supports multiple operating systems (Linux and Windows nodes) out-of-the-box. This broad compatibility addresses the needs of enterprises running multi-cloud or heterogeneous clusters.
* **DevOps & SecOps Use-Cases** – Provide value for both operational performance monitoring (DevOps/SRE) and security/compliance monitoring (SecOps) use-cases. Retina enables on-demand troubleshooting (e.g. packet capture for a failing microservice) as well as continuous monitoring (e.g. metrics and alerts on dropped packets, DNS errors, or suspicious traffic spikes).
* **Open Source Collaboration** – By open-sourcing Retina (under the MIT License), Microsoft invites the community to innovate and contribute plugins, integrations, and new capabilities. The project is intended to evolve through community feedback, with extensibility as a core principle (allowing users to add new elemetry sources or exporters easily). Ultimately, the goal is a robust, community-driven observability toolkit tailored for Kubernetes networking.

Retina was first announced in early 2024 as part of Microsoft’s effort to embrace cloud-native open source. It shares similar goals with other eBPF-based observability tools (e.g. Pixie, Red Hat’s NetObserv) but is unique in its plug-in architecture and tight integration with the Cilium Hubble observability stack while remaining independent of any specific network stack. Next, we’ll dive into Retina’s architecture to see how it achieves these goals.

## System Architecture and Design

![High-level Retina architecture](/retina-arch.webp)
_High-level architecture of the Retina agent’s data plane and control plane._

The data plane (orange, bottom) uses eBPF programs in the kernel to capture network events, which are buffered in maps or perf rings. In user space, a Plugin Manager coordinates multiple plugins (yellow) that retrieve data from eBPF and system events, producing flow records. A set of watchers (yellow) also runs to gather contextual info (e.g. pod interfaces, API server IPs). The output “flow” records are sent either to a local Enricher (green, standard mode) or to an external channel consumed by Hubble (green, Hubble mode), depending on the chosen control plane.

At a high level, Retina’s architecture consists of a **kernel-space data collection layer** and a **user-space processing layer**, with two modes of operation for how data is exported. In simplest terms, the Retina agent on each node collects raw network telemetry from the host kernel and then processes/exports it for consumption by monitoring tools like Prometheus, Grafana, or Hubble UI. The design emphasizes modularity and extensibility, using a plugin system to instrument various kernel events via eBPF. Below, we break down the main components and how they interact:

* **eBPF Data Plane (Kernel Space)**: On Linux nodes, Retina loads a set of custom eBPF programs into the kernel at startup (each provided by a plugin). These programs attach to various hooks (such as socket events, network stack hooks, or tracepoints) to capture relevant events and metrics with minimal overhead. The eBPF code records data into shared memory structures– for example, updating counters in **eBPF maps** (for aggregate metrics) or writing event records into a **perf ring buffer** (for streaming detailed events). This approach allows Retina to observe network traffic (and even OS kernel events like drops or DNS lookups) without running an agent inside every pod. In fact, one eBPF probe per node can monitor all pods on that host, eliminating the need for per-container sidecars or instrumentation libraries. (On Windows nodes, where eBPF is not available, Retina uses Windows networking APIs like HNS and VFP to gather similar data— more on Windows support later.)

* **Plugin Manager and Plugins (User Space)**: In user-space, the Retina *Plugin Manager* initializes and controls a set of **plugins** (each implemented as a Go module). Each plugin has a focused responsibility (for example, measuring packet drops, tracking DNS queries, or counting traffic on an interface). On startup, each plugin may set up necessary kernel hooks (loading its eBPF program and related maps) and then begin collecting data. The plugin reads metrics or events from the eBPF maps/perf buffers and converts them into a unified **flow record** format. (Retina adopts Cilium’s “flow” data structure for consistency, which includes fields like source/destination IPs, ports, protocols, byte/packet counts, timestamps, etc., plus any plugin-specific data such as DNS query names or drop reasons.) The plugin manager ensures all enabled plugins run and can also **reconcile** plugins (e.g. reload eBPF programs if needed). This pluggable design makes Retina easy to extend – if a new telemetry source is required, a developer can add a new plugin implementing the standard interface. Some **built-in plugins** in the latest version of Retina include: **Drop Reason** (counts packets/bytes dropped and categorizes the drop reason and direction), **DNS** (tracks DNS request/response counts, error codes, and response IPs), **Packet Forward** (measures traffic throughput on each node’s main interface), among others. Additional plugins can be created to capture new metrics without altering the core agent.

* **Watchers and Kubernetes Context**: Alongside plugins, Retina runs *watchers* (via a Watcher Manager) to collect Kubernetes cluster metadata that’s essential for enriching the raw network data. For example, an **Endpoint Watcher** monitors the creation/deletion of pod network interfaces (veth pairs) on the node and emits events when pods come and go. A **Kubernetes API Watcher** monitors the API server’s advertised IP addresses. These watchers populate an in-memory **IP-to-object cache** that maps IP addresses to pod names, namespaces, node names, labels, etc. By maintaining this cache of Kubernetes objects keyed by IP, Retina can later join the network flow data with relevant context (e.g. “10.3.2.5 = Pod X in Namespace Y”) for more meaningful observability. This enrichment is done in real-time as flows are processed, and it enables cluster-wide views like “traffic by namespace” or “top talkers by service.” (Under the hood, to scale in large clusters, Retina can reduce the load on the API server by using a custom lightweight CRD called **RetinaEndpoint** that mirrors just the needed pod info. This avoids watching all Pod metadata and instead tracks only the minimal fields needed for network mapping, improving efficiency for big deployments.)

### Control Plane

A unique aspect of Retina is that it can operate in one of two **control plane** modes for processing and exporting the collected flow data. The choice can be configured at deployment and determines how the data flows after the initial capture by plugins.

#### Hubble Control Plane (for Linux)

In this mode, Retina effectively integrates with **Cilium’s Hubble** observability stack. Instead of handling all data internally, the Retina plugins write flow records into an **external channel** (a shared memory ring) that a **Monitor Agent** process watches. The Monitor Agent (a component in Retina’s code, not to be confused with Cilium’s agent) acts as a bridge – as soon as new flow events are detected in the channel, it forwards them to a local **Hubble Observer** (which is essentially Hubble’s core engine running alongside Retina). The Hubble Observer component in turn parses the flows (using Hubble’s own set of parsers for L4 flows, DNS events, drop events, etc.) and *enriches* them further with Kubernetes context (using Cilium libraries and the Retina IP cache).

![Retina Hubble Control-Plane](/retina-hubble-control-plane.webp)
_Retina running in Hubble Control Plane mode on a Linux cluster. The green components are Retina’s integration pieces, interfacing between the data plane (orange) and Hubble (blue). The Monitor Agent captures flows from Retina’s external channel and feeds them to the Payload Parser and Hubble Enricher, which use Cilium libraries (pink) and Retina’s IP cache to attach Kubernetes context. The Hubble stack (blue) then provides a Flow Buffer, generates Prometheus metrics (served on port 9965), and allows a Hubble Relay (for multi-node aggregation via port 4244) and Hubble UI/CLI to retrieve flow logs. This mode effectively turns Retina into a “CNI-agnostic Hubble” data source._

The enriched flow data can then be used to produce Hubble metrics and logs, just as if Cilium were present. In fact, when running in Hubble mode, each Retina node will expose:

1. **Prometheus metrics** on the standard Hubble metrics port (TCP **9965**), labeled as hubble_*metrics. These include node-level stats like forwarded packets, drops (with reasons), TCP/UDP connection metrics, as well as pod-level metrics such as DNS query rates and API server latency.

2. **Flow logs** accessible via Hubble’s APIs – e.g. a local UNIX domain socket (at `/var/run/cilium/` hubble.sock) for node-specific flow queries, and port **4244** for cluster-wide flows (which
Hubble Relay can connect to in order to aggregate flows from all nodes). This means tools like the Hubble CLI or Hubble UI can connect to Retina-powered nodes and retrieve live flow data as if Cilium were providing it.

In essence, Retina’s Hubble mode brings the **full power of Hubble** (flow visibility, Hubble UI, CLI, and metrics) **to clusters that are not running Cilium**. It uses Retina as the data source and Hubble as the presentation layer. For AKS users, this feature is part of “Advanced Network Observability,” allowing Hubble’s interface to work even on Azure CNI or other CNIs by using Retina as the eBPF data
plane. One important note: the Hubble control plane requires Linux (since Hubble and Cilium libraries are Linux-based). Therefore, on Windows nodes Retina cannot run in Hubble mode – those will default to the standard mode (described next).

#### Standard Control Plane

In this mode, Retina handles the entire flow processing pipeline itself (without using Hubble). The plugins send their flow records to an **internal Enricher** component (instead of to an external channel). The Enricher in standard mode takes each flow record and attaches the Kubernetes metadata from Retina’s cache (pod name, namespace, etc.), similar
to what the Hubble Enricher would do. After enrichment, the flows are passed to a built-in **Metrics Module**. The Metrics Module aggregates and converts these flows into Prometheus
metrics, which the Retina agent then exposes on a metrics HTTP endpoint (by default, on each node). These metrics have standard Prometheus formats (counters, histograms, etc.) for various networking stats. For example, Retina exports metrics like `retina_packets_forwarded_total`, `retina_packets_dropped_total` (with drop reason labels), `retina_dns_request_total` (by query type/outcome), and so on – broken down by node or by pod depending on configuration. Operators can scrape these metrics with Prometheus and visualize trends on Grafana dashboards or set up alerts. There is no Hubble-specific output in this mode (no flow API or hubble CLI usage), but it covers most monitoring needs and works on **all platforms** (Linux *and* Windows).

![Retina Standard Control-Plane](/retina-standard-control-plane.webp)
_A simplified architecture of Retina Standard Control Plane, note the enricher is responsible to add Kubernetes metadata to flow object._

The Standard mode is useful for clusters where you prefer a simpler deployment (just metrics to Prometheus) or need Windows node support. However, it offers slightly less detail than Hubble (for instance, Hubble’s flow logs and some advanced metrics may not be available in standard mode). In fact, the Retina maintainers have noted that Hubble mode provides additional features and metrics not in standard mode, and **the plan is to eventually deprecate the standard control plane in favor of Hubble** as the project matures. This suggests that in the future, we may see deeper integration and possibly Windows support via Hubble if eBPF for Windows or similar capabilities become available. For now, both modes coexist – ensuring all users have an option regardless of their environment.


### Data Plane

The data plane in Retina is responsible for the actual collection and processing of network telemetry data. It utilizes eBPF programs to capture and analyze network traffic at the kernel level, providing high-performance and low-overhead observability.

1. **eBPF Programs**: Retina uses eBPF programs to hook into various points in the kernel's networking stack. These programs are written in a restricted C-like language and are compiled into eBPF bytecode, which is then loaded into the kernel. The eBPF programs are responsible for capturing network packets, extracting relevant metadata, and storing this information in eBPF maps.

2. **eBPF Maps**: eBPF maps are key-value stores that are used to share data between eBPF programs and user-space applications. Retina uses these maps to store network telemetry data, such as packet headers, flow records, and connection tracking information. The data stored in these maps can be accessed and processed by user-space applications for further analysis and visualization.

3. **Data Aggregation**: The data plane in Retina supports different levels of data aggregation, allowing users to control the granularity of the collected telemetry data. This can range from low-level packet captures to high-level flow records, depending on the use case and performance requirements.

#### Plugins Model

Retina's plugins model is designed to be highly extensible, allowing users to customize and extend the functionality of the observability platform. Plugins are responsible for processing the data collected by the data plane and exporting it to various storage and visualization backends.

1. **Plugin Lifecycle**: The lifecycle of a Retina plugin can be summarized as follows:
  * **Initialize**: During the initialization phase, the plugin sets up the necessary eBPF maps, sockets, and other resources required for data collection and processing.
  * **Start**: In the start phase, the plugin begins reading data from the eBPF maps and arrays, processing this data, and sending it to the appropriate location based on the control plane configuration.

2. **Plugin Interface**: Retina plugins implement a common interface, which defines the methods and data structures used for data collection and processing. This interface ensures that plugins can be easily integrated into the Retina framework and can work seamlessly with the data plane.

3. **Data Export**: Depending on the control plane being used, the data processed by the plugins can be sent to various destinations. For example, it can be sent to a Retina Enricher for further processing or written to an external channel consumed by a Hubble observer. Plugins can also use syscalls or other API calls to interact with the kernel and collect additional telemetry data.

4. **Extensibility**: Retina's plugins model is designed to be highly extensible, allowing users to develop custom plugins to address specific use cases. This extensibility is achieved through a well-defined plugin interface and a modular architecture that supports the integration of third-party plugins.

Retina's BPF implementation leverages eBPF for distributed network observability in Kubernetes, focusing on non-intrusive data collection and a modular plugins architecture.

#### BPF Implementation

Retina uses eBPF probes attached to kernel hooks to monitor network activity without modifying applications or requiring sidecars. Key features:

* **Dual-stack instrumentation**: Collects metrics from both Kubernetes API server interactions and kernel-level packet processing
* **Low-overhead design**: Operates with <1% CPU/memory footprint per node through optimized BPF verifier configurations
* **Windows compatibility**: Implements alternative data collection for Windows nodes while maintaining Linux eBPF core

Retina's plugin architecture enables extensible data processing.

1. **Pluggable BPF Programs**: Each plugin loads/unloads specific BPF probes dynamically
2. **Shared Maps**: Uses BPF maps (hash, array) for cross-plugin data sharing
3. **Conditional Attachment**: Plugins enable hooks only when needed (e.g., DNS monitoring activates on port 53)

#### BPF Hook Points

Retina attaches probes at critical networking stack layers:

| Hook Point              | Layer            | Use Case                        |
|-------------------------|------------------|----------------------------------|
| XDP                     | Driver           | High-speed packet drops counting |
| TC Ingress              | Network Stack    | Connection tracking & latency    |
| TC Egress               | Network Stack    | Packet transmission metrics      |
| socket/connect          | Application      | DNS resolution tracking          |
| kprobe/tcp_v4_connect   | Kernel           | TCP connection establishment     |

Key implementation details:

* TC Hook Chaining: Uses TC_ACT_OK to preserve existing CNI chain integrations
* CO-RE Compliance: Leverages libbpf's BPF CO-RE for cross-kernel compatibility
* Per-Pod Filtering: Uses cgroup IDs to isolate metrics per workload

The architecture enables granular observability while maintaining cloud-agnostic operation through its plugin-defined BPF attachment strategy.

#### Integration with Control Plane

Retina's data plane and plugins model are designed to work seamlessly with the control plane, which is responsible for managing the configuration and operation of the observability platform. The control plane provides a centralized hub for monitoring application health, network health, and security, and it integrates with various storage and visualization backends to provide a comprehensive observability solution.

In summary, Retina's BPF implementation leverages the power of eBPF to provide a high-performance and low-overhead networking observability platform for Kubernetes environments. The data plane and plugins model are designed to be highly extensible, allowing users to customize and extend the functionality of the platform to address specific use cases.

### Architecture Design Priorities

A few design aspects are worth highlighting:

* **Extensibility**: From the outset, Retina was built to be modular. Adding new **data sources** (e.g. a plugin for a new protocol) or new **export integrations** (e.g. sending data to a different metrics backend) is intended to be straightforward. This is reflected in the clean separation of plugins and the control plane interface – the core data plane always produces flow records, and new consumers can be written to handle those flows. For example, organizations could write custom exporters if they want to feed Retina data into systems like Elasticsearch or Splunk, thanks to this extensibility. The plugin interface is documented and welcomes contributions for new metrics.

* **Low Overhead**: By leveraging eBPF in the kernel, Retina avoids heavy per-packet processing in user space. eBPF runs in a sandboxed VM in kernel context and efficiently aggregates events, meaning the overhead on each node is minimal (the Azure team reports a minimal CPU/memory footprint even at scale). The metrics collection is on-demand and configurable – you can disable plugins you don’t need, and enable advanced metrics only when necessary. This design ensures that even large clusters can run Retina continuously without significant performance impact. The use of Prometheus for metrics means storage and querying are offloaded to a robust external system, and the on-node agent does not need to store large amounts of data (except small ring buffers and caches in memory). Packet captures (which are heavier) are explicitly on-demand to avoid any continuous cost when not in use.

* **CNI/OS Independence**: Unlike some tools that tie deeply into a specific CNI or kernel build, Retina operates at the generic kernel level and uses standard hooks. This means it does *not require Azure CNI* or any particular data plane – it **works with Calico, Flannel, Azure CNI, AWS VPC CNI, Cilium, or any other** because it is monitoring actual kernel events rather than CNI-specific logs. Similarly, by providing a Windows implementation, Retina recognizes that modern
Kubernetes clusters can have Windows worker nodes (for .NET or Windows container workloads), and it provides observability for those as well. On Windows, eBPF isn’t available,
so Retina uses alternate means (the Host Networking Service and Virtual Filtering Platform in Windows Server) to collect metrics. The metrics on Windows include things like bytes/packets forwarded and basic flow counts, though at present Windows does not support all the deep packet tracing features that Linux eBPF does (and as noted, Hubble integration is Linux-only at the moment).

In summary, the architecture of Retina is a carefully layered system: **plugins and eBPF** capture raw data; **watchers and caches** add contextual metadata; then either the **Hubble observer** or Retina’s own **enricher/metrics module** turn that into user-facing metrics and logs. This design achieves a powerful outcome: you get detailed network insight (down to specific flows or failed connections) in a cloud-neutral way, with the choice of consuming the data via industry-standard tools (Prometheus/Grafana) or via the rich Hubble UI/CLI experience – all from the same underlying data. Next, we’ll look at how one would deploy Retina and use it in practice.

## Deployment and Typical Usage Patterns

Retina is built to be Kubernetes-native and is typically deployed as a DaemonSet (one Retina agent Pod per node) plus some optional supporting components. The project provides a Helm chart to simplify installation.

The following commanand is installing Retina with Standard control-plane and Basic metrics mode.

```sh
VERSION=$( curl -sL https://api.github.com/repos/microsoft/retina/releases/latest | jq -r .name)
helm upgrade --install retina oci://ghcr.io/microsoft/retina/charts/retina \
    --version $VERSION \
    --namespace kube-system \
    --set image.tag=$VERSION \
    --set operator.tag=$VERSION \
    --set logLevel=info \
    --set enabledPlugin_linux="\[dropreason\,packetforward\,linuxutil\,dns\]"
```


A typical workflow to get started is:

1. **Installation via Helm**: You can install Retina from the official container registry (GHCR) using Helm. For example, one can fetch the latest version number and run
`helm upgrade --install retina ghcr.io/microsoft/retina/charts/retina ...` with appropriate values. The Retina Helm chart will deploy the **retina-agent DaemonSet** in (for example)
the `kube-system` namespace, scheduling a pod on every node (Linux and optionally Windows nodes). You can configure the release via Helm values – e.g. enabling Windows support
( `--set os.windows=true` ), choosing the control plane mode ( `standard` vs `hubble` ), and enabling an operator component if you plan to use CRD-driven captures ( `--set operator.enabled=true` ). By default, the DaemonSet’s pods run with the necessary privileges to attach eBPF programs (on Linux) or access low-level network info (on Windows). They also expose a metrics endpoint on each pod (by default on port **10093**) and are annotated so that Prometheus can autodiscover and scrape these metrics.

2. **Integrating with Monitoring Tools**: After deploying the Retina agents, you will typically set up a Prometheus instance to gather the metrics. In many Kubernetes setups, a Prometheus Operator or other stack is already present – you just ensure that Prometheus is scraping the Retina metrics port on each pod (the Helm chart’s annotations facilitate this). Grafana can then be used to visualize the data. The Retina project provides example dashboard configurations; for instance, a Grafana dashboard (available via Grafana’s dashboard repository) can be imported to show cluster-wide drop rates, DNS errors, top talkers, etc. If using Azure, you have the alternative of using Azure Monitor’s managed Prometheus and Grafana services to automatically scrape and visualize Hubble/Retina metrics. In **Hubble mode**, you might additionally deploy a Hubble Relay (for multi-node aggregation) and the *Hubble UI if you want a graphical service dependency map* or to query flows visually. In **standard mode**, you would rely on *Grafana for dashboards*, since flow logs are not exposed – though you can still use the kubectl retina trace commands to get some info (more on CLI below).

3. **Retina CLI (kubectl plugin)**: Retina offers a CLI that integrates with kubectl via the Kubernetes SIG Krew plugin system. By installing the plugin (`kubectl krew install retina`), you gain a new command `kubectl retina ...` for
interacting with Retina. The CLI can be used to manage captures, check the status of the Retina system, and even launch an interactive debug shell. For example, after installation you can run kubectl retina version to check connectivity, or `kubectl retina capture create ...` to initiate a packet capture. The CLI is essentially a convenience that wraps Kubernetes API calls (it will create and manage the custom resources under the hood). It’s worth
noting that the CLI currently requires a Linux environment for full functionality (since it expects to interact with Linux-based endpoints for things like the debug shell). Typically, you would run it from a Linux machine or cloud shell that has kubectl access to the cluster.

4. Using **Metrics Mode** (Continuous Monitoring): Once installed, Retina immediately begins collecting metrics. Out-of-the-box it will collect a default set of Basic metrics on each node (packets, bytes, drops, etc., aggregated by node). You can optionally enable more granular
metrics:
  * **Advanced Pod-level metrics (remote context)**: which track source↔destination pod pairs (this provides very detailed flow metrics but can produce a high volume of time series in large clusters).
  * **Advanced Pod-level metrics (local context)**: which only attribute traffic to the local pod (source for outbound, dest for inbound), reducing cardinality and allowing you to focus on particular pods by adding a Retina annotation to them.
These modes are configurable via the `MetricsConfiguration` CRD or Helm values, and you would choose based on your scale and needs. In practice, many users start with basic cluster-wide metrics (sufficient for alerts like “node X is dropping packets” or “overall DNS error rate spiked”) and selectively enable pod-level metrics for critical workloads or during troubleshooting sessions. All metrics collected are exposed as Prometheus metrics on each node, which Prometheus scrapes periodically. You can set up **Prometheus alerting rules** for certain Retina metrics – e.g. an alert if any pod experiences sustained packet drops, or if API server latency exceeds a threshold (Retina measures the latency of Kubernetes API calls from each node). Over time, you might integrate these metrics into your broader monitoring dashboards alongside application metrics.

5. On-Demand Packet Captures (Troubleshooting Workflow): One of Retina’s most powerful features is the ability to trigger **distributed packet capture** jobs when deeper network
debugging is needed. Instead of manually ssh’ing to nodes and running tcpdump (which is tedious and often impossible in managed k8s), you can let Retina orchestrate it. There are two
ways to initiate a capture:
  * **Via CLI**: Use kubectl retina capture create with appropriate flags. For example, you might run: `kubectl retina capture create --name mycapture --pod-selectors app=my-service --duration 2m --blob-upload <SecretContainingSAS>` to capture 2 minutes of traffic for pods with label `app=my-service`, and have the **pcap files** uploaded to a cloud storage blob. The CLI will translate this into a Capture custom resource and create a Kubernetes Job on each relevant node to perform the capture. You’ll see output confirming that capture jobs have been created on the selected nodes. When the duration elapses, each job will terminate and either store the capture file in the specified location (Azure Blob, Amazon S3, a PVC, or simply a hostPath on the node) according to the output settings you provided. The CLI will also tell you where the files went and remind you to clean up the jobs and temporary secrets afterwards.
  * **Via CRD**: Alternatively, you can create a `Capture` YAML manifest and apply it with `kubectl apply`. The Capture CRD object encapsulates all the capture parameters (selectors for nodes/pods, filters, duration, output location, etc.). When you apply this, the Retina operator (if installed) will see the new Capture object and then create the Kubernetes Job(s) to execute the capture on the appropriate nodes. This declarative approach is useful for automation or integrating captures into a workflow (for example, triggering a capture via GitOps or as part of a CI/CD diagnostic step).
In both cases, under the hood each capture is executed by running a privileged pod (using the `retina-agent` container image) on the target nodes which attaches an **eBPF pcap tracer** (for Linux, leveraging technology from the Inspektor Gadget project) or uses **Windows Pktmon** for Windows nodes. The result is one or more standard .pcap files containing the raw packets matching your criteria (optionally including packet metadata). These can be downloaded from the blob storage or PVC you configured, and then **analyzed with Wireshark** or other tools offline. The capability to do this in one command cluster-wide (as opposed to manually coordinating multiple node captures) drastically reduces time-to-diagnose for complex network issues.
A typical use-case is: *“Pods A and B can’t communicate; let’s capture all traffic between them across all nodes for 60 seconds”* – which Retina can accomplish with a single CRD or CLI invocation, whereas traditionally an engineer might spend hours hopping between nodes with tcpdump.

6. **Trace and Shell (Advanced Debugging)**: The Retina CLI also offers some experimental but handy debugging commands. For example, `kubectl retina trace` can report the status of
recent captures or the health of Retina components on the nodes. More interestingly, `kubectl retina shell` opens an interactive shell environment for network debugging on a node or pod. This is somewhat akin to a `kubectl debug` but tailored for network issues – it provides an environment where you can run networking tools on the target (though this feature is still marked experimental). Such tools further reduce the friction in troubleshooting by giving cluster admins quick access to diagnostic data in situ.

#### Deployment Workflow

Overall, the typical deployment workflow is:
1. deploy the Retina DaemonSet
2. set up Prom/Grafana, and let it run continuously to monitor network health

When an anomaly is detected (e.g. an alert on high packet drops):
1. use Retina’s CLI or CRDs to drill down
2. maybe switch to a more detailed metrics mode for specific pods
3. run a packet capture
4. query flows via the Hubble CLI/UI if enabled

Retina fits naturally into Kubernetes workflows: it’s packaged and interacted with just like other Kubernetes resources (via kubectl, CRDs, Helm, etc.), which lowers the barrier for adoption among DevOps teams.

## Supported Platforms and Integrations

One of Retina’s selling points is **broad platform support** – it works across different Kubernetes distributions, cloud providers, and operating systems

![Retina Support](/retina-deep-dive.webp)
_Overview of Retina support._

* Kubernetes Distributions: Retina is **Kubernetes distribution-agnostic**. It can be deployed on managed services like Azure Kubernetes Service (AKS), Amazon EKS, Google GKE, or on self-managed
clusters (kubeadm, Rancher, etc.). As long as the cluster is a fairly standard Kubernetes (v1.20+ for CRDs, etc.), Retina can be installed. It does not require any Azure-specific services to function (despite being built by Azure team) – it is not tied to Azure directly; users can run Retina in any Kubernetes instance, on-premises or in AWS, Azure, or GCP. This
flexibility is important for hybrid-cloud scenarios and aligns with the multi-cloud goal.

* Operating Systems: Both **Linux and Windows** node types are supported in the latest Retina versions. On Linux, Retina uses eBPF for deep kernel telemetry. It supports a variety of Linux
distributions (Ubuntu, Debian, Mariner (Azure Linux), etc.) as long as the kernel is recent enough to support required eBPF features. (Azure’s testing on AKS covers Ubuntu and Azure Linux nodes, for instance.) On Windows, Retina runs a slightly different code path using built-in Windows networking instrumentation (HNS for container networking info and VFP for virtual switch telemetry). This means you can get basic network metrics from Windows containers too (like bytes in/out, flows, etc., and even initiate captures using Windows’ pktmon). Windows support is still evolving – currently Windows nodes only work with the standard control plane (no Hubble), and not every metric is available as on Linux (due to the lack of eBPF). Nonetheless, the inclusion of Windows is a major advantage for shops running mixed-node clusters, ensuring no
blind spots.

* **CNI / Network Stack**: Retina works with any CNI plugin or Kubernetes network implementation. Whether your cluster is using Kubenet, Azure CNI, Calico, Flannel, Cilium, WeaveNet, or others, Retina focuses on capturing traffic at the kernel level, so it doesn’t depend on or interfere with
the CNI’s own components. For example, in a Cilium cluster you could actually use either Cilium’s built-in Hubble or Retina (though you likely wouldn’t run both simultaneously in production to avoid redundant overhead). In a cluster using Azure’s native networking or Calico, Retina steps in to provide the observability that those CNIs lack natively, all without needing any configuration specific to that CNI. This CNI-agnostic approach is a deliberate design to make Retina a drop-in addition to any Kubernetes environment.

* **Cloud Integration**: While Retina runs on any cloud, it provides specific integration points for popular cloud monitoring services. Notably, it can export metrics to Azure Monitor and Azure Log Analytics, which is useful on AKS. In fact, Azure Monitor’s managed Prometheus can scrape Retina metrics, and Azure provides pre-built Grafana dashboards for Retina/Hubble as part of its “Advanced Networking” offerings. Outside of Azure, you could integrate Retina with AWS CloudWatch Container Insights or GCP’s Stackdriver by forwarding metrics, though the primary method in multi-cloud setups is to use Prometheus as a common denominator.

* **Observability Stacks**: Retina was built to play well with existing observability tools:
  1. **Prometheus & Grafana**: As mentioned, all metrics are exposed in Prometheus format. There are Grafana dashboards and PromQL queries available to visualize Retina metrics (for example, dashboards to show a heatmap of packet drops by node, or top 10 services by egress traffic). This allows teams to incorporate network observability into their existing monitoring dashboards, alongside CPU/memory and application metrics.
  2. **Cilium Hubble**: Retina can feed into Hubble – effectively acting as “Hubble for everyone.” If you enable Hubble mode, you can use the Hubble CLI ( `hubble observe ...` ) to query flows, or spin up the Hubble UI to get a live map of communications in your cluster. This is a powerful integration because Hubble provides a user-friendly way to explore network data (e.g. a UI graph of services and their connections, and the ability to drill into specific flow details). **Previously, Hubble’s benefits were mostly limited to clusters running Cilium as the CNI** – Retina breaks that limitation. In non-Cilium clusters, Retina + Hubble give you similar insight without a full Cilium deployment. Many users will appreciate this, as the learning curve for Hubble’s interface is lower than parsing raw metrics or logs.
  ![Hubble CLI](/hubble-cli.webp)
_An example of Hubble CLI accessible via Retina deployed with Hubble control plane._
  3. **Other Exporters**: Retina’s telemetry can be sent to “other vendors” as well . For example, it could push metrics to a third-party APM solution if configured. The plugin architecture also suggests that communities could create exporters for systems like Datadog or New Relic by consuming the flow data and reformatting it. The documentation hints at multiple storage options for telemetry – so far, Prometheus and Azure Monitor are explicitly mentioned, but others can be plugged in.

To sum up, Retina fits into the ecosystem as a **versatile component**: it doesn’t replace your monitoring stack but rather enhances it with network-specific data. You deploy it alongside your existing tools. It’s supported on a wide range of infrastructure, reflecting the reality that large organizations run many flavors of Kubernetes. Whether you run all Linux on Azure, or a hybrid Linux/Windows cluster on-prem, Retina is intended to “just work” and deliver a consistent set of insights. Its integration with Hubble further allows it to leverage a popular open-source UI and CLI, avoiding the need to reinvent those aspects.

## Real-World Use Cases and Scenarios

Retina’s value becomes most clear when examining common scenarios that DevOps, SREs, or network engineers face. Here are a few real-world use cases where Retina provides high value, by simplifying or accelerating what used to be painful processes:

* **Debugging Pod Connectivity Issues**: Scenario: Suddenly, two services that used to communicate (microservice A calling B) can no longer reach each other. Perhaps after a network policy change or an unknown issue, connections time out. The traditional approach involves checking logs, verifying network policy YAMLs, and if needed, running `tcpdump` on each node that might host those pods – a time-consuming and technically challenging process (requires access to nodes and careful filtering). With Retina: This becomes much easier. An engineer can run a one-line **Retina capture** targeting pods A and B. Retina will automatically deploy capture jobs on all nodes where those pods reside and collect packet traces (for example, capturing any TCP SYN packets from A to B and seeing if they get replies). Within minutes, the engineer can retrieve a pcap file to identify if packets are being dropped (and if so, where and why – e.g. no SYN-ACK means the request never reached B, possibly a networking rule issue). Retina can also concurrently show **drop metrics** – for instance, the `retina_drop_reason` metric might show incrementing counts for a specific drop cause on a node, pointing directly to the problem (e.g. packets dropped due to an iptables rule or an MTU issue). By **automating distributed packet capture**, Retina dramatically shortens the feedback loop for network troubleshooting from hours to minutes.

* **Continuous Monitoring of Network Health**: Scenario: You want to ensure the cluster’s networking is healthy over time and get alerted to any anomalies. This includes detecting things like: a spike in DNS failures (which could indicate an external DNS outage or CoreDNS issues), increased API server latency (which might impact all components), or a surge in dropped packets possibly due to misrouting or saturated links. With Retina: All these are exposed as **Prometheus metrics with Kubernetes context**. An operator can set up **alerts** – e.g., “Alert if any namespace sees >100 dropped packets in 5 minutes” or “Alert if `hubble_dns_error_total` > 0 for 10 minutes in the production namespace”. Grafana dashboards can continuously display these metrics per cluster, and one can correlate them with deployments or incidents. For example, Retina measures **DNS query/response counts and error codes**; if a rollout causes a spike in NXDOMAIN errors, you’d see it immediately and could investigate if an application is querying wrong DNS names. Likewise, Retina’s metric for **API server latency** allows cluster operators to notice if the control plane is under stress (perhaps etcd issues) as it starts affecting pods’ ability to communicate with the API server. Another example: you can track **namespace-level traffic** – Retina can tell you how much traffic each namespace is sending/receiving, and even alert if a normally quiet namespace suddenly starts transmitting large volumes (which could be a sign of a compromised pod exfiltrating data). All of this monitoring happens continuously, so you gain **ongoing visibility** into the network internals of your cluster, akin to having a cluster-wide packet watchdog that never sleeps.

* **Security Auditing and Compliance**: Scenario: Your security team wants to ensure that certain sensitive services only talk to expected dependencies and nothing else. Or, during an incident, you need to quickly determine what connections a potentially compromised pod made. With Retina: The combination of **flow logs (in Hubble mode)** and metrics can greatly aid security observations. For instance, using the Hubble CLI, one could run `hubble observe --pod suspicious-pod --last 5m` to get a list of all connections from that pod in the last 5 minutes (assuming Hubble mode enabled). Even without Hubble, one could use Retina’s capture feature in a forensic way: deploy a capture targeting the suspicious pod with a filter to log all its outbound connections over a period. Retina’s data (especially when stored in Azure Monitor or a SIEM) can be used to demonstrate compliance (e.g. show that database pods only communicated within their permitted range, by analyzing flow logs or metrics labels for unexpected targets). Alerting can be set up for security as well – for example, if a usually quiet service suddenly starts connecting to an external IP or transferring a large amount of data Retina’s metrics (like bytes transferred per pod) could trigger an alert. Essentially, Retina provides a network telemetry trail that is invaluable for security reviews and incident response.

* **Multi-Cluster / Multi-Cloud Visibility**: Scenario: A company runs Kubernetes clusters on multiple clouds (say some on AKS, some on EKS). Networking issues can differ per environment and it’s hard to have a consistent tool to monitor all. With Retina: you can deploy the same solution in all clusters and export metrics to a central Prometheus or monitoring system. Since Retina is cloud-agnostic, your dashboards and processes can be standardized. If a network slowdown happens on EKS vs AKS, the Retina metrics are the same format, so your SRE team can quickly compare. In multi-cloud scenarios, Retina offers a unifying layer of network insight that is otherwise missing (as cloud provider-specific tools only work in their own environment). This is aligned with the vision of being multi-cloud from day one.

These scenarios highlight how Retina reduces pain points and investigation time. Many of these tasks (packet capture, flow analysis, cross-cloud monitoring) used to be very hard or impossible without specialized tools. Retina brings them within reach of the average Kubernetes operator using familiar
interfaces. Early users have noted that Retina, Pixie, and similar eBPF tools “aren’t tied to a specific CNI”
and thereby fill a gap, though they caution about running too many such tools at once due to potential conflicts – a topic we’ll discuss as a challenge.

In summary, Retina’s real-world value is in giving engineering teams x-ray vision into their cluster’s networking. From performance optimization (finding the latency bottlenecks) to security (auditing traffic) to plain debugging of outages, Retina accelerates insight. It does so in a way that aligns with how teams already work (metrics, alerts, `kubectl` commands, etc.), which helps drive adoption across cross-functional groups – engineers, DevOps, product managers interested in reliability, and researchers analyzing system behavior can all leverage Retina’s data.

## Contributing to the Retina Project

Retina is an open-source project on GitHub (github.com/microsoft/retina) and welcomes contributions from the community. If you are interested in contributing, here are some key points about the **community and process**:

* **Project Governance and License**: Retina is released under the MIT License (a permissive open-source license), as noted in the repository. Contributions are governed by Microsoft’s standard open-source **Code of Conduct**, meaning all community members should adhere to respectful collaboration guidelines. Before contributing, you’ll be asked to sign a **Contributor License Agreement (CLA)** which affirms you have rights to the contributed code and permit Microsoft to include it in the project. This is a one-time step (the CLA bot will prompt you automatically when you open a pull request).

* **Contributing Guide**: The repository provides a `CONTRIBUTING.md` and documentation on how to set up a development environment. Typically, you would fork the repo, make your changes (perhaps adding a plugin or fixing a bug), test them, and submit a pull request. The maintainers actively review PRs. They have also set up continuous integration (linting, tests) and even security analysis (CodeQL) on the repo, so contributions should pass those checks. New contributors might start with issues labeled as good-first-issue or help-wanted.

* **Community Engagement**: The Retina team has opened channels for community discussion. Notably, they hold **weekly Office Hours** (for example, every Friday at 11:30 AM PST) where anyone can join a call to ask questions, get help, or discuss feature requests. This is a great opportunity to interact directly with the maintainers and other users. Meeting links (often via Teams) are provided in the contributing docs. They also encourage questions by opening issues or using the GitHub Discussions tab for design conversations. You can reach the maintainers via email as well at **retina@microsoft.com**.

### Contribution Areas

* **Code contributions**: Perhaps you want to add support for a new metric or protocol. For example, writing a new eBPF plugin to capture HTTP request stats could be a contribution. Or improving Windows support. All such contributions are welcome – the extensible design is meant to accommodate them.

* **Documentation**: As with any open project, documentation improvements (tutorials, examples, clarifications) are highly valued. The docs are in the repo (and hosted on retina.sh), so you can submit PRs to improve them. Community support: Answering questions from new users, writing blog posts or how-to guides, and sharing use-cases are also valuable contributions. Building a library of real-world knowledge (for instance, how Retina helped in your specific scenario) can help others adopt it.

* **Integration and Testing**: If you try Retina on a platform or with a setup others haven’t, you can contribute by reporting issues or ensuring Retina works in those environments. For instance, test it on the latest Kubernetes version and report compatibility, or integrate it with a new monitoring tool and share the steps.

The **community model** around Retina appears to be friendly and growing. Since it’s a relatively new project (initial release in early 2024), contributors have a chance to significantly influence its direction. Microsoft’s team explicitly stated that by open-sourcing Retina, they “aim to share our knowledge and vision … and hope that Retina will evolve and grow through collaboration with other developers and organizations”. This indicates that the maintainers are eager to accept outside input and make Retina a broad community effort, not just a Microsoft-internal project.

![Retina Commits March 2024 - May 2025](/retina-commits.webp)
_Summary of Retina commits from GitHub repo._

When contributing, be sure to follow the coding style (mostly Go conventions) and any guidelines in `CONTRIBUTING.md`. Also note that all code contributions undergo review and require passing automated tests (so writing tests for new features is encouraged).

In summary, if you’re an engineer or researcher interested in Kubernetes networking, contributing to Retina is a great way to get involved in eBPF and cloud-native observability. The project is open, with regular meetings and active maintainers, making it an inviting open-source community.

## Challenges, Limitations, and Roadmap

While Retina is a powerful tool, it’s important to recognize its current limitations and areas of ongoing development. Being an emerging project, there are challenges to be aware of and exciting improvements on the roadmap:

* **Maturity and Stability**: As of May 2025 Retina is still in a 0.x version (for example, releases are v0.0.30+). It’s under heavy development, which means some features may change or there may be rough edges. Early adopters should be prepared for frequent updates. The fundamentals (metrics and capture) are fairly stable, but as the community grows, best practices on scaling and tuning are still being refined. A likely roadmap milestone will be a 1.0 release once the team and community are confident in stability and have incorporated initial feedback.

* **Dual Control Plane (Transition to Hubble)**: Retina currently supports two modes (Standard and Hubble), which adds complexity in documentation and user choice. The stated plan is to deprecate the Standard control plane in favor of the Hubble control plane going forward. This implies that the project’s roadmap includes making Hubble mode the primary (perhaps only) mode, unifying the codebase around it. The challenge here is ensuring that everything Hubble needs can work without Cilium. There may be tasks such as abstracting any remaining Cilium-specific assumptions in Hubble libraries or developing a solution for Windows compatibility. For now, Windows forces use of Standard mode, so a roadmap question is: Will Retina ever support Hubble on Windows? This likely depends on the progress of eBPF for Windows or alternate approaches. It may be that Windows support will lag behind or have a different observability story until the ecosystem catches up.

* **Windows Limitations**: Current Retina support on Windows, while valuable, is not as feature-rich as on Linux. Windows nodes rely on event tracing and the `pktmon` utility, which means they can’t capture as low-level details as eBPF can on Linux. Also, the absence of Hubble on Windows means no flow logs from Windows pods, only metrics. Organizations using Windows containers should be aware of these limitations: for example, you can get overall traffic counts from Windows pods, but you might not get granular drop reasons or a pcap of Windows traffic (pktmon captures are possible but different in format). The roadmap might include deeper integration with eBPF for Windows (an ongoing Microsoft research project) – if and when that matures, Retina could potentially unify its approach. Until then, a limitation is that mixed OS clusters have a slightly uneven feature set. In practice, this is often acceptable (Windows workloads might be a subset of functions, and you still get metrics for them, just not full Hubble flows).

* **Performance and Scalability**: While Retina is designed for efficiency, one has to be mindful of **metrics cardinality** and resource usage in large clusters. If you enable the most detailed metrics mode (tracking every pod-to-pod pair), the number of time-series in Prometheus could blow up in a large microservices deployment. The Retina docs explicitly warn about cardinality and provide modes to limit it (like local pod context and annotation-based opt-in for metrics). This requires the user to thoughtfully configure Retina based on their cluster size. Similarly, packet capture jobs consume resources and write potentially large files – they should be used sparingly. An operational challenge is ensuring that enabling Retina doesn’t overwhelm your monitoring system: this is more of a Prometheus scaling concern than a Retina bug, but it’s part of using any observability tool. In testing, the overhead on cluster nodes has been minimal, but Prometheus load might increase with lots of metrics. The roadmap could include more dynamic tuning – for instance, Retina might auto-detect large scale and suggest defaulting to lower granularity metrics to protect the user. Additionally, as more plugins are added, they will need to be vetted for performance (eBPF generally is low overhead, but a poorly written plugin could cause issues).

* **eBPF Interference and Kernel Constraints**: A subtle challenge with any eBPF-based tool is that the Linux kernel has a limit on the number and complexity of eBPF programs that can run. If a cluster already has other eBPF programs running (for example, Cilium, Pixie, Falco, BPF-based security agents, etc.), there is potential for **conflicts or resource contention**. A comment from the community noted that as more eBPF tools proliferate, managing them is a concern, and efforts like **bpfman** (a BPF manager) are looking to address this. For Retina users, this means you should audit what other eBPF utilities are in your cluster. Running Retina alongside Cilium (which itself uses eBPF) is generally fine – they hook different parts of the kernel – but running multiple observability eBPF programs that hook the same events could lead to issues or simply higher overhead. This is a new area where best practices are evolving. The Retina team will likely work with the broader community on standards for coexisting (perhaps via that bpfman project or upstream kernel improvements). For now, a limitation to note is: avoid running multiple eBPF-based observability tools for the exact same purpose simultaneously, or do so only after testing.

* **User Experience and UI**: Out of the box, Retina itself doesn’t have a fancy UI (it relies on Grafana for metrics and optionally Hubble UI for flows). Some users might desire a more integrated UI. On the roadmap, one could envision improvements such as a **consolidated dashboard** that comes with Retina to show the key info (maybe an open-source Grafana dashboard is already provided, which counts, but some might want a web UI without Grafana). Given Microsoft’s strategy, they might lean on existing solutions (Grafana, Hubble UI) rather than build something new, but feedback from users will guide this. Another aspect is **integration with Kubernetes UIs** – perhaps in the future, Retina data could surface in Azure’s portal for AKS, or in tools like Lens or Octant via plugins. These are speculative, but show where user experience could be enhanced.

* **Comparison with Alternatives**: It’s worth noting that Retina enters a space with other projects tackling network observability (Cilium/Hubble for Cilium users, Pixie for application-layer telemetry, Weave Scope, etc.). One challenge will be to differentiate and possibly integrate with these. The maintainers seem focused on not re-inventing wheels (hence adopting Hubble). The roadmap might include deeper integration with Pixie or others – for instance, imagine combining Pixie’s application tracing with Retina’s network tracing for a full-stack picture. As of now, such integration isn’t present, but open-source allows the community to bridge these if
desired.

### Future Direction

While not explicitly stated, we can infer some future directions:

* More **plugins/metrics**: e.g. support for HTTP/gRPC tracing at L7, capturing network policy violations as metrics, or integrating with kernel features like connection tracking to report connection churn.
* **Enhanced CRD Automation**: Perhaps additional CRDs to schedule recurring captures (like “capture 1 min of traffic every hour” for audit purposes), or CRDs to configure alerts directly in the system.
* **Edge cases**: Support for things like IPv6 clusters (ensuring all metrics cover IPv6 addresses), dual-stack networking, etc., if not already fully handled.
* **Documentation and Guides**: As adoption grows, expect more guidance for specific scenarios (the official docs are good but more real-world tutorials will likely emerge via the community or Microsoft).

In terms of roadmap transparency, contributors can look at the GitHub issues and discussions where maintainers sometimes tag features for upcoming releases. Microsoft’s announcement blogs and Azure updates also give clues. For example, the Azure blog announcing Retina mentioned that “extensibility was key from the outset and will remain”, indicating a commitment to pluggable design as new use cases arise. Another Azure update blog positioned Retina as part of a larger “Advanced Networking” suite for AKS – implying that Microsoft will continue investing in it and integrating it with Azure services(so AKS customers can expect Retina to become a more native part of the platform over time, possibly with easier enablement via the Azure portal or CLI).

### Summary of Limitations

- Windows support: Only standard metrics, no flow
logs; uses different tech (HNS/Pktmon) which is less feature-rich than eBPF.
- Hubble dependency: Full
functionality on Linux comes with the complexity of running Hubble components; standard mode is simpler but will be phased out.
- Resource tuning: Users need to configure metrics modes wisely to avoid high Prometheus load in very large environments.
- Multi-tool conflicts: Caution when running alongside
other eBPF tools (watch for kernel limits and compatibility).
- Early-stage project: Expect rapid changes and keep an eye on updates for bug fixes and new features; some polish (ease of use, UI integrations) still in progress.

### Outlook

Despite these challenges, the roadmap for Retina is promising. The move toward a single Hubble-based control plane suggests a cleaner, more powerful future state. Community contributions could rapidly add capabilities (for example, someone may contribute support for OpenTelemetry export, or new analytics). Given it addresses a real need in Kubernetes ops, Retina is likely to mature quickly. Microsoft’s backing and use of Retina in Azure AKS offerings means it will get production hardening and long-term support. We can expect regular releases that improve performance and add compatibility with new Kubernetes versions (the team will likely keep it in sync with kernel changes and K8s API changes).

## Conclusions

In conclusion, Retina already offers immense value by shedding light on Kubernetes networks, and with ongoing development, its few gaps are likely to close. Users and contributors can look forward to a more unified, feature-rich Retina – one that might become a de-facto standard for network observability in the cloud-native world. It’s an exciting project to watch (and participate in) as it evolves on the cutting edge of eBPF and cloud networking.

## References

* [eBPF distributed networking observability tool for Kubernetes (GitHub - microsoft/retina)](https://github.com/microsoft/retina)
* [What is Retina? (Retina Docs)](https://retina.sh/docs/Introduction/intro)
* [Retina (Retina Docs)](http://retina.sh)
* [Architecture (Retina Docs)](https://retina.sh/docs/Introduction/architecture)
* [Metrics (Retina Docs)](https://retina.sh/docs/Metrics/metrics-intro)
* [Standard Metric Modes (Retina Docs)](https://retina.sh/docs/Metrics/modes/)
* [Capture (Retina Docs)](https://retina.sh/docs/Concepts/CRDs/Capture)
* [Contributing (Retina Docs)](https://retina.sh/docs/Contributing/overview)
* [Microsoft Azure Introduces Retina: a Cloud Native Container Networking Observability Platform - InfoQ](https://www.infoq.com/news/2024/04/microsoft-retina-observability/)
* [A look at Retina on AKS (Teknews Blog)](https://blog.teknews.cloud/aks/network/2024/06/29/A_look_at_Retina_on_AKS.html)
* [Announcing Advanced Container Networking Services for your Azure Kubernetes Service clusters (Microsoft Azure Blog)](https://azure.microsoft.com/en-us/blog/announcing-advanced-container-networking-services-for-your-azure-kubernetes-service-clusters/)
* [Microsoft open sources Retina: A cloud-native container networking observability platform (Microsoft Azure Blog)](https://azure.microsoft.com/en-us/blog/microsoft-open-sources-retina-a-cloud-native-container-networking-observability-platform/)
* [eBPF-Powered Observability Beyond Azure: A Multi-Cloud Perspective with Retina (Microsoft Azure Blog)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/ebpf-powered-observability-beyond-azure-a-multi-cloud-perspective-with-retina/4403361)