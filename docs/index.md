# Welcome

This website hosts the official documentation for the FirecrackerSharp SDK.

FirecrackerSharp is a C# SDK for the innovative Firecracker microVM hypervisor developed by Amazon. It allows developers to:

- Boot up and shutdown microVMs
- Use different virtual machine hosts, bypassing the need for a Linux developer machine
- Use microVM jailing in order to tighten security without much extra effort
- Manage the microVM's lifecycle with fine-grained logging capabilities and error handling
- Connect to the microVM's HTTP server over a Unix domain socket for managing it, with (mostly) complete bindings to the OpenAPI specification and robust static typing and error handling
- Automate installing, updating and tracking installations of Firecracker via GitHub's API
- Interact with a microVM through its serial console in a thread-safe manner with:
    - Advanced output buffering, allowing piping out output to different sources and not necessarily into RAM
    - State management via completion tracking, allowing for complete thread-safety and reliable primary and intermittent writes
- Split up a microVM's serial console into multiple terminals via TTY multiplexing (_will be available in 1.0_)
- Automatically spin up host and guest network interfaces via CNI plugins (_will be available in 1.0_)

To learn what Firecracker is and what capabilities it offers for applications that need virtualization, check out its official website: [https://firecracker-microvm.github.io/](https://firecracker-microvm.github.io/).

To get started with using this (unofficial) .NET SDK for Firecracker, check out [the tutorial in order to set up your development environment for Firecracker and .NET](./getting-started/environment-setup.md).

## Why Firecracker

While QEMU, for example, supports (comparative to Firecracker) few VMs with full feature set, Firecracker is a alternative also using KVM internally that focuses on incredibly high amounts of atomic, lightweight and security-proofed VMs for so-called "edge workloads". This is why Firecracker is being used in AWS for the AWS Lambda service, providing real and secure host-to-guest isolation (unlike containers) without the overhead of QEMU, VMWare, VirtualBox or Hyper-V virtual machines.

## Stability of this SDK

FirecrackerSharp is already quite stable and offers a variety of thoroughly unit- and integration-tested features, however there are still some things that need to be ironed out and thus **it's still in beta**. **Breaking changes** happen regularly, so using it in production is not yet recommended.
