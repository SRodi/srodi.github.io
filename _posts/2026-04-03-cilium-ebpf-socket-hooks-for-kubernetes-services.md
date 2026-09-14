---
title: "How Cilium Uses eBPF Socket Hooks for Kubernetes Service Load Balancing"
date: 2026-04-03 09:00:00 +0100
categories: [cilium, networking]
tags: [kubernetes, networking, cilium, ebpf]
mermaid: true
image:
  path: /ebpf-socket-hooks.webp
  alt: "Kubernetes CNI"
---
## Introduction

When an application connects to a Kubernetes Service, the first useful interception point can occur before a packet exists. Cilium's socket load balancer attaches eBPF programs to cgroup socket-address hooks and can replace a Service virtual IP (VIP) with a backend Pod IP while the kernel is processing `connect()` or `sendmsg()`.

That early translation is only one part of the datapath. Cilium still handles connection tracking, network policy, routing, and optional L7 proxy redirection in packet-processing programs attached at TC or other hooks. Keeping those layers separate is the key to understanding both the performance benefit and the limits of socket-level load balancing.

This article builds that model from Linux sockets upward and finishes with a test you can run on a Cilium cluster.

## Linux Sockets: The Application's Network Endpoint

The `socket()` system call creates a communication endpoint and returns a file descriptor. Its arguments select an address family such as `AF_INET`, a type such as `SOCK_STREAM`, and a protocol such as TCP.

User space works with the file descriptor. Inside Linux, a file references a `struct socket`, which refers to protocol state represented by `struct sock` or a derived type such as `struct tcp_sock`. It is therefore more precise to call a socket a kernel-managed endpoint than to equate it with one structure. Cilium's connection tracking is separate again: it lives in eBPF maps, not in the socket object.

### Client, Server, and Datagram Lifecycles

```mermaid
flowchart LR
    subgraph Server["TCP server"]
        S1["socket()"] --> S2["bind()"] --> S3["listen()"] --> S4["accept()"] --> S5["I/O"] --> S6["close()"]
    end
    subgraph Client["TCP client"]
        C1["socket()"] --> C2["optional bind()"] --> C3["connect()"] --> C4["I/O"] --> C5["close()"]
    end
    subgraph Datagram["UDP socket"]
        U1["socket()"] --> U2["optional bind()/connect()"] --> U3["sendmsg()/recvmsg()"] --> U4["close()"]
    end
```

`listen()` and `accept()` belong to passive stream servers. An unconnected UDP socket can choose a destination for each message. This matters because Cilium uses `connect` hooks for TCP and connected UDP, and `sendmsg`/`recvmsg` hooks for unconnected UDP.

### Quick Local Test

This standard-library Go program shows the local and peer addresses observed at both ends of one TCP connection:

<!-- markdownlint-disable MD010 -->
```go
package main

import (
	"fmt"
	"io"
	"net"
)

func main() {
	listener, err := net.Listen("tcp4", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}
	defer listener.Close()

	serverResult := make(chan error, 1)

	go func() {
		connection, err := listener.Accept()
		if err != nil {
			serverResult <- err
			return
		}
		defer connection.Close()

		fmt.Printf("server socket: %s <- %s\n",
			connection.LocalAddr(), connection.RemoteAddr())

		_, err = connection.Write([]byte("hello from the server\n"))
		serverResult <- err
	}()

	client, err := net.Dial("tcp4", listener.Addr().String())
	if err != nil {
		panic(err)
	}
	defer client.Close()

	fmt.Printf("client socket: %s -> %s\n",
		client.LocalAddr(), client.RemoteAddr())

	message, err := io.ReadAll(client)
	if err != nil {
		panic(err)
	}
	fmt.Print(string(message))

	if err := <-serverResult; err != nil {
		panic(err)
	}
}
```
<!-- markdownlint-enable MD010 -->

Save it as `socket_demo.go` and run it with `go run socket_demo.go`. The ephemeral client port changes, but both endpoints should report the same connection tuple from opposite directions.

## Conventional and eBPF Kubernetes Datapaths

Before looking at socket hooks, it helps to establish what changes when Cilium replaces the conventional Kubernetes Service datapath.

“Non-eBPF” and “non-Cilium” are not exact synonyms. A non-Cilium CNI may also offer an eBPF dataplane, while Cilium can coexist with kube-proxy in configurations where it does not replace Service handling. In this article, **conventional datapath** means kube-proxy implementing Services with Netfilter rules or IPVS, and **Cilium eBPF datapath** means Cilium implementing Services with eBPF programs and maps.

### Conventional kube-proxy Path

In a conventional iptables or nftables deployment, the application connects to the Service VIP and Linux initially constructs a packet for that VIP. The packet reaches Netfilter, where rules installed by kube-proxy select a backend and apply destination NAT (DNAT). Linux conntrack records the translation so replies can be reverse-translated.

```mermaid
flowchart LR
    A["Application"] -->|"connect(Service VIP)"| S["Linux socket"]
    S -->|"packet dst = Service VIP"| N["Netfilter / kube-proxy rules"]
    N -->|"DNAT + backend selection"| C["Linux conntrack"]
    C -->|"packet dst = Pod IP"| B["Backend Pod"]
    B -->|"reply"| C
    C -->|"reverse NAT"| A
```

The CNI plugin still provides Pod interfaces, addresses, and reachability. kube-proxy is the component implementing the Service VIP in this model. They are related parts of cluster networking, but they are not the same component.

### Cilium eBPF Paths

Cilium replaces kube-proxy's Service rules with eBPF maps and programs. It has two relevant opportunities to translate a Service:

1. **Socket-level LB:** a cgroup program rewrites the destination during `connect()` or `sendmsg()`, before Linux constructs a packet.
2. **Per-packet LB:** a TC eBPF program sees a packet addressed to the Service VIP and rewrites it in the packet path. This is Cilium's fallback or companion path for traffic socket LB cannot handle.

```mermaid
flowchart LR
    A["Application"] -->|"connect(Service VIP)"| H{"Socket LB handles it?"}
    H -- Yes --> S["Rewrite socket to Pod IP"]
    S --> P1["Packet created for Pod IP"]
    H -- No --> P2["Packet created for Service VIP"]
    P2 --> T["TC eBPF Service lookup"]
    T --> P3["Rewrite packet to Pod IP"]
    P1 --> D["Cilium policy, CT, and forwarding"]
    P3 --> D
    D --> B["Backend Pod"]
```

Both Cilium paths use eBPF, but only the first is socket-level load balancing. The second is closer in timing to conventional kube-proxy because it acts after packet construction, although its implementation uses eBPF maps and programs rather than a linear Netfilter ruleset.

| Property | Conventional kube-proxy | Cilium per-packet eBPF | Cilium socket eBPF |
| --- | --- | --- | --- |
| First Service decision | Netfilter/IPVS packet path | TC packet path | Socket syscall path |
| Packet initially targets | Service VIP | Service VIP | Backend Pod IP |
| Service state | iptables/nftables rules or IPVS tables | Cilium eBPF LB maps | Cilium eBPF LB maps |
| Flow/NAT state | Linux conntrack/IPVS state | Cilium CT and NAT maps | Socket reverse-LB map; packets still use Cilium CT |
| Translation frequency | Packet path, accelerated by connection state | Packet path, accelerated by Cilium CT | Usually once for TCP; potentially per message for unconnected UDP |
| Network policy | Separate firewall/CNI mechanism | Cilium packet datapath | Still enforced later in Cilium's packet datapath |

This comparison is architectural, not a universal performance ranking. Actual results depend on kernel version, cluster size, enabled policy, routing mode, traffic pattern, and whether traffic can use the socket path.

## eBPF Hooks Around Sockets

A **program type** defines the context and helpers available to an eBPF program. An **attach type** identifies the operation that triggers it.

| Program type | Example attach types | Typical purpose |
| --- | --- | --- |
| `BPF_PROG_TYPE_CGROUP_SOCK` | `BPF_CGROUP_INET_SOCK_CREATE`, `BPF_CGROUP_INET_SOCK_RELEASE` | Control socket creation and release |
| `BPF_PROG_TYPE_CGROUP_SOCK_ADDR` | `BPF_CGROUP_INET4_CONNECT`, `BPF_CGROUP_UDP4_SENDMSG`, `BPF_CGROUP_INET4_GETPEERNAME` | Inspect or rewrite socket addresses |
| `BPF_PROG_TYPE_CGROUP_SOCKOPT` | `BPF_CGROUP_GETSOCKOPT`, `BPF_CGROUP_SETSOCKOPT` | Inspect or change socket options |
| `BPF_PROG_TYPE_SOCK_OPS` | `BPF_CGROUP_SOCK_OPS` | React to TCP events and tune behavior |

Cilium's libbpf section names are more compact:

```c
__section("cgroup/connect4")
int cil_sock4_connect(struct bpf_sock_addr *ctx) { /* ... */ }

__section("cgroup/sendmsg4")
int cil_sock4_sendmsg(struct bpf_sock_addr *ctx) { /* ... */ }

__section("cgroup/recvmsg4")
int cil_sock4_recvmsg(struct bpf_sock_addr *ctx) { /* ... */ }
```

These are section names, not additional BPF program types.

### The `bpf_sock_addr` Context

Socket-address programs receive this kernel UAPI context. Fields prefixed with `user_` represent the address supplied by the application and are writable at supported hooks:

```c
struct bpf_sock_addr {
    __u32 user_family;
    __u32 user_ip4;
    __u32 user_ip6[4];
    __u32 user_port;       /* network byte order */
    __u32 family;
    __u32 type;
    __u32 protocol;
    __u32 msg_src_ip4;
    __u32 msg_src_ip6[4];
    struct bpf_sock *sk;
};
```

`user_port` occupies 32 bits even though a TCP or UDP port is 16 bits. Cilium wraps access to it in helpers and rewrites `user_ip4`/`user_ip6` plus `user_port` after selecting a backend.

## Cilium Socket-Level Load Balancing

With kube-proxy replacement enabled, Cilium maintains Service and backend entries in eBPF maps. A TCP connection to a ClusterIP follows this path:

```mermaid
sequenceDiagram
    participant App as Application
    participant Kernel as Linux connect()
    participant SockBPF as Cilium cgroup/connect4
    participant Maps as Cilium LB maps
    participant TCP as TCP/IP stack
    participant Backend as Backend Pod

    App->>Kernel: connect(Service VIP:port)
    Kernel->>SockBPF: bpf_sock_addr context
    SockBPF->>Maps: Look up Service and select backend
    Maps-->>SockBPF: Backend Pod IP:port
    SockBPF->>SockBPF: Rewrite user_ip4 and user_port
    SockBPF-->>Kernel: Allow operation
    Kernel->>TCP: Create connection to backend
    TCP->>Backend: SYN to backend Pod IP
```

The hook runs **inside the kernel while handling the socket operation, before packet construction**. It does not run before the kernel or outside the networking stack.

### Forward Translation

The IPv4 path in Cilium's `bpf/bpf_sock.c` has this shape:

```c
svc = lb4_lookup_service(&key, true);
if (!svc)
    return -ENXIO;

/* Backend selection and affinity handling are omitted here. */
backend = __lb4_lookup_backend(backend_id);
if (!backend)
    return -EHOSTUNREACH;

sock4_update_revnat(ctx, backend, dst_ip, dst_port, svc->rev_nat_index);
ctx->user_ip4 = backend->address;
ctx_set_port(ctx, backend->port);
```

The current implementation also handles NodePort and HostPort wildcard lookups, session affinity, Local Redirect Policies, IPv4-in-IPv6, and L7 exceptions. The result is that TCP constructs its SYN for the backend rather than the Service VIP.

This avoids Service DNAT and reverse DNAT on every packet in that connection. It does **not** mean the complete route is NAT-free: masquerading or other SNAT may still occur elsewhere.

### TCP and UDP Use Different Hooks

| Traffic | Forward hook | Reverse socket translation |
| --- | --- | --- |
| TCP | `connect4` / `connect6` | Optional `getpeername4` / `getpeername6` handling |
| Connected UDP | `connect4` / `connect6` | Receive/name hooks as configured |
| Unconnected UDP | `sendmsg4` / `sendmsg6` for each destination | `recvmsg4` / `recvmsg6` |

Backend selection normally occurs once for TCP. Unconnected UDP can be translated for every message, so “once per connection” is not a universal property.

Cilium stores socket reverse-translation state in `cilium_lb4_reverse_sk` and `cilium_lb6_reverse_sk`. These maps let relevant socket operations recover the original Service address. They are distinct from Cilium's packet connection-tracking maps.

### Local Redirect Policies

A Local Redirect Policy (LRP) can direct a Service or frontend address to node-local backends. Socket LB recognizes LRP entries and can select those backends in `bpf_sock.c`.

The condition below is instead from the **per-packet** path in `bpf_lxc.c`:

```c
if (CONFIG(enable_lrp) && is_defined(ENABLE_SOCKET_LB_FULL) &&
    unlikely(lb4_svc_is_localredirect(svc)))
    goto skip_service_lookup;
```

It prevents per-packet load balancing from overriding an LRP decision already made by socket LB; it does not implement the redirect itself.

## Where Policy and Connection Tracking Happen

After socket LB selects a backend, Linux constructs packets with that backend as the destination. Those packets still traverse Cilium's endpoint datapath:

```mermaid
flowchart TD
        A["Application calls connect(Service VIP)"] --> B["cgroup socket hook"]
        B --> C["Service lookup and backend selection"]
        C --> D["Linux creates packet for backend"]
        D --> E["TC program on Pod veth"]
        E --> F["Cilium CT lookup"]
        F --> G["Identity and network-policy evaluation"]
        G --> H{"L7 policy?"}
        H -- Yes --> I["Redirect packet to Envoy"]
        H -- No --> J["Route or redirect to backend"]
        I --> J
```

This boundary prevents several common misconceptions:

* **Cilium connection tracking** uses maps such as `cilium_ct4_global` and `cilium_ct_any4_global`; the Linux socket does not own those entries.
* **Network policy** functions such as `policy_can_egress4()` run in packet programs such as `bpf_lxc.c`.
* **L7 redirection** functions such as `ctx_redirect_to_proxy4()` redirect packets to Cilium's Envoy proxy; they are not actions of the `connect4` hook.
* **Packet reverse NAT** in packet LB and NodePort paths is separate from socket reverse translation.

### Why Per-Packet Load Balancing Still Exists

Not every Service can be handled entirely at the socket layer. Current Cilium source retains per-packet load balancing for cases including:

* socket LB restricted to the host namespace;
* SCTP, which Cilium's socket LB does not handle;
* selected L7 load-balancing behavior;
* configurations where full socket LB is unavailable;
* cluster-aware addressing paths requiring packet handling.

Setting `socketLB.hostNamespaceOnly=true` bypasses socket LB in Pod namespaces. This supports service meshes that need to observe the original Service VIP; translation then occurs later in the TC datapath.

## Validate Socket LB on a Cilium Cluster

This test requires a Linux cluster running Cilium with socket LB enabled for Pod namespaces. It creates two HTTP backends, opens a persistent TCP connection to their Service, and inspects the destination Linux actually connected to.

### 1. Confirm Cilium's Configuration

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
kubectl -n kube-system exec ds/cilium -- cilium-dbg config --all \
    | grep -E 'KubeProxyReplacement|bpf-lb-sock|SocketLB'
```

Diagnostic labels can change between releases. The status must report `Socket LB: Enabled`, and socket LB must not be restricted to the host namespace.

An installation created with the Cilium CLI is Helm-backed. Prefer `cilium upgrade` when continuing to manage it with that CLI, and pin the currently installed chart version so this configuration change does not also upgrade Cilium:

```bash
CILIUM_VERSION=$(cilium status --output json \
    | jq -r '.helm_chart_version')

cilium upgrade \
    --version "$CILIUM_VERSION" \
    --reuse-values \
    --set socketLB.enabled=true \
    --set socketLB.hostNamespaceOnly=false \
    --restart \
    --wait

cilium status --wait
```

For an installation managed directly with Helm, the equivalent procedure is:

```bash
CILIUM_VERSION=$(helm -n kube-system list \
    --filter '^cilium$' --no-headers | awk '{print $NF}')

helm repo add cilium https://helm.cilium.io/
helm repo update cilium

helm upgrade cilium cilium/cilium \
    --namespace kube-system \
    --version "$CILIUM_VERSION" \
    --reuse-values \
    --set socketLB.enabled=true \
    --set socketLB.hostNamespaceOnly=false \
    --set rollOutCiliumPods=true

kubectl -n kube-system rollout status daemonset/cilium
kubectl -n kube-system exec ds/cilium -- \
    cilium-dbg status --verbose \
    | grep -E 'KubeProxyReplacement|Socket LB|Socket LB Coverage'
```

`socketLB.hostNamespaceOnly=false` is important for this test: setting it to `true` deliberately bypasses socket LB for ordinary Pods. If Cilium was installed by another lifecycle manager, apply the equivalent values through that manager rather than editing `cilium-config` directly.

Socket LB can be enabled independently, while Cilium's full kube-proxy replacement depends on socket LB. If the goal is also to replace kube-proxy, set `kubeProxyReplacement=true` through the installation manager and follow Cilium's kube-proxy-free migration procedure. Do not blindly enable it on a live cluster that still runs kube-proxy: the two implementations maintain independent NAT state, existing connections can break during the transition, and Cilium must have a directly reachable Kubernetes API server configured before kube-proxy is removed.

### 2. Create the Test Workloads

```bash
kubectl create namespace socket-lb-demo

kubectl -n socket-lb-demo create deployment echo \
    --image=registry.k8s.io/e2e-test-images/agnhost:2.53 \
    --replicas=2 -- /agnhost netexec --http-port=8080

kubectl -n socket-lb-demo expose deployment echo \
    --name=echo --port=8080 --target-port=8080

kubectl -n socket-lb-demo run client \
    --image=nicolaka/netshoot:v0.13 --restart=Never -- sleep 3600

kubectl -n socket-lb-demo rollout status deployment/echo
kubectl -n socket-lb-demo wait --for=condition=Ready pod/client --timeout=120s
```

Record the virtual and backend addresses:

```bash
kubectl -n socket-lb-demo get service echo -o wide
kubectl -n socket-lb-demo get endpointslice \
    -l kubernetes.io/service-name=echo -o wide
```

### 3. Exercise and Inspect the Service

The `/hostname` endpoint identifies the selected backend:

```bash
for request in 1 2 3 4; do
    kubectl -n socket-lb-demo exec client -- \
        curl -fsS http://echo:8080/hostname
done
```

Open a TCP connection without completing an HTTP request, then inspect it:

```bash
kubectl -n socket-lb-demo exec client -- bash -c \
    'exec 3<>/dev/tcp/echo/8080; ss -tnp; exec 3>&-'
```

With full socket LB active for the client Pod, `ss` should show a remote **backend Pod IP**, not the Service ClusterIP. DNS still resolved `echo` to the Service VIP; the socket hook performed the subsequent rewrite. If `ss` shows the ClusterIP instead, verify that `cilium-dbg status --verbose` reports `Socket LB: Enabled` and that socket LB is not restricted to the host namespace.

### 4. Inspect Cilium's eBPF State

Cilium's service and socket maps are node-scoped. Select the Cilium agent on the node that hosts the client Pod so the map and cgroup state correspond to the connection being tested:

```bash
CLIENT_NODE=$(kubectl -n socket-lb-demo get pod client \
    -o jsonpath='{.spec.nodeName}')

CILIUM_POD=$(kubectl -n kube-system get pod \
    -l k8s-app=cilium \
    --field-selector "spec.nodeName=${CLIENT_NODE}" \
    -o jsonpath='{.items[0].metadata.name}')

SERVICE_IP=$(kubectl -n socket-lb-demo get service echo \
    -o jsonpath='{.spec.clusterIP}')

printf 'client node: %s\ncilium pod: %s\nservice IP: %s\n' \
    "$CLIENT_NODE" "$CILIUM_POD" "$SERVICE_IP"
```

First confirm the feature state and inspect the Service frontend and backend entries programmed on that node:

```bash
kubectl -n kube-system exec "$CILIUM_POD" -- \
    cilium-dbg status --verbose \
    | grep -E 'KubeProxyReplacement|Socket LB'

kubectl -n kube-system exec "$CILIUM_POD" -- \
    cilium-dbg bpf lb list \
    | grep -F "$SERVICE_IP"

kubectl -n kube-system exec "$CILIUM_POD" -- \
    cilium-dbg map list \
    | grep -E 'cilium_lb[46]_(services|backends|reverse_sk)'
```

The LB listing confirms that the Service and its backends are present in the node's maps; by itself, it does not prove that this connection used socket LB. When socket LB is enabled, the agent Pod can also inspect the cgroup programs attached on its node:

```bash
kubectl -n kube-system exec "$CILIUM_POD" -- \
    bpftool cgroup tree /run/cilium/cgroupv2 effective \
    | grep -E 'connect[46]|sendmsg[46]|recvmsg[46]|getpeername[46]'
```

Finally, keep a connection open briefly and inspect the socket reverse-NAT map while it is active:

```bash
kubectl -n socket-lb-demo exec client -- bash -c \
    'exec 3<>/dev/tcp/echo/8080; sleep 5' &
SOCKET_HOLDER_PID=$!

sleep 1
kubectl -n kube-system exec "$CILIUM_POD" -- \
    cilium-dbg bpf socknat list

wait "$SOCKET_HOLDER_PID"
```

An active socket-LB connection should add a `Backend -> Frontend` entry that maps the selected Pod backend back to the Service address. A header with no entries means there was no socket reverse-NAT state at the time of inspection; verify that socket LB is enabled, applies inside Pod namespaces, and that the connection remained open. Service maps are normally populated on every Cilium node, so another agent may show the LB entries, but the client-node agent is the correct target for connection-specific cgroup and socket state.

Clean up when finished:

```bash
kubectl delete namespace socket-lb-demo
```

## Summary

* Socket LB moves Service backend selection to socket operations, before packet construction.
* TCP and connected UDP use `connect` hooks; unconnected UDP also needs per-message hooks.
* Socket reverse translation and packet connection tracking are separate map-backed mechanisms.
* Network policy, packet CT, routing, and Envoy redirection remain packet-datapath responsibilities.
* Per-packet load balancing remains necessary for unsupported protocols and fallback paths.
* Avoiding per-packet Service NAT does not guarantee a completely NAT-free route.

The useful mental model is not “Cilium networking happens at the socket.” Cilium makes the Service decision unusually early, then passes packets addressed to the selected backend through its normal security and forwarding datapath.

## References

* **Linux socket API:** [`socket(2)`](https://man7.org/linux/man-pages/man2/socket.2.html), [`connect(2)`](https://man7.org/linux/man-pages/man2/connect.2.html), and [`listen(2)`](https://man7.org/linux/man-pages/man2/listen.2.html)
* **Linux eBPF UAPI:** [`include/uapi/linux/bpf.h`](https://github.com/torvalds/linux/blob/master/include/uapi/linux/bpf.h)
* **Linux cgroup socket-address programs:** [Program type documentation](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_CGROUP_SOCK_ADDR/)
* **Linux cgroup socket-option programs:** [Kernel documentation](https://docs.kernel.org/bpf/prog_cgroup_sockopt.html)
* **Kubernetes virtual IPs and Service proxies:** [Official documentation](https://kubernetes.io/docs/reference/networking/virtual-ips/)
* **Kubernetes CNI plugins:** [Official documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
* **Netfilter connection tracking:** [Linux kernel documentation](https://docs.kernel.org/networking/nf_conntrack-sysctl.html)
* **Cilium kube-proxy replacement:** [Official documentation](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
* **Cilium Local Redirect Policy:** [Official documentation](https://docs.cilium.io/en/stable/network/kubernetes/local-redirect-policy/)
* **Cilium eBPF maps:** [Official documentation](https://docs.cilium.io/en/stable/network/ebpf/maps/)
* **Cilium life of a packet:** [Official documentation](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)
* **Cilium socket datapath:** [`bpf/bpf_sock.c`](https://github.com/cilium/cilium/blob/main/bpf/bpf_sock.c)
* **Cilium endpoint packet datapath:** [`bpf/bpf_lxc.c`](https://github.com/cilium/cilium/blob/main/bpf/bpf_lxc.c)
