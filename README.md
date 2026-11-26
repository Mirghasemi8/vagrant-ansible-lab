# Multi-Node DevOps Lab (Vagrant & Ansible)

Automated multi-node Debian 12 lab using Vagrant (`vmware_desktop`) and Ansible, featuring configurable worker node count (`NODE_COUNT`), SSH key distribution via `.keys/`, and on-demand Docker installation triggered from `control-node`.

## Architecture & Topology

| Hostname       | Role                 | Private IP      | Purpose                                                                                    |
| :------------- | :------------------- | :-------------- | :----------------------------------------------------------------------------------------- |
| `control-node` | Ansible control node | `192.168.56.10` | Installs Ansible, sets `ANSIBLE_CONFIG`, generates `ansible-key` (ed25519), runs playbooks |
| `node-1`       | Worker node          | `192.168.56.11` | Receives `ansible-key.pub` in `~/.ssh/authorized_keys`; target for Docker installation     |
| `node-2`       | Worker node          | `192.168.56.12` | Receives `ansible-key.pub` in `~/.ssh/authorized_keys`; target for Docker installation     |
| `node-3`       | Worker node          | `192.168.56.13` | Receives `ansible-key.pub` in `~/.ssh/authorized_keys`; target for Docker installation     |

When `NODE_COUNT` changes, worker nodes follow `node-<i>` at IP `192.168.56.<10 + i>`.

## Project Structure

```text
.
├── Vagrantfile                  # VM definitions, NODE_COUNT loop, SSH key setup, and run: "never" Docker trigger
├── README.md                    # Lab documentation and usage instructions
├── .keys/                       # Shared folder for exchanging ansible-key.pub between control-node and worker nodes
└── ansible/
    ├── ansible.cfg              # Ansible configuration file
    ├── inventory.ini            # Target hosts ([nodes]) and SSH connection variables ([nodes:vars])
    └── install-docker.yml       # Playbook to install docker.io, start/enable docker, and add vagrant to docker group
```

## Prerequisites

- **Git** installed on the host machine to clone the repository.
- **Vagrant** installed on the host machine.
- **VMware Desktop** installed with the Vagrant VMware provider/utility configured (`vmware_desktop`).
- **Base Box:** `bento/debian-12` (pulled automatically on initial `vagrant up` when online).

## Step-by-Step Guide After Cloning

### 1. Clone and Enter the Repository

```bash
git clone https://github.com/Mirghasemi8/vagrant-ansible-lab.git
cd vagrant-ansible-lab
```

### 2. Validate and Bring Up the Nodes

Validate the `Vagrantfile` syntax and provision `control-node` plus all worker nodes (`node-1` through `node-3`):

```bash
vagrant validate
vagrant up
```

What happens during `vagrant up`:

- `control-node` boots with IP `192.168.56.10`, installs Ansible, configures `ANSIBLE_CONFIG`, mounts `./ansible` and `./.keys`, and generates `/home/vagrant/.ssh/ansible-key` (copying `ansible-key.pub` into `.keys/`).
- `node-1` through `node-3` boot with IPs `192.168.56.11`–`192.168.56.13`, mount `./.keys`, and append `ansible-key.pub` to `/home/vagrant/.ssh/authorized_keys`.
- The `install-docker` provisioner has `run: "never"`, so Docker is not installed automatically during initial `vagrant up`.

### 3. Install Docker on the Nodes

Once `control-node` and all worker nodes are up and provisioned, install Docker on the nodes using the `install-docker` provisioner on `control-node`:

```bash
vagrant provision control-node --provision-with install-docker
```

This triggers the `install-docker` provisioner on `control-node`, which runs `/home/vagrant/ansible/install-docker.yml` against the `[nodes]` group in `ansible/inventory.ini` to install `docker.io`, start and enable the `docker` service, and add the `vagrant` user to the `docker` group.

Alternatively, you can SSH into `control-node` and run the playbook directly:

```bash
vagrant ssh control-node -c "export ANSIBLE_CONFIG=/home/vagrant/ansible/ansible.cfg && cd /home/vagrant/ansible && ansible-playbook install-docker.yml"
```

If an SSH session to a node was already open before the `vagrant` user was added to the `docker` group, reconnect (`vagrant ssh node-1`) so the group membership takes effect.

## Verification

1. **Check VM states:**

   ```bash
   vagrant status
   ```

   Confirm `control-node`, `node-1`, `node-2`, and `node-3` show `running`.

2. **Test Ansible ping from `control-node`:**

   ```bash
   vagrant ssh control-node -c "cd /home/vagrant/ansible && ansible nodes -m ping"
   ```

   Confirm each node responds with `"ping": "pong"`.

3. **Test Docker on worker nodes:**
   ```bash
   vagrant ssh node-1 -c "docker --version"
   vagrant ssh node-1 -c "docker run --rm hello-world"
   ```
   Confirm Docker runs without `sudo` and prints the `Hello from Docker!` message.

## Scaling, Troubleshooting & Cleanup

- **Scale worker nodes:** Change `ENV['NODE_COUNT']` in `Vagrantfile` (e.g., `"2"` or `"4"`), update `[nodes]` in `ansible/inventory.ini` to match the exact node names and IPs (`192.168.56.<10 + i>`), and run `vagrant up`.
- **Adjust OS or hardware resources:** Edit `ENV['DEFAULT_DISTRO']`, `v.memory`, or `v.cpus` in `Vagrantfile` before creating or recreating VMs.
- **Stale keys or VM conflicts:** If recreating VMs from scratch, remove old keys or destroy existing VMs cleanly first so SSH `authorized_keys` and private network IPs stay consistent.
- **Destroy all VMs:**
  ```bash
  vagrant destroy -f
  ```
