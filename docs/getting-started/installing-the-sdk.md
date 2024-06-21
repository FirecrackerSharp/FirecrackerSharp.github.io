# Installing the SDK

## Installing packages

To use FirecrackerSharp, you'll need the core package (`FirecrackerSharp`):

```dotnet add package FirecrackerSharp```

And then, one of the hosts, the local host is installed as follows:

```dotnet add package FirecrackerSharp.Host.Local```

And the SSH host is installed as follows:

```dotnet add package FirecrackerSharp.Host.Ssh```

## Configuring the host

For now, the hosts are configured via static classes (`LocalHost` or `SshHost`). An ASP.NET Core-specific extension that will use Microsoft's DI to do it is planned for the 1.0 release, but not available yet.

### Local host

Simply call:

```cs
LocalHost.Configure();
```

No extra configuration is needed, everything should "just work".

### SSH host

The SSH host needs three main configurations to work, the first is a connection pool configuration that will specify the settings of the SSH and SFTP connection pool and the credentials used to connect to the destination:

```cs
public sealed record ConnectionPoolConfiguration(
    ConnectionInfo ConnectionInfo,
    uint SshConnectionAmount,
    uint SftpConnectionAmount,
    TimeSpan KeepAliveInterval);
```

The second configuration is used to create PTYs (virtual terminals) that are needed to launch and interact with the firecracker/jailer processes. There's a sensible default configuration at `ShellConfiguration.Default`:

```cs
public sealed record ShellConfiguration(
    string Terminal,
    uint Columns,
    uint Rows,
    uint Width,
    uint Height,
    int BufferSize)
{
    public static readonly ShellConfiguration Default = new(
        "/bin/bash", 1000, 1000, 1000, 1000, 1000);
}
```

The third configuration is used for the `curl` tool that is needed to make HTTP requests to Unix domain sockets of the Firecracker process. SSH.NET needs to support Unix socket forwarding for this not to be necessary, but the [related issue](https://github.com/sshnet/SSH.NET/issues/238) hasn't been resolved for more than 7 years, so `curl` is used over SSH instead. A sensible default configuration is available at `CurlConfiguration.Default`:

```cs
public sealed record CurlConfiguration(
    TimeSpan PollFrequency,
    int RetryAmount,
    int RetryDelaySeconds,
    int RetryMaxTimeSeconds)
{
    public static readonly CurlConfiguration Default = new(TimeSpan.FromMilliseconds(5), 1, 0, 20);
}
```

Lastly, **ensure the following packages are installed on the SSH Linux machine!**:

- `tar` (working with archives)
- `untar` or `unar` (same thing)
- `curl` (requests)

To set up the host, call `SshHost` like so:

```cs
SshHost.Configure(new ConnectionPoolConfiguration(...), new CurlConfiguration(...), new ShellConfiguration(...));
```

The connection pool keeps up the constant amount of SSH and SFTP connections through your app's lifetime. **Ensure the connection pool is disposed of like so:**

```cs
SshHost.Dispose();
```
