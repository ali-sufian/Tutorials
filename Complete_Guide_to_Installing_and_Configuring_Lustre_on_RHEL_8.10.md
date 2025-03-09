# **Complete Guide to Installing and Configuring Lustre on RHEL 8.10**

## **1. Introduction**

Lustre is a high-performance parallel file system commonly used in HPC environments. This guide covers installing Lustre on **two servers** (`node1-lstr-server` & `node2-lstr-server`) and a **client** (`node3-lstr-client`) on **RHEL 8.10**.

## **2. Prerequisites**

### **Hardware Requirements**

- **node1-lstr-server & node2-lstr-server**
  - 2 CPUs, 8GB RAM (minimum)
  - `/dev/nvme0n2` (MGS+MDT: 50GB)
  - `/dev/nvme0n3` (OST: 100GB+)
- **node3-lstr-client**
  - 2 CPUs, 4GB RAM

### **Software Requirements**

- RHEL 8.10 installed on all nodes
- Root or sudo privileges
- Internet access to download packages

## **3. Setting Up Repositories**

Run the following command **on all nodes** to configure Lustre and e2fsprogs repositories:

```bash
sudo tee /etc/yum.repos.d/lustre.repo <<EOF
[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/latest-release/el8.10/server/
enabled=1
gpgcheck=0

[lustre-client]
name=lustre-client
baseurl=https://downloads.whamcloud.com/public/lustre/latest-release/el8.10/client/
enabled=1
gpgcheck=0

[e2fsprogs]
name=e2fsprogs
baseurl=https://downloads.whamcloud.com/public/e2fsprogs/latest/el8/
enabled=1
gpgcheck=0
EOF
```

## **4. Install Kernel, Lustre, and e2fsprogs**

Run the following command **on all nodes**:

```bash
sudo dnf install -y kernel kernel-devel kernel-headers
sudo dnf install -y e2fsprogs e2fsprogs-devel e2fsprogs-libs
```

Reboot after installation:

```bash
sudo reboot
```

## **5. Install Lustre Server Components**

Run this on **node1-lstr-server & node2-lstr-server**:

```bash
sudo dnf install -y lustre kmod-lustre
```

## **6. Format Lustre Storage Devices**

### **On `node1-lstr-server` (MGS + MDT + OST)**
```bash
sudo mkfs.lustre --fsname=lustrefs --mgs --mdt --index=0 --backfstype=ldiskfs /dev/nvme0n2
sudo mkfs.lustre --fsname=lustrefs --ost --index=0 --mgsnode=<MGS_IP> --backfstype=ldiskfs /dev/nvme0n3
```

### **On `node2-lstr-server` (MDT + OST)**
```bash
sudo mkfs.lustre --fsname=lustrefs --mdt --index=1 --mgsnode=<MGS_IP> --backfstype=ldiskfs /dev/nvme0n2
sudo mkfs.lustre --fsname=lustrefs --ost --index=1 --mgsnode=<MGS_IP> --backfstype=ldiskfs /dev/nvme0n3
```

## **7. Mount Lustre Targets**

### **On `node1-lstr-server`**

```bash
sudo mkdir -p /mnt/mgsmdt
sudo mount -t lustre /dev/nvme0n2 /mnt/mgsmdt

sudo mkdir -p /mnt/ost
sudo mount -t lustre /dev/nvme0n3 /mnt/ost
```

### **On `node2-lstr-server`**

```bash
sudo mkdir -p /mnt/mdt
sudo mount -t lustre /dev/nvme0n2 /mnt/mdt

sudo mkdir -p /mnt/ost
sudo mount -t lustre /dev/nvme0n3 /mnt/ost
```

Verify the mounts:

```bash
df -h | grep mdt
df -h | grep ost
```

## **8. Install Lustre Client**

Run this **on node3-lstr-client**:

```bash
sudo dnf install -y lustre-client kmod-lustre-client
```

## **9. Mount Lustre on Client**

Replace `<MGS_IP>` with `node1-lstr-server`'s IP:

```bash
sudo mkdir -p /mnt/lustre
sudo mount -t lustre <MGS_IP>@tcp:/lustrefs /mnt/lustre
```

Verify:

```bash
df -h | grep lustre
```

## **10. Configure Persistent Mounting**

Edit `/etc/fstab` on each node:

### **On Servers (`node1-lstr-server` & `node2-lstr-server`)**

```bash
echo "/dev/nvme0n2 /mnt/mgsmdt lustre defaults,_netdev 0 0" | sudo tee -a /etc/fstab
echo "/dev/nvme0n3 /mnt/ost lustre defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

### **On Client (`node3-lstr-client`)**

```bash
echo "<MGS_IP>@tcp:/lustre /mnt/lustre lustre defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

## **11. Final Verification**

Reboot all nodes:

```bash
sudo reboot
```

Check Lustre mounts:

```bash
df -h | grep lustre
```

Test file creation on the client:

```bash
touch /mnt/lustre/testfile
ls -l /mnt/lustre/
```

## **12. Troubleshooting**

If any mount fails, check logs:

```bash
dmesg | tail -50
journalctl -xe
```

Ensure the network is reachable and ports are open between nodes.

## **Conclusion**

Lustre is now set up with two servers and one client. This provides a functional parallel file system for testing and experimentation.
