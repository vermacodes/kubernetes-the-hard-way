# Prerequisites

In this lab you will review the machine requirements necessary to follow this tutorial.

## Virtual or Physical Machines

This tutorial requires four (4) virtual or physical ARM64 or AMD64 machines running Debian 12 (bookworm). The following table lists the four machines and their CPU, memory, and storage requirements.

| Name    | Description            | CPU | RAM   | Storage |
|---------|------------------------|-----|-------|---------|
| jumpbox | Administration host    | 1   | 512MB | 10GB    |
| server  | Kubernetes server      | 1   | 2GB   | 20GB    |
| node-0  | Kubernetes worker node | 1   | 2GB   | 20GB    |
| node-1  | Kubernetes worker node | 1   | 2GB   | 20GB    |
| node-2  | Kubernetes worker node | 1   | 2GB   | 20GB    |

How you provision the machines is up to you, the only requirement is that each machine meet the above system requirements including the machine specs and OS version.

I am using Raspberry Pis as servers and they have following configurations. Also, I am using WSL on my windows computer as jump server.

| Name    | Description            | CPU | RAM    | Storage |
|---------|------------------------|-----|--------|---------|
| server  | Kubernetes server      | 4   | 8GB    | 128GB   |
| node-0  | Kubernetes worker node | 4   | 16GB   | 256GB   |
| node-1  | Kubernetes worker node | 4   | 8GB    | 128GB   |
| node-2  | Kubernetes worker node | 4   | 8GB    | 128GB   |

Once you have all four machines provisioned, verify the OS requirements by viewing the `/etc/os-release` file:

```bash
cat /etc/os-release
```

You should see something similar to the following output:

```text
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
```

Next: [setting-up-the-jumpbox](02-jumpbox.md)
