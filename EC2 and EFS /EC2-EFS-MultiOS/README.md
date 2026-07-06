ASK : 

Create an EFS and connect it to 3 different EC2 instances. Make sure that all instances have different operating systems. For instance, Ubuntu, Red Hat Linux and Amazon Linux 2. 

That diagram shows the setup: one EFS file system with mount targets, shared across three EC2 instances (Ubuntu, Red Hat, Amazon Linux 2) sitting in the same VPC, all mounting over NFS on port 2049, gated by a security group that allows that traffic within the VPC.


<img width="2720" height="1720" alt="image" src="https://github.com/user-attachments/assets/d0610530-253b-4fb6-ba28-e3ced9a6488a" />


## Theory behind it

**Amazon EFS (Elastic File System)** is a managed, elastic **NFS (Network File System)**. Unlike EBS, which attaches to exactly one EC2 instance at a time, EFS is designed for **concurrent, shared access** — many instances, even across different Availability Zones and different operating systems, can mount the same file system at once and see the same files in real time.

Key concepts:
- **Mount target**: EFS creates a network interface (ENI) in each subnet/AZ you choose. Instances mount the file system through the mount target in their own AZ, which keeps latency low.
- **NFSv4.1 protocol**: EFS speaks standard NFS, so any Linux distribution with an NFS client (`nfs-common` on Debian/Ubuntu, `nfs-utils` on RHEL/Amazon Linux) can mount it with the same basic `mount` command — this is exactly why it works identically across Ubuntu, Red Hat, and Amazon Linux 2 despite them being different distros.
- **Security groups govern access**, not IAM alone: the mount target's security group must allow inbound TCP 2049 (NFS) from the EC2 instances' security group, and the EC2 instances need outbound access to the same port.
- **Elastic throughput mode** scales I/O automatically with usage, so you don't have to pre-provision capacity.
- **POSIX file system semantics**: writes from one instance are immediately visible to the others, since it's the same underlying storage, not a copy — this is what your `ubantu.txt`, `redhat.txt`, and `linux.txt` files appearing together on all three instances demonstrates.

## Steps to do this

1. **Launch 3 EC2 instances** in the same VPC (different subnets/AZs is fine), one each running Ubuntu, Red Hat Enterprise Linux, and Amazon Linux 2/2023.
2. **Create the EFS file system** (`aws efs create-file-system` or via console), selecting the same VPC as the instances, and let AWS create mount targets in the relevant subnets automatically.
3. **Configure security groups**:
   - EFS mount target SG: inbound NFS (TCP 2049) from the EC2 instances' SG or the VPC CIDR.
   - EC2 instance SG: outbound allowed (default), and SSH inbound for management.
4. **Install the NFS client on each instance**:
   - Ubuntu: `sudo apt update && sudo apt install nfs-common -y`
   - Red Hat / Amazon Linux: `sudo yum install -y nfs-utils` (or `dnf`)
5. **Create a mount point and mount the file system** on each instance:
   ```
   sudo mkdir -p /efs-assignment
   sudo mount -t nfs4 -o nfsvers=4.1 <fs-id>.efs.<region>.amazonaws.com:/ /efs-assignment
   ```
   (Alternatively use the EFS mount helper: `sudo mount -t efs -o tls <fs-id>:/ efs`)
6. **Verify** with `df -h` — you should see the EFS filesystem mounted with an "8.0E" (exabyte, effectively unlimited) size.
7. **Test shared access**: create a file from each instance (e.g. `sudo touch ubuntu.txt`) inside the mounted directory, then `ls` from the other two instances — all files should appear on every instance, confirming shared storage.
8. *(Optional, for persistence)* Add an entry to `/etc/fstab` so the mount survives reboots.

## Use cases

- **Shared web content / application code**: multiple web or app servers behind a load balancer serving the same files (images, uploads, logs) without duplicating storage.
- **Content management systems**: WordPress, Drupal, etc., where several servers need to read/write the same media library.
- **Big data and analytics**: multiple compute nodes (e.g. Hadoop, machine learning training jobs) reading the same shared datasets in parallel.
- **CI/CD build farms**: build agents sharing a common cache or artifact directory.
- **Container workloads**: ECS/EKS pods across multiple hosts mounting a common persistent volume.
- **Home directories / user profiles**: centralizing `/home` across a fleet of Linux instances so users get the same environment wherever they log in.
- **Cross-OS collaboration** (as in your assignment): demonstrating that Ubuntu, Red Hat, and Amazon Linux can all interoperate through a common NFS-based storage layer, which is useful in heterogeneous enterprise environments running mixed distros.
