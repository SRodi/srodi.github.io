---
title: "What's a Linux container and how to write one?"
date: 2024-11-13 09:00:00 +0100
categories: [containers]
tags: [containers, bash, linux] ## always lowercase !!
mermaid: true
image:
  path: /containers.webp
  alt: "what's a Linux container?"
---

## Introduction
A Linux container is a lightweight, portable, and self-sufficient software package that includes everything needed to run a piece of software, including the code, runtime, system tools, libraries, and settings. Containers leverage features of the Linux kernel, such as namespaces and cgroups, to provide isolated environments for applications. This isolation ensures that applications running in containers do not interfere with each other and can run consistently across different environments. Containers are often used to deploy and manage applications in a microservices architecture, providing benefits such as scalability, efficiency, and ease of deployment.

## Plan
In this tutorial, we'll go through a step-by-step approach to creating a basic Linux container using Bash. The steps include configuring Linux cgroups, creating network namespaces, and setting up a root filesystem. We will create a basic example script that does all of this and then starts an interactive shell inside the container.

1. [Root Filesystem](#filesystem-setup) set up a minimal filesystem with debootstrap.
2. [Cgroups](#cgroups): configure CPU and memory cgroups to limit the container's resources.
3. [Network Namespace](#network-namespace): create network namespace, and a pair of virtual Ethernet interfaces (veth) to connect the host to the container's namespace.
4. [Unshare](#unshare): use unshare with multiple namespaces (PID, network, mount, etc.), and chroot into the container's root filesystem, starting a shell inside.

This is a very basic container and is not production-ready, but it demonstrates core containerization principles using namespaces and cgroups in a Bash script.

![container](/demo.gif)
_Custom basic container demo_

## Prerequisites
- Linux `kernel version >= 5.4`
- `libcgroup-dev` is a set of utilities for managing control groups (cgroups v2) in Linux.
- `debootstrap` is a tool that installs a Debian-based Linux distribution into a subdirectory of another, already installed system

```sh
sudo apt update && sudo apt upgrade -y
sudo apt install -y libcgroup-dev debootstrap
```

### Filesystem Setup
`debootstrap` can be used to create a minimal filesystem for a container or chroot environment. `debootstrap` downloads and extracts the necessary packages to set up a basic Debian system, which can then be customized and used as the root filesystem for a container.

In this step we create a directory at `/mnt/mycontainer` and download an Alpine Linux filesystem archive from a specified URL. The archive is then extracted into the created directory. The script ensures that the necessary directories, such as `/proc`, are created within the container's filesystem.

```sh
CONTAINER_NAME=mycontainer
FS_PATH=/mnt/$CONTAINER_NAME
FS_DL_URL=http://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64
FS_VERSION=alpine-minirootfs-3.20.0-x86_64.tar.gz

# Create a directory for the filesystem
mkdir -p $FS_PATH
# Fetch Alpine Linux filesystem and save it in $FS_PATH
wget -P $FS_PATH $FS_DL_URL/$FS_VERSION
# Extracts the contents of the tarball into $FS_PATH
tar -xzf $FS_PATH/$FS_VERSION -C $FS_PATH
# Create proc directory and mount proc filesystem
mkdir -p $FS_PATH/proc
```

### Cgroups
`Cgroups` are a feature of the Linux kernel that allow you to allocate resources—such as CPU time, system memory, disk I/O, and network bandwidth—among user-defined groups of tasks (processes). `libcgroup-dev` is a development package for the `libcgroup` library, which provides tools and libraries to control and monitor control groups (cgroups) in Linux.

In this step we create and configure control groups (cgroups) to manage resource limits for the container. `$CGROUP_DIR` represents the directory path where the cgroup settings for the container are located. The command `echo "+memory +cpu" > $CGROUP_DIR/cgroup.subtree_control` is used to enable specific resource controllers for a cgroup. In this case, it is enabling the memory and CPU controllers.

Next we set a memory limit for the container by executing the command `echo 1024 > $CGROUP_DIR/memory.max`, we instruct the system to restrict the container's memory usage to `1KB`. Writing the value `1024` to the `memory.max` file within this directory enforces the memory limit.

Then we set a CPU usage limit for the container. The command `echo 10000 > $CGROUP_DIR/cpu.max` restricts the container to use only `10ms` of CPU time for every 100 milliseconds, effectively limiting it to `10%` of a CPU's capacity. Similar to the memory limit, the `$CGROUP_DIR` variable points to the cgroup directory, and writing the value `10000` to the `cpu.max` file enforces the CPU usage limit.

These resource limits help ensure that the container does not consume more than its allocated share of system resources, which is crucial for maintaining system stability and performance, especially in multi-tenant environments.

```sh
# Set up cgroups v2
CGROUP_DIR=/sys/fs/cgroup/$C_NAME
mkdir -p $CGROUP_DIR
# instruct the cgroup to control memory and CPU usage for its child cgroups
echo "+memory +cpu" > $CGROUP_DIR/cgroup.subtree_control
# Set memory limit to 1KB
echo 1024 > $CGROUP_DIR/memory.max
# Set CPU limit to 10ms per 100ms (10% of a CPU)
echo 10000 > $CGROUP_DIR/cpu.max
```

### Network Namespace
The script creates the network namespace for the container with the command `ip netns add`. Network namespaces allow for the isolation of network interfaces, routing tables, and other networking components.

Next, the script creates a pair of virtual Ethernet devices (veth pairs) using the `ip link add` command. Veth pairs are always created in pairs and act like a virtual network cable connecting two network interfaces. Here, `veth0` and `veth1` are the names of the two ends of the veth pair.

The script then assigns one end of the veth pair (veth1) to the newly created network namespace using the `ip link set` command. This effectively moves `veth1` into the isolated network namespace, while `veth0` remains in the default namespace. One end of the veth pair is assigned to the container's network namespace, while the other end is configured on the host.

Following this, the script configures the network interfaces within the namespace. It uses the `ip netns exec` command to execute commands within the context of the specified namespace. The `ip addr add` command assigns the IP address `10.0.0.1/24` to `veth1`, and the `ip link set veth1 up` command brings the interface up.

Finally, the script configures the `veth0` interface in the default namespace. It assigns the IP address `10.0.0.2/24` to `veth0` and brings the interface up using the `ip link set veth0 up` command. This setup allows for communication between the default namespace and the isolated network namespace through the veth pair.

```sh
# Set up network namespace
ip netns add "${CONTAINER_NAME}_ns"
# Create veth pairs
ip link add veth0 type veth peer name veth1
# Assign veth1 to the container's network namespace
ip link set veth1 netns "${CONTAINER_NAME}_ns"
# Set up veth interfaces
ip netns exec "${CONTAINER_NAME}_ns" ip addr add 10.0.0.1/24 dev veth1
ip netns exec "${CONTAINER_NAME}_ns" ip link set veth1 up
ip addr add 10.0.0.2/24 dev veth0
ip link set veth0 up
```



### Unshare
The last command of the script is used to execute a process within a network namespace and a new set of namespaces for various system resources. This is done to isolate processes from the rest of the system.

The command starts with `ip netns exec "${CONTAINER_NAME}_ns"`, which executes the next commands within the container network namespace. This ensures that the process has its own isolated network stack.

Next, the `unshare` command is used with several flags: `--mount`, `--pid`, `--fork`, `--uts`, and `--mount-proc=$FS_PATH/proc`. The unshare command creates new namespaces for the process, isolating it from the parent namespaces. The `--mount` flag creates a new mount namespace, `--pid` creates a new process ID namespace, `--fork` forks a new process, `--uts` creates a new UTS namespace (for hostname and domain name isolation), and `--mount-proc=$FS_PATH/proc` mounts the proc filesystem from the specified path.

Finally, the `chroot $FS_PATH /bin/ash` command changes the root directory of the process to the specified filesystem path and starts a new shell `/bin/ash`. This effectively confines the process to the specified filesystem, providing further isolation.

Overall, this sequence of commands is used to create a highly isolated environment for running processes, which is a fundamental aspect of containerization.

```sh
# Run the unshare command within the network namespace.
ip netns exec "${CONTAINER_NAME}_ns" \
    unshare --mount --pid --fork --uts --mount-proc=$FS_PATH/proc \
    chroot $FS_PATH /bin/ash
```

## Conclusions
In conclusion, this tutorial walks you through a script to set up a basic container environment using a bash. It begins by creating a directory for the container's filesystem and downloading the Alpine Linux miniroot filesystem. The filesystem is then extracted, and a /proc directory is created to mount the proc filesystem. The script also sets up control groups (cgroups) to limit the container's memory and CPU usage, ensuring it operates within specified resource constraints.

Additionally, the script configures a network namespace for the container, creating virtual Ethernet (veth) pairs to establish network interfaces. One end of the veth pair is assigned to the container's network namespace, while the other end is configured on the host, providing network connectivity. Finally, the script uses the unshare command to create a new namespace for the container, isolating its processes, mount points, and hostname, and then changes the root directory to the new filesystem using chroot. This demonstrates the basics of container creation and management on a Linux machine.

For those interested in exploring the complete implementation, the GitHub code repository is available at [https://github.com/SRodi/container-101](https://github.com/SRodi/container-101).

## References

- [libcgroup-dev](https://github.com/libcgroup/libcgroup/blob/main/README)
- [debootstrap](https://wiki.debian.org/Debootstrap)
- [alpine-minirootfs](https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/)
- [cgroups](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [netns](https://www.man7.org/linux/man-pages/man8/ip-netns.8.html)
- [unshare](https://www.man7.org/linux/man-pages/man1/unshare.1.html)
- [chroot](https://www.man7.org/linux/man-pages/man2/chroot.2.html)
- [This tutorial code](https://github.com/SRodi/container-101)
