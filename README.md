# Proxmox-Commands
Proxmox VE Commands
To **check all containers (CTs)** in **Proxmox**, you can use the **command line** with `pct` and `qm` tools. Here’s how you can list and check all LXC containers (CTs):

---

### 🔍 **List All Containers**

Run this command on the Proxmox host:

```bash
pct list
```

This will output something like:

```
VMID       Status     Lock         Name
100        running                 debian-ct
101        stopped                 ubuntu-ct
```

---

### 🔍 **Check Container Status**

To check detailed info for each container:

```bash
for ct in $(pct list | awk 'NR>1 {print $1}'); do
  echo "==== CT $ct ===="
  pct status $ct
done
```

You can also get more details:

```bash
for ct in $(pct list | awk 'NR>1 {print $1}'); do
  echo "==== CT $ct ===="
  pct config $ct
done
```

---

### 🛠 Optional: List All CTs with IP Address

If you want to get the IPs of running containers:

```bash
for ct in $(pct list | awk 'NR>1 && $2=="running" {print $1}'); do
  echo "==== CT $ct ===="
  pct exec $ct ip a | grep inet
done
```

---

Let me know if you want to filter by status (running/stopped) or include VMs (`qm list`) too.

To **forcefully destroy a container (CT) with VMID 144** in **Proxmox**, follow these steps carefully. This will permanently delete the container, including all of its data.

---

### ⚠️ **Warning**

This operation **cannot be undone**. Make sure you really want to delete CT `144`.

---

### ✅ **Steps to Force Destroy CT 144**

1. **Stop the CT (if it's running):**

```bash
pct stop 144
```

If that fails, you can **force kill** the container process:

```bash
kill -9 $(lxc-info -n 144 -pH)
```

> `lxc-info` might need to be installed: `apt install lxc-utils`

2. **Remove the CT:**

```bash
pct destroy 144
```

This removes the container and associated config.

---

### 🛠️ **If Container Is Locked**

If you get an error about a **lock**, remove it manually:

```bash
rm /etc/pve/nodes/$(hostname)/lxc/144.conf.lock
pct destroy 144
```

Or use this safer unlock command:

```bash
pct unlock 144
pct destroy 144
```

---

Let me know if you're using **Proxmox GUI** and prefer to do it from there, or if there's a specific error during deletion.

You're seeing:

```
storage 'local-lvm' is not available on node 'node2'
```

This means the container **144 was created on a different node**, or its **storage backend ('local-lvm') is not present or accessible on node2**.

Here’s how to handle this step by step:

---

### ✅ 1. **Check Where CT 144 Is Actually Located**

Run this:

```bash
find /etc/pve/nodes/ -name 144.conf
```

This will tell you **which node** the CT belongs to (e.g., `node1`, `node2`, etc.)

---

### ✅ 2. **Move to the Correct Node**

If CT `144` is on `node1`, SSH into `node1` and run the destroy command there:

```bash
ssh root@node1
pct destroy 144
```

---

### ✅ 3. **If You're Sure the CT Is on `node2`, But Storage Is Missing**

If `local-lvm` is missing on this node, you can try one of the following:

#### Option A: **Temporarily recreate a dummy `local-lvm` entry**

Not ideal, but possible for forcing deletion.

Or better:

#### Option B: **Edit the config manually (only if you're sure it's orphaned)**

```bash
rm /etc/pve/nodes/node2/lxc/144.conf
```

Then manually remove the container data (be very careful here):

```bash
lvremove /dev/pve/vm-144-disk-0  # Or whatever the LVM volume is named
```

You can find the actual LVM name with:

```bash
lvs | grep 144
```

---

### ✅ 4. **Check if Container Is in a "Zombie" State in the GUI**

If you're using the web UI, and it shows the CT stuck in a weird state:

* Go to **Datacenter → CT 144 → More → Remove**.
* Or check under **Datacenter → Storage** to confirm if `local-lvm` is shared.

---

### 🔍 Summary

* The error means **you’re trying to destroy a container from the wrong node**, or its **storage doesn’t exist on that node**.
* Use `find /etc/pve/nodes/ -name 144.conf` to identify where it is.
* Run `pct destroy` **only on the correct node** where the CT and its storage exist.

---

Let me know what the output of `find /etc/pve/nodes/ -name 144.conf` is, and I can guide you from there.
Thanks. Since the config file is located at:

```
/etc/pve/nodes/node2/lxc/144.conf
```

…it **means container 144 is assigned to `node2`**, but the error shows that `storage 'local-lvm' is not available on node2`.

So Proxmox is trying to delete the CT and also clean up its associated storage on `local-lvm` — **but `local-lvm` does not exist on node2**, which is causing the `pct destroy` to fail.

---

## ✅ Solution

### 🛠 Option 1: **Manually Delete the CT Without Access to Storage**

If you are **sure** the actual storage (disk volumes) for CT 144 no longer exist or are not accessible from this node, and you just want to **clean up the config**, do:

```bash
rm /etc/pve/nodes/node2/lxc/144.conf
```

> ⚠️ This will **not delete any disk** — it only removes the configuration and unregisters the container.

Then check:

```bash
ls /dev/pve | grep 144
```

If there is a leftover disk like `vm-144-disk-0`, and your `pve` volume group exists, you can remove it with:

```bash
lvremove /dev/pve/vm-144-disk-0
```

---

### 🛠 Option 2: **Make `local-lvm` Available on Node2 Temporarily (Not Recommended Unless Needed)**

If you're trying to do a clean destroy and the `local-lvm` was accidentally removed or disabled from `node2`, you could try re-adding it under:

```
Datacenter → Storage → Add → LVM-Thin
```

Make sure the name matches `local-lvm`, and that it points to the correct volume group (e.g., `pve`).

But again, this is only useful if the storage physically exists and you want to restore it.

---

### ✅ Quick Summary

* Container config exists on `node2`
* Storage `local-lvm` doesn't exist on `node2`
* So `pct destroy` fails
* Manually remove config with:

```bash
rm /etc/pve/nodes/node2/lxc/144.conf
```

Let me know if you want help confirming if the disk is still present or how to reconfigure the storage.

Great — that tells us the disk for CT 144 still exists:

```
/dev/pve/vm-144-disk-0
```

So here's the complete cleanup process:

---

## ✅ Final Steps to Force-Delete CT 144 (Including Disk)

### 🔹 1. **Remove the CT Config**

This unregisters the container from Proxmox:

```bash
rm /etc/pve/nodes/node2/lxc/144.conf
```

---

### 🔹 2. **Delete the Leftover Disk**

Since the disk is on the `pve` volume group and named `vm-144-disk-0`, remove it with:

```bash
lvremove /dev/pve/vm-144-disk-0
```

You’ll get a confirmation prompt — type `y` to proceed.

> 💡 If you want to skip confirmation:

```bash
lvremove -f /dev/pve/vm-144-disk-0
```

---

### 🔹 3. **(Optional) Verify Cleanup**

Check that everything is gone:

```bash
pct list | grep 144       # Should show nothing
ls /dev/pve | grep 144    # Should show nothing
```

---

You're done! CT 144 is now **fully and cleanly removed**, including its disk.

Let me know if you want help finding and cleaning up orphaned CTs or VMs.

