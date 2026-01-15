You’re dealing with a **Realtek RTL8111/8168 NIC**, which by default uses the **r8169** kernel driver. You want to **switch to the vendor r8168 driver** using `.deb` packages. That’s a common and valid move when r8169 is unstable or slow.

I’ll walk you through this cleanly, Linux-first, step by step.

---

## 1️⃣ Check which driver is currently in use

### Identify your Ethernet controller

```bash
lspci -nn | grep -i ethernet
```

Typical output:

```
02:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168 [10ec:8168]
```

### Check the active kernel driver

```bash
lspci -k -nn | grep -A3 -i ethernet
```

You’ll see something like:

```
Kernel driver in use: r8169
Kernel modules: r8169
```

### Alternative (module-based check)

```bash
lsmod | grep r816
```

or

```bash
ethtool -i eth0
```

(or replace `eth0` with your interface, e.g. `enp2s0`)

---

## 2️⃣ Install required tools (important)

Make sure DKMS is installed — **r8168 depends on it**:

```bash
sudo apt update
sudo apt install dkms build-essential
```

---

## 3️⃣ Install the r8168 `.deb` package

Go to the directory where you downloaded the driver:

```bash
cd ~/Downloads   # or wherever the .deb is
```

Install it:

```bash
sudo dpkg -i r8168-dkms*.deb
```

If you get dependency errors:

```bash
sudo apt --fix-broken install
```

This:

* Builds the r8168 kernel module
* Registers it with DKMS so it survives kernel updates

---

## 4️⃣ Disable r8169 (CRITICAL STEP)

If you don’t blacklist r8169, the kernel will **keep loading it first**.

Create a blacklist file:

```bash
sudo nano /etc/modprobe.d/blacklist-r8169.conf
```

Add this line:

```
blacklist r8169
```

Save and exit.

---

## 5️⃣ Force r8168 to load

Unload r8169 (if currently loaded):

```bash
sudo modprobe -r r8169
```

Load r8168:

```bash
sudo modprobe r8168
```

---

## 6️⃣ Update initramfs and reboot

This ensures the correct driver loads early at boot:

```bash
sudo update-initramfs -u
sudo reboot
```

---

## 7️⃣ Verify r8168 is now active

After reboot:

```bash
lspci -k | grep -A3 -i ethernet
```

Expected:

```
Kernel driver in use: r8168
Kernel modules: r8168
```

Also:

```bash
lsmod | grep r8168
```

And:

```bash
ethtool -i enp2s0
```

You should see:

```
driver: r8168
```

---

## 8️⃣ If network does not come up immediately

Restart NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

Or bring the interface up:

```bash
sudo ip link set enp2s0 up
```

---

## 9️⃣ How to revert back to r8169 (if needed)

```bash
sudo apt remove r8168-dkms
sudo rm /etc/modprobe.d/blacklist-r8169.conf
sudo update-initramfs -u
sudo reboot
```

---

## ⚠️ Notes from experience (important)

* **r8168 is vendor-specific** → often more stable on Realtek chips
* **r8169 is generic** → good but sometimes buggy with power saving
* On laptops, r8169 may cause:

  * Random disconnects
  * Low throughput
  * Link drops after suspend

Switching to r8168 usually fixes all of that.

---

If you want, tell me:

* Your Ubuntu version
* Exact NIC model (`lspci -nn`)
* Whether this is desktop or laptop

I’ll tune it further (power management, offloading, ASPM).
