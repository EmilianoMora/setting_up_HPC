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
Once slurm and all other programs are installed and running correctly, we can create a snapshot to reuse that configuration for future instances that can be used for controller or worker nodes.
In the S3IT page, we just need to select the instance we just created and click on *Create Snapshot*. Once the snaphot has been created, we can just launch another instance even with a different flavor.

## Set up HPC
Once we have more than one instance, we will need to edit the /etc/hosts file to ensure proper hostname resolution between nodes.
```sh
sudo nano /etc/hosts
```
And add the following information to the end. Add the IP instead of the *xxx.xx.xxx.xxx*. Here, is where we decide which instance will be the controller and the worker node(s). This step needs to be done in all nodes independently.
```
xxx.xx.xxx.xxx  primrose-frontend001
xxx.xx.xxx.xxx  primrose-med128compute001
xxx.xx.xxx.xxx  primrose-med128compute002
```
Test if you can log into the other instance
```ssh
# From the primrose-frontend001
ssh primrose-med128compute001
logout
# From the primrose-med128compute001
ssh primrose-frontend001
logout
```
Check that munge is working on the worker node(s).
```sh
# Restart munge
sudo systemctl enable munge
sudo systemctl restart munge

# Test munge
munge -n | ssh primrose-frontend001 unmunge
```
You should see
```
STATUS:          Success (0)
ENCODE_HOST:     primrose-med128compute001 (IP_ADDRESS)
ENCODE_TIME:     2026-07-28 07:54:52 +0000 (1785225292)
DECODE_TIME:     2026-07-28 07:54:52 +0000 (1785225292)
TTL:             300
CIPHER:          aes128 (4)
MAC:             sha256 (5)
ZIP:             none (0)
UID:             ubuntu (1000)
GID:             ubuntu (1000)
LENGTH:          0
```
Start slurm
On the controller node
```sh
sudo systemctl restart munge
sudo systemctl enable slurmctld
sudo systemctl restart slurmctld
sudo systemctl status slurmctld
```
On the worker node
```sh
sudo systemctl enable slurmd
sudo systemctl restart slurmd
sudo systemctl status slurmd
```
Once slurm is up and running. We just need to configure shared storage. This is done so that all the volumes mounted and the programs installed in the controller node (primrose-frontend001) can be accessed by all the worker nodes(s).
For the creation and mounting of volumes to an instance, we can follow the instructions from the [S3IT](https://docs.s3it.uzh.ch/cloud/user_guide/volumes/). When the wanted volumes have been created, and mounted into the controller node, we can run the following commands.
1) In the controller node
Edit the */etc/exports* file
```sh
sudo nano /etc/exports
```
And add the following. Make sure to change the subnet (172.23.0.0/16) accordingly based on the IP of all instances.
```
/home/ubuntu/volume_1    172.23.0.0/16(rw,sync,no_subtree_check,no_root_squash)
/home/ubuntu/volume_2    172.23.0.0/16(rw,sync,no_subtree_check,no_root_squash)
```
Explanation:

172.23.0.0/16 → allows your HPC private network
* rw: workers can read/write
* sync: safer writes
* no_subtree_check: recommended for HPC
* no_root_squash: root on workers remains root (common in HPC)

```sh
sudo exportfs -ra #Reload exports
sudo systemctl restart nfs-kernel-server #Restart NFS
sudo exportfs -v #Check if volumes have been loaded
```
2) In the worker node
Create the necessary directories.
```sh
sudo mkdir -p /home/ubuntu/volume_1
sudo mkdir -p /home/ubuntu/volume_2
```
Edit the */etc/fstab*.
```sh
sudo nano /etc/fstab
```
And add the following. Change the IP address accordingly by adding the IP of the controller node.
```
xxx.xx.xxx.xxx:/home/ubuntu/volume_1      /home/ubuntu/volume_1      nfs4  defaults,_netdev 0 0
xxx.xx.xxx.xxx:/home/ubuntu/volume_2  /home/ubuntu/volume_2  nfs4  defaults,_netdev 0 0
```
Mount the volumes
```sh
sudo systemctl daemon-reload
sudo mount -a
```
Check if volumes were mounted
```sh
mount | grep nfs
```
Expected:
```
172.23.205.219:/home/ubuntu/volume_1 on /home/ubuntu/volume_1
172.23.205.219:/home/ubuntu/volume_2 on /home/ubuntu/volume_2
```
3) Test the setup
On the controller:
```sh
echo "hello HPC" | sudo tee /home/ubuntu/emiliano/test.txt
```
On worker:
```sh
cat /home/ubuntu/emiliano/test.txt
```
You should get.
```
hello HPC
```
Then test write access.
Worker:
```sh
echo "written from worker" > /home/ubuntu/emiliano/worker_test.txt
```
Controller:
```
cat /home/ubuntu/emiliano/worker_test.txt
```

If you want, I can:
- Commit this edited README to the branch `update/readme-proofread-slurm` and open a pull request, or
- Make a smaller change that only fixes grammar and critical typos (no structural additions).
