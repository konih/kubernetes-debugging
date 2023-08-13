<div align="center">
<img src="images/header.png"/>
</div>

# The Problem with using minimal container images

For many years, the standard practice for building container images was to use a minimal base image, such as Alpine or Distrosless. This practice was driven by the desire to reduce the size of the container image, and thus reduce the attack surface of the container. 
Up until Kubernetes 1.20, this practice had a significant drawback: it made debugging containers more difficult. This lead to a common practice of using a different base image for development and production, which is not ideal and will not be helpful in the case of an issue on production.

Fortunately, Kubernetes 1.20 introduced a new feature called ephemeral containers, which allows you to attach a debugging container to a running pod. This means that you can now use a minimal base image for your production container, and still be able to debug it if needed.
This article will show you how to use ephemeral containers to debug common networking issues and also show you how to use ephemeral containers to debug a Java and a Go application.

# Debugging in Kubernetes with Ephemeral Containers: A Practical Guide

Understanding the complexities of Kubernetes, the leading platform for orchestrating and automating container
deployment, can be a daunting task, especially for software engineers who may not be entirely familiar with its
administrative aspects. However, thanks to recent improvements and features in Kubernetes, we can simplify some of these
complex operations.

One such feature that has proven useful for debugging in Kubernetes is the introduction of ephemeral containers. This
article aims to unpack the mystery behind ephemeral containers and how to utilize them effectively in a Kubernetes
environment. But before we get to that, let's take a moment to understand the fundamental concept of Linux namespaces
that underpin Kubernetes operations.

## Linux Namespaces and Kubernetes: A Container's Perspective

Linux namespaces are powerful features of the Linux kernel that encapsulate system resources to provide isolated
environments, each with unique global resources like process IDs (PID) or network stacks. Kubernetes leverages this
technology via container runtimes like Docker, containerd, or CRI-O, creating a new set of namespaces for each pod. This
ensures that each pod has its own isolated network, IPC, UTS, and PID environments.

In this context, there are two important Kubernetes constructs: sidecar and ephemeral containers. Both exist within the
pod's shared namespaces, allowing inter-container communication as if on the same host, while remaining isolated from
other pods.

**Sidecar containers** are secondary containers deployed in the same pod as the primary application container. Their
shared environment means they can freely communicate over `localhost`, signal processes in other containers if PID
namespace sharing is enabled, and share volumes defined at the pod level.

The diagram below provides a visual representation of this concept. In it, you can see how a sidecar container shares
most namespaces with the primary application container, including the PID, network, and IPC namespaces. They are
separate only in their mount namespaces and Unix Time-Sharing (UTS) namespaces, which are unique to each container.

![pod_sidecar_namespaces.svg](images%2Fpod_sidecar_namespaces.svg)

This namespace sharing enables sidecar containers to closely interact with the primary application container, enabling
them to supplement or extend the primary container's behavior in various ways.

## Ephemeral Containers: Temporary Sidecars for Practical Use

Ephemeral containers, as the name suggests, are short-lived containers. Unlike traditional containers in a pod,
ephemeral containers are not defined in the pod's specification and do not share
in the pod's lifecycle. Instead, they come to life as needed, sharing the namespaces of an existing container in the
same pod.

These short-lived containers offer powerful tools for understanding and diagnosing the behavior of your applications.
For instance, if you need to inspect a running pod that's experiencing issues, you can spin up an ephemeral container
without interrupting the pod's operation or requiring specific tools not included in your production image.

Like sidecars, ephemeral containers can communicate over `localhost`, use Interprocess Communication (IPC), inspect or
signal processes in the pod (if PID namespace sharing is enabled), and access shared volumes. The extent of their
interaction, however, is governed by the pod's security context and Kubernetes' security settings.

Creating an ephemeral container requires the use of the `kubectl` command-line tool. Before you can create an ephemeral
container, you need to make sure `kubectl` is installed and configured to interact with your Kubernetes cluster.

Limitations of Ephemeral Containers:
- No readiness or liveness probes
- Resources are not configurable
- Can not expose ports
- No restart policy
- Can not be removed for audit purposes

## Advanced Debugging in Kubernetes with Kubectl Debug

In order to understand how ephemeral containers operate, we'll be using the `kubectl debug` command. This allows you to
add an ephemeral container to a running pod without altering its existing containers.

Let's see how this works:

1. Identify the name of the pod you want to debug.

```bash
kubectl get pods
```

2. Add an ephemeral container to the pod. Replace `<pod-name>` with the name of your pod.

```bash
kubectl debug -it <pod-name> --image=busybox --target=<pod-name>
```

The `-it` flag allows for an interactive session, `--image` specifies the container image, and `--target` designates the
pod to add the ephemeral container to.

You will see an ephermeralContainer section added to the pod's YAML file.

```yaml
ephemeralContainers:
  - command:
    - sh
    image: busybox
    imagePullPolicy: IfNotPresent
    name: debug-container
    resources: {}
    securityContext:
      privileged: true
    stdin: true
    terminationMessagePolicy: File
    tty: true
```

The `kubectl debug` command also includes additional options to further refine your debugging:

- **Debugging Nodes:** Use `kubectl debug node/my-node` to create a new pod running in the host namespaces, like host
  PID, host network, etc. This is akin to running a privileged pod on the node. However, it's best to limit its use due
  to security concerns.

- **--share-processes:** This flag, used when creating a pod, enables all containers in the pod to share a single
  process namespace, allowing for inter-process interaction as if on the same host.

- **--profile:** This option allows users to set "Debugging Profiles," providing configuration for the debug container
  to suit your needs. The available profiles
  include `general`, `baseline`, `restricted`, `auto`, `sysadmin`, `netadmin`, and `legacy`. Each profile offers a
  variety of configurations based on your debugging journey or security standard.

-- **--copy-to** This option creates a copy of the target pod with a new name. This 

## Which images can I use?

- Netshoot
- Koolkits by LightRun

## Switching to a different root while bringing the debugging binaries

```bash

```

## Introduction to Nicolaka/Netshoot and Utilizing Tcpdump in Kubernetes

The [nicolaka/netshoot](https://github.com/nicolaka/netshoot) is a container image that contains an impressive
collection of Linux network debugging and performance analysis tools. Some of these tools
include `curl`, `iperf`, `iproute2`, `iptables`, `nmap`, `tcpdump`, and `tshark`, among many others. Its extensive
toolkit makes it ideal for network troubleshooting and performance tuning.

![Linux Performance Observability Tools included in netshoot](images%2Fnetshoot.png)

Let's now turn our attention to `tcpdump`, a powerful tool for network traffic analysis, and understand how to use it
with Kubernetes using `kubectl debug`.

### Understanding Tcpdump

`Tcpdump` is a common packet analyzer that runs under the command line. It allows the user to display TCP/IP and other
packets being transmitted or received over a network to which the computer is attached. Tcpdump is especially powerful
because it provides a detailed timestamp, allows you to read or write packet data from files, and lets you filter
packets based on specific parameters.

### Debugging with Tcpdump and Wireshark

Tcpdump is one of the first tools you traditionally reach for when debugging network issues. One of the security
best practices related to containers is to not install any unnecessary tools in your production images. This means that
you should not have `tcpdump` installed in your production images. However, this doesn't mean you can't use it to debug
issues in your production environment.

In this section, we'll demonstrate how to use `tcpdump` with `kubectl debug` to capture network traffic from a pod and
analyze it locally using `Wireshark`.

1. Identify the pod you want to debug.

```bash
kubectl get pods
```

2. Launch an interactive debugging session using `kubectl debug` and the `nicolaka/netshoot` image.

```bash
kubectl debug -it <pod-name> --image=nicolaka/netshoot --target=<pod-name>
```

3. Once inside the debug session, initiate a `tcpdump` capture on the interface of interest (e.g., eth0). Save the
   capture to a file.

```bash
tcpdump -i eth0 -w /tmp/capture.pcap
```

4. After capturing sufficient data, exit the debug session. Copy the captured file from the pod to your local machine
   using `kubectl cp`.

```bash
kubectl cp <pod-name>:/tmp/capture.pcap ./capture.pcap
```

5. Open the copied `.pcap` file locally using Wireshark for in-depth packet analysis.

Remember, `tcpdump` captures all the packets going through the network interface, so ensure to use it responsibly and
consider potential privacy issues.

With `tcpdump`, `kubectl debug`, and `Wireshark`, you have powerful tools at your disposal to identify and diagnose
network issues within your Kubernetes environment.

# The proc filesystem

# Setting securityContext, adding capabilities, and using privileged containers

# Mouting volumes

```bash
ln -s /proc/$$/root/bin /proc/1/root/debug_image_bin # $$ refers to the current process ID
export PATH=${PATH}:/proc/1/root/debug_image_bin
chroot /proc/1/root sh # set the root filesystem to the root of the container
```

---

# Debugging in Kubernetes with Go KoolKit: A Practical Guide

Demystifying the intricacies of Kubernetes, the preeminent platform for orchestrating and automating container deployment, is often challenging, especially for software engineers who might not be deeply familiar with its administrative facets. Thanks to Kubernetes' recent advancements, we can streamline some of these complex tasks.

This article delves into the Go KoolKit, an efficient debugging tool available in Kubernetes. But first, let's understand Linux namespaces, the fundamental concept underlying Kubernetes operations.

## Linux Namespaces and Kubernetes: A Container's Perspective

Linux namespaces are powerful Linux kernel features that encapsulate system resources to provide isolated environments, each with unique global resources such as process IDs (PID) or network stacks. Kubernetes uses this technology via container runtimes like Docker, containerd, or CRI-O, creating a new set of namespaces for each pod. This ensures each pod has its own isolated network, IPC, UTS, and PID environments.

Within this context, Kubernetes constructs such as sidecar and ephemeral containers become important. These exist within the pod's shared namespaces, enabling inter-container communication as if on the same host while remaining isolated from other pods.

This article previously introduced sidecar and ephemeral containers; here we will explore how to debug using Go KoolKit.

## Getting Started with Go KoolKit

Go KoolKit is part of the Kubernetes Toolkits (KoolKits), a set of language-specific container images with highly opinionated debugging tools for applications running in Kubernetes pods. To use the Go KoolKit, add the shorthand `kk` command to your shell by pasting the following snippet:

```bash
echo "## KoolKits - Shorthand
kk() {
	kubectl debug -it $1 --image=lightruncom/koolkits:$2 --image-pull-policy=Never --target=$3
}" >> ~/.bashrc
source ~/.bashrc
```

Now, you can run the Go KoolKit with your pod:

```bash
kk <POD-NAME> golang <DEPLOYMENT-NAME>
```

## Go KoolKit Debugging Techniques

The Go KoolKit features numerous utilities, including delve, pprof, fzf, alp, go-callvis, and gvm. Apart from these, it includes additional tools on top of the official ubuntu:20.04 image. Let's examine how to use some of these tools to debug your Kubernetes applications effectively.

### Chroot /proc/1/root

The `chroot` command changes the apparent root directory for the current running process and its children. Here's how to use it:

```bash
chroot /proc/1/root
```

By issuing this command, you're changing the root directory to the directory where the first process (PID 1) is running.

### Iftop

`Iftop` is a command-line tool that shows a table of current network usage by pairs of hosts. To run `iftop`, use the following command:

```bash
iftop
```

### Drill

`Drill` is a tool similar to `dig` that queries DNS servers for information. It can be useful for diagnosing DNS-related issues in your Kubernetes environment. Here's how to use it:

```bash
drill example.com
```

Replace `example.com` with the domain name you want to investigate.

### Debugging a Pod in CrashLoopBackOff

When a pod is in a `CrashLoopBackOff` state, it means the pod is crashing repeatedly. Debugging in this scenario can be challenging but essential. You can use the `kubectl logs` command to inspect the logs of the crashed pod:

```bash


kubectl logs <POD-NAME>
```

Additionally, you can use `kubectl describe` to get more information about the pod:

```bash
kubectl describe pod <POD-NAME>
```

These commands will provide valuable information about what's causing the pod to crash.

### Delve

Delve is a debugger for Go programs. It allows you to dissect your program and understand what it is doing at runtime. Here's a simple way to run Delve:

```bash
dlv debug
```

You can set breakpoints, step through your code, and evaluate variables using Delve.

### Pprof

`Pprof` is a visualization tool for profiling data, useful for understanding the CPU usage or memory consumption of your Go applications. Here's how to use it:

First, import the net/http/pprof package in your application:

```go
import _ "net/http/pprof"
```

Then, start a web server with `http.ListenAndServe()` in your `main()` function:

```go
go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

After starting your application, you can navigate to `http://localhost:6060/debug/pprof/` in your web browser to see the profiling data.
