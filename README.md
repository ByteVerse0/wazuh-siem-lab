# wazuh-siem-lab

> **Status:** work in progress. More detection scenarios will be added as they are completed.

A hands-on SIEM lab built on a Proxmox homelab. Wazuh monitors a Linux endpoint, an attacker machine generates suspicious activity, and the resulting alerts are used to practice detection and analysis.

## Objective

Learn blue team fundamentals by simulating realistic attack scenarios, collecting the logs in a SIEM, and documenting how each activity is detected.

## Environment

| Host | Role | OS | Notes |
|------|------|----|-------|
| Proxmox VE host | Hypervisor | Proxmox VE 9 | Runs all the virtual machines below |
| `wazuh-server` | SIEM, all-in-one | Ubuntu Server 24.04 | 2 vCPU, 4 GB RAM, 50 GB disk (see the note on requirements) |
| `vm-ubuntu` | Monitored endpoint with Wazuh agent | Ubuntu 24.04 | |
| `kali` | Attack simulation | Kali Linux | |

**Wazuh version:** 4.14.6, single node (indexer, server and dashboard on one machine).

## Architecture

```
Internet
    |
Home router
    |
Proxmox VE
    ├── wazuh-server  (indexer + server + dashboard)
    ├── vm-ubuntu     (Wazuh agent, monitored endpoint)
    └── kali          (attacker)

vm-ubuntu --logs--> wazuh-server <--browser-- analyst
kali --simulated attacks--> vm-ubuntu
```

All machines share one network segment. Placeholders used below: `<SERVER_IP>` is the address of `wazuh-server`, `<GATEWAY_IP>` is the router.

## Setup

### 1. Create the virtual machines

Create three virtual machines on Proxmox: `wazuh-server` and `vm-ubuntu` from the Ubuntu Server 24.04 ISO, and `kali` from the Kali Linux installer image. Attach them to the same bridge (`vmbr0` by default).

The Wazuh quickstart recommends 4 vCPU, 8 GB of RAM and 50 GB of disk for up to 25 endpoints. This lab used 2 vCPU and 4 GB, which is below the recommendation (see step 2).

### 2. Install Wazuh on `wazuh-server`

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

`-a` installs the three components (indexer, server, dashboard) on the same machine. At the end the installer prints the `admin` user and its password: save them.

If the installer stops because the hardware check fails, you can skip the check on a lab machine with `-i`:

```bash
sudo bash ./wazuh-install.sh -a -i
```

The generated passwords can be read again later from the installer archive:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

The dashboard is then available at `https://<SERVER_IP>` (the certificate is self-signed, so the browser shows a warning).

### 3. Give `wazuh-server` a fixed address

With DHCP, the server's address can change after a reboot, which breaks the agents' configuration. Edit the network configuration:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - <SERVER_IP>/24
      routes:
        - to: default
          via: <GATEWAY_IP>
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
```

The interface name (`ens18` here) may differ: check it with `ip addr`. Optionally set a readable hostname:

```bash
sudo hostnamectl set-hostname wazuh-server
```

Do the same on `vm-ubuntu` with its own fixed address.

### 4. Install the agent on `vm-ubuntu`

In the dashboard open **Agents**, then **Deploy new agent**. Choose the operating system and architecture, enter `<SERVER_IP>` as the server address, and run the generated commands on `vm-ubuntu`. For a Debian-based system they have this shape:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_amd64.deb && sudo WAZUH_MANAGER='<SERVER_IP>' dpkg -i ./wazuh-agent_4.14.6-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Always use the exact command shown by your dashboard. The agent should appear as **Active** in the **Agents** page.

### 5. Prepare the attacker machine

Enable SSH on `kali` so it can be reached and used for login-based scenarios:

```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

## Detections

| # | Scenario | MITRE ATT&CK | Status |
|---|----------|--------------|--------|
| 1 | Privilege escalation reconnaissance | T1548.003 | Detected |

### Scenario 1: privilege escalation reconnaissance

Simulates an attacker who already has access to a system and looks for a way to gain more privileges. Run on `vm-ubuntu`:

```bash
sudo cat /etc/shadow
```

`/etc/shadow` holds the hashed passwords of all users. An attacker reads it to crack them offline.

```bash
sudo find / -perm -4000 2>/dev/null
```

This lists the binaries with the SUID bit set. A SUID binary runs with the permissions of its owner (often root) instead of the user who starts it, so attackers search for them as a way to become root.

**What Wazuh showed:** both commands were recorded with timestamp, user and working directory, and mapped to MITRE ATT&CK T1548.003. In the dashboard, look under the security events of the `vm-ubuntu` agent.

**Known noise:** Wazuh also raised rootkit-detection alerts on system binaries such as `/bin/cat` and `/bin/md5sum`. They are false positives: the rootkit signatures match strings like `/bin/sh` that legitimately appear inside the scripts of recent Ubuntu releases.

**Vulnerability Detector:** Wazuh reported 57 vulnerabilities in the kernel running on `vm-ubuntu`.

## Planned scenarios

- Brute force SSH login attempts against `vm-ubuntu` from `kali`
- Suspicious process execution
- Creation of unauthorized users
- File integrity monitoring alerts

## Tools

- [Wazuh](https://wazuh.com/) — SIEM and XDR
- [Proxmox VE](https://www.proxmox.com/) — hypervisor
- [Kali Linux](https://www.kali.org/) — attack simulation

## Notes

- Run these simulations only on machines you own, in an isolated lab network.
- The Wazuh admin password is generated at install time and is not stored in this repository.
