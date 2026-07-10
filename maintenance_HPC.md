# Adding and Removing Nodes in a Slurm HPC Cluster

One of the main advantages of using **Slurm** is that adding and removing compute nodes is a routine operation. The exact process depends on whether your cluster is **static** (a fixed set of virtual machines) or **dynamic** (virtual machines created and destroyed on demand).

The cluster described in this tutorial is a **static cluster**, where each worker node is manually configured.

---

# Overview

Every new worker node must satisfy four basic requirements before Slurm can schedule jobs on it:

1. It can communicate with the head NODE.
2. It shares the same MUNGE authentication key.
3. It uses the same Slurm configuration.
4. It has access to the shared filesystem.

Everything else in the setup process supports one or more of these requirements.

---

# Adding a New Worker Node

Assume the cluster currently contains:

```
frontend001
compute001
compute002
```

and a new node called `compute003` is being added.

## Step 1. Launch the VM

Create a new virtual machine using the same operating system image as the existing worker nodes.

Ideally, the new node should have:

- The same Ubuntu version
- The same Slurm version
- The same system packages installed

Using VM snapshots or cloud images makes this much easier.

---

## Step 2. Connect the Node to the Private Network

The new node must be able to communicate with the controller node (`frontend001`) and vice versa.

If using `/etc/hosts`, add the new node to every machine:

```text
172.23.x.x compute003
```

Every node should have the same host table.

> **Note:** For larger clusters, DNS is recommended instead of manually maintaining `/etc/hosts`.

---

## Step 3. Install Required Software

Install the required packages on the new worker:

```bash
sudo apt update

sudo apt install \
    slurm-wlm \
    munge \
    nfs-common \
    openssh-server
```

At this point the software is installed but not yet configured.

---

## Step 4. Copy the MUNGE Key

Copy the controller's authentication key:

```
/etc/munge/munge.key
```

to the new worker.

Restart MUNGE:

```bash
sudo systemctl restart munge
```

Verify authentication:

```bash
munge -n | ssh frontend001 unmunge
```

If the command succeeds, the worker and controller share the same authentication key.

---

## Step 5. Copy the Slurm Configuration

Copy

```
/etc/slurm/slurm.conf
```

from the controller to the new worker.

All nodes must use exactly the same configuration file.

---

## Step 6. Update `slurm.conf`

On the controller, add the new node:

```text
NodeName=compute003 CPUs=4 RealMemory=15000 State=UNKNOWN
```

Then add it to the partition:

```text
PartitionName=main \
Nodes=compute001,compute002,compute003
```

For larger clusters, node ranges are easier to maintain:

```text
Nodes=compute[001-003]
```

---

## Step 7. Distribute the Updated Configuration

Copy the updated `slurm.conf` to every worker node.

The configuration file should be identical across the entire cluster.

---

## Step 8. Restart Slurm

Restart the controller daemon:

```bash
sudo systemctl restart slurmctld
```

Restart the worker daemon:

```bash
sudo systemctl restart slurmd
```

---

## Step 9. Mount the Shared Filesystem

Mount the NFS export:

```bash
sudo mount frontend001:/home /home
```

For permanent mounts, add the entry to `/etc/fstab`.

---

## Step 10. Verify the Node

Check that Slurm recognizes the new node:

```bash
sinfo
```

The output should contain:

```text
compute001
compute002
compute003
```

Additional information can be obtained with:

```bash
scontrol show nodes
```

---

# Removing a Worker Node

There are two ways to remove a node depending on whether the removal is temporary or permanent.

---

## Option 1. Temporary Removal (Maintenance)

If the node will return later, place it into the **DRAIN** state:

```bash
scontrol update NodeName=compute002 State=DRAIN Reason="maintenance"
```

This prevents new jobs from being scheduled while allowing existing jobs to finish.

Once maintenance is complete:

```bash
scontrol update NodeName=compute002 State=RESUME
```

This is the recommended procedure for hardware maintenance or system updates.

---

## Option 2. Permanent Removal

If the worker node is being deleted:

Stop the Slurm daemon:

```bash
sudo systemctl stop slurmd
```

Delete the virtual machine.

Edit `slurm.conf` and remove:

```text
NodeName=compute002
```

Remove the node from its partition:

```text
PartitionName=main \
Nodes=compute001,compute003
```

Distribute the updated configuration to all remaining nodes.

Restart the controller:

```bash
sudo systemctl restart slurmctld
```

---

# Recovering a Failed Node

If a worker unexpectedly crashes, Slurm will eventually mark it as:

```text
DOWN
```

After the node has been repaired or rebooted:

```bash
sudo systemctl restart slurmd
```

If necessary, return it to service:

```bash
scontrol update NodeName=compute003 State=RESUME
```

---

# Scaling the Cluster

The current setup requires manually:

- Editing `/etc/hosts`
- Copying the MUNGE key
- Copying `slurm.conf`
- Installing packages

This approach is perfectly acceptable for small clusters (2–10 nodes) but becomes difficult to maintain as the cluster grows.

Most production HPC systems automate these tasks using:

- **Ansible** for package installation and configuration management
- **Cloud-init** for automatic configuration during VM creation
- **DNS** instead of `/etc/hosts`
- Configuration management systems such as **Puppet** or **SaltStack**

---

# Recommended Workflow for Cloud HPC

Since this cluster runs on the **S3IT ScienceCloud**, worker nodes can be treated as disposable virtual machines.

A common workflow is:

1. Create a "golden" worker image containing:
   - Ubuntu
   - Slurm
   - MUNGE
   - NFS client
   - Required software

2. Launch new worker VMs from this image whenever additional resources are needed.

3. Use cloud-init or Ansible to:
   - Assign the hostname
   - Install the current `munge.key`
   - Install the latest `slurm.conf`
   - Start the `slurmd` service

4. Verify that Slurm recognizes the new node using:

```bash
sinfo
```

With this approach, adding a worker node becomes largely automatic, making the cluster easier to scale and maintain.
