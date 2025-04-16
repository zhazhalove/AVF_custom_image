# Run a Custom Linux Image in Android Virtualization Framework (AVF) with USB Attachment

## Description

This guide walks you through building and running a custom ARM64 Linux image using QEMU and deploying it on an Android device using the Android Virtualization Framework (AVF), with support for USB peripheral passthrough.

```
Disclaimer:

This guide is intended for educational and research purposes only. Proceeding with these steps involves unlocking and rooting your Android device, which can void your warranty, render your device inoperable, or expose it to security risks. The author assumes no liability for any damage or data loss that may occur. Use at your own risk.
```

## Dependencies

- Linux development system with QEMU ARM64 packages installed.
- Linux ISO image (Debian is used in this guide).
- [Rooted](https://xdaforums.com/t/guide-unlock-bootloader-root-google-pixel-tablet-with-magisk-adb-fastboot-command-lines.4615841/) Android 14/15 device (Google Pixel Tablet used).
- [F-Droid](https://f-droid.org/en/) installed on the rooted Android device.
- [Termux](https://f-droid.org/en/packages/com.termux/) installed on the rooted Android device.
- USB Debugging enabled in Developer Options.
- [Android SDK Platform Tools](https://developer.android.com/tools/releases/platform-tools) on the development machine.
- [Google USB Driver](https://developer.android.com/studio/run/win-usb)

## Step-by-Step Process

### Build Custom ARM64 Linux Image

```
Note:
In this guide, the debian-12.10.0-arm64 (Bookworm) ISO image is used to build the rootfs. This should work for any ARM64 Linux distro.
```

```
Note:
The Linux development environment is Ubuntu 24.04.2.
```

1. Start your Linux development environment.
2. Confirm required QEMU ARM64 packages are installed:

```
sudo apt update
sudo apt install qemu-system-arm qemu-utils edk2-aarch64
```

3. Create a working directory:

```
mkdir ~/avf_custom_image
cd ~/avf_custom_image
```

4. Download your Linux distro ISO:

```
wget https://cdimage.debian.org/debian-cd/current/arm64/iso-cd/debian-12.10.0-arm64-netinst.iso
```

5. Copy EFI firmware files:

```
sudo cp -p /usr/share/AAVMF/AAVMF_CODE.fd ./AAVMF_CODE.fd
sudo cp -p /usr/share/AAVMF/AAVMF_VARS.fd ./AAVMF_VARS.fd
```

6. Make the VARS file writable:

```
sudo chown $USER:$USER AAVMF_VARS.fd
```

7. Build the rootfs disk:

```
qemu-img create -f raw ./debian-arm64.img 10G
```

8. Boot the custom Linux installer:

```
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 2G \
  -nographic \
  -drive if=pflash,format=raw,readonly=on,file=AAVMF_CODE.fd \
  -drive if=pflash,format=raw,file=AAVMF_VARS.fd \
  -drive file=debian-arm64.img,format=raw,if=virtio \
  -cdrom debian-12.10.0-arm64-netinst.iso \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0 \
  -device virtio-rng-pci \
  -boot order=d
```

9. Manually boot EFI loader from CD-ROM:

![EFI Boot](distro_efi_boot.png)

```
cd FS0:\efi\boot\
ls
bootaa64.efi
```

10. Select non-graphical installation:

![debian_installation_no_graphic](distro_menu_select_install.png)

11. Create an ESP (EFI System Partition):

![EFI boot partition](distro_efi_boot_part.png)

```
Note:
To exit QEMU: Ctrl+A then X
```

### Set Up Android Device Environment

1. Reboot into the custom Linux image:

```
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 2G \
  -nographic \
  -drive if=pflash,format=raw,readonly=on,file=AAVMF_CODE.fd \
  -drive if=pflash,format=raw,file=AAVMF_VARS.fd \
  -drive file=debian-arm64.img,format=raw,if=virtio \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0 \
  -device virtio-rng-pci \
  -device vhost-vsock-pci,guest-cid=42 \
  -boot order=c
```

2. Log in and edit sources list:

```
sudo nano /etc/apt/sources.list
```

Add `contrib non-free non-free-firmware` to each line. Then:

```
Example:

deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian/ bookworm-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ bookworm-updates main contrib non-free non-free-firmware
```

```
sudo apt update
sudo apt install firmware-misc-nonfree
```

3. Install socat:

```
sudo apt install socat
```

4. Add VSOCK systemd service:

```ini
cat <<EOF | sudo tee /etc/systemd/system/vsock-shell.service
[Unit]
Description=Start socat vsock shell
After=network.target

[Service]
ExecStart=/usr/bin/socat VSOCK-LISTEN:8000,reuseaddr,fork EXEC:'/bin/bash -i',pty,stderr,setsid,sigint,sane
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable vsock-shell
sudo systemctl start vsock-shell
```

5. Install networking tools:

```
sudo apt install wireless-tools iw wpasupplicant
```

6. Add fallback EFI boot entries:

```
# may need to add fallback EFI BOOT to ESP parition
# /EFI/BOOT/BOOTAA64.EFI is actually a copy of shimaa64.efi.
# /EFI/BOOT/grubaa64.efi is the real GRUB, intended to be loaded by shim.
# /EFI/BOOT/mmaa64.efi is MokManager, available if shim needs it.

# confirm this mount point exists
df | grep /boot/efi

sudo mkdir /boot/efi/EFI/BOOT
sudo cp -p /boot/efi/EFI/debian/shimaa64.efi /boot/efi/EFI/BOOT/BOOTAA64.EFI
sudo cp -p /boot/efi/EFI/debian/grubaa64.efi /boot/efi/EFI/BOOT/grubaa64.efi
sudo cp -p /boot/efi/EFI/debian/mmaa64.efi /boot/efi/EFI/BOOT/mmaa64.efi
```

7. Power off the VM:

```
sudo poweroff
```

### Transfer Files to Android

1. Connect the Android device via USB-C.
2. Push the image:

```
adb push path_to_custom_linux_image.img /data/local/tmp
```

### Run Custom Linux Image with AVF

1. Launch Termux
2. In Termux, install and configure OpenSSH:

```
pkg update && pkg upgrade
pkg install openssh
passwd
sshd
```

3. In Termux, check IP and username:

```
ifconfig
whoami
```

4. In Termux, install socat:

```
pkg install socat
```

5. From the development computer, SSH into Android:

```
ssh u0_a256@<IP> -p 8022
su
```

6. Run Crosvm:

```
Note:

Update --block path to the correct Linux custom image file.
```

```
VM_SOCKET="/data/local/tmp/crosvm.sock"

/apex/com.android.virt/bin/crosvm run \
    --name debian_vm \
    --disable-sandbox \
    --mem 2048 \
    --bios /apex/com.android.virt/etc/u-boot.bin \
    --serial type=stdout,hardware=serial,num=1 \
    --params "console=ttyS0 root=/dev/vda rw" \
    --block path=/data/local/tmp/debian-bootable.img \
    --vsock cid=42 \
    -s "$VM_SOCKET"
```

7. Wait for the VM login prompt.
8. From the development computer, connect a second SSH session.
```
ssh u0_a256@<IP> -p 8022
su
```
9. Plug in USB WiFi adapter.
10. Find device path:

```
lsusb
```

![lsusb output](lsusb_output.png)

11. Attach USB Wifi adapter to VM:

```

VM_SOCKET="/data/local/tmp/crosvm.sock" # Make sure this matches the VM guest

BUS_NUM="001" # Example

DEV_NUM="002" # Example

/apex/com.android.virt/bin/crosvm usb attach 00:00:00:00 /dev/bus/usb/${BUS_NUM}/${DEV_NUM} ${VM_SOCKET}
```

12. Connect to the VM shell:

```
Note:

The CID value MUST match the running VM guest value.

[CID value]:8000
```

```
/data/data/com.termux/files/usr/bin/socat -,raw,echo=0 VSOCK-CONNECT:42:8000
```

### Configure Wi-Fi in Guest VM

```
ip link
ip link set <interface> up
ip link show <interface>
iwlist <interface> scan
wpa_passphrase 'YourSSID' 'YourPassword' > /etc/wpa_supplicant.conf
wpa_supplicant -B -i <interface> -c /etc/wpa_supplicant.conf
dhclient <interface>
```

![ip_addr_output](ip_addr_output.png)

## Testing

- Only lite IP packet traffic works across the Wifi adapter (ping).
- Moderate IP packet load causes crosvm emulated USB controller to collapse and takes down the guest VM network interface.
- Further research is required.

## Resources

- [[Guide] Unlock Bootloader + Root Google Pixel Tablet with Magisk](https://xdaforums.com/t/guide-unlock-bootloader-root-google-pixel-tablet-with-magisk-adb-fastboot-command-lines.4615841/)
- [Android Virtualization Framework Overview](https://source.android.com/docs/core/virtualization)
- [F-Droid](https://f-droid.org/en/)
- [Termux](https://f-droid.org/en/packages/com.termux/)
- [Book of Crosvm](https://crosvm.dev/book/)

