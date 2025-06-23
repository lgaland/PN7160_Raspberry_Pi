# Porting PN7160 to Raspberry Pi

This project aims to interface NXP's NCI-based NFC controller PN7160 with a Raspberry Pi, enabling the use of NXP’s Linux NFC stack. The process involves three main steps:

1. Installing the PN7160 I2C kernel driver  
2. Editing the device tree to interface the driver with the hardware  
3. Installing NXP’s Linux NFC stack  

---

## 🧰 Hardware Setup

- Raspberry Pi 5 Model B Rev 1.1 (BCM2712, HW revision d04171)  
- NXP PN7160 I2C Development Board: OM27160A1HN  
- NXP Raspberry Pi Interface Board: OM29110RPI-B  
- NFC Forum–compliant NFC tag

<figure style="text-align: center;">
  <img src="https://raw.githubusercontent.com/lgaland/PN7160_Raspberry_Pi/refs/heads/main/assets/pi.png" width="300">
  <figcaption>Hardware setup: Raspberry Pi 5 + OM29110RPI-B + OM27160A1HN</figcaption>
</figure>

> **Note:** The host used here is a Raspberry Pi 5, but the code can be easily adapted for other platforms.

---

## 💻 Software Setup

- **OS**: Raspberry Pi OS (Debian GNU/Linux 12) - 64-bit
- **Kernel**: Version 6.12.33-v8-16k+  

Open a shell on the Pi and clone the repository:

```bash
git clone https://github.com/lgaland/PN7160_Raspberry_Pi.git
```

---

## 🔧 Step 1: Install the PN7160 I2C Kernel Driver

NXP’s Linux NFC stack interfaces with the PN7160 via an I2C kernel driver.

Clone the Raspberry Pi Linux kernel sources:

```bash
git clone --depth=1 https://github.com/raspberrypi/linux.git -b rpi-6.12
```

Copy the `pn71xx` I2C kernel driver sources to the appropriate directory:

```bash
cp -r ./PN7160_Raspberry_Pi/pn71xx ./linux/drivers/nfc
```

Edit the build system:

- In `./linux/drivers/nfc/Kconfig`, add before `endmenu`:

  ```bash
  source "drivers/nfc/pn71xx/Kconfig"
  ```

- In `./linux/drivers/nfc/Makefile`, add:

  ```bash
  obj-y += pn71xx/
  ```

### 🧱 Build the Kernel

First, install the build dependencies:

```bash
sudo apt install bc bison flex libssl-dev make
```

> Depending on your Raspberry Pi model, some steps may vary. Refer to the [official guide](https://www.raspberrypi.com/documentation/computers/linux_kernel.html) for details.

```bash
cd linux
KERNEL=kernel_2712
make bcm2712_defconfig
make menuconfig
```

Navigate to:

```
Networking support
 └──> NFC subsystem support
      └──> Near Field Communication (NFC) devices
```

Enable:

```
<NFC PN71XX device support (I2C)>
```

Save and exit menuconfig.

Then to build and install the kernel, in `./linux` create a bash script named `rebuild.sh` with this content:

```bash
#!/bin/bash

set -e

KERNEL=kernel_2712

# Rebuild Linux kernel
make -j6 Image.gz modules dtbs

# Install kernel modules
sudo make -j6 modules_install

# Copy new kernel into place
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image.gz /boot/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/overlays/
```

Now execute it:

```bash
./rebuild.sh
```

> ⚠️ Kernel compilation may take some time on the Raspberry Pi.

---

## 🌲 Step 2: Device Tree Configuration

We’ll bind the I2C interface and necessary GPIOs using a device tree overlay.

When mounted on the OM29110RPI-B interface board, the OM27160A1HN uses:

- I2C interface: `i2c1`
- GPIO23 → IRQ (input)  
- GPIO24 → VEN (output)  
- GPIO25 → DWL_REQ (output)

### 📦 Create the Overlay

Install the Device Tree Compiler:

```bash
sudo apt update
sudo apt install device-tree-compiler
```

Compile the overlay:

```bash
cd ../PN7160_Raspberry_Pi/dts
dtc -I dts -O dtb -o i2c1-pn71xxa.dtbo i2c1-pn71xxa.dts
```

Copy the blob to the overlays directory:

```bash
sudo cp i2c1-pn71xxa.dtbo /boot/firmware/overlays/
```

### 🔧 Enable the Overlay

Edit `/boot/firmware/config.txt` and append under `[all]`:

```ini
dtoverlay=i2c1-pn71xxa
```

Also ensure this line is not commented:

```ini
dtparam=i2c_arm=on
```

### 🔐 Device Node Permissions

By default, the `/dev/pn71xx` node is only accessible to root. You can change this using a `udev` rule:

Create `/etc/udev/rules.d/pn71xx.rules` an append this line:

```bash
ACTION=="add", KERNEL=="pn71xx", MODE="0666"
```

Reboot:

```bash
sudo reboot
```

### 🧪 Testing

Check if the driver loaded correctly:

```bash
dmesg | grep -i nfc
```

Expected output:

```
[    2.811942] nfc: nfc_i2c_dev_init: Loading NXP NFC I2C driver
[    2.812010] nfc: nfc_parse_dt: irq 592
[    2.812015] nfc: nfc_parse_dt: 592, 593, 594
[    2.812153] nfc: nfc_i2c_dev_probe: requesting IRQ 186
[    2.842502] nfc: nfc_i2c_dev_probe: probing nfc i2c successfully
```

If you see something like the above—congratulations! The hardware is now correctly interfaced and the correponding device file is `/dev/pn71xx`.

---

## 📚 Step 3: Install NXP’s Linux NFC Stack

Now install `libnfc-nci`, NXP’s Linux NFC stack.

Refer to **Chapter 5** of [AN13287 – PN7160 Linux Porting Guide (PDF)](https://www.nxp.com/docs/en/application-note/AN13287.pdf).

### 📦 Demos and Tools

- [Examples for libnfc-nci](https://github.com/NXPNFCLinux/linux_libnfc-nci_examples)  
- [Factory test tool](https://github.com/NXPNFCLinux/linux_NfcFactoryTestApp)

---

## 🔗 Useful Links

Those links helped me, maybe they will help you too:

### Application notes and manuals
- [AN13287 – PN7160 Linux Porting Guide (PDF)](https://www.nxp.com/docs/en/application-note/AN13287.pdf)
- [AN11697  – PN71xx Linux Software Stack Integration Guidelines (PDF)](https://www.nxp.com/docs/en/application-note/AN13287.pdf)
- [UM11496 – PN7160 evaluation board (PDF)](https://www.nxp.com/docs/en/user-manual/UM11496.pdf)
- [UM10956 – OM29110 NFC's SBC interface boards (PDF)](https://www.nxp.com/docs/en/user-manual/UM11496.pdf)

### Git repos
- [nxpnfc kernel driver](https://github.com/NXPNFCLinux/nxpnfc)  
- [linux_libnfc-nci](https://github.com/NXPNFCLinux/linux_libnfc-nci)  
- [linux_NfcFactoryTestApp](https://github.com/NXPNFCLinux/linux_NfcFactoryTestApp)
- [linux_libnfc-nci_examples](https://github.com/NXPNFCLinux/linux_libnfc-nci_examples)
### Others
- [Jeff Geerling - How to Recompile Linux (on a Raspberry Pi)](https://www.jeffgeerling.com/blog/2025/how-recompile-linux-on-raspberry-pi)
