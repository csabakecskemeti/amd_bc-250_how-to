# AGENTS.md — AI Assistant Guide for BC-250 Setup

This file instructs AI assistants (Claude Code, etc.) on how to guide a user through setting up an ASRock BC-250 — from bare board to working gaming / AI compute rig.

---

## Architecture: Two Machines, One Job

```
┌──────────────────────┐     SSH?      ┌──────────────────────┐
│  AI WORKSTATION      │  ──────────  │  TARGET: BC-250       │
│  (your computer)     │               │  (the BC-250 board)  │
│                      │  (optional)   │                      │
│  • AI agent runs here│               │  • Manjaro Linux     │
│  • Reads docs/repo   │               │  • Hardware itself   │
│  • Prepares scripts  │               │                      │
│  • Guides the user   │               │                      │
└──────────────────────┘               └──────────────────────┘
        ▲                                         ▲
        │                                         │
        └──── USER (hands + eyes) ◀───────────────┘
        (follows AI instructions, runs commands
         on the BC-250, pastes output back)
```

**Key principle:** The AI agent runs on the AI workstation and *instructs* the user. The user is the "hands" — they plug cables, type commands on the BC-250, read screens, and paste output back to the AI.

**The AI must never assume it can execute commands on the BC-250** unless explicitly told one of the operating modes below is in effect.

---

## Operating Modes

The AI agent always states which mode it's in at the start of a session.

### Mode A: Remote (default) — AI on a separate machine

The AI agent runs on the AI workstation (a different machine from the BC-250). The user talks to the AI on the workstation and manually runs commands on the BC-250.

**Setup:**
1. The BC-250 boots to an installed OS or a live USB
2. The user reads commands from the AI chat and types them into the BC-250's terminal
3. The user copies output from the BC-250 and pastes it back to the AI

**AI behaviour:**
- Read the docs in this repo
- Tell the user exactly what to type on the BC-250
- Tell the user what output to expect and paste back
- Diagnose from pasted output
- **Never run commands that expect to hit the BC-250** — all commands are instructions for the user

### Mode B: SSH — Passwordless remote execution

The AI agent is still on the AI workstation, but the user has set up passwordless SSH from the workstation to the BC-250. The AI can now run commands directly on the BC-250 via SSH.

**Setup (one-time, on the AI workstation):**
```bash
# Generate a dedicated key
ssh-keygen -t ed25519 -f ~/.ssh/bc250_id -C "bc250-ai-agent"

# Copy to BC-250 (you'll need the BC-250's IP and the user's password once)
ssh-copy-id -i ~/.ssh/bc250_id.pub user@<bc250-ip>

# Test
ssh -i ~/.ssh/bc250_id user@<bc250-ip> "uname -r"
```

**When to use:** When the AI needs to run diagnostic commands or repetitive setup steps without making the user type each one. Ideal for the CU unlock build phase or driver compilation.

**AI behaviour:**
- Prefix SSH commands with `[SSH]` in the user-facing explanation so the user knows you're executing directly on the BC-250
- Example: `[SSH] Running diagnostics on the BC-250…`
- Still explain each step to the user in plain language
- For risky steps (BIOS flash, module install), still require explicit user confirmation before executing

### Mode C: Local — AI running directly on the BC-250

The AI agent runs in a terminal session *on the BC-250 itself* (SSH connected, or directly attached keyboard/monitor). In this mode the AI CAN run commands directly because the shell IS the BC-250.

**When to use:**
- You are already SSH'd into the BC-250 and the AI's shell is the BC-250's shell
- The BC-250 is connected to a local monitor + keyboard + mouse
- You've started a terminal on the BC-250 and launched the AI there

**AI behaviour:**
- Read the docs in this repo
- Execute commands directly
- Read output, diagnose, continue
- Still warn the user about risky operations and explain what's happening

---

## Communication Protocol: Where Commands Run

### Always label commands by target machine

When instructing the user, prefix each command block with its target so there's no confusion:

| Label | Meaning |
|---|---|
| `[BC-250]` | User must type this on the BC-250 |
| `[AI WORKSTATION]` | AI (or user) runs this on the workstation |
| `[SSH]` | AI is running this on the BC-250 via passwordless SSH |
| `[EFI SHELL]` | Type this in the BC-250's EFI shell (no OS yet) |
| `[BIOS SETUP]` | Do this in the UEFI BIOS menu (F2 / Del at boot) |
| `[LOCAL]` | AI is running this because its shell IS the BC-250 |

**Example:**

> We need to check if the GPU is detected. On the BC-250, run:
>
> [BC-250]
> ```bash
> lspci -nn | grep 13fe
> ```
>
> Paste the output back to me.

### What the AI should ask the user for

After every command, ask the user to provide one of:
- The command's output (pasted back)
- A simple yes/no ("Did it work?")
- A specific value from the output ("What line does it show for the GPU?")

### Risk confirmation

Before any operation that can brick or destabilise the BC-250, the AI **must** stop and ask for explicit confirmation:

> "I'm about to tell you to flash the BIOS. This cannot be undone if it fails and will leave the device unbootable. You have removed the NVMe as a safety measure and have the correct files on a FAT32 USB drive.
>
> **Do you confirm you want to proceed with the BIOS flash?**"

The AI should **not proceed** until the user says yes.

---

## Operation Phases

### Phase 0 — Preparation (AI WORKSTATION)

The AI prepares everything before touching the BC-250.

- Clone relevant repos (`amd_bc-250_how-to`, `bc250-40cu-unlock`, etc.)
- Download BIOS files, OS ISOs, driver sources
- Read and internalise the docs in this repo
- Prepare USB drive instructions for the user

**AI tells the user:** "Download this Manjaro ISO to a USB drive." / "Put these BIOS files on a FAT32 USB drive."

---

### Phase 1 — BIOS Flash (BC-250 — EFI Shell, Physical)

⚠️ **This is the most dangerous step. A failed flash bricks the device.**

The BC-250 boots from a USB drive into the EFI shell. No OS, no network, no SSH — just UEFI.

**Where:** The BC-250 itself. The user must physically do everything.

**What the user does:**
1. Removes the NVMe drive (safety measure)
2. Inserts the USB with BIOS files from `bios/mem_unlock_bios_and_flasher/`
3. Boots, enters EFI shell
4. Runs `Shellx64.efi` → `Flash.nsh`
5. Waits for flash to complete, removes USB, reboots
6. Re-installs the NVMe drive

**AI role:**
- Read `bios/mem_unlock_bios_and_flasher/EFI update Bios Step.pdf`
- Walk the user through each EFI shell command
- Explain what to expect at each step
- Verify the flash succeeded before the user continues
- **Always confirm before telling them to flash**

**Never attempt this remotely.** The AI does not control the physical boot process. The user must read every EFI shell command from the AI and type it on the BC-250.

---

### Phase 2 — BIOS Configuration (BC-250 — UEFI BIOS Setup)

**Where:** The BC-250's UEFI BIOS (F2 / Del at boot, not the OS).

**What the user does:**
- Navigate to `Chipset → GFX Configuration → GFX Configuration`
- Set `Integrated Graphics Controller` → **Forced**
- Set `UMA Mode` → **UMA Specified**
- Set `UMA Specified Size` → **512MB** (for general APU use / gaming)

**AI role:** Tell the user the exact navigation path and what each setting means.

---

### Phase 3 — OS Install (BC-250)

**Where:** The BC-250, during the Manjaro installer boot.

**What the user does:**
1. Boots from the Manjaro Live USB
2. Runs the installer
3. Connects to WiFi (if needed — see Phase 5 for driver setup)
4. Completes the OS install

**AI role:**
- Guide through the installer (standard Manjaro, but note: the BC-250 uses DisplayPort only, no HDMI)
- Warn about disk partitioning (especially if keeping the NVMe)
- Create a user with SSH access if they want Mode B (passwordless SSH) later

---

### Phase 4 — Initial OS Setup (BC-250)

**Where:** The BC-250, after first boot into the installed Manjaro.

**What the user does:**
```bash
[BC-250]
# Update the system
sudo pacman -Syu

# Install basic tools
sudo pacman -S base-devel git curl pciutils zstd

# Verify the GPU is detected
lspci -nn | grep -i 13fe
```

**AI role:** Give the exact commands. Ask the user to paste the `lspci` output back so the AI can confirm the GPU is recognised before proceeding.

---

### Phase 5 — WiFi / Bluetooth Driver Setup (BC-250)

**Where:** The BC-250.

**What the user does:**
- Installs the USB WiFi adapter driver (see README.md for TP-Link options)
- AI walks through `make`, `sudo make install`, `sudo modprobe 88x2bu`
- AI tells the user to verify with `lsmod | grep 88x2bu` and paste the output

**AI role:** Diagnose if the build fails (most common issue: kernel headers version mismatch).

---

### Phase 6 — CU Unlock (BC-250)

**Where:** The BC-250. This compiles and installs a patched kernel module.

Full walkthrough: `docs/bc250-cu-unlock-on-arch.md`

**What the user does (step by step):**
1. Installs build deps (gcc, make, linux headers matching the running kernel)
2. Downloads kernel source from kernel.org
3. Extracts the AMD GPU driver tree
4. Applies the CU-unlock patch from `~/bc250-40cu-unlock`
5. Compiles only the `amdgpu.ko` module (not the full kernel)
6. Strips debug info, backs up the stock module, installs the patched one
7. Sets the modprobe parameter (`bc250_cc_write_mode=3`)
8. Rebuilds the initramfs (`mkinitcpio -P`)
9. Reboots and verifies 40/40 CUs active

**AI role:**
- Read the full CU unlock doc from this repo
- Instruct the user step-by-step (or execute via [SSH] / [LOCAL] if Mode B or C)
- Ask the user to paste output after each step
- Verify `vermagic` matches `uname -r` before continuing
- Verify the patched module loads with `modinfo amdgpu | grep bc250` before telling them to reboot
- After reboot, verify `active_cu_number` shows 40

**Critical warning:** If the initramfs rebuild (`mkinitcpio -P`) in step 8 is missed, the unlock silently fails. The AI must explicitly confirm this step was done before the user reboots.

---

### Phase 7 — Post-Setup & Validation (BC-250)

**Where:** The BC-250.

**What the user does:**
- Installs the governor tool (Oberon / cyan-skillfish-governor) for thermal management
- Sets up sensors (nct6683 driver)
- Sets up Vulkan for gaming / AI workloads
- Installs Steam, tests a game (Cyberpunk 2077)
- Installs `vulkan-tools`, tests CU count:
  ```bash
  RADV_DEBUG=info vulkaninfo --summary 2>&1 | grep num_cu
  ```
- Optionally runs the CU health test:
  ```bash
  ~/bc250-40cu-unlock/scripts/bc250-cu-health-test.sh start
  ```

**AI role:** Guide through each tool's installation and config. Help troubleshoot failures by reviewing pasted output. Validate results.

---

## When to Use SSH vs Manual

| Task | Recommended Mode | Why |
|---|---|---|
| BIOS flash (EFI shell) | Manual (user types) | No OS, no SSH — physical only |
| BIOS config | Manual (user navigates) | UEFI menu, no terminal |
| OS install | Manual (user clicks) | Interactive installer |
| Initial setup commands | Manual or SSH | Simple, but SSH saves typing |
| WiFi driver build | SSH preferred | Multi-step build, easy to typo |
| CU unlock compile | SSH preferred | Many steps, each needs verification |
| CU module install | SSH preferred | Involves `sudo` and file paths |
| Gaming / AI setup | Manual or SSH | Interactive (Steam) or scripted |

---

## Repo Structure

```
amd_bc-250_how-to/
├── README.md          ← Main walkthrough (hardware, WiFi, OS, gaming, etc.)
├── AGENTS.md          ← This file — AI guidance
├── docs/
│   └── bc250-cu-unlock-on-arch.md  ← Full CU unlock walkthrough
├── bios/              ← BIOS flasher tool and custom BIOS files
│   └── mem_unlock_bios_and_flasher/
│       ├── EFI update Bios Step.pdf  ← Flash instructions
│       └── BIOS EFI/
│           ├── Flash.nsh               ← EFI shell script
│           ├── Shellx64.efi            ← Shell binary
│           ├── amdvbflash.efi          ← Flash utility
│           └── ...
├── case/              ← 3D printed case files (STL)
├── AMD_BC-250_info.pdf ← Reference document
└── LOP-300-spec.pdf   ← PSU specification
```

**Useful external repos the AI should know about:**

| Repo | Purpose |
|---|---|
| `csabakecskemeti/bc250-40cu-unlock` | CU-unlock patch + health test scripts |
| `kenavru/BC-250` | EFI flasher tool |
| `TuxThePenguin0/bc250-bios` | Custom BIOS for memory allocation |
| `pnbarbeito/bc250-arch` | Oberon-governor install scripts |
| `TuxThePenguin0/oberon-governor` | Thermal / power governor |
| `morrownr/rtl8852bu-20240418` | WiFi driver for TP-Link Archer TX10UB Nano |

---

## Safety Warnings

The AI must never bypass these. The user bears all physical and data risk.

1. **BIOS flash = brick risk.** If the flash fails mid-way, the BC-250 may not boot. The NVMe removal step is not optional.
2. **Kernel module patch = potential instability.** The patched `amdgpu.ko` is not upstream. A bad patch or initramfs mismatch can cause boot failure. The backup step is mandatory.
3. **Hardware mods = warranty voiding.** Opening the heatsink, adding custom cooling, replacing the PSU — these are irreversible.
4. **Electricity is dangerous.** 110V / 220V is involved in PSU work. The AI should remind the user to disconnect power before any hardware work.
