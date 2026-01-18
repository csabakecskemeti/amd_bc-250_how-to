# AMD BC-250 How To
(-details in AMD_BC-250_info.pdf)

The ASRock AMD BC-250 is a crypto mining hardware. It's not a card but a full computer based on the PlayStation 5. As mining is not efficient on this any more, the boards are selling out in the range of $60-$100 each on eBay (as of November 2025).

The BC-250 APU is slightly tuned down to 6 CPU cores (PS5 has 8) and 24 GPU compute units (PS5 has 32). The memory is 16GB integrated GDDR6.

As this is a full computer it also has 4 USB, one Ethernet and a DisplayPort port on its I/O plate. Additionally it has an M.2 NVMe slot for storage (tested with 1TB stick). All of these makes it an ideal small form factor budget mini gaming PC or a budget travel gaming PC.

> **IMPORTANT!**
>
> The following description includes hardware modifications involving sharp tools and exposure to 110V electrical components, as well as software modifications that can render the BC-250 device non-functional. Proceed only if you fully understand the risks and have the necessary skills and safety knowledge.
>
> By using this information, you acknowledge that you are solely responsible for your actions. I do not accept any liability for injuries, electrical hazards, data loss, device damage, or any other consequences resulting from following the procedures in this document. If you are not completely confident in performing these modifications safely, do not attempt them.

---

## Hardware

### Case

I found 2 nice 3D designs for the BC-250. Both offer integrated PSU and cooling.

#### Mini case with integrated LOP 300-12 PSU

Probably the most compact shell case with integrated PSU. The PSU that matches the design is the LOP 300-12 (300W 12V). It does not come with cooling, one central 120mm fan provides the cooling both for the PSU and the BC-250 board. This looks ideal to make it a travel PC.

No modification needed for the finstack.

To use the LOP 300-12V power supply you have to remove the factory pinout and solder a PCIe power connector instead. Same thing with the 110V power plug.

I've found that the middle shell piece was very tight so I've modified the original model. This new version is easier to install. The design uses M3 size heated inserts to assemble the case and hold the parts. The 3D model files are included in the repo with the original author's instruction.

**Credit:** https://www.printables.com/model/1228207-asrock-amd-bc-250-shell-case/files

#### Case with Flex PSU

Good size case. The cooling performance is probably better than with the mini shell case. The Flex PSU is probably more reliable. Finstack has to be opened up which is not easy to do. I found a finstack scooper 3D design and used that. I only opened up the middle piece for the single fan design.

The technique I've used was to get the scooper in position at the small openings at the middle section of the finstack, and then hammered it through till the other opening. That gave me a ~120x120mm open fin in the middle section as this case requires it.

The case is made up of a front and back piece and 2 custom adapters. Printing the front and back piece was over 2 days individually.

The 3D model files are included in the repo with the original author's instruction.

**Credit:**
- https://www.printables.com/model/1423572-minimal-case-for-bc250-and-flex-psu/files
- https://www.printables.com/model/1282906-bc-250-scooper/files

### Cooling

Depending on your choice of case you might have to open the closed server style finstack. Some open up completely and use 2x120mm fans. Some just open the middle section which is a 120x120 area that perfectly fits a single 120mm fan.

#### Open the heatsink

It's not easy to do. This scooper has helped a lot:
https://www.printables.com/model/1282906-bc-250-scooper/files

#### Heatpads replacement

- **Front (heatsink side):** 1.5mm heatpad for VRMs, thermal paste for APU
- **Back (memory side):** 2mm heatpad for memory modules

---

## BIOS

In order to use this as a gaming device, the shared memory allocation has to be changed. For that you need an unlocked 3.0 BIOS version.

### Flash

There's no need for hardware flashing the BIOS any more. Lots of descriptions and videos from a year ago has shown that. I found an EFI Flasher tool that works with the "unlocked" 3.0 BIOS.

**Based on:**
- https://github.com/kenavru/BC-250 - This is the flasher tool
- https://gitlab.com/TuxThePenguin0/bc250-bios - This has the custom BIOS to be able to change the memory allocation

I've created this "custom" BIOS flash, that has the unlocked 3.0 BIOS version:
https://github.com/csabakecskemeti/amd_bc-250_how-to/tree/main/bios

Follow the instructions here:
[EFI update BIOS Step.pdf](bios/mem_unlock_bios_and_flasher/EFI%20update%20Bios%20Step.pdf)

> **Note:** Additionally remove the NVMe drive just to be safe.

### Memory allocation for gaming

From: https://github.com/mothenjoyer69/bc250-documentation

> VRAM allocation is configured within: `Chipset -> GFX Configuration -> GFX Configuration`. Set `Integrated Graphics Controller` to **forced**, and `UMA Mode` to **UMA_SPECIFIED**, and set the VRAM limit to your desired size. 512MB is best for general APU use.

You have to do this to be able to game on the device. If this is set, even Cyberpunk 2077 has a playable framerate.

---

## OS

Manjaro Linux is a good choice. It knows the hardware and has drivers. I've used the KDE Plasma version:
https://manjaro.org/products/download/x86

---

## WiFi and Bluetooth

The BC-250 doesn't have built-in WiFi or Bluetooth, so you'll need a USB adapter. I've tested two options:

### Option 1: TP-Link TL-WN725N (N150) - Plug and Play

**TP-Link USB WiFi Adapter for PC (TL-WN725N), N150 Wireless Network Adapter**

| Pros | Cons |
|------|------|
| Plug and play - drivers included in kernel 6.12+ | Slower speeds (N150) |
| No installation required | Single band only |
| Very compact | No Bluetooth |

#### Setup

Simply plug in the adapter. The `6.12.64-1-MANJARO` kernel already contains the drivers natively.

#### Verify the adapter is recognized

```bash
lsusb
```

Check if the network interface is available:

```bash
ip link
```

#### Connect to WiFi

If you experience issues with saving the network password through the GUI, use `nmcli`:

```bash
nmcli device wifi connect "YOUR_WIFI_NETWORK" password "YOUR_PASSWORD" ifname wlp0s16f0u2
```

> **Note:** Replace `wlp0s16f0u2` with your actual interface name from `ip link`.

---

### Option 2: TP-Link Archer TX10UB Nano (AX900) - Recommended

**TP-Link 2-in-1 USB Bluetooth WiFi Adapter Archer TX10UB Nano | AX900 WiFi 6 BT 5.3**

| Pros | Cons |
|------|------|
| Faster WiFi speeds (AX900) | Requires driver installation |
| Dual band support (2.4GHz + 5GHz) | |
| WiFi 6 support | |
| Bluetooth 5.3 included | |
| Single USB port for both WiFi + BT | |
| Great for gaming with BT headphones | |

#### Step 1: Prepare the system

Install all necessary build tools and kernel headers:

```bash
sudo pacman -Syu
sudo pacman -S base-devel linux-headers git
```

- `base-devel` → compilers, make, etc.
- `linux-headers` → kernel headers for module compilation (e.g., `linux612-headers` for kernel 6.12)
- `git` → to clone the driver repo

> **Note:** Make sure kernel headers match your running kernel version.

#### Step 2: Clone the driver repository

The `morrownr` driver is actively maintained and supports kernels ≥6.11:

```bash
cd ~
git clone https://github.com/morrownr/rtl8852bu-20240418.git
cd rtl8852bu-20240418
```

**Alternative mirror:** https://github.com/csabakecskemeti/rtl8852bu-20240418

#### Step 3: Build and install the driver

```bash
make
sudo make install
```

- `make` → compiles the module
- `sudo make install` → installs it to `/lib/modules/$(uname -r)/kernel/...`

#### Step 4: Load the WiFi module

```bash
sudo modprobe 88x2bu
```

Verify it's loaded:

```bash
lsmod | grep 88x2bu
```

Expected output:
```
88x2bu       <some numbers>
```

#### Step 5: Power and auto-load tweaks (Recommended)

Disable Realtek USB power-saving (prevents disconnects and suspend/resume issues):

```bash
echo "options 88x2bu rtw_power_mgnt=0 rtw_enusbss=0" | sudo tee /etc/modprobe.d/88x2bu.conf
```

Ensure module loads automatically at boot:

```bash
echo "88x2bu" | sudo tee /etc/modules-load.d/88x2bu.conf
```

#### Step 6: Connect to WiFi

Restart NetworkManager to detect the adapter:

```bash
sudo systemctl restart NetworkManager
```

Then use the system tray/network icon to select your network, or use `nmcli`:

```bash
nmcli device wifi list
nmcli device wifi connect "YOUR_WIFI_NETWORK" password "YOUR_PASSWORD"
```

#### Summary Notes

- Kernel headers must match your running kernel
- `rtw89` and other Realtek drivers will **not** work for this USB device
- `morrownr/rtl8852bu` (88x2bu) is actively patched for modern kernels
- DKMS isn't strictly necessary — module builds directly and works immediately
- Power-saving tweaks are critical for USB WiFi stability

---

## Utilities

### Oberon-Governor

https://gitlab.com/TuxThePenguin0/oberon-governor

This repo has an install script:
https://github.com/pnbarbeito/bc250-arch

---

## Gaming

You can install Steam via the UI - there's an Add/Remove Software app where you can install Steam from.

### Gaming performance

Cyberpunk 2077 has playable framerate:

| Metric | Value |
|--------|-------|
| Average FPS | 85.94 |
| Min FPS | 70.85 |
| Max FPS | 105.02 |
| Resolution | 1920x800 |
| Texture Quality | Medium |
| Windowed Mode | Windowed Borderless |
| AMD FSR 3.1 Frame Generation | Yes |
| AMD FidelityFX Super Resolution 3.0 | Auto |
| AMD FSR 3.0 Sharpness | 0.50 |
| Ray Tracing Enabled | Yes |
| Ray-Traced Local Shadows | Yes |

---

## AI

### Vulkan (Manjaro Linux)

```bash
sudo pacman -S amdvlk spirv-tools vulkan-headers vulkan-radeon vulkan-tools vkd3d lib32-vulkan-radeon
```

### Llama.cpp for Vulkan

```bash
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release
```

---

## Sensors

Here are the exact shell commands you need, plus the config file entries:

### 1. Load the driver immediately

```bash
sudo modprobe nct6683 force=true
```

### 2. Make the `force=true` option persistent

Create (or edit) `/etc/modprobe.d/sensors.conf`:

```bash
sudo nano /etc/modprobe.d/sensors.conf
```

Add:

```
options nct6683 force=true
```

### 3. Ensure the module loads at boot

Create (or edit) `/etc/modules-load.d/99-sensors.conf`:

```bash
sudo nano /etc/modules-load.d/99-sensors.conf
```

Add:

```
nct6683
```

### 4. Regenerate initramfs

For Arch-based systems:

```bash
sudo mkinitcpio -P
```

### 5. After reboot, verify with:

```bash
sensors
```

### 6. Use it with psensor

Install psensor for a graphical interface to monitor temperatures.

---

## Useful links

- https://github.com/mothenjoyer69/bc250-documentation
- https://github.com/kenavru/BC-250
- https://gitlab.com/TuxThePenguin0/bc250-bios
- https://gitlab.com/TuxThePenguin0/oberon-governor
- https://github.com/pnbarbeito/bc250-arch
- https://github.com/morrownr/rtl8852bu-20240418 - WiFi driver for TP-Link Archer TX10UB Nano
- https://github.com/csabakecskemeti/rtl8852bu-20240418 - WiFi driver mirror
