# Creating a basic app

In this tutorial, we'll create a sample app that:

- Boots up an unrestricted VM (_lifecycle_)
- Pauses and resumes the VM (_management_)
- Runs a basic buffered command via the VM's TTY (_classic TTY_)
- Shuts down the unrestricted VM (_lifecycle_)

This is a basic quickstart for working with the most common features of this SDK. To learn more about each of these topics (which you should, in order to use the SDK's full capabilities), refer to other sections in the documentation.

The sample's source code is available in the [sample GitHub repository](https://github.com/FirecrackerSharp/DocumentationSamples/tree/main).

## Getting started

First, create a new CLI project, install the core SDK package and the host you'll be using [according to the installation instructions](./2-installing-the-sdk.md). We'll be using the local host here as well as in all other samples.

Create the sample class:

```cs
namespace CreatingABasicApp;

public static class BasicAppSample
{
    public static async Task RunAsync()
    {
    }
}
```

Invoke it from `Program.cs`:

```cs
using CreatingABasicApp;

await BasicAppSample.RunAsync();
```

And start by configuring your host in `RunAsync`:

```cs
// Start by setting up the virtual host. Here, a local host is used, replace with another host if necessary
LocalHost.Configure();
```

## Download all necessary resources

To boot a microVM, several resources are required:

- An uncompressed Linux kernel, preferably either 5.10 or 6.1
- **A rootfs**, also known as the root filesystem, this contains your distribution, init system, installed packages, and is the entirety of `/` on your VM that will be used as a tmpfs
- (_Optionally_) An initrd (initial RAM disk), which we won't use in this case

Usually, you'd build a rootfs yourself (sometimes also the Linux kernel), but for these purposes we're going to use the one Firecracker's devs use in their CI, which has Ubuntu 22.04 with systemd inside. And we'll also download the prebuilt kernel:

```bash
ARCH="$(uname -m)"

# Download a linux kernel binary
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.9/${ARCH}/vmlinux-5.10.217

# Download a rootfs
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.9/${ARCH}/ubuntu-22.04.ext4
```

For the sake of simplicity, create `/opt/res` with sudo and save the kernel to `/opt/res/kernel` and the rootfs to `/opt/res/rootfs.ext4`.

Now, we'll need the Firecracker hypervisor itself: go to the [release page](https://github.com/firecracker-microvm/firecracker/releases), download the latest release, extract the archive and rename the binary starting with `firecracker` to just `firecracker` and place it under `/opt/res`. Do the same for the binary starting with `jailer`. Later, you'll learn how this process can be automated with the SDK.

**Ensure the `firecracker` and `jailer` binaries are executable with `chmod +x`!**

## Configure the VM

The first element of the VM is a `VmConfiguration`, this contains the properties of the future VM and will be passed down to Firecracker (in different possible ways, which we won't cover here).

Create your `VmConfiguration` in `RunAsync`:

```cs
// Configure the future VM
var vmConfiguration = new VmConfiguration(
    // boot data: kernel, args, initrd
    BootSource: new VmBootSource(
        KernelImagePath: "/opt/res/kernel",
        BootArgs: "console=ttyS0 reboot=k panic=1 pci=off",
        InitrdPath: null /* we aren't using an initrd */),
    // hardware configuration
    new VmMachineConfiguration(
        MemSizeMib: 128 /* 128 MB of ram */,
        VcpuCount: 1 /* 1 vCPU */),
    // drives, here: only the rootfs
    Drives: [
        new VmDrive(
            DriveId: "rootfs",
            IsRootDevice: true,
            PathOnHost: "/opt/res/rootfs.ext4")
    ]
);
```

The second element is a link to the Firecracker installation:

```cs
var firecrackerInstall = new FirecrackerInstall(
    Version: "v1.7.0", // replace with the one you're using
    FirecrackerBinary: "/opt/res/firecracker",
    JailerBinary: "/opt/res/jailer");
```

The third element are the arguments that need to be passed to the Firecracker binary:

```cs
var firecrackerOptions = new FirecrackerOptions(
    SocketFilename: $"{Guid.NewGuid()}",
    SocketDirectory: "/tmp");
```

Lastly, we'll need the VM ID, which should be unique for every VM. Let's generate it randomly:

```cs
var vmId = Random.Shared.Next(1, 10000).ToString();
```

## Booting up the VM

First, we'll create an instance of the VM. Since we aren't using jailing, we're using `UnrestrictedVm`. Then we boot it up. In the period between these actions, we can perform configuration of lifecycle logging, but this is out of scope of this guide:

```cs
// Create the VM instance and boot it
var vm = new UnrestrictedVm(vmConfiguration, firecrackerInstall, firecrackerOptions, vmId);
var bootResult = await vm.BootAsync();
if (bootResult.IsFailure())
{
    Console.WriteLine("Couldn't boot VM!");
    return;
}
```

## Managing the VM

Now, let's try a couple actions via the bindings to the VM's HTTP server. This server allows management of the VM, here are just a few possibilities, with a basic example of using the error handling response mechanisms the SDK provides.

Get the VM's information:

```cs
// Get the VM's info
var vmInfoResponse = await vm.Management.GetInfoAsync();
vmInfoResponse
    .IfSuccess(vmInfo =>
    {
        Console.WriteLine($"The VM is called {vmInfo.Id}");
        Console.WriteLine($"The VMM version is {vmInfo.VmmVersion}");
    })
    .IfError(error =>
    {
        Console.WriteLine($"Received an error when trying to get info: {error}");
    });
```

Pause and resume the VM, plus a demonstration of response chaining:

```cs
// Pause and resume VM
var pauseResponse = await vm.Management.UpdateStateAsync(new VmStateUpdate(VmStateForUpdate.Paused));
var resumeResponse = await vm.Management.UpdateStateAsync(new VmStateUpdate(VmStateForUpdate.Resumed));
pauseResponse
    .ChainWith(resumeResponse)
    .IfSuccess(() => Console.WriteLine("Paused and resumed VM!"))
    .IfError(error => Console.WriteLine($"Got an error when pausing and resuming: {error}"));
```

## Using the VM's TTY

The SDK's TTY system is highly advanced and is a huge topic of its own, with output buffering, completion tracking and multiplexing capabilities. Here, just for demonstration, we'll simply run a `cat --help` command, await it and print out its output:
```cs
// Run a command in the VM's TTY
var output = await vm.TtyClient.RunBufferedCommandAsync("cat --help");
Console.WriteLine("Received from command:");
Console.WriteLine(output);
```

## Shutting down the VM

It's as simple as:

```cs
// Shut down the VM
var shutdownResult = await vm.ShutdownAsync();
if (shutdownResult.IsFailure() || shutdownResult.IsSoftFailure())
{
    Console.WriteLine("Couldn't shut down VM!");
}
```
