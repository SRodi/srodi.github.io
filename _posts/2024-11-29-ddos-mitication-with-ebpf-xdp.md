---
title: "Protect from DDoS attacks with eBPF XDP"
date: 2024-11-29 09:00:00 +0100
categories: [ebpf]
tags: [networking, c, linux, security] ## always lowercase !!
mermaid: true
image:
  path: /xdp-ddos.webp
  alt: "XDP fror DDoS protection"
---

## Introduction
XDP (eXpress Data Path) is a high-performance, in-kernel packet processing framework in Linux. It enables the execution of user-defined programs at the earliest point of the network stack, right at the NIC (Network Interface Card) driver level, before the kernel processes the packets.

Key Features of XDP:
1. Performance: XDP operates directly in the kernel, bypassing many layers of the Linux networking stack. This allows for ultra-low latency and high-throughput packet processing.
2. Flexibility: It leverages eBPF (extended Berkeley Packet Filter), allowing developers to write custom packet processing logic in C (or other supported languages).
3. Efficiency: By processing packets before they reach the network stack, XDP reduces CPU usage and memory overhead.
Programmability: Developers can use eBPF to handle tasks such as filtering, forwarding, or load balancing.

Common Use Cases
- DDoS (Distributed Denial of Service) Protection: Quickly drop or filter malicious packets at the NIC level to protect servers.
- Load Balancing: Distribute traffic across multiple backends with minimal latency.
- Traffic Shaping: Enforce rate limits or prioritize specific types of traffic.
- Packet Monitoring: Analyze or log traffic patterns directly on the NIC.

## Plan
In this tutorial we use XDP to detect and mitigate DDoS attacks targeting a specific server by monitoring unusually high traffic from specific IP addresses. The XDP program monitors incoming packets for the target server, tracks the rate of incoming packets per source IP in a hash map, and drops packets from IPs exceeding a threshold.

![container](/demo-xdp.gif)
_Custom basic container demo_

## Prerequisites
- Linux `kernel version >= 5.15`
- Debian/Ubuntu distribution
- [clang](https://clang.llvm.org/index.html) is a language front-end and tooling infrastructure for languages in the C language family.
- [llvm](https://llvm.org/) is a collection of modular and reusable compiler and toolchain technologies.
- [libbpf-dev](https://packages.debian.org/sid/libbpf-dev) is a library for loading eBPF programs and reading and manipulating eBPF objects from user-space.

```sh
sudo apt-get install clang llvm libbpf-dev -y
```


## XDP program
This program is an eBPF application written in C, designed to mitigate DDoS attacks using XDP. It defines a rate-limiting mechanism that tracks the number of packets received from each source IP address within a one-second time window. The program uses a hash map to store rate limit entries, which include the timestamp of the last update and the packet count for each source IP. When a packet arrives, the program checks if the source IP has exceeded the defined threshold of 250 packets per second. If the threshold is exceeded, the packet is dropped; otherwise, it is allowed to pass. The program ensures efficient packet processing at the network interface level, providing a high-performance solution for DDoS mitigation.

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

#define THRESHOLD 250 // Max packets per second
#define TIME_WINDOW_NS 1000000000 // 1 second in nanoseconds

struct rate_limit_entry {
    __u64 last_update; // Timestamp of the last update
    __u32 packet_count; // Packet count within the time window
};

// Hash map to track rate limits for each source IP
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32); // Source IP
    __type(value, struct rate_limit_entry);
} rate_limit_map SEC(".maps");

SEC("xdp") int ddos_protection(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data; // Parse Ethernet header
    struct ethhdr *eth = data;
    
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;
    
    // Check for IP packets
    if (eth->h_proto != __constant_htons(ETH_P_IP))
        return XDP_PASS;
    
    // Parse IP header
    struct iphdr *iph = (void *)(eth + 1);
    if ((void *)(iph + 1) > data_end)
        return XDP_PASS;
        
    __u32 src_ip = iph->saddr; // Source IP address
    
    // Lookup rate limit entry for this IP
    struct rate_limit_entry *entry = bpf_map_lookup_elem(&rate_limit_map, &src_ip);
    
    // Get current time in nanoseconds
    __u64 current_time = bpf_ktime_get_ns();
    
    if (entry) {
        // Check if we're in the same time window
        if (current_time - entry->last_update < TIME_WINDOW_NS) {
            entry->packet_count++;
            if (entry->packet_count > THRESHOLD) {
                return XDP_DROP; // Drop packet if rate exceeds threshold
            }
        } else {
            // New time window, reset counter
            entry->last_update = current_time;
            entry->packet_count = 1;
        }
    } else {
        // Initialize rate limit entry for new IP
        struct rate_limit_entry new_entry;
        // Zero out padding bytes
        __builtin_memset(&new_entry, 0, sizeof(new_entry));
        new_entry.last_update = current_time;
        new_entry.packet_count = 1;
        bpf_map_update_elem(&rate_limit_map, &src_ip, &new_entry, BPF_ANY);
    }
    return XDP_PASS; // Allow packet if under threshold   
}

char _license[] SEC("license") = "GPL";
```

### Program logic
The `eBPF XDP program` includes the necessary Linux kernel headers and defines constants for rate limiting, such as the maximum packets per second (`THRESHOLD`) and the time window in nanoseconds (`TIME_WINDOW_NS`).The program maintains a hash map (`rate_limit_map`) to track the rate limit for each source IP address, storing the last update timestamp and packet count within the time window.

The `ddos_protection` function, marked with the `SEC("xdp")` section, processes incoming packets, starting by parsing the `Ethernet header`. It checks if the packet is an `IP packet` and ensures that the packet data is within bounds before proceeding with further processing. If the packet is not an IP packet or the data is out of bounds, it passes the packet without any action.

If none of the above conditions are met, the function then extracts the source IP address from the IP header of the incoming packet. It then looks up a rate limit entry for this IP address in the `rate_limit_map` BPF map.

The current time is fetched in nanoseconds using `bpf_ktime_get_ns()`. If an entry for the source IP exists, the code checks if the current time is within the same time window defined by `TIME_WINDOW_NS`. If it is, the packet count for this IP is incremented. If the packet count exceeds a predefined `THRESHOLD`, the packet is dropped by returning `XDP_DROP`. If the time window has elapsed, the packet count is reset, and the time of the last update is set to the current time.

If no entry exists for the source IP, a new rate limit entry is initialized with the current time and a packet count of one. This new entry is then added to the rate_limit_map. If the packet count is within the threshold, the packet is allowed to pass by returning `XDP_PASS`. This mechanism helps in mitigating DDoS attacks by limiting the rate of packets from any single IP address.

## Compile the program
This command allows to compile the eBPF program written in C. The command utilizes `clang`, which is a compiler front end for the C, C++, and Objective-C programming languages.

Here's a breakdown of the command:

- `clang`: This is the compiler being used. Clang is part of the LLVM project and is known for its fast compilation times and excellent diagnostics.
- `-O2`: This flag tells the compiler to optimize the code for performance. The -O2 level is a moderate optimization level that balances compilation time and the performance of the generated code.
- `-g`: This flag includes debugging information in the compiled output. This is useful for debugging the eBPF program later.
- `-target bpf`: This specifies the target architecture for the compilation. In this case, it is bpf, which stands for Berkeley Packet Filter. This is necessary because eBPF programs run in a virtual machine inside the Linux kernel.
- `-c xdp_ebpf_prog.c`: This tells the compiler to compile the source file xdp_ebpf_prog.c without linking. The -c flag indicates that the output should be an object file.
- `-o xdp_ebpf_prog.o`: This specifies the name of the output file. In this case, the compiled object file will be named xdp_ebpf_prog.o.

In summary, this command compiles the xdp_ebpf_prog.c source file into an object file xdp_ebpf_prog.o with optimizations and debugging information, targeting the BPF architecture. This is a crucial step in developing eBPF programs, which are often used for tasks like network packet filtering and monitoring within the Linux kernel.

```sh
# Compile the XDP program with clang
# -O2: Optimize the code for better performance
# -g: Generate debug information
# -target bpf: Specify the target architecture as BPF
# -c: Compile the source file without linking
# -o: Specify the output file
clang -O2 -g -target bpf -c xdp_ebpf_prog.c -o xdp_ebpf_prog.o
```

## Load and Attach program
To attach an eBPF program to a network interface we use `ip`. The `ip` command is a powerful utility for network configuration in Linux, and the `link set dev` subcommand is used to modify the properties of a network device.

Here's a breakdown of the command:

- `sudo`: This command is run with superuser privileges, which are necessary for modifying network interfaces and loading eBPF programs.
- `ip link set dev "$IFACE"`: This part of the command specifies the network interface `$IFACE` that you want to configure. The `ip link set dev` command is used to change the settings of the specified network device.
- `xdp obj xdp_ebpf_prog.o sec xdp`: This part attaches an eBPF program to the network interface. `obj xdp_ebpf_prog.o` specifies the object file containing the compiled eBPF program. `sec xdp` indicates the section of the object file that contains the XDP program.

```sh
IFACE=eth0
# Attach the XDP program to the network interface
sudo ip link set dev "$IFACE" xdp obj xdp_ebpf_prog.o sec xdp
```

This command loads an eBPF program from the specified object file and attaches it to the given network interface using XDP.

## Test
To test a DDoS attack, you can use [hping3](https://www.kali.org/tools/hping3/) tool. `hping3` is a network tool able to send custom ICMP/UDP/TCP packets and to display target replies like ping does with ICMP replies.

```sh
sudo apt install hping3
```

This command runs an `Nginx` web server inside a `Docker` container and makes it accessible via port `1234` on the host machine. This setup is useful for testing and development purposes, allowing you to run and interact with a web server in an isolated environment.
```sh
docker run -p 1234:80 nginx
```

>NOTE: Instructions to [Install Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Now we need to attach the XDP program to the `lo` interface
```sh
sudo ip link set dev lo xdp obj xdp_ebpf_prog.o sec xdp
```

Then run the test:
```sh
sudo hping3 -i u1000 -S -p 1234 127.0.0.1
```

The `-i u1000` option in the `hping3` command specifies the interval between packets. Here, `u1000` means 1000 microseconds (1 millisecond) between each packet. This option is used to control the rate of packet sending, which is useful for simulating high traffic or DDoS attacks.

The `-S` option in the `hping3` command specifies that the SYN flag should be set in the TCP packets. This is used to simulate a SYN flood attack, which is a type of DDoS attack where many SYN packets are sent to a target to overwhelm it.

The test result, depending on your parameters and the `TRESHOLD` configured in the eBPF program, should look similar to this:

```sh
--- 127.0.0.1 hping statistic ---
3176 packets transmitted, 332 packets received, 90% packet loss
round-trip min/avg/max = 0.2/4.4/11.4 ms
```

The output shows statistics from a test run where packets were sent to the local host (127.0.0.1). The first line, --- 127.0.0.1 hping statistic ---, indicates the start of the statistics for the test targeting the IP address 127.0.0.1, which is the loopback address used to test network software without sending packets over the network.

The next line, 3176 packets transmitted, 332 packets received, 90% packet loss, provides a summary of the test results. It shows that out of 3176 packets sent, only 332 were received, resulting in a 90% packet loss. This high packet loss percentage suggests that the system is under heavy load, due to a DDoS attack, and our XDP program is dropping packets once the `TRESHOLD` is reached for the given `TIME_WINDOW_NS`.

The final line, `round-trip min/avg/max = 0.2/4.4/11.4 ms`, gives the round-trip time statistics for the packets that were successfully received. The minimum round-trip time was 0.2 milliseconds, the average was 4.4 milliseconds, and the maximum was 11.4 milliseconds. These values provide insight into the latency experienced during the test.

## Cleanup
To detach the program, use the `sudo` command to execute a privileged operation, which is necessary for modifying network interface settings. The `ip link set dev "$IFACE" xdp off` command disables XDP on the specified network interface (`eth0` in this case). By setting `xdp off`, the script ensures that any XDP programs currently attached to the interface are detached, effectively turning off XDP processing for that interface.

```sh
IFACE=eth0
```sh
# Detach the XDP program from the network interface
sudo ip link set dev "$IFACE" xdp off
```

Detach the program used for the local test.
```sh
sudo ip link set dev lo xdp off
```

## Considerations
The test presented above uses `lo` which is a virtual network interface - by default, XDP is designed to work on physical and virtual interfaces that send and receive packets from the network. The loopback interface behaves differently because it is a purely software interface used for local traffic within the system.

Traffic on the loopback interface is not "real" network traffic, it is handled entirely within the kernel. As a result, certain packet processing steps (like those involving hardware offload) are bypassed, and this can affect how XDP interacts with the loopback interface.

The performance benefits of XDP may be less pronounced for loopback traffic compared to physical interfaces. This means the performance is greater on regular network interfaces like `eth0`, which represent a physical hardware device. 

## Conclusions
In conclusion, this tutorial has demonstrated how to leverage eBPF and XDP to create a high-performance solution for mitigating DDoS attacks. By attaching an eBPF program to the network interface, we can efficiently monitor and control incoming traffic at the kernel level, providing a robust defense mechanism against malicious activities. The steps outlined in this guide, from setting up the environment to compiling and deploying the eBPF program, offer a comprehensive approach to implementing eBPF-based security solutions.

For those interested in further exploring the capabilities of eBPF and XDP, the complete implementation and additional resources are available in the [GitHub repository](https://github.com/SRodi/xdp-ddos-protect/). This repository includes detailed code examples and instructions to help you get started with your own eBPF projects. By understanding and utilizing these powerful tools, you can enhance the security and performance of your network infrastructure, ensuring a more resilient and responsive system.
