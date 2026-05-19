# Disk Image Clone — Full Step-by-Step Runbook

Cloning **LENOVO F0HN00S0RU** (Windows 11 Pro, RETAIL) to a new PC.
Source disk: 477 GB NVMe, ~93 GB used → image ≈ 55–70 GB compressed.

> **Golden rule:** do NOT wipe or sell the old PC until the new one is fully
> verified and working. The old PC is your safety net.

---

## Phase 0 — Shopping list

- [ ] **USB stick, ≥8 GB** — becomes the bootable rescue media (erased).
- [ ] **External USB drive/SSD, ≥128 GB** — holds the image. Strongly recommended;
      makes the restore fully offline and reliable.
- [ ] *(Only if you skip the external drive)* an **Ethernet cable** to connect both
      PCs during the restore — WinPE WiFi is unreliable.

---

## Phase 1 — Check BitLocker on the OLD PC

1. Open an **Administrator** terminal (Start → type `powershell` → right-click →
   Run as administrator).
2. Run: `manage-bde -status C:`
3. Read the **Protection Status** line:
   - **Protection Off** → nothing to do, skip to Phase 2.
   - **Protection On** → do BOTH of these before continuing:
     a. Get the recovery key: go to `https://account.microsoft.com/devices`,
        find this PC, "View BitLocker keys". Save the 48-digit key somewhere
        OFF this machine (phone, paper, email to yourself).
     b. Suspend BitLocker so imaging is clean: `manage-bde -protectors -disable C:`

---

## Phase 2 — Prep the NEW PC

The restore **erases the new PC's disk completely.** Before you start:

1. Copy off anything already on the new PC you want to keep.
2. Note its own Windows key as a fallback (admin terminal):
   `powershell -Command "(Get-CimInstance SoftwareLicensingService).OA3xOriginalProductKey"`
   If it returns blank, its license is digital/linked to a Microsoft account — fine.
3. Confirm the new PC's internal disk is **≥93 GB** (any modern SSD is).

---

## Phase 3 — Create the system image (OLD PC)

1. Download **Hasleo Backup Suite Free** from `https://www.hasleo.com/` and install it.
2. Plug in the **external drive**.
3. Open Hasleo Backup Suite → click **System Backup**.
   - It auto-selects every partition Windows needs (EFI, C:, recovery). Leave as-is.
4. **Destination:** click the destination box → pick the external drive.
5. Open **Backup Options** → make sure **Compression** is enabled (normal/medium).
6. Click **Backup Now**. Takes ~15–40 min. Do not sleep/shut down the PC.
7. When done, confirm the image file exists on the external drive.

> **No external drive? (WiFi path)** Save the image to `C:\` instead, then share
> that folder: right-click folder → Properties → Sharing → Share. You'll pull it
> over **Ethernet** during Phase 5 (keep the old PC powered on and cabled).

---

## Phase 4 — Create the bootable rescue USB (OLD PC)

1. Plug in the **8 GB USB stick** (back up anything on it first — it gets erased).
2. In Hasleo Backup Suite → **Tools** → **Create Bootable Media**.
3. Choose **WinPE** (Windows-based — better driver support than Linux).
4. Select the USB stick as the target → **Create**. Takes a few minutes.
5. Label the stick so you don't confuse it with the image drive.

---

## Phase 5 — Restore the image onto the NEW PC

1. On the new PC, plug in BOTH: the **rescue USB** and the **external drive**.
   *(WiFi path: plug in the rescue USB and the Ethernet cable instead.)*
2. Power on the new PC and immediately tap **F12** repeatedly → the Lenovo boot
   menu appears → choose the **USB stick**.
   - If the USB isn't listed: enter BIOS setup (tap **F2** or **F1** at power-on),
     and either enable USB boot or temporarily **disable Secure Boot**. Save & exit,
     retry F12.
3. The Hasleo WinPE environment loads. Click **Restore**.
4. **Select the image:** browse to the external drive → pick the backup image.
   *(WiFi path: choose network/share, enter `\\OLD-PC-NAME\SharedFolder`.)*
5. **Select the target:** the new PC's **internal disk**. Double-check you picked
   the internal disk, NOT the USB or external drive.
6. **Enable the dissimilar-hardware option** — look for a checkbox labelled
   **"Restore to dissimilar hardware"** / **"Universal Restore"** in the wizard.
   Tick it. This injects generic drivers so Windows boots on the new hardware.
7. Confirm you understand the target disk will be wiped → **Start**.
8. Restore takes ~20–40 min. Do not interrupt power.

---

## Phase 6 — First boot on the NEW PC

1. When the restore finishes, **remove the USB stick** and reboot.
2. Windows boots, detects new hardware, installs drivers, and reboots itself a
   couple of times. This is normal — let it settle.
3. If BitLocker prompts for a recovery key → enter the 48-digit key from Phase 1.
4. Once at the desktop, run **Windows Update** fully (several rounds) for drivers.
5. Install the new PC vendor's update tool (Lenovo System Update if it's a Lenovo,
   otherwise the new brand's equivalent) and let it install hardware drivers.

### If the new PC won't boot after restore
- Almost always a boot-mode mismatch. Enter BIOS → set to **UEFI mode** with
  **Secure Boot ON**, save, retry.
- Still failing → boot the rescue USB again → use Hasleo's **boot repair** option.

---

## Phase 7 — Reactivate Windows

Your license is **RETAIL**, which is transferable to new hardware.

1. Settings → System → **Activation**.
2. If it shows "not activated": click **Troubleshoot** →
   **"I changed hardware recently"** → sign in with your Microsoft account →
   select this device from the list → **Activate**.
3. If troubleshooter doesn't appear, make sure you're signed into Windows with the
   Microsoft account the license is linked to.

---

## Phase 8 — Re-auth machine-bound services

A clone copies files, but some things are tied to the old hardware and must be
re-done on the new PC:

- [ ] **Windows Hello / PIN** — re-create it (Settings → Accounts → Sign-in options).
- [ ] **Tailscale** — it re-registers as a new node. The old IP `100.121.110.58`
      stays with the old machine; the new PC gets its own IP. Update any SSH
      configs/bookmarks that referenced the old IP.
- [ ] **Claude Code** — should work as-is (config is in the image). If auth is
      stale, run `/login`.
- [ ] **Docker Desktop** — re-launch; it may re-initialize the WSL backend.
- [ ] **GitHub CLI / git** — `gh auth status`; re-run `gh auth login` if needed.
- [ ] **VS Code Claude extension patch** — the `webview/index.js` patch is
      per-install; re-apply if the extension updated.

---

## Phase 9 — Verify, then retire the old PC

1. Confirm on the new PC: Windows activated, internet works, your files are
   present, key apps launch (Docker, VS Code, Chrome, Claude Code).
2. Run a small git clone + commit to confirm git identity carried over.
3. **Only now**, once everything is verified:
   - On the OLD PC, re-enable BitLocker if you suspended it:
     `manage-bde -protectors -enable C:`
   - Sign out of accounts, then reset/wipe the old PC if you're disposing of it
     (Settings → System → Recovery → Reset this PC → Remove everything).

---

## Time budget

| Phase | Rough time |
|-------|-----------|
| Imaging (Phase 3) | 15–40 min |
| Rescue USB (Phase 4) | 5–10 min |
| Restore (Phase 5) | 20–40 min |
| First boot + drivers (Phase 6) | 30–60 min |
| **Total hands-on** | **~2–3 hours** |
