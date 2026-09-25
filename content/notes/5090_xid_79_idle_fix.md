+++
title = "RTX 5090 Xid 79 on idle fix"
date = 2026-09-24
+++

I run [CachyOS](https://cachyos.org/) btw.

For now it is on kernel 7.2.7-1-cachyos and Nvidia driver 615.71.09

I have RTX 5090 ASUS ROG Astral with Ryzen 9950X3D on ASRock Taichi Lite x870e and the only thing annoying me on this magnificent setup is Xid 79 errors from time to time when the display is idle for 3 or more hours. Xid 79 can happen while idle, or when the display wakes up from idle after a long idle stay.

It looks like eventually I figured out how to fix this. The solution is to disable ASPM, but in the proper way.

I have ASPM disabled in BIOS like this
```
Advanced->AMD PBS->AMD Common Platform Module->PM L1 SS: Disabled
```

But it turns out it only disables L1.1 and L1.2 states of ASPM, but not ASPM itself.

Also, the kernel parameter `pcie_aspm=off` does not disable ASPM, it disables the kernel's ability to manipulate ASPM.
If you use this parameter, you have to remove it.

To disable ASPM, use the kernel boot parameter `pcie_aspm.policy=performance`

If your GPU is PCI device 01:00.0, you can check ASPM state with this command
```
sudo lspci -vvv -s 01:00.0 | grep -A3 LnkCtl
```

We are going to achieve a result like this
```
LnkCtl: ASPM Disabled; RCB 64 bytes, LnkDisable- CommClk+
    ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt- FltModeDis-
```

### Bonus: things that could be connected to stability I achieved

I disabled Dynamic Power Management in nvidia kernel module parameters

Create the file
```
sudo nvim /etc/modprobe.d/nvidia.conf
```

With this content
```
options nvidia NVreg_DynamicPowerManagement=0x00
```

Regenerate initramfs image
```
sudo mkinitcpio -P
```

After reboot check that it was applied with this command
```
cat /proc/driver/nvidia/params | grep -i DynamicPowerManagement
```

And undervolt GPU with [nvoc](https://github.com/martinstark/nvoc)

You can install it from AUR `paru -S nvoc-cli`

I run `sudo nvoc -c 800,2900 -o 1000 -m 2000 -p 100` and got almost 100W less under heavy load compared to without it.

Check stability with [gpu-burn](https://github.com/wilicc/gpu-burn)

Monitor GPU in real time
```
watch -n 1 nvidia-smi --query-gpu=clocks.gr,clocks.max.gr,power.draw --format=csv
```

To run it as a systemd service, create the file
```
sudo nvim /etc/systemd/system/nvidia-clocks.service
```

With this content
```
[Unit]
Description=Nvidia GPU clocks
After=systemd-udev-settle.service local-fs.target
Wants=systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/usr/bin/nvoc -c 800,2900 -o 1000 -m 2000 -p 100
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and run it
```
sudo systemctl enable --now nvidia-clocks.service
```
