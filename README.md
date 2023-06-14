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

In sum, ephemeral containers serve as a vital tool in the Kubernetes ecosystem, providing flexibility and practical
benefits in managing and debugging applications in a live environment.

## Advanced Debugging in Kubernetes with Kubectl Debug

In order to understand how ephemeral containers operate, we'll be using the `kubectl debug` command. This allows you to add an ephemeral container to a running pod without altering its existing containers.

Let's see how this works:

1. Identify the name of the pod you want to debug.

```bash
kubectl get pods
```

2. Add an ephemeral container to the pod. Replace `<pod-name>` with the name of your pod.

```bash
kubectl debug -it <pod-name> --image=busybox --target=<pod-name>
```

The `-it` flag allows for an interactive session, `--image` specifies the container image, and `--target` designates the pod to add the ephemeral container to.

The `kubectl debug` command also includes additional options to further refine your debugging:

- **Debugging Nodes:** Use `kubectl debug node/my-node` to create a new pod running in the host namespaces, like host PID, host network, etc. This is akin to running a privileged pod on the node. However, it's best to limit its use due to security concerns.

- **--share-processes:** This flag, used when creating a pod, enables all containers in the pod to share a single process namespace, allowing for inter-process interaction as if on the same host.

- **--profile:** This option allows users to set "Debugging Profiles," providing configuration for the debug container to suit your needs. The available profiles include `general`, `baseline`, `restricted`, `auto`, `sysadmin`, `netadmin`, and `legacy`. Each profile offers a variety of configurations based on your debugging journey or security standard.
