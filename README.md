# Setting Up a SLURM Cluster

To setup the new computational cluster in the new [S3IT ScienceCloud](https://docs.s3it.uzh.ch/cloud/overview/), I followed [this](https://github.com/SergioMEV/slurm-for-dummies) GitHub page and modified accordingly. I wrote an earlier version here, but this one is easier and install slurm 25.05.8 on Ubuntu 24.04.4 LTS.

## Create a golden-image with all of the pre-requisite programs
In the older version, I was installing the programs one by one in each node. However, S3IT allows to create an image where all programs can be installed and then it is just easier to launch a new node from this image.

1) Launch instance

Access the [SEIT](https://cloud.science-it.uzh.ch/) using your credentials and make sure you have been added as an user to be able to launch instances and create volumes. [Here](https://docs.s3it.uzh.ch/cloud/user_guide/instances/) are the S3IT guidelines to launch an instance. In brief, click on *Instances* and then *Launch Instance*. Specify *instance name*, *source* (in my case ***Ubuntu 24.04 (2026-05-08)), *flavor* (pick a small VM, I picked 2cpu-8ram-hpcv3), *networks* (uzh-only) and *key pair*.

Once the instance is created one can access it via
```sh
ssh ubuntu@assigned.ip.address
```
Once one is within the instance, run the following command.
```sh
sudo apt-get update
```
Then create a new SSH key
```
ssh-keygen -t rsa -b 4096

# Change permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
```
In order for the other instances (nodes) to be able to reach is other we need to add public key (id_rsa.pub) into the authorized_keys file. First copy the result of the following command.
```sh
cat ~/.ssh/id_rsa.pub
```
And paste it in a new line within the *~/.ssh/authorized_keys* file and save.
```sh
sudo nano ~/.ssh/authorized_keys
```
2) Install necessary programs and dependencies
```sh
sudo apt install build-essential libhwloc-dev libjson-c-dev libcurl4-openssl-dev libyaml-dev libsystemd-dev libdbus-1-dev pkg-config libmysqlclient-dev libssl-dev libpam0g-dev
```
3) Install SSH
```sh
sudo apt install openssh-server openssh-client #Install SSH
```
4) Install Munge
```sh
sudo apt install munge libmunge2 libmunge-dev #Install Munge

# Change Munge Key permissions
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key

# Check Munge status
munge -n | unmunge | grep STATUS
```
5) Install NFS
```sh
sudo apt install nfs-kernel-server nfs-common #Install NFS-client
```
6) Install slurm v25.05.8. **NOTE**: for no reason install slurm from the Linux online repository (sudo apt install slurm-wlm). This will install slurm v23 that will then affect how v25 is installed.
```
# Download the source (use the same version 23.11.4 to match your system)
wget https://download.schedmd.com/slurm/slurm-25.05.8.tar.bz2
tar -xjf slurm-25.05.8.tar.bz2
cd slurm-25.05.8

# Configure, compile, and install
./configure \
    --prefix=/usr \
    --sysconfdir=/etc/slurm \
    --with-munge
    
make -j$(nproc)
sudo make install
```
Check that the cgroup_v2 was configured (this is essential for the worker nodes to work).
```sh
find src/plugins -name "*.so" | grep v2

# The result should look like this
./src/plugins/cgroup/v2/.libs/cgroup_v2.so

# Also Run
stat -fc %T /sys/fs/cgroup

# You should see
cgroup2fs

# Also Run
find /usr -name "cgroup*.so" 2>/dev/null

# You should see
/usr/lib/slurm/cgroup_v2.so
/usr/lib/slurm/cgroup_v1.so
```
Create the slurm user and group
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

sudo cp etc/slurmctld.service /etc/systemd/system/
sudo cp etc/slurmd.service /etc/systemd/system/
```
Create a cgroup.conf file
```sh
sudo nano /etc/slurm/cgroup.conf
```
Add the following information and save
```
CgroupPlugin=cgroup/v2

ConstrainCores=yes
ConstrainRAMSpace=yes

AllowedRAMSpace=100
AllowedSwapSpace=0
```
Create a slurm.conf file
```
sudo nano /etc/slurm/slurm.conf
```
[Here](./slurm.conf), is a template for the most basic configuration file for slurm v25. Some aspects will need to be modified depending on the instances that will be added to the HPC. Specifically, the *NODES* and *PARTITIONS* sections.
The information for the *NODES* section can be found for the **worker nodes** with the following commands.
```sh
lscpu | grep -E "CPU\(s\)|Socket|Core|Thread"

# And
free -h
```
