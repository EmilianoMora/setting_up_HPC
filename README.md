# settig_up_cluster
To setup the new computational cluster in the new S3IT ScienceCloud, I followed (this)[https://github.com/SergioMEV/slurm-for-dummies] github page and modified accordingly.

## Launch instances
Following the S3IT recommendations to launch instances I had to launch one instance that will serve as a controller node and two (or more) that will be worker nodes. 
## Set up private network
Edit the '/etc/hosts' file of all nodes (controller and worker) and add the IPs and names of all the nodes in the cluster:
```
xxx.xx.xxx.xx1	frontend001
xxx.xx.xxx.xx2	med32compute001
xxx.xx.xxx.xx3	med32compute002
```
## 1) Setyp SSH
Run the following command on all nodes:
```
sudo apt install openssh-server openssh-client
```
Test if it works by running the following command:
```
ssh med32compute001 #If you are in the frontend001 node
```
# 2) Install and setup Munge
Install munge in **all nodes** running the following commands:
```
sudo apt install munge libmunge2 libmunge-dev
munge -n | unmunge | grep STATUS
```
Then ensure that all munge files in the **controller node** have the correct permissions by:
```
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key
```
Then restart the munge service and configure it to run at startup. And test
```
systemctl enable munge
systemctl restart munge
systemctl status munge
```
It is necessary to have the exact same */etc/munge/munge.key* from the controller node in all the worker nodes. One way of doing it is to transfer the file from the *controller node* into each working node using rsync:
```
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/munge/munge.key ubuntu@xxx.xx.xxx.xx2:
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/munge/munge.key ubuntu@xxx.xx.xxx.xx3:
```
In each node eliminate and replace the */etc/munge/munge.key* file running the following commands:
```
sudo rm /etc/munge/munge.key
sudo mv munge.key /etc/munge

#And ensure correct permissions

sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key

# Restart munge
sudo systemctl enable munge
sudo systemctl restart munge

# Test munge
munge -n | ssh frontend001 unmunge 
```
# 3) Install and setup Slurm
On **all nodes (controller and workers)** run the following command:
```
sudo apt install slurm-wlm

sudo systemctl enable slurmd
sudo systemctl restart slurmd
sudo systemctl status slurmd
```
Modify the configuration file for slurm in the controller node (sudo nano /etc/slurm/slurm.conf) and the copy it in all worker nodes. Here is an example for version, but it may need to be adjusted accroding to all the worker nodes:
```
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
Copy the configuration file into all the worker nodes. I do this by using rsync:
```
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/slurm/slurm.conf ubuntu@xxx.xx.xxx.xx2:/etc/slurm
sudo rsync -avz -e "ssh -i .ssh/id_rsa" /etc/slurm/slurm.conf ubuntu@xxx.xx.xxx.xx3:/etc/slurm
```
Restart slurm on all working nodes:
```
sudo systemctl enable slurmd
sudo systemctl restart slurmd
sudo systemctl status slurmd
```
# 4) Configure shared storage
To create a shared file system (i.e. so thatt all nodes have the same folders) first we need to setup NFS. In the **controller node** install/update NFS:
```
sudo apt install nfs-kernel-server
```
Modidy the */etc/exports* file:
```
sudo nano /etc/exports

# Add the following line
/home 172.23.208.0/24(rw,sync,no_root_squash) #Modify this accordingly by creating a subnet of all IPs of the worker nodes

# Apply cahnges
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```
On all **worker nodes** run the following commands:
```
# Install NFS
sudo apt install nfs-common
# Mount the frontend into the node
sudo mount primrose-frontend001:/home /home
```

