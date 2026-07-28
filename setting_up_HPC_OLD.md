# Setting Up a SLURM Cluster

To setup the new computational cluster in the new [S3IT ScienceCloud](https://docs.s3it.uzh.ch/cloud/overview/), I followed [this](https://github.com/SergioMEV/slurm-for-dummies) GitHub page and modified accordingly.

## Prerequisites

Before starting, ensure you have:
- SSH access between all nodes
- Sudo privileges on all nodes
- Network connectivity between controller and worker nodes
- Ubuntu OS installed on all nodes

## Launch Instances

Following the S3IT recommendations, launch:
- **One controller node** (acts as the cluster manager)
- **Two or more worker nodes** (execute computational tasks)

## Set Up Private Network

**On all nodes** (controller and workers), edit the `/etc/hosts` file and add the IPs and names of all the nodes in the cluster:

```
xxx.xx.xxx.xx1	frontend001
xxx.xx.xxx.xx2	med32compute001
xxx.xx.xxx.xx3	med32compute002
```

Replace the IPs with your actual network addresses.

## 1) Setup SSH

Run the following command on all nodes:

```bash
sudo apt install openssh-server openssh-client
```

Test if it works by running the following command from the controller node:

```bash
ssh med32compute001  # Test connection to a worker node
```

## 2) Install and Setup Munge

Munge is a cryptographic authentication service used by SLURM to authenticate communication between nodes.

### Install Munge on all nodes

Run the following commands on **all nodes** (controller and workers):

```bash
sudo apt install munge libmunge2 libmunge-dev
```

Test the installation by verifying Munge authentication:

```bash
munge -n | unmunge | grep STATUS
```

This should return a `STATUS=Success` message.

### Configure Munge permissions on controller node

Ensure that all munge files in the **controller node** have the correct permissions:

```bash
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key
```

### Enable and test Munge on controller node

```bash
sudo systemctl enable munge
sudo systemctl restart munge
sudo systemctl status munge
```

### Synchronize Munge key to worker nodes

It is necessary to have the exact same `/etc/munge/munge.key` on all worker nodes. Copy the key from the **controller node** to each **worker node**:

```bash
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/munge/munge.key ubuntu@xxx.xx.xxx.xx2:
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/munge/munge.key ubuntu@xxx.xx.xxx.xx3:
```

### Configure Munge on worker nodes

On each **worker node**, replace the `/etc/munge/munge.key` file:

```bash
sudo rm /etc/munge/munge.key
sudo mv munge.key /etc/munge

# Ensure correct permissions
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key

# Enable and restart Munge
sudo systemctl enable munge
sudo systemctl restart munge

# Test Munge authentication across nodes
munge -n | ssh frontend001 unmunge
```

## 3) Install and Setup SLURM

### Install SLURM on all nodes

On **all nodes** (controller and workers), run:

```bash
sudo apt-get update
sudo apt install slurm-wlm
```

### Enable SLURM controller daemon

In the **controller node**, enable and start the SLURM controller:

```bash
sudo systemctl enable slurmctld
sudo systemctl restart slurmctld
sudo systemctl status slurmctld
```

### Enable SLURM compute daemons

On all **worker nodes**, enable and start the SLURM compute daemons:

```bash
sudo systemctl enable slurmd
sudo systemctl restart slurmd
sudo systemctl status slurmd
```

### Configure SLURM

On the **controller node**, edit the SLURM configuration file:

```bash
sudo nano /etc/slurm/slurm.conf
```

Use the following example configuration as a template. Adjust the values according to your cluster specifications (node names, CPU counts, memory, etc.):

```ini
############################
# BASIC CLUSTER SETTINGS
############################

ClusterName=primrose
ControlMachine=primrose-frontend001
SlurmUser=slurm

AuthType=auth/munge

StateSaveLocation=/var/spool/slurm
SlurmdSpoolDir=/var/lib/slurm/slurmd

############################
# NODES
############################

NodeName=med32compute001 CPUs=2 Sockets=2 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=7700 State=UNKNOWN
NodeName=med32compute002 CPUs=2 Sockets=2 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=7700 State=UNKNOWN

############################
# PARTITIONS
############################

PartitionName=main \
  Nodes=med32compute001,med32compute002 \
  Default=YES \
  MaxTime=INFINITE \
  State=UP

############################
# SCHEDULER
############################

SchedulerType=sched/backfill

SelectType=select/cons_res
SelectTypeParameters=CR_Core

############################
# PRIORITY / FAIRSHARE
############################

PriorityType=priority/multifactor
PriorityWeightAge=12500
PriorityWeightFairshare=100000
PriorityWeightJobSize=12500
PriorityWeightPartition=0
PriorityWeightQOS=20000

PriorityDecayHalfLife=14-0
PriorityUsageResetPeriod=NONE
PriorityMaxAge=7-0
PriorityFavorSmall=NO

############################
# ACCOUNTING
############################

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=frontend001
AccountingStoragePort=6819
AccountingStorageEnforce=associations,limits,qos
AccountingStoreFlags=job_comment

JobCompType=jobcomp/none

############################
# ACCOUNTING GATHERING
############################

JobAcctGatherType=jobacct_gather/linux
JobAcctGatherFrequency=60

AcctGatherEnergyType=acct_gather_energy/none
AcctGatherInfinibandType=acct_gather_infiniband/none
AcctGatherFilesystemType=acct_gather_filesystem/none
AcctGatherProfileType=acct_gather_profile/none

############################
# GRES
############################

GresTypes=gpu

############################
# JOB CONTROL
############################

JobRequeue=1
MaxArraySize=1000
MaxJobCount=10000

ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup

PropagateResourceLimits=NONE
ReturnToService=2
TmpFs=/tmp
VSizeFactor=101

############################
# TIMEOUTS
############################

BatchStartTimeout=60
CompleteWait=35
InactiveLimit=0
KillWait=30
MinJobAge=300
Waittime=0

############################
# CONTROLLER DAEMON
############################

SlurmctldPort=6817
SlurmctldTimeout=300
SlurmctldDebug=error
SlurmctldLogFile=/var/log/slurm/slurmctld.log
DebugFlags=backfill,cpu_bind,priority,reservation,selecttype,steps

MailProg=/usr/bin/mail

############################
# COMPUTE DAEMON
############################

SlurmdPort=6818
SlurmdTimeout=300
SlurmdDebug=error
SlurmdLogFile=/var/log/slurm/slurmd.log

DisableRootJobs=NO
```

### Copy SLURM configuration to worker nodes

From the **controller node**, copy the SLURM configuration to all worker nodes using rsync:

```bash
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/slurm/slurm.conf ubuntu@xxx.xx.xxx.xx2:/etc/slurm
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/slurm/slurm.conf ubuntu@xxx.xx.xxx.xx3:/etc/slurm
```

### Restart SLURM on all worker nodes

```bash
sudo systemctl enable slurmd
sudo systemctl restart slurmd
sudo systemctl status slurmd
```

## 4) Configure Shared Storage

To create a shared file system so that all nodes have access to the same home directory, we need to set up NFS (Network File System).

### Install and configure NFS server on controller node

On the **controller node**, install NFS:

```bash
sudo apt install nfs-kernel-server
```

Edit the `/etc/exports` file:

```bash
sudo nano /etc/exports
```

Add the following line (modify the IP subnet to match your worker nodes' network):

```
/home 172.xx.xxx.x/xx(rw,sync,no_root_squash)
```

Apply the changes:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

### Install NFS client and mount on worker nodes

On all **worker nodes**, run the following commands:

```bash
# Install NFS client
sudo apt install nfs-common

# Create mount point if it doesn't exist
sudo mkdir -p /home

# Mount the controller's home directory
sudo mount primrose-frontend001:/home /home
```

### Make NFS mount permanent

To ensure the NFS mount persists after reboot, add the following line to `/etc/fstab` on each **worker node**:

```bash
sudo nano /etc/fstab
```

Add this line:

```
primrose-frontend001:/home /home nfs defaults 0 0
```

## Troubleshooting

### Munge key synchronization issues
- Ensure the `/etc/munge/munge.key` file has identical permissions (0700) on all nodes
- Check that the munge service is running: `sudo systemctl status munge`
- Test Munge authentication: `munge -n | unmunge | grep STATUS`

### SSH connection failures
- Verify SSH keys are properly configured for passwordless login
- Check SSH service is running: `sudo systemctl status ssh`
- Test connectivity: `ssh nodename` from the controller node

### NFS mount issues
- Check NFS server is running on controller: `sudo systemctl status nfs-kernel-server`
- Verify firewall allows NFS traffic on port 2049
- Test mount manually: `sudo mount -t nfs primrose-frontend001:/home /mnt/test`
- Check mount status: `mount | grep nfs`

### SLURM connectivity issues
- Verify all nodes have synchronized clocks (use `ntpdate` or `chronyc`)
- Check SLURM logs: `sudo tail -f /var/log/slurm/slurmctld.log`
- Ensure Munge is working on all nodes before troubleshooting SLURM

