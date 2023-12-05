---
title: "Configure a local Docker container image registry"
date: 2023-10-13 09:00:00 +0100
categories: [DevOps, Docker]
tags: [docker, kubernetes, minikube] ## always lowercase !!
image:
  path: /assets/img/post/intro-image-1.png
  alt: "Local container registry for Kubernetes related development"
---

## Introduction
Open Source development for large projects often requires building docker images and pushing images to a registry to allow for local testing in a Kubernetes environment.

One possible solution to avoid configuring authentication to the registry with the imagePullSecrets for each image is to deploy a local container registry which is accessible from both Kubernetes and host development machine.

> This blog post is based on trisberg's Gist [Using a Local Registry with Minikube](https://gist.github.com/trisberg/37c97b6cc53def9a3e38be6143786589) which includes all relevant steps. However, I decided to write a post since I encountered issues with Rancher Desktop. The aim of this post is to provide some additional context and references when required.
{: .prompt-info }

## Requirements
This tutorial is based on macOS and does NOT cover any other operating systems. However, the high-level approach is quite similar for Linux distributions, with the main difference that host development machine file names will most likely differ.

Docker should be installed on the host development machine, this can be done via [Docker Desktop](https://docs.docker.com/desktop/) or [Rancher Desktop](https://docs.rancherdesktop.io/getting-started/installation/) for example.

We will use Minikube as a local Kubernetes clusters. For instructions on how to install Minikube on your host development machine follow the [official Minikube documentation](https://minikube.sigs.k8s.io/docs/start/)

## Tutorial steps
1. Run a container registry locally
2. Start a local Kubernetes cluster
3. Configure the host and cluster VM networking
4. Create a Kubernetes Service and Endpoint
5. Create a test Docker image
6. Test creating a Kubernetes Job

### Run a container registry locally
This step uses Docker CLI to start a container which uses image `registry:2` from Docker. The container will listen on port 5000, and will persist images using a volume mount on local directory `~/.registry/storage`.

```bash
docker run -d -p 5000:5000 --restart=always --volume ~/.registry/storage:/var/lib/registry registry:2
```

On your host development machine edit the `/etc/hosts` file to add the name `registry.dev.svc.cluster.local` on the same line as `localhost` entry. Later we will create a Kubernetes Service and necessary host networking configuration so that name `registry.dev.svc.cluster.local` will resolve to the cluster IP of the Service. See [Kubernetes docs: DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

Validate the container registry is running on the host development machine

```bash
❯ docker ps | grep registry
5ee4531b4a93   registry:2  "/entrypoint.sh /etc…"   About an hour ago   Up 54 minutes   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp stoic_chatelet
```

Test the registry is accessible from the host development machine by sending an HTTP request with cURL. See [Docker Registry HTTP API V2 Specifications](https://distribution.github.io/distribution/spec/api/) for additional available endpoints.

```bash
❯ curl registry.dev.svc.cluster.local:5000/v2/_catalog
{"repositories":[]}
```

Configure `insecure registry` for Docker daemon by modifying the `~/.docker/daemon.json` as follows:

```json
{
  "insecure-registries": ["registry.dev.svc.cluster.local:5000"]
}
```

> If you are using Rancher Desktop you will have to ssh into Rancher Desktop VM and edit the config file `/etc/conf.d/docker` setting `DOCKER_OPTS="--insecure-registry=registry.dev.svc.cluster.local:5000"`. See [this GitHub issue comment](https://github.com/rancher-sandbox/rancher-desktop/discussions/1477#discussioncomment-2106389) with the instructions. Remember to stop and restart Rancher Desktop after updating `/etc/conf.d/docker`
{: .prompt-warning }

### Start a local Kubernetes cluster
To test the local registry, start a local Kubernetes cluster with access to the `insecure registry`. This can be done as follows with Minikube

```bash
minikube start --insecure-registry registry.dev.svc.cluster.local:5000
```

Make sure all pods in `kube-system` namespace are running fine

```bash
❯ k get po -A
NAMESPACE     NAME                               READY   STATUS      RESTARTS      AGE
kube-system   coredns-5dd5756b68-9s8hw           1/1     Running     0             67m
kube-system   etcd-minikube                      1/1     Running     0             67m
kube-system   kube-apiserver-minikube            1/1     Running     0             67m
kube-system   kube-controller-manager-minikube   1/1     Running     0             67m
kube-system   kube-proxy-t7dp8                   1/1     Running     0             67m
kube-system   kube-scheduler-minikube            1/1     Running     0             67m
kube-system   storage-provisioner                1/1     Running     0             67m
```

### Configure the host and cluster VM networking
It is recommended to configure a fixed IP to allow Minikube to reach the registry running on the host development machine. This will prevent having issues when the host connects to a different network. Make sure to use an available IP. 

To create an alias on macOS:
```bash
export HOST_DEV_MACHINE_IP=172.16.1.1
sudo ifconfig lo0 alias $HOST_DEV_MACHINE_IP
```

When the host development machine is restarted, the alias will have to be recreated. This can be prevented by [creating a Launchd Job](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html).

In addition, an entry needs to be added in `/etc/hosts` of Minikube VM. The entry will resolve registry name `registry.dev.svc.cluster.local` to the IP address `172.16.1.1` of the host, enabling Docker containers in Minikube to pull images from the local registry.

```bash
minikube ssh "echo \"$HOST_DEV_MACHINE_IP       registry.dev.svc.cluster.local\" | sudo tee -a  /etc/hosts"
```

### Create a Kubernetes Service Endpoint
In order to enable access to the local container registry from within Minikube cluster, it is required to create a Kubernetes service which will correspond to name `registry.dev.svc.cluster.local`.

The Service has no selectors and is named `registry` in the `dev` namespace. Additionally, a Kubernetes Endpoint with the same name must be created, to point to the static IP address `$HOST_DEV_MACHINE_IP` of the host development machine.

As a result, Kubernetes requests to `registry.dev.svc.cluster.local` will resolve to the host development machine, allowing Minikube cluster Pods to be created using container images pulled directly from the local container registry on the host development machine.

```bash
# create dev namespace
kubectl create namespace dev

# create Service
cat <<EOF | kubectl apply -n dev -f -
---
kind: Service
apiVersion: v1
metadata:
  name: registry
spec:
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
kind: Endpoints
apiVersion: v1
metadata:
  name: registry
subsets:
  - addresses:
      - ip: $HOST_DEV_MACHINE_IP
    ports:
      - port: 5000
EOF
```

### Create a test Docker image
In this step we create a Docker image and push the image to the local container registry. We can do so by using a pre-existing image that can be pulled from the public Docker container registry.

```bash
docker pull hello-world
```

Once the image is available locally, we tag it and push it to the local registry.

```bash
docker tag hello-world registry.dev.svc.cluster.local:5000/srodi-test:v0.0.1
```

Finally, we upload the Docker image to the local container registry

```bash
docker push hello-world registry.dev.svc.cluster.local:5000/srodi-test:v0.0.1
```

Using the Docker registry HTTP API we send a GET request to the local container registry to check if the version of the container image we upload is available.

```bash
❯ curl registry.dev.svc.cluster.local:5000/v2/srodi-test/manifests/v0.0.1
{
   "schemaVersion": 1,
   "name": "srodi-test",
   "tag": "v0.0.1",
   "architecture": "amd64",
   "fsLayers": [
      ...
   ],
   "history": [
      ...
   ],
   "signatures": [
      ...
   ]
}
```

### Test creating a Kubernetes Job
To validate that Minikube has access to the container registry we need to create a Kubernetes resource that requires to download an image from the local container registry.

Create a Kubernetes job using the image we uploaded to the local registry in the previous step

```bash
❯ k create job test --image registry.dev.svc.cluster.local:5000/srodi-test:v0.0.1
job.batch/test created
```

Verify the image was pulled from the local insecure container registry

```bash
❯ k get po
NAME          READY   STATUS      RESTARTS   AGE
test-547fd    0/1     Completed   0          104m

❯ k get po test-547fd -oyaml |grep image
  - image: registry.dev.svc.cluster.local:5000/srodi-test:v0.0.1
    imagePullPolicy: IfNotPresent
    image: registry.dev.svc.cluster.local:5000/srodi-test:v0.0.1
    imageID: docker-pullable://registry.dev.svc.cluster.local:5000/srodi-test@sha256:7e9b6e7ba2842c91cf49f3e214d04a7a496f8214356f41d81a6e6dcad11f11e3
```

## Conclusions
In this short tutorial we covered how to run a Docker container registry locally do that it is accessible from a local Kubernetes cluster, using Minikube as an example. We also configured the host and VM to use a static IP so that a Kubernetes Service and Endpoint resources allow containers images to be downloaded from a local container registry. Finally, we tested the access to the registry by creating a Kubernetes Job consuming an image from the local registry.

A use case for this local configuration can be Open Source development for large projects. I have personally used this configuration for Istio development given that several container images are built and pushed to registry at each iteration.

## References

* [Using a Local Registry with Minikube](https://gist.github.com/trisberg/37c97b6cc53def9a3e38be6143786589)
* [Edit Docker config in Rancher Desktop](https://github.com/rancher-sandbox/rancher-desktop/discussions/1477#discussioncomment-2106389)
* macOS: [Creating a Launchd Job](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
* [Docker Registry HTTP API V2 Specifications](https://distribution.github.io/distribution/spec/api/)
* [official Minikube documentation](https://minikube.sigs.k8s.io/docs/start/)
* Kubernetes docs: [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)