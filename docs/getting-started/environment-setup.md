# Environment setup

## About FirecrackerSharp's hosts

Firecracker is a highly performant hypervisor, however it is **Linux-only and relies on KVM**, a Linux kernel-exclusive module for virtualization.

FirecrackerSharp, unlike any other Firecracker SDK, allows you to use virtual hosts **other than a Linux desktop/server**. This is implemented via a VM host abstraction that has several implementations:

- The local host (`FirecrackerSharp.Host.Local` on NuGet), this uses a Linux desktop/server without any layers of indirection. Prefer to use this in production in all possible cases, as it's (understandably) the most resilient virtual host there is.
- The SSH host (`FirecrackerSharp.Host.Ssh` on NuGet), this connects to a remote Linux server over SSH (using [SSH.NET](https://github.com/sshnet/SSH.NET)), uses its filesystem over SFTP and runs commands (or virtual terminals) over SSH.

In the future, more hosts are planned (such as a Docker/Podman rootful container host).

Using these hosts, here are the most common ways to configure your environment:

## 1. A Linux desktop with hardware virtualization

This is the setup the developers of FirecrackerSharp use, and it's the one we **highly recommend** if using Linux for .NET development isn't a severe issue (considering that all of .NET is cross-platform and there are better IDE options than Visual Studio, using Linux would not be a problem).

When possible, try to **adhere to [Firecracker's supported host and guest kernels](https://github.com/firecracker-microvm/firecracker/blob/main/docs/kernel-policy.md)**. While it's absolutely possible to run it on, for example, the 6.9 host Linux kernel instead of the latest supported 6.1, it may cause some issues the SDK can't deal with.

1. Enable virtualization in your BIOS, refer to the instructions of your BIOS's manufacturer and the type of your CPU to see which option to use, then reboot.
2. Install KVM to your Linux distribution. The easiest route would be to install QEMU alongside it. Follow your distribution's documentation on how to install QEMU, then reboot.
3. Verify you've installed KVM by running: ```lsmod | grep kvm```. The output should contain something like `kvm_amd` or `kvm_intel` or just `kvm` for you to be sure that KVM is enabled.
4. Gain access to KVM as your rootless user (running the jailer will still require the **root password**, however the SDK supports escalation of privileges).
    - For access managed through an access control list: `sudo setfacl -m u:${USER}:rw /dev/kvm`
    - For access managed through a kvm group: `[ $(stat -c "%G" /dev/kvm) = kvm ] && sudo usermod -aG kvm ${USER} && echo "Access granted."`
5. Verify you have KVM access via `[ -r /dev/kvm ] && [ -w /dev/kvm ] && echo "OK" || echo "FAIL"`.

Now you should be good to go! Firecracker doesn't depend on any particular Linux distribution, but it's most often used in either Debian and Ubuntu for the 6.1 host kernel.

## 2. A Linux VM

Here, you will need to create a virtual machine running Linux (preferably Debian, since it uses the 6.1 kernel).

On Windows, use Hyper-V, VMWare or VirtualBox. On Linux, simply install QEMU and virt-manager to create the virtual machine.

_On a Linux host_, **ensure the VM is on a bridge network interface**. Then, run `ip addr` to get its address on your local network and install SSH. For different distributions, the process will vary, for Debian use the following instructions:

1. `sudo apt-get install -y openssh-server`
2. Edit `/etc/ssh/sshd_config` and set `PermitRootLogin yes` so you that you can log into the root account with the root password
3. `sudo systemctl enable ssh`
4. `sudo systemctl restart ssh`
5. Go into your router's DHCP configuration and give the VM's local IP a static lease

_On a Windows host_, follow the instructions on how to assign a static local IP to your VM, then analogously install SSH inside the VM.

After installing the VM, **enable nested virtualization for the VM** so that it has KVM access. This will depend on which hypervisor you're using. Then, run `lsmod | grep kvm` to ensure KVM is accessible inside the VM.

## 3. WSL2 with nested virtualization and SSH

**Warning:** WSL is a pretty specific type of Linux VM and you may encounter issues using it for nested virtualization you wouldn't have with a conventional VM made in Hyper-V, VirtualBox or VMWare. If possible, opt for a traditional VM instead when running Windows. The WSL + SSH approach hasn't been thoroughly tested and we don't provide official support for it.

Follow the instructions on how to set up WSL2, enable KVM nested virtualization, and then install SSH in it. More detailed instructions will be added later.
