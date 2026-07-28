# Setting up a SLURM Cluster

This document describes how I set up a SLURM cluster on the S3IT ScienceCloud. I followed and adapted material from https://github.com/SergioMEV/slurm-for-dummies. The instructions below are presented as a sequence you can follow to build a golden image and launch worker instances.

Prerequisites
- An account on the S3IT ScienceCloud with permission to launch instances and create volumes.
- Ubuntu (or other Debian-based) instance image — these steps assume the apt package manager.
- Basic familiarity with SSH, sudo, and system administration.

Overview
1. Launch an instance and connect via SSH.
2. Create SSH keys and share the public key among nodes.
3. Install build tools and dependencies.
4. Install and configure Munge (authentication service).
5. Install and configure NFS (shared file systems for /var/spool or home directories if needed).
6. Build and install Slurm from source (recommended version: 25.05.8).
7. Create Slurm users and directories, add configuration files, and enable services.
8. Verify cgroup v2 support, Slurm plugins, and services are running.

1) Launch an instance
- Use the ScienceCloud web UI (https://cloud.science-it.uzh.ch/) to create a VM. Ensure you have the required permissions.
- Connect using:
```sh
ssh ubuntu@<assigned.ip.address>
```
Replace `<assigned.ip.address>` with your instance IP.

2) Update the system and create SSH keys
```sh
sudo apt-get update
sudo apt-get upgrade -y
```

Generate an SSH key pair (avoid overwriting an existing key unless intended):
```sh
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

Fix SSH directory and file permissions:
```sh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

Copy the public key to the authorized_keys file (append to the file on each node that should allow passwordless access):
```sh
cat ~/.ssh/id_rsa.pub
# copy the output and paste into ~/.ssh/authorized_keys on the other instance(s)
```
You can edit authorized_keys locally:
```sh
nano ~/.ssh/authorized_keys
# paste the public key on a new line, save and exit
```

3) Install necessary packages (build tools and libraries)
```sh
sudo apt install -y build-essential libhwloc-dev libjson-c-dev libcurl4-openssl-dev libyaml-dev libsystemd-dev libdbus-1-dev pkg-config libmysqlclient-dev libssl-dev libpam0g-dev
```

4) Install OpenSSH (if not already installed)
```sh
sudo apt install -y openssh-server openssh-client
sudo systemctl enable --now ssh
```

5) Install and configure Munge
Munge provides authentication between Slurm daemons. You must create a single shared munge key and distribute it to every node (controller and workers) with identical permissions.

Install:
```sh
sudo apt install -y munge libmunge2 libmunge-dev
```

Create a munge key (example — run on the controller, then copy /etc/munge/munge.key to workers):
```sh
sudo dd if=/dev/urandom of=/etc/munge/munge.key bs=1 count=1024
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

Set directory permissions:
```sh
sudo chown -R munge:munge /etc/munge /var/log/munge /var/lib/munge /run/munge
sudo chmod 0700 /etc/munge /var/lib/munge /var/log/munge
sudo chmod 0755 /run/munge
```

Enable and start munge:
```sh
sudo systemctl enable --now munge
```

Verify munge:
```sh
munge -n | unmunge | grep STATUS
# Should return a STATUS line indicating successful encrypt/decrypt
```

6) Install NFS (if you plan to export shared directories)
```sh
sudo apt install -y nfs-kernel-server nfs-common
# edit /etc/exports on the server to publish any exported directories, then
sudo exportfs -a
sudo systemctl enable --now nfs-kernel-server
```

7) Build and install Slurm from source (recommended)
Important: Do NOT install the distribution package `slurm-wlm` if you plan to use Slurm v25. The distro package may provide an older version that conflicts.

Download, build, and install Slurm v25.05.8:
```sh
wget https://download.schedmd.com/slurm/slurm-25.05.8.tar.bz2
tar -xjf slurm-25.05.8.tar.bz2
cd slurm-25.05.8

./configure \
    --prefix=/usr \
    --sysconfdir=/etc/slurm \
    --with-munge

make -j$(nproc)
sudo make install
sudo ldconfig
```

Notes:
- If you need a different Slurm version, change the version string consistently in the download and documentation.
- After `make install`, ensure Slurm libraries and plugins are in the expected library path.

8) Verify cgroup v2 support & Slurm cgroup plugin
Check for the compiled plugin:
```sh
find src/plugins -name "*.so" | grep cgroup
# e.g., ./src/plugins/cgroup/v2/.libs/cgroup_v2.so
```

Check the running cgroup filesystem type:
```sh
stat -fc %T /sys/fs/cgroup
# expected: cgroup2fs
```

Also verify installed plugins (example):
```sh
find /usr -name "cgroup*.so" 2>/dev/null
# expect: /usr/lib/slurm/cgroup_v2.so (and maybe cgroup_v1.so)
```

9) Create Slurm user, groups and directories
```sh
sudo groupadd -r slurm
sudo useradd -r -g slurm -d /var/lib/slurm -s /usr/sbin/nologin slurm

sudo mkdir -p /var/spool/slurm
sudo chown slurm:slurm /var/spool/slurm
sudo chmod 700 /var/spool/slurm

sudo mkdir -p /var/lib/slurm/slurmd
sudo chown slurm:slurm /var/lib/slurm/slurmd

sudo mkdir -p /var/log/slurm
sudo chown slurm:slurm /var/log/slurm
sudo chmod 755 /var/log/slurm

sudo mkdir -p /etc/slurm
```

Install systemd service files:
```sh
# from the slurm source directory (if these files exist in etc/)
sudo cp etc/slurmctld.service /etc/systemd/system/
sudo cp etc/slurmd.service /etc/systemd/system/
sudo systemctl daemon-reload
```

10) Configure cgroup.conf and slurm.conf
Create `/etc/slurm/cgroup.conf` and add:
```
CgroupPlugin=cgroup/v2

ConstrainCores=yes
ConstrainRAMSpace=yes

AllowedRAMSpace=100
AllowedSwapSpace=0
```

Create `/etc/slurm/slurm.conf`:
- Use the provided template (link or include file).
- Edit the ControlMachine, NodeName, and partitions to match your cluster.
- For worker node resources, run on each worker:
```sh
lscpu | grep -E "CPU\(s\)|Socket|Core|Thread"
free -h
```
and use those outputs to define CPUs and memory in the `NodeName` entries.

11) Enable and start Slurm services
On the controller:
```sh
sudo systemctl enable --now slurmctld
```
On each worker:
```sh
sudo systemctl enable --now slurmd
```

Verification and troubleshooting
- Check service status:
```sh
sudo systemctl status munge slurmctld slurmd nfs-kernel-server
```
- Check Slurm: `sinfo`, `squeue`, `scontrol show nodes`
- Check logs in `/var/log/slurm` and `/var/log/munge`.

Notes & recommendations
- Keep the munge key identical on all nodes; protect it and use secure transfer (scp with root or sudo).
- Prefer building Slurm from source only if you require a specific newer version; otherwise use distro packages that match your environment.
- Consider automating image creation (golden image) with a script so new nodes are identical.

If you want, I can:
- Commit this edited README to the branch `update/readme-proofread-slurm` and open a pull request, or
- Make a smaller change that only fixes grammar and critical typos (no structural additions).
