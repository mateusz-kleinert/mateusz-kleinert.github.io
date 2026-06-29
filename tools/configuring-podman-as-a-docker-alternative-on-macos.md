# Configuring Podman as a Docker Alternative on macOS

Podman is a daemonless, Docker-compatible container engine. On macOS it runs containers inside a lightweight Linux VM — the *Podman machine* — but, unlike Docker Desktop, gives you direct control over that VM. This guide covers installing the engine, sizing the VM's CPU and memory, getting it to trust your organization's CA certificates, and running existing Compose projects.

## Installing the Podman engine

There are two common ways to install the CLI on macOS.

The official route is the `.pkg` installer from [podman.io](https://podman.io) or the [GitHub releases page](https://github.com/containers/podman/releases). Podman's own documentation prefers this over Homebrew because the project controls the package directly.

The most common route in practice is Homebrew:

```bash
# Install the Podman CLI
brew install podman

# Confirm it installed and check your architecture
podman --version
uname -m          # arm64 = Apple Silicon, x86_64 = Intel
```

Installing the CLI does not start anything on its own. On macOS the `podman` command is a remote client that talks to a `podman` service running inside the Linux VM, so the next step is creating that VM.

If you also want a graphical interface, Podman Desktop is available (`brew install --cask podman-desktop`) and can create and manage machines through a UI, but everything below is done from the terminal so it works regardless of which you choose.

## Understanding the virtual machine model

Each Podman machine is a self-contained Linux VM, by default a custom Fedora CoreOS image. The machine is created with `podman machine init`, started with `podman machine start`, and managed as a named object — the default name is `podman-machine-default`.

macOS offers more than one virtualization provider, and which one you get by default depends on your chip:

- **Apple Silicon (M1–M4):** the default provider is `libkrun` (listed as "GPU enabled" in Podman Desktop), with Apple's Hypervisor framework (`applehv`) available as an alternative.
- **Intel Macs:** the default is `applehv`; `libkrun` is not available.
- **QEMU** is also available as a cross-architecture option and is the only provider that supports some features such as USB passthrough and live resource resizing.

You can override the default at creation time with `--provider` (`applehv`, `libkrun`, or `qemu`). For most users the default is the right choice; the provider mainly matters because it determines whether you can resize CPU and memory in place, as discussed below.

## Creating the machine with proper CPU and memory settings

The most reliable way to size a Podman machine is to set its resources when you create it. The relevant flags are `--cpus` (number of cores), `--memory` (RAM in **MiB**), `--disk-size` (in **GB**), and optionally `--swap` (in MiB).

```bash
# Create a machine sized for real work and start it immediately
podman machine init \
  --cpus 4 \
  --memory 8192 \
  --disk-size 100 \
  --now
```

The `--now` flag starts the machine right after initialization, so you don't need a separate `podman machine start`.

You can keep more than one machine for different profiles and switch between them:

```bash
podman machine init dev-machine  --cpus 4 --memory 8192
podman machine init ci-machine   --cpus 2 --memory 4096
podman machine list
```

After starting, confirm the VM actually received what you asked for:

```bash
podman machine inspect            # shows Resources block
podman machine ssh -- nproc       # CPU count seen inside the VM
podman machine ssh -- free -m     # memory seen inside the VM
```

### Changing resources after creation

To adjust an existing machine, stop it first and use `podman machine set`:

```bash
podman machine stop
podman machine set --cpus 6 --memory 16384 --disk-size 150
podman machine start
```

Two caveats are worth knowing. Disk size can generally only grow, never shrink. And live resizing of CPU and memory depends on the provider and Podman version — it is fully supported on QEMU but is not always honored on `applehv`/`libkrun`. If `podman machine inspect` shows the old values after a `set`, fall back to recreating the machine, which always works:

```bash
podman machine stop
podman machine rm
podman machine init --cpus 6 --memory 16384 --disk-size 150 --now
```

Recreating destroys the VM's local images and volumes, so push anything you need to a registry or back up named volumes first.

## Setting trusted CA bundles for the virtual machine

This is the configuration that most often blocks people behind a corporate network. Tools like Zscaler and other TLS-inspecting proxies re-sign HTTPS traffic with an internal certificate authority. You may have already trusted that CA in the macOS keychain, so your browser and host tools work — but the Podman *VM* has its own, separate trust store. Commands run inside it (pulling images, `podman login`, `curl` from a container) will fail with the telltale error:

```
x509: certificate signed by unknown authority
```

The fix is to make the VM trust the same CA(s). The dependable, version-independent way is to add the certificate to the VM's trust store yourself; recent Podman builds also offer a flag that automates it, covered afterward.

### Option 1: Add a CA bundle to the VM's trust store

The VM is Fedora-based, so trusted anchors live in `/etc/pki/ca-trust/source/anchors/` and are activated with `update-ca-trust`. This works on every current Podman release.

The most convenient way is to pipe the certificate from the host straight into the VM, no interactive editing required:

```bash
# Copy a host-side CA file (PEM/CRT) into the VM's anchors directory
cat corporate-ca.crt | \
  podman machine ssh "sudo tee /etc/pki/ca-trust/source/anchors/corporate-ca.crt > /dev/null"

# Rebuild the VM's trust store
podman machine ssh "sudo update-ca-trust"

# Restart so all services in the VM pick up the change
podman machine stop && podman machine start
```

A concatenated bundle (multiple certificates in one `.pem`) works just as well as a single certificate — drop the whole bundle into the anchors directory under one filename.

Then restart the machine from the host as shown above. If a pull still fails after editing the trust store, a restart is almost always the missing step — `update-ca-trust` alone does not always reload every running service.

### Option 2: Import the host's CA certificates automatically (newer Podman)

Recent Podman builds add an `--import-native-ca` flag to `podman machine init` and `podman machine set` that copies the host's trusted CAs into the VM on every boot, keeping the two trust stores in sync without manual editing. Availability is version-dependent, so confirm your build has it before relying on it:

```bash
podman machine init --help | grep import-native-ca
```

If the flag is listed, you can enable it at creation time, or on an existing machine:

```bash
# At creation
podman machine init --import-native-ca --cpus 4 --memory 8192 --now

# Or on an existing machine
podman machine stop
podman machine set --import-native-ca
podman machine start
```

It is off by default, so you have to opt in. If `--help` does not list it, your release predates the feature — use the manual method above, which works everywhere.

### Option 3: Trust a CA for a single registry only

If you only need to talk to one private registry rather than trusting a CA system-wide, Podman reads per-registry certificates from a `certs.d` directory. On the host this lives under your config directory:

```bash
mkdir -p ~/.config/containers/certs.d/registry.company.com
cp corporate-ca.crt ~/.config/containers/certs.d/registry.company.com/ca.crt

# For mutual TLS, add a client cert/key in the same directory
# cp client.cert ~/.config/containers/certs.d/registry.company.com/client.cert
# cp client.key  ~/.config/containers/certs.d/registry.company.com/client.key
```

The directory name must match the registry hostname (and port, if non-standard, as `host:port`). This is narrower than the system trust store and a good fit when a single internal registry is the only thing using a private CA.

As a last resort for throwaway local testing you can skip verification with `podman pull --tls-verify=false ...` or `podman login --tls-verify=false ...`, but never leave that in place for anything real — it disables the protection the certificate exists to provide.

## Using Podman as a Docker drop-in

With the machine running and trust configured, the engine behaves like Docker. The simplest bridge is a shell alias, since the CLIs are command-compatible:

```bash
alias docker=podman
```

For tools and SDKs that expect a Docker socket, expose Podman's socket and point `DOCKER_HOST` at it. Find the socket path with:

```bash
podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}'
export DOCKER_HOST="unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')"
```

Podman Desktop also offers a Docker compatibility toggle that aliases the default `/var/run/docker.sock` location so unmodified Docker tooling keeps working.

## Running multi-container apps with Compose

If your project already uses a `docker-compose.yml`, you can keep it. There are two distinct things called "compose" in the Podman world, and the difference matters.

The first is the built-in `podman compose` subcommand. As of current Podman it is a *thin wrapper* around an external compose provider rather than its own implementation — it locates an installed provider, sets up the environment so that provider talks to your local Podman socket, and passes your command and arguments straight through. The default providers are `docker-compose` and `podman-compose`, and if both are installed `docker-compose` takes precedence because it is the reference implementation of the Compose specification.

The second is `podman-compose`, a standalone Python tool that parses Compose files and translates them into native Podman CLI commands. It is daemonless, leans on Podman's rootless model, and by default places a project's services into a single pod. It implements a large subset of the Compose spec but is explicitly not a 1:1 replacement — newer or more advanced features (some health-check-based `depends_on` conditions, `deploy`/`replicas`, certain profile and network behaviors) may need adjustment or may not work.

Install whichever provider you prefer:

```bash
# The standalone translator
pip3 install podman-compose

# …or use Docker's compose plugin as the provider, if it is installed
```

Then drive it through Podman. Because `podman compose` just forwards to the provider, the commands look exactly like Docker's:

```bash
podman compose up -d
podman compose ps
podman compose down

# Or call the standalone tool directly
podman-compose up -d
```

On macOS, remember Compose still runs against the VM, so the Podman machine must be started first. If you want to pin the provider rather than let precedence decide, set it explicitly — either with the `PODMAN_COMPOSE_PROVIDER` environment variable or the `compose_providers` field in `containers.conf`:

```bash
export PODMAN_COMPOSE_PROVIDER=podman-compose
```

### A note on `COMPOSE_FILE` and `COMPOSE_PATH_SEPARATOR`

Because `podman compose` hands off to a Compose provider, the standard Compose environment variables are honored by that provider — including the two that govern *which* files get loaded.

`COMPOSE_FILE` lets you name one or more Compose files without repeating `-f` on the command line. When you list multiple files, later files are merged over earlier ones, so ordering is significant:

```bash
export COMPOSE_FILE=compose.yaml:compose.override.yaml:compose.prod.yaml
podman compose up -d        # loads and merges all three, left to right
```

`COMPOSE_PATH_SEPARATOR` controls the character that separates those paths. The default is platform-dependent: a colon (`:`) on macOS and Linux, and a semicolon (`;`) on Windows. On a Mac you can therefore rely on the colon, but it is worth setting the separator explicitly in a shared `.env` file so the same configuration behaves identically across a mixed macOS/Windows team:

```bash
# .env — make the file list portable across operating systems
COMPOSE_PATH_SEPARATOR=:
COMPOSE_FILE=compose.yaml:compose.override.yaml
```

A couple of practical cautions. A path that legitimately contains a colon (less common on macOS, but possible) will be misread as a separator, which is the main reason to set `COMPOSE_PATH_SEPARATOR` deliberately. And these variables are read from the shell environment and the project's `.env`, with the shell taking precedence — so an exported `COMPOSE_FILE` will silently override what's in `.env`, which can be a confusing surprise when a teammate's setup loads different files than yours.

## Verifying the setup

A quick check confirms the engine and resources are in order:

```bash
podman machine list                       # machine is Running with expected CPU/MEM
podman info                               # full engine and host details
```

To confirm the CA is trusted, exec into the VM and inspect its trust store directly rather than triggering a pull. The Fedora-based VM ships the `trust` tool, which enumerates every anchor the system trusts; filter it by your CA's name or organization:

```bash
# List trusted anchors inside the VM and look for your CA
podman machine ssh "trust list | grep -i 'Corporate CA'"
```

If `trust list` shows no match, the CA isn't trusted yet — revisit the trust configuration above, and make sure you restarted the machine afterward, since `update-ca-trust` alone does not always reload every running service.
