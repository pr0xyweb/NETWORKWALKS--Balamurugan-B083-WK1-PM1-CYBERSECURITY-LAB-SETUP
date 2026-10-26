# 🔐 Building My Own Pentest Lab — VirtualBox + Kali Linux

**A practical virtual penetration-testing lab I built, broke, fixed, and documented.**

This repository documents how to build a Kali Linux lab with Oracle VirtualBox, configure a predictable private network, enable host–guest file sharing, troubleshoot common setup failures, and create a clean snapshot for repeatable testing.

The goal is reproducibility: document the small details that are often skipped in tutorials, including `vboxsf` permissions, NAT vs. NAT Network, static IP persistence, and clipboard/drag-and-drop troubleshooting.

> **Scope:** This repository covers lab infrastructure and setup. Add target VMs only when you have confirmed that they are intentionally vulnerable and isolated for authorized practice.

---

## 📚 Table of Contents

- [Why This Repository Exists](#-why-this-repository-exists)
- [What This Lab Provides](#-what-this-lab-provides)
- [Architecture](#-architecture)
- [Lab Configuration](#️-lab-configuration)
- [Prerequisites](#-prerequisites)
- [Setup Walkthrough](#-setup-walkthrough)
  - [1. Install VirtualBox](#step-1--install-virtualbox)
  - [2. Download Kali Linux](#step-2--download-kali-linux)
  - [3. Create the NAT Network](#step-3--create-the-nat-network)
  - [4. Import Kali and Configure Its Adapters](#step-4--import-kali-and-configure-its-adapters)
  - [5. Configure Guest Additions and Shared Folders](#step-5--configure-guest-additions-and-shared-folders)
  - [6. Configure a Static IP Address](#step-6--configure-a-static-ip-address)
  - [7. Take a Clean Snapshot](#step-7--take-a-clean-snapshot)
- [Verification Checklist](#-verification-checklist)
- [Troubleshooting: Problems I Hit](#-troubleshooting-problems-i-hit)
- [What I Learned](#-what-i-learned)
- [Safety and Ethical Use](#-safety-and-ethical-use)
- [Official Resources](#-official-resources)
- [Roadmap](#-roadmap)
- [Author](#-author)

---

## 📌 Why This Repository Exists

I got tired of cybersecurity lab tutorials that worked only on the author's machine. Many guides skipped the small details that can make a setup fail:

- The `vboxsf` group required to access shared folders.
- The difference between **NAT** and **NAT Network**.
- A static IP address that changes again after reboot.
- Clipboard and drag-and-drop settings that appear enabled but do not work.
- Host-specific differences when running `VBoxManage`.

I built the lab from scratch, ran into problems, fixed them, and documented the process—including the failures. This repository is intended to help other learners reproduce the setup and troubleshoot it without starting over.

## 🎯 What This Lab Provides

A virtual lab running on one host computer:

- **Attacker VM:** Kali Linux.
- **Virtual network:** `10.0.0.0/24` using a VirtualBox NAT Network.
- **Planned Kali address:** `10.0.0.2/24`.
- **Gateway:** `10.0.0.1` (verify the actual NAT Network configuration on your host).
- **Internet access:** outbound connectivity through VirtualBox NAT, when configured correctly.
- **Host–guest clipboard:** bidirectional, when supported and working.
- **Drag and drop:** bidirectional, when supported and working.
- **Shared folder:** host Downloads folder mounted in Kali at `/media/sf_downloads`.
- **Snapshot:** a clean baseline to return to after experiments.

Future target VMs may use addresses in the `10.0.0.3–10.0.0.99` range. Use this range only after checking for address conflicts and confirming that the network's DHCP pool does not assign the same addresses.

> **Network note:** A VirtualBox NAT Network commonly uses its own gateway address, but the gateway is not guaranteed to be `10.0.0.1` for every configuration. Verify it in VirtualBox Network Manager before applying the static IP and DNS settings below. If the gateway or DHCP configuration differs, adjust the commands accordingly.

---

## 🏗️ Architecture

```text
┌──────────────────────────────────────────────────────────┐
│ Host: Windows / Linux / macOS                            │
│                                                          │
│  Host Downloads folder                                   │
│          │                                               │
│          ▼                                               │
│  VirtualBox shared folder                                │
│          │                                               │
│          ▼                                               │
│  Kali guest: /media/sf_downloads                         │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ VirtualBox NAT Network: PentestLab                 │  │
│  │ Subnet: 10.0.0.0/24                                │  │
│  │                                                    │  │
│  │ 10.0.0.1       Gateway (if configured this way)    │  │
│  │ 10.0.0.2       Kali Linux (planned static IP)      │  │
│  │ 10.0.0.3–99    Reserved range for future targets   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

A NAT Network allows attached VMs to communicate with each other while providing outbound NAT connectivity. It is different from attaching each VM to an independent plain NAT adapter.

**Isolation reminder:** NAT is not a complete security boundary. Configure target VMs deliberately, avoid bridged networking for vulnerable targets, and verify the effective network adapters before testing.

---

## ⚙️ Lab Configuration

| Component | Planned configuration |
|---|---|
| Hypervisor | Oracle VirtualBox 7.0+ |
| Guest OS | Kali Linux (rolling) |
| Kali RAM | 4096 MB recommended; 2048 MB may work for lighter tasks |
| Kali vCPUs | 2 |
| Network type | NAT Network |
| Network name | `PentestLab` |
| Subnet | `10.0.0.0/24` |
| Kali IP | `10.0.0.2/24` |
| Gateway | `10.0.0.1` if confirmed in Network Manager |
| DNS | `10.0.0.1` and `1.1.1.1`, if reachable and permitted |
| Clipboard | Bidirectional, subject to Guest Additions support |
| Drag and drop | Bidirectional, subject to Guest Additions support |
| Shared folder | Host Downloads → guest `/media/sf_downloads` |
| Snapshot | `clean-network-setup` |

These are the intended settings, not a claim that every VirtualBox host will use identical defaults. Check the actual network configuration before applying them.

---

## 🧰 Prerequisites

Before you begin, prepare:

- A computer with hardware virtualization enabled.
- Oracle VirtualBox installed on the host.
- The matching VirtualBox Extension Pack if you need features that require it.
- A Kali Linux VirtualBox image downloaded from the official Kali website.
- Enough free disk space for the VM and snapshots.
- Administrator access on the host and `sudo` access inside Kali.
- A basic understanding of terminal commands and private IP addresses.

---

## 🪜 Setup Walkthrough

The steps below describe the intended configuration. Menu names can vary slightly between VirtualBox versions and host operating systems.

### Step 1 — Install VirtualBox

Download VirtualBox from the [official VirtualBox downloads page](https://www.virtualbox.org/wiki/Downloads).

1. Install the version appropriate for your host operating system.
2. If you need the Extension Pack, use the version that matches your installed VirtualBox version.
3. Start VirtualBox and confirm that it opens successfully.

**Why:** Version mismatches can cause confusing feature or device issues. Install extensions only when needed and follow the official compatibility guidance.

### Step 2 — Download Kali Linux

Download a pre-built VirtualBox image from [Kali Linux — Get Kali](https://www.kali.org/get-kali/).

1. Choose the official VirtualBox image.
2. Extract the downloaded archive with a compatible archive utility, such as [7-Zip](https://www.7-zip.org/download.html).
3. Locate the extracted `.vbox` and virtual disk files.
4. Add or open the `.vbox` file in VirtualBox.

**Why use a pre-built image?** It can be quicker than installing from an ISO and may already include useful guest integration packages. However, package versions and preinstalled components can change, so verify Guest Additions rather than assuming they are current.

### Step 3 — Create the NAT Network

In VirtualBox, open **Tools → Network** or **Network Manager**, then locate the **NAT Networks** section. Create a network with these intended values:

```text
Name:        PentestLab
IPv4 Prefix: 10.0.0.0/24
DHCP:        Enabled initially, if desired
IPv6:        Disabled for this IPv4-only lab
```

The exact menu layout depends on your VirtualBox version.

You can also use the command line on systems where `VBoxManage` is available:

```bash
VBoxManage natnetwork add \
  --netname "PentestLab" \
  --network "10.0.0.0/24" \
  --enable \
  --dhcp on
```

If the network already exists, inspect it rather than running the creation command repeatedly. Use `VBoxManage natnetwork list` and the relevant `VBoxManage natnetwork` help output to check your version's available management commands.

**Why NAT Network?** Plain NAT typically gives each VM its own NAT environment. A NAT Network provides a shared virtual network for attached VMs, allowing lab guests to communicate with each other while NAT handles outbound connectivity.

**Important DHCP detail:** If you plan to give Kali `10.0.0.2` statically, confirm that this address is not in use or likely to be assigned by DHCP. You can reserve the address outside the DHCP pool or configure the DHCP pool appropriately. Do not assume the pool starts at a particular address.

### Step 4 — Import Kali and Configure Its Adapters

Add the extracted Kali `.vbox` file to VirtualBox, then open the VM settings.

**Settings → Network → Adapter 1**

```text
Attached to:  NAT Network
Name:         PentestLab
Adapter Type: Intel PRO/1000 MT Desktop (if supported and appropriate)
```

Use an adapter type supported by the guest and VirtualBox version. The exact choice may vary.

**Settings → General → Advanced**

```text
Shared Clipboard: Bidirectional
Drag'n'Drop:       Bidirectional
```

These settings alone do not guarantee that clipboard or drag-and-drop will work; the guest-side integration tools must also be working.

**Settings → Shared Folders → Add**

```text
Folder Path:   C:\Users\<your-user>\Downloads   (Windows example)
               ~/Downloads                      (Linux/macOS example)
Folder Name:   downloads
Auto-mount:    Enabled
Make Permanent: Enabled
```

Replace the example path with the actual Downloads folder on your host. The examples use a folder name of `downloads`, which commonly produces the guest mount path `/media/sf_downloads` when auto-mount is supported and enabled.

### Step 5 — Configure Guest Additions and Shared Folders

Boot Kali and open a terminal.

Update the package index and install the guest integration packages available for your Kali installation:

```bash
sudo apt update
sudo apt install -y virtualbox-guest-x11 virtualbox-guest-utils
```

If the packages are already installed, review the output for errors. A reboot may be required after installation or updates.

Add your current user to the shared-folder group:

```bash
sudo usermod -aG vboxsf "$USER"
```

Then reboot:

```bash
sudo reboot
```

After logging back in, verify your group membership:

```bash
id
groups
```

Check the shared folder:

```bash
ls -ld /media/sf_downloads
ls -la /media/sf_downloads
```

**Why `vboxsf` matters:** VirtualBox shared folders are commonly accessible through the `vboxsf` group. If the mount exists but your account lacks the required group membership, access may fail with `Permission denied`. Group changes generally take effect after a new login; rebooting is a simple way to ensure the new membership is applied.

**Clipboard and drag-and-drop:** Guest Additions versions, desktop session support, and the active display environment can affect these features. If they fail, confirm that the guest integration packages are installed and compatible with the host version, then restart the guest session or reboot.

### Step 6 — Configure a Static IP Address

Kali commonly uses NetworkManager. Editing `/etc/network/interfaces` may not control a connection managed by NetworkManager, so use `nmcli` for a NetworkManager-managed connection.

First identify the connection profile:

```bash
nmcli connection show
```

Find the correct wired connection name. In this example, it is `Wired connection 1`. Replace it if your system uses a different name.

Before making changes, confirm the actual gateway address and DNS requirements in your NAT Network configuration. The following example assumes the gateway is `10.0.0.1` and that the selected address is available:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 1.1.1.1"
```

Apply the connection profile:

```bash
sudo nmcli connection up "Wired connection 1"
```

Verify the address, route, and DNS settings:

```bash
ip -br address
ip route
nmcli device show
```

You should see `10.0.0.2/24` on the interface connected to the lab network.

**Troubleshooting tip:** If the connection fails after changing it, use the VirtualBox console to restore a working configuration. Check the actual interface and connection profile before assuming the interface is named `eth0` or the profile is named `Wired connection 1`.

**DNS note:** `1.1.1.1` is an example public DNS resolver. If the lab environment or network policy blocks it, use an allowed resolver. A working IP route and working DNS are separate things.

### Step 7 — Take a Clean Snapshot

Once the network and shared folder are working, take a snapshot before installing or changing additional tools.

List your VMs to confirm the exact name:

```bash
VBoxManage list vms
```

Create a snapshot, replacing `Kali-Linux` if your VM has a different registered name:

```bash
VBoxManage snapshot "Kali-Linux" take "clean-network-setup"
```

To restore a snapshot, first check the available snapshots and confirm the exact VM and snapshot names:

```bash
VBoxManage snapshot "Kali-Linux" list
```

Then restore the intended snapshot:

```bash
VBoxManage snapshot "Kali-Linux" restore "clean-network-setup"
```

**Warning:** Restoring a snapshot discards changes made after that snapshot in the affected VM state. Back up any scripts, reports, or files you want to keep before restoring.

---

## ✅ Verification Checklist

Run the following checks from Kali before using the lab.

### 1. Check the IP address

```bash
ip -br address
```

**Expected:** the lab-connected interface has `10.0.0.2/24`.

### 2. Check the gateway

```bash
ping -c 3 10.0.0.1
```

**Expected:** replies from the configured gateway, if ICMP is allowed and the gateway is actually `10.0.0.1`.

### 3. Check outbound IP connectivity

```bash
ping -c 3 1.1.1.1
```

**Expected:** replies if outbound connectivity is configured and ICMP is allowed. A failed ping alone does not prove that all Internet access is unavailable.

### 4. Check DNS resolution

```bash
getent hosts kali.org
```

Alternatively, if `nslookup` is installed:

```bash
nslookup kali.org
```

**Expected:** a DNS response for `kali.org`.

### 5. Check shared-folder access

```bash
id
ls -la /media/sf_downloads
```

**Expected:** your account has the required group membership and can read the shared folder.

### 6. Check clipboard and drag-and-drop

- Test copying and pasting plain text between the host and guest.
- Test transferring a harmless sample file using the configured drag-and-drop direction.
- If either fails, check Guest Additions, the desktop session, and VirtualBox settings.

### Verification summary

- [ ] Kali has the intended static IP.
- [ ] The configured gateway is reachable.
- [ ] Outbound IP connectivity works.
- [ ] DNS resolution works.
- [ ] The shared folder is readable.
- [ ] Clipboard transfer works, if supported.
- [ ] Drag-and-drop works, if supported.
- [ ] The clean snapshot exists.

---

## 🐞 Troubleshooting: Problems I Hit

The following are practical failure scenarios encountered or anticipated during setup. The causes listed are common possibilities; check your own configuration before assuming a single cause.

### Problem 1 — Shared folder exists, but access is denied

**Symptom:** `/media/sf_downloads` exists, but listing or entering it returns `Permission denied`.

**Likely cause:** Your user is not a member of the `vboxsf` group, or the shared folder is not mounted with the expected permissions.

**Fix:**

```bash
sudo usermod -aG vboxsf "$USER"
sudo reboot
```

After reboot, check:

```bash
id
ls -ld /media/sf_downloads
ls -la /media/sf_downloads
```

**Lesson:** When a directory exists but cannot be accessed, inspect its ownership, permissions, mount status, and group membership before changing permissions indiscriminately.

### Problem 2 — No Internet after setting a static IP

**Symptom:** The gateway responds, but external IP addresses or domain names cannot be reached.

**Possible causes:**

- The configured gateway is incorrect.
- The static IP conflicts with another guest.
- The DNS server is unreachable.
- The NAT Network is disabled or incorrectly configured.
- The destination blocks ICMP, even though other traffic may work.

**Check routes and DNS:**

```bash
ip route
nmcli connection show "Wired connection 1"
nmcli device show
```

If the gateway is confirmed to be `10.0.0.1`, you can update DNS with:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.dns "10.0.0.1 1.1.1.1"

sudo nmcli connection up "Wired connection 1"
```

Replace the connection name and DNS values as needed.

**Lesson:** An IP address, default gateway, and DNS resolver serve different purposes. Check each separately rather than treating every connectivity problem as a DNS issue.

### Problem 3 — Clipboard and drag-and-drop do not work

**Symptom:** VirtualBox settings show bidirectional integration, but nothing transfers.

**Possible causes:**

- Guest Additions are missing, incompatible, or not running.
- The guest session or desktop environment does not support the expected integration.
- The VM needs a restart.
- Clipboard or drag-and-drop is disabled for the VM.

**Fix checklist:**

1. Confirm that the VM settings enable the required feature.
2. Confirm that the guest integration packages are installed.
3. Check compatibility between the host VirtualBox version and guest integration components.
4. Reboot Kali and test again.
5. If the commands exist, try restarting the relevant guest client processes from the Kali desktop session:

```bash
VBoxClient --clipboard
VBoxClient --draganddrop
```

If these commands are unavailable or return errors, inspect the installed guest packages and session logs rather than assuming the commands are supported in every configuration.

**Lesson:** Host-side settings and guest-side integration both matter. One does not guarantee the other is working.

### Problem 4 — `VBoxManage: command not found` on macOS

**Symptom:** VirtualBox opens in the GUI, but shell scripts cannot find `VBoxManage`.

**Possible cause:** The VirtualBox command-line binary is installed inside the application bundle and is not on the current shell's `PATH`.

**Check the common application-bundle location:**

```bash
ls /Applications/VirtualBox.app/Contents/MacOS/VBoxManage
```

If the file exists, add its directory to your Zsh path:

```bash
echo 'export PATH="$PATH:/Applications/VirtualBox.app/Contents/MacOS"' >> ~/.zshrc
source ~/.zshrc
```

Verify:

```bash
command -v VBoxManage
VBoxManage --version
```

The path may differ if VirtualBox was installed in another location.

**Lesson:** A working GUI does not guarantee that command-line tools are on `PATH`.

### Problem 5 — “VT-x is not available” or virtualization error

**Symptom:** The VM refuses to start and displays a hardware-virtualization or hypervisor error.

**Possible causes:** Hardware virtualization is disabled in firmware, another hypervisor is affecting VirtualBox, or the host configuration is incompatible with the installed version.

**Troubleshooting steps:**

1. Check the host's BIOS/UEFI settings for Intel VT-x or AMD-V/SVM and enable it if necessary.
2. Reboot the host after changing firmware settings.
3. On Windows, check the installed VirtualBox version's guidance for Hyper-V, Windows Hypervisor Platform, and other virtualization-based security features.
4. Review the VirtualBox VM log for the specific error before disabling Windows security or virtualization features.

**Important:** Do not automatically disable Hyper-V or security features as a first step. Modern VirtualBox versions may work with the Windows hypervisor, depending on the configuration. Follow current official documentation for your version.

**Lesson:** Read the exact error and check firmware and host compatibility before reinstalling the guest OS.

### Problem 6 — Static IP reverts after reboot

**Symptom:** After reboot, Kali receives a DHCP address instead of `10.0.0.2`.

**Possible causes:**

- The wrong NetworkManager connection profile was edited.
- Another profile automatically connects first.
- The static configuration was applied to the wrong interface.
- A different network manager or configuration is controlling the interface.

**Check the profiles:**

```bash
nmcli connection show
nmcli connection show "Wired connection 1"
nmcli device status
```

Inspect the IPv4 settings:

```bash
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns \
  connection show "Wired connection 1"
```

If needed, reapply the static configuration using the correct connection profile and confirmed gateway values from Step 6.

**Lesson:** Use the network manager that actually controls the interface and verify the saved connection profile after reboot.

---

## 💡 What I Learned

### 1. NAT vs. NAT Network is not a minor detail

A shared NAT Network allows lab VMs to communicate with each other. Independent NAT adapters do not automatically place VMs on the same guest network.

### 2. Static networking needs more than an IP address

The address, subnet, gateway, and DNS settings have different roles. Verify them independently.

### 3. Linux group membership can block access

Groups such as `vboxsf` control access to certain resources. Use `id`, `groups`, and `ls -ld` to diagnose permissions before making broad permission changes.

### 4. Snapshots make experimentation repeatable

A clean baseline makes it easier to recover from configuration mistakes. Keep separate backups for important scripts and reports because a snapshot is not a substitute for a backup.

### 5. Documentation is part of the engineering

A setup that works only on one machine is difficult to reproduce. Documenting the assumptions, failures, and verification steps makes the lab more useful to other learners.

---

## 🔐 Safety and Ethical Use

This lab is intended for cybersecurity education and authorized testing.

- Test only systems you own or have explicit permission to assess.
- Keep intentionally vulnerable target VMs on the intended isolated lab network.
- Avoid bridged networking for vulnerable targets unless you have a specific, controlled reason and understand the exposure.
- Verify every VM's network adapters before starting a test.
- Do not scan public IP addresses or third-party systems without authorization.
- Do not store real credentials, personal information, or production secrets in lab snapshots or shared folders.
- Back up important work before restoring snapshots or deleting virtual machines.

**A NAT Network helps with lab separation, but it should not be treated as a complete security boundary.**

---

## 🔗 Official Resources

- [Oracle VirtualBox Downloads](https://www.virtualbox.org/wiki/Downloads)
- [Oracle VirtualBox User Manual](https://www.virtualbox.org/manual/)
- [Kali Linux — Get Kali](https://www.kali.org/get-kali/)
- [Kali Linux Documentation](https://www.kali.org/docs/)
- [7-Zip Downloads](https://www.7-zip.org/download.html)

---

## 🗺️ Roadmap

- [x] Document the base Kali and VirtualBox setup.
- [x] Document NAT Network and static IP configuration.
- [x] Document shared-folder permissions.
- [x] Add verification and troubleshooting steps.
- [x] Document snapshot creation and recovery.
- [ ] Add a deliberately vulnerable target VM.
- [ ] Document target network isolation and connectivity checks.
- [ ] Add repeatable lab setup scripts.
- [ ] Add a lab network diagram with verified settings.
- [ ] Document sample exercises and findings from authorized practice.

Items marked as planned are not claims that the work is already complete.

---

## 👤 Author

**[K.Balamurugan]**  
Cybersecurity learner — building things, breaking them in a controlled lab, and documenting what I learn.

- **GitHub:** [https://github.com/pr0xyweb](https://github.com/pr0xyweb)


---

*If this guide helps you reproduce the lab or solve a problem, feel free to open an issue or suggest an improvement.*
