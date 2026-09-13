# OpenShift SNO Agent-Based Installer (Disconnected)

Ansible playbook that generates a bootable agent ISO for deploying a Single Node OpenShift (SNO) cluster in a disconnected (air-gapped) environment.

The agent-based installer is fully CLI-driven — no web UI or external services required. The generated ISO contains RHCOS and an embedded assisted installer agent that performs the installation autonomously.

## Architecture

```
┌─────────────────────┐      ┌──────────────────────┐
│   Workstation        │      │   Mirror Registry    │
│                      │      │                      │
│  generate-agent-iso  │      │  OCP release images  │
│  playbook            │      │  (podman registry)   │
│         │            │      │        ▲              │
│         ▼            │      └────────┼──────────────┘
│    agent.iso         │               │
│         │            │               │
└─────────┼────────────┘               │
          │                            │
          ▼                            │
┌─────────────────────┐                │
│   Target Node       │                │
│                      │    pulls images│
│  Boots ISO ──────────┼────────────────┘
│  Installs RHCOS      │
│  Bootstraps OCP      │
│  Runs SNO cluster    │
└──────────────────────┘
```

**How it works:**

1. The playbook templates `install-config.yaml` and `agent-config.yaml` from your variables
2. `openshift-install agent create image` generates a bootable ISO (~1.3 GB)
3. Boot the target machine from the ISO
4. The embedded agent writes RHCOS to disk, reboots, and bootstraps the cluster
5. The cluster pulls OCP container images from the mirror registry during installation
6. After ~30 minutes, a fully operational single-node OpenShift cluster is running

## Prerequisites

- `openshift-install` CLI (matching your target OCP version)
- `oc` CLI
- `nmstate` package (for NMState network config validation)
- Ansible (`ansible-playbook`)
- A mirror registry with OCP release images (see [Mirror Registry Setup](#mirror-registry-setup))
- DNS records for the cluster (see [DNS Requirements](#dns-requirements))
- Pull secret from [console.redhat.com](https://console.redhat.com/openshift/downloads)

### Installing the CLI tools

Download from the [OpenShift mirror](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/):

```bash
OCP_VERSION="4.18.54"
curl -sL https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-linux.tar.gz | sudo tar xz -C /usr/local/bin/
curl -sL https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux.tar.gz | sudo tar xz -C /usr/local/bin/
sudo dnf install -y nmstate   # RHEL/Fedora
```

## Quick Start

```bash
# 1. Clone and configure
git clone <this-repo>
cd sno-agent-based-installer
cp vars/cluster-config.example.yml vars/cluster-config.yml

# 2. Edit vars/cluster-config.yml with your environment values

# 3. Place your secret files
cp ~/Downloads/pull-secret.txt files/pull-secret.json
cp /path/to/mirror-registry-ca.crt files/mirror-ca.crt

# 4. Ensure your pull secret includes auth for the mirror registry
#    (see "Pull Secret Configuration" below)

# 5. Generate the ISO
ansible-playbook generate-agent-iso.yml

# 6. Boot the target machine from output/<cluster_name>/agent.x86_64.iso

# 7. Monitor the installation
openshift-install --dir output/<cluster_name> agent wait-for bootstrap-complete --log-level=info
openshift-install --dir output/<cluster_name> agent wait-for install-complete
```

## Configuration

Copy `vars/cluster-config.example.yml` to `vars/cluster-config.yml` and fill in your values:

| Variable | Description | Example |
|----------|-------------|---------|
| `cluster_name` | Cluster name (used in DNS) | `my-cluster` |
| `base_domain` | Base DNS domain | `example.com` |
| `ocp_version` | Target OpenShift version | `4.18` |
| `node_ip` | Static IP for the SNO node | `192.168.1.100` |
| `prefix_length` | Subnet prefix length | `24` |
| `gateway` | Default gateway | `192.168.1.1` |
| `dns_server` | DNS server | `192.168.1.1` |
| `interface_name` | Network interface name | `enp1s0` |
| `mac_address` | NIC MAC address | `aa:bb:cc:dd:ee:ff` |
| `machine_network_cidr` | Node network CIDR | `192.168.1.0/24` |
| `root_device` | Install target disk (optional) | `/dev/sda` |
| `mirror_registry` | Mirror registry hostname:port | `registry.example.com:5000` |
| `image_content_sources` | Mirror mappings from `oc adm release mirror` output | See example file |

## DNS Requirements

Three DNS records must point to the SNO node IP **before** booting the ISO:

| Record | Example |
|--------|---------|
| `api.<cluster>.<domain>` | `api.my-cluster.example.com → 192.168.1.100` |
| `api-int.<cluster>.<domain>` | `api-int.my-cluster.example.com → 192.168.1.100` |
| `*.apps.<cluster>.<domain>` | `*.apps.my-cluster.example.com → 192.168.1.100` |

The wildcard `*.apps` record is required for the OpenShift router (ingress) to expose routes for the console, OAuth, and any applications you deploy.

## Mirror Registry Setup

A disconnected install requires a local registry hosting the OCP release images. Any OCI-compliant registry works (Docker registry, Quay, Harbor, etc.).

### 1. Set up the registry

```bash
# Generate TLS certs (must include SAN for both hostname and IP)
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout registry.key -out registry.crt \
  -subj "/CN=registry.example.com" \
  -addext "subjectAltName=DNS:registry.example.com,IP:192.168.1.50"

# Run a simple registry
sudo mkdir -p /opt/registry/{data,certs}
sudo cp registry.{crt,key} /opt/registry/certs/

sudo podman run -d --name mirror-registry \
  -p 5000:5000 --restart=always \
  -v /opt/registry/data:/var/lib/registry:z \
  -v /opt/registry/certs:/certs:z \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  docker.io/library/registry:2
```

### 2. Mirror the OCP release

```bash
OCP_VERSION="4.18.54"
REGISTRY="registry.example.com:5000"

oc adm release mirror \
  --from=quay.io/openshift-release-dev/ocp-release:${OCP_VERSION}-x86_64 \
  --to=${REGISTRY}/openshift/release \
  --to-release-image=${REGISTRY}/openshift/release:${OCP_VERSION}-x86_64 \
  --insecure=true
```

This takes 15-30 minutes and downloads ~19 GB of images. The command output includes the `imageContentSources` block to copy into your `cluster-config.yml`.

### 3. Copy the CA certificate

```bash
cp registry.crt files/mirror-ca.crt
```

## Pull Secret Configuration

Your pull secret (`files/pull-secret.json`) **must** include an auth entry for the mirror registry. Without it, the agent installer rejects the configuration.

If your mirror registry doesn't require authentication, add a placeholder entry:

```bash
# Add mirror registry auth to pull secret
REGISTRY="registry.example.com:5000"
PULL_SECRET=$(cat files/pull-secret.json)

echo "$PULL_SECRET" | jq --arg reg "$REGISTRY" \
  --arg auth "$(echo -n 'unused:unused' | base64)" \
  '.auths[$reg] = {"auth": $auth}' > files/pull-secret.json
```

If your registry requires real credentials, replace `unused:unused` with `username:password`.

## Boot Order (VM or Bare Metal)

The ISO must remain attached throughout the installation. However, after RHCOS is written to disk, the node must reboot **from the disk**, not the ISO.

**For VMs (libvirt/KVM):** Set the boot order to disk first, CDROM second:

```xml
<os>
  <type arch='x86_64' machine='pc-q35-...'>hvm</type>
  <boot dev='hd'/>
  <boot dev='cdrom'/>
</os>
```

Do **not** set `<boot order='1'/>` on the CDROM device — this overrides the `<os>` boot order and causes the node to reboot into the ISO in a loop, restarting the install each time.

**For bare metal:** Configure the BIOS/UEFI boot order to prioritize the local disk over USB/CD. Use one-time boot override to initially boot from the ISO.

## Installation Timeline

| Phase | Duration | What happens |
|-------|----------|--------------|
| Boot & validation | ~2 min | Agent boots, discovers hardware, validates config |
| Write to disk | ~2 min | RHCOS is written to the target disk |
| Reboot to disk | ~1 min | Node reboots from installed OS |
| Bootstrap | ~15 min | etcd, kube-apiserver, and bootstrap control plane start |
| CVO rollout | ~15 min | Cluster Version Operator deploys all 34 operators |
| **Total** | **~30-35 min** | Fully operational SNO cluster |

## Post-Install

After installation completes:

```bash
# Credentials are in the output directory
export KUBECONFIG=output/<cluster_name>/auth/kubeconfig
cat output/<cluster_name>/auth/kubeadmin-password

# Verify the cluster
oc get nodes
oc get co
oc get clusterversion

# Access the web console
# https://console-openshift-console.apps.<cluster>.<domain>
# Login: kubeadmin / <password from above>
```

## Project Structure

```
.
├── ansible.cfg                         # Ansible configuration (local only)
├── generate-agent-iso.yml              # Main playbook
├── files/
│   ├── .gitkeep
│   ├── pull-secret.json                # Your pull secret (git-ignored)
│   └── mirror-ca.crt                   # Mirror registry CA cert (git-ignored)
├── templates/
│   ├── install-config.yaml.j2          # OpenShift install config template
│   └── agent-config.yaml.j2            # Agent config template (NMState networking)
├── vars/
│   ├── cluster-config.example.yml      # Example configuration (committed)
│   └── cluster-config.yml              # Your configuration (git-ignored)
└── output/                             # Generated ISO and auth files (git-ignored)
```

## Troubleshooting

**SSH into the node during install** (RHCOS uses the `core` user):
```bash
ssh core@<node_ip>
```

**Check assisted service logs** (while booted from ISO):
```bash
sudo podman logs service
```

**Check installation stage:**
```bash
sudo podman logs service 2>&1 | grep "reached installation stage"
```

**Check bootkube progress** (after reboot to disk):
```bash
sudo journalctl -u bootkube.service -f
```

**Common issues:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| "pull secret must contain auth for registry" | Pull secret missing mirror registry entry | Add auth entry (see [Pull Secret Configuration](#pull-secret-configuration)) |
| Node reboots into ISO repeatedly | CDROM boot order takes priority over disk | Fix boot order to disk first (see [Boot Order](#boot-order-vm-or-bare-metal)) |
| "Unable to read from the discovery media" | ISO was ejected during install | Keep ISO attached until install completes |
| "nmstatectl: executable file not found" | nmstate not installed on build host | `sudo dnf install -y nmstate` |
| Bootstrap stuck "Waiting for kube-apiserver" | Normal — takes 10-15 min | Wait; check `sudo crictl ps \| wc -l` is growing |
| API cert errors with saved kubeconfig | Kubeconfig from a previous ISO generation | Get fresh kubeconfig from the installed node |

## License

MIT
