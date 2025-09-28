# OpenMediaVault: Public Guest Share + Two Private Per‑User Shares (w/ Optional Client‑Side Encryption)

A clean, reproducible setup for a home NAS on **OpenMediaVault (OMV 6/7)** where you keep one **public, guest‑only** SMB share for media/DLNA, and add two **private, per‑user** SMB shares that only each user can access. Includes macOS connection tips and optional **VeraCrypt/Cryptomator** client‑side encryption so that even NAS admins cannot read the private data at rest.

> Works great on OMV running as a VM on Proxmox. No VLANs required. Keep the media share open for the whole LAN, and secure personal data with proper SMB permissions and (optionally) client‑side encryption.

---

## Table of Contents
1. [What You Get](#what-you-get)
2. [Prerequisites](#prerequisites)
3. [Quick OMV Concepts](#quick-omv-concepts)
4. [Step‑by‑Step Setup](#step-by-step-setup)
   - [1) Create Users](#1-create-users)
   - [2) Create Per‑User Shared Folders](#2-create-per-user-shared-folders)
   - [3) Grant Privileges (Login Rights)](#3-grant-privileges-login-rights)
   - [4) Create SMB Shares (Private)](#4-create-smb-shares-private)
   - [5) Keep/Adjust the Public Media Share](#5-keepadjust-the-public-media-share)
5. [macOS: Mount Private and Guest Shares at the Same Time](#macos-mount-private-and-guest-shares-at-the-same-time)
6. [Optional: “Even the Admin Can’t Read” – Client‑Side Encryption](#optional-even-the-admin-cant-read--client-side-encryption)
   - [Option A: VeraCrypt Container](#option-a-veracrypt-container)
   - [Option B: Cryptomator Vault](#option-b-cryptomator-vault)
7. [SMB Advanced Options: What to Use](#smb-advanced-options-what-to-use)
8. [Troubleshooting](#troubleshooting)
9. [Backups & Restore Notes](#backups--restore-notes)
10. [Security & Hygiene Tips](#security--hygiene-tips)
11. [Appendix: Minimal Checklist](#appendix-minimal-checklist)

---

## What You Get

- **MEDIA** (guest‑only, no credentials): Everyone on the LAN can read (and optionally write) for DLNA/streaming.
- **ALICE** (private): only user `alice` can authenticate and read/write.
- **BOB** (private): only user `bob` can authenticate and read/write.
- Optional: **client‑side encryption** inside ALICE/BOB using **VeraCrypt** or **Cryptomator** so the NAS (and its admin/root) cannot read the contents at rest.

```text
LAN
 ├─ SMB: \\omv\MEDIA        (Guest‑Only)
 ├─ SMB: \\omv\ALICE        (alice only)
 └─ SMB: \\omv\BOB          (bob only)
```

---

## Prerequisites

- OMV 6 or 7 (Web UI) with SMB/CIFS service enabled.
- A data disk mounted in OMV for shares.
- Two user accounts you plan to use (we’ll create them in the guide: `alice`, `bob`).
- (Optional) macOS client; Windows works similarly.

> **Naming convention** in this guide: shared folders `alice_private`, `bob_private`; SMB share names `ALICE`, `BOB`, `MEDIA`.

---

## Quick OMV Concepts

- **Shared Folders** are OMV’s base objects (backed by POSIX files/dirs).
- **Privileges** (in *Shared Folders → Permissions*) control **who may log in via SMB** and whether they get **read/write** or **read‑only**. Use this for SMB access control.
- **SMB Share settings** (in *Services → SMB/CIFS → Shares*) decide whether the share is **Public/Guest** or **Private**, browseable, read‑only, and advanced options like **transport encryption**, **inheritance**, **recycle bin**.
- **Guest account** maps to Linux `nobody`. It’s **read‑only by default**; to allow guest **writes**, the underlying shared folder POSIX mode must permit it (e.g., `0777`).

---

## Step‑by‑Step Setup

### 1) Create Users
**OMV → Users → Users → Create**
- Create `alice` and `bob`.
- Suggested shell: **/usr/sbin/nologin** (no shell access).
- Set passwords here (keeps Linux + Samba in sync).

### 2) Create Per‑User Shared Folders
**OMV → Storage → Shared Folders → Add**
- **`alice_private`** → pick your data filesystem/device → Save
- **`bob_private`**   → pick your data filesystem/device → Save
- Leave default POSIX perms (or `0770` if you prefer a tighter base).

### 3) Grant Privileges (Login Rights)
**OMV → Storage → Shared Folders → select folder → Permissions (aka Privileges)**
- For `alice_private`: **alice = Read/Write**, **bob = No access** (and **users group = No** if present).
- For `bob_private`: **bob = Read/Write**, **alice = No access**.
- Apply/save changes.

> **Tip:** This is the critical step that enforces per‑user access via SMB.

### 4) Create SMB Shares (Private)
**OMV → Services → SMB/CIFS → Shares → Add** (twice)
- **Shared folder:** `alice_private` → **Name:** `ALICE`
  - **Public:** **No** (login required)
  - **Read only:** No
  - **Browseable:** Yes (or No if you want it hidden from network browse lists)
- **Shared folder:** `bob_private` → **Name:** `BOB`
  - Same toggles as above.
- Save → Apply (yellow banner).

### 5) Keep/Adjust the Public Media Share
Open your existing media share (*Services → SMB/CIFS → Shares → Edit*):
- **Public:** **Only guests** (Guest‑Only).
- If you want guests to **write** (not typical for DLNA): ensure the **Shared Folder POSIX mode** allows it (e.g., `0777`). Otherwise leave guest read‑only.

---

## macOS: Mount Private and Guest Shares at the Same Time

macOS holds **one SMB login per server name** at a time. Use two aliases for the same box:

1) **Private share**: Finder → Go (⌘K) → `smb://omv/ALICE` → Connect as **Registered User** (`alice`)  
2) **Guest share**: Finder → Go (⌘K) → `smb://omv.local/MEDIA` (or use the NAS **IP**, or a DNS alias like `nas-guest`) → Connect as **Guest**

Now both are mounted concurrently (different aliases = different credential sessions). If Finder reused the wrong creds, eject (⌘E) and reconnect.

> **Pro tip:** Create two DNS names on your router (e.g., `nas-private` and `nas-guest`) → mount `smb://nas-private/ALICE` and `smb://nas-guest/MEDIA`.

---

## Optional: “Even the Admin Can’t Read” – Client‑Side Encryption

NAS/root can always read unencrypted files. To prevent that, encrypt **before** data leaves your Mac.

### Option A: VeraCrypt Container
- Install **VeraCrypt** on macOS.
- Create a **container file** in `ALICE` (e.g., `~/Volumes/ALICE/Private.vc` via the SMB mount).
- Choose size & strong passphrase.
- Mount the container locally in VeraCrypt; work inside the mounted volume.
- **Backups:** any change touches the container; backup tools may re‑copy the whole file. Consider multiple smaller containers if that’s an issue.

### Option B: Cryptomator Vault
- Install **Cryptomator** on macOS.
- Create a **vault folder** inside `ALICE` (filenames and contents are encrypted).
- Mount the vault locally; work as usual.
- **Backups:** more incremental‑friendly than a single big container.

> You can enable SMB **Transport encryption** (see below) *as well*—that protects the traffic on the wire. VeraCrypt/Cryptomator protect the data **at rest on the NAS**.

---

## SMB Advanced Options: What to Use

Edit per share: **OMV → Services → SMB/CIFS → Shares → (Edit) → Advanced**

- **Transport encryption (SMB3)**
  - **What it does:** Encrypts SMB traffic on the network; legacy clients that can’t do SMB3 encryption are denied.
  - **Private shares (ALICE/BOB):** **Enable** for on‑wire privacy if all clients are modern; small CPU/throughput cost on NAS.
  - **MEDIA:** **Disable** (maximize streaming performance; content isn’t sensitive).

- **Inherit ACLs**
  - Keep **ON** (default). Ensures new files/folders inherit parent ACLs; harmless and consistent.

- **Inherit permissions**
  - Default **OFF**. For single‑user private shares, leave it off.
  - Turn **ON** only if you later build a collaborative share and want new files to copy parent ownership/perm bits.

- **Enable recycle bin**
  - **OFF** for shares containing **VeraCrypt containers** or **Cryptomator vaults** (deleting large container files can explode space usage).
  - **ON** (with limits) if you want a safety net for normal files. Consider max size/age to avoid disk bloat.

---

## Troubleshooting

- **“Access denied” on private shares**
  1) Re‑check *Shared Folders → Permissions (Privileges)* for that folder (owner user must be **Read/Write**; others **No access**).
  2) If you tinkered with POSIX/ACLs under the path, overly restrictive perms can still block; reset to sane defaults.
  3) Rarely, Samba creds drift; fix via SSH: `sudo smbpasswd alice` then retry.

- **macOS keeps connecting as Guest**
  - Eject the share and reconnect as Registered User.
  - Use different server aliases (`omv` vs `omv.local` vs IP) for separate guest vs private sessions.

- **Can’t mount both guest + private at once**
  - Use two different server names/aliases (see macOS section).

- **Performance drop after enabling Transport encryption**
  - That’s expected; try disabling it on high‑throughput shares like MEDIA. Keep client‑side encryption for privacy.

---

## Backups & Restore Notes

- **Proxmox**: snapshot/backup the OMV VM regularly (Proxmox Backup Server is ideal).  
- **Inside OMV**: if you back up **encrypted** private data, you’re backing up **ciphertext**—which is good for privacy. Test restores: you’ll need the **VeraCrypt/Cryptomator passphrase** on a client to read files.
- **VeraCrypt containers**: consider multiple smaller containers to make backups more incremental‑friendly.
- **Cryptomator**: generally better for incremental backup and cloud sync patterns.

> **Never lose the passphrases.** Store in a reputable password manager; keep a printed emergency kit in a safe place.

---

## Security & Hygiene Tips

- Keep **SMB1 disabled** (modern OMV templates do); use SMB3.
- Restrict SMB to your LAN interface(s) only (OMV SMB “Interfaces” option) if you have multiple NICs.
- Consider **transport encryption ON** for ALICE/BOB if you often use Wi‑Fi.
- Avoid server‑side ACL complexity unless needed; rely on *Privileges* for SMB access control.
- Keep OMV updated; enable basic auth rate‑limiting (Fail2ban) if you ever expose services (generally: don’t).

---

## Appendix: Minimal Checklist

- [ ] Create users `alice`, `bob` in OMV Users.
- [ ] Create shared folders `alice_private`, `bob_private` on data disk.
- [ ] Set *Privileges*: `alice_private` → **alice RW**, others **No**; `bob_private` → **bob RW**, others **No**.
- [ ] SMB shares: `ALICE` (Public **No**), `BOB` (Public **No**). Browseable as you prefer.
- [ ] MEDIA share: Public **Only guests**; guest writes only if POSIX mode allows (e.g., `0777`).
- [ ] (Optional) For ALICE/BOB: enable SMB **Transport encryption**.
- [ ] (Optional) Set up **VeraCrypt** container or **Cryptomator** vault inside each private share.
- [ ] Test from macOS with two aliases (`omv` vs `omv.local`/IP) for simultaneous private + guest mounts.
- [ ] Configure backups (Proxmox VM; plus any file‑level backup you need).

---

**Author’s note:** This guide is intentionally minimal, reproducible, and GitHub‑friendly. Adapt share names and disk paths as you see fit.
