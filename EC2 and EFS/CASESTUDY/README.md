
Case study : 

Problem Statement: 
You work for XYZ Corporation. Your corporation is working on an application and they require secured web servers on Linux to launch the application. 
Tasks To Be Performed: 
	1. Create an instance in the US-East-1 (N. Virginia) region with Linux OS and manage the requirement of web servers of your company using AMI. 
	2. Replicate the instance in the US-West-2 (Oregon) region. 
	3. Build two EBS volumes and attach them to the instance in the US-East-1 
(N. Virginia) region. 
	4. Delete one volume after detaching it and extend the size of the other 
volume. 
Take backup of this EBS volume


<img width="1360" height="760" alt="image" src="https://github.com/user-attachments/assets/61a41d1f-586e-4058-8e76-6e7d5cee2fd3" />
<img width="1360" height="840" alt="image" src="https://github.com/user-attachments/assets/ede4ba68-9075-47ca-a20e-508f0f93b649" />

That first diagram covers the region-replication part of the case study. The second piece — EBS volume lifecycle — is its own flow, so here's a separate diagram for that.The two diagrams together map the whole case study: cross-region AMI replication on top, EBS volume lifecycle management on the bottom.



## Theory behind it

**AMIs and cross-region replication**: An **AMI (Amazon Machine Image)** is a template that captures an EC2 instance's root volume (and any attached volumes), OS, and installed software as a point-in-time snapshot. AMIs are region-scoped by default — you can't launch an instance in `us-west-2` directly from an AMI that lives only in `us-east-1`. **Copying an AMI** across regions duplicates the underlying EBS snapshots into the destination region, after which you can launch new instances there. This is the standard mechanism for disaster recovery, multi-region deployment, and low-latency access for geographically distributed users, since the copy preserves the exact configuration (installed packages, web server config, files) of the source instance.

**EBS volumes**: Amazon EBS volumes are block storage devices that attach to a *single* EC2 instance at a time (unlike EFS). Key behaviors relevant to this case study:
- **Attach/detach**: A volume can be attached to a running instance at a specific device name (e.g. `/dev/sdb`), then formatted (`mkfs`) and mounted.
- **Delete**: Once detached, a volume can be deleted — this permanently destroys the data, so it's typically done only after confirming the data is no longer needed or has been backed up.
- **Online resize (extend)**: EBS supports **elastic volumes** — you can increase a volume's size in the console without detaching it (`Modify volume`), but the OS-level filesystem still needs to be told to use the new space (`xfs_growfs` for XFS, `resize2fs` for ext4). Until that step, `df -h` still reports the old size.
- **Snapshots**: An EBS snapshot is an incremental, point-in-time backup of a volume stored durably in S3 behind the scenes. Snapshots are the basis for both AMIs and standalone volume backups, and can be used to restore a new volume later.

## Steps to do this

**Part 1 — instance + AMI + cross-region replication**
1. In **us-east-1 (N. Virginia)**, launch an EC2 instance with a Linux AMI (e.g. Amazon Linux 2023), a security group allowing SSH (22) and HTTP (80), and a key pair.
2. Connect via EC2 Instance Connect/SSH, then install and start a web server:
   ```
   sudo yum install httpd -y
   sudo systemctl start httpd
   sudo systemctl enable httpd
   echo "XYZ Corporation Web Server" | sudo tee /var/www/html/index.html
   ```
3. From the instance's **Actions → Image and templates → Create image**, generate an AMI (this snapshots the attached EBS volumes).
4. Once the AMI is `available`, select it and choose **Actions → Copy AMI**, setting the destination region to **us-west-2 (Oregon)**.
5. Switch console region to us-west-2, wait for the copied AMI to become available, then **Launch instance from AMI** to create the replica.

**Part 2 — EBS volumes**
6. Back in us-east-1, create two EBS volumes (e.g. 8 GiB and 100 GiB, `gp3`) in the same Availability Zone as the running instance.
7. Attach both volumes to the instance, then on the instance:
   ```
   sudo mkfs -t xfs /dev/sdb
   sudo mkdir /data1 && sudo mount /dev/sdb /data1
   sudo mkfs -t xfs /dev/sdc
   sudo mkdir /data2 && sudo mount /dev/sdc /data2
   ```
8. **Detach** the second volume from the instance, then **delete** it from the EBS console.
9. **Extend** the remaining volume (e.g. 100 GiB → 110 GiB) via **Modify volume**, then grow the filesystem on the instance:
   ```
   sudo xfs_growfs /data1
   ```
10. **Back up** the extended volume by creating a **snapshot** (Actions → Create snapshot) — this is your point-in-time, durable backup.

## Use cases

- **Disaster recovery / business continuity**: keeping a warm or cold standby in a second region so you can fail over if `us-east-1` has an outage.
- **Multi-region deployment**: serving users on the US West Coast with lower latency by running a replica closer to them.
- **Golden image pipeline**: baking a "golden AMI" once (OS + patches + web server + app), then reusing it to launch consistent instances across regions or Auto Scaling Groups.
- **Storage right-sizing**: attaching extra EBS volumes temporarily for a migration or batch job, then detaching and deleting them once no longer needed, instead of over-provisioning the root volume.
- **Non-disruptive capacity growth**: extending a volume online as an application's data grows, without downtime.
- **Compliance and recovery backups**: scheduled EBS snapshots (often automated via Data Lifecycle Manager) satisfy backup/retention requirements and let you restore a volume to a known-good state after corruption or accidental deletion.

### A few things worth tightening up if you want this assignment to be more complete

- **Security groups**: your screenshots show `0.0.0.0/0` open for SSH — worth noting in the write-up that production setups should restrict SSH to a known IP range (as AWS itself flags in the console).
- **AZ awareness for EBS**: volumes and instances must be in the same Availability Zone — worth explicitly stating which AZ you used (`us-east-1d` in your screenshots) since it's a common source of "can't attach" errors.
- **Cost note**: an extended volume can't be shrunk back down, and EBS billing is by provisioned size regardless of usage — a one-line note on this shows awareness of the trade-off in step 4/9.
- **Snapshot lifecycle**: mentioning AWS Data Lifecycle Manager for automating recurring snapshots (rather than one manual snapshot) would round out the "backup" task nicely for a case-study writeup.
