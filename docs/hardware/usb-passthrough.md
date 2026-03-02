# USB & Target Boards

Alloy supports passing USB devices (debuggers, development boards, programmers) from your host machine into the VM. You can flash firmware and run a debug session without leaving your development environment.

---

## Prerequisites

=== "VirtualBox (macOS / Linux)"

USB passthrough is built into VirtualBox. No additional setup needed.

Make sure you're in the `vboxusers` group on Linux:

```bash
sudo usermod -aG vboxusers $USER
# log out and back in for the change to take effect
```

=== "WSL2 (Windows)"

WSL2 requires **usbipd-win** to share USB devices with WSL:

```powershell
winget install usbipd
```

After installing, reboot your machine.

---

## List connected USB devices

Connect your board or debugger, then:

```bash
alloy-host usb list
```

```
ID    DEVICE                              STATUS
1     J-Link Ultra+ (SEGGER)              available
2     STM32 STLink                        available
3     CP2102 USB to UART Bridge           available
```

---

## Attach a device to the VM

```bash
alloy-host usb attach arm-dev --id 1
```

The device is now accessible inside the VM. Verify from inside the VM:

```bash
alloy-host ssh arm-dev -- lsusb
```

```
Bus 002 Device 003: ID 1366:0105 SEGGER J-Link Ultra+
```

---

## Flash firmware

Once the device is attached, use your normal flash tool from inside the VM:

```bash
alloy-host ssh arm-dev
# inside the VM:
JLinkExe -device nRF9160 -if SWD -speed 4000 -commandfile flash.jlink
# or with OpenOCD:
openocd -f interface/jlink.cfg -f target/nrf9160.cfg -c "program firmware.hex verify reset exit"
# or with pyocd:
pyocd flash --target nrf9160 firmware.bin
```

The device behaves exactly as it would if you were running these tools directly on your host machine.

---

## Detach a device

```bash
alloy-host usb detach arm-dev --id 1
```

The device is released back to the host and is available to other applications.

---

## Check attachment status

```bash
alloy-host usb status arm-dev
```

```
ID    DEVICE                              STATUS
1     J-Link Ultra+ (SEGGER)              attached → arm-dev
2     STM32 STLink                        available
```

---

## Troubleshooting

**Device shows as "available" but `lsusb` doesn't show it inside the VM**

Try detaching and re-attaching:
```bash
alloy-host usb detach arm-dev --id 1
alloy-host usb attach arm-dev --id 1
```

**On Linux: permission denied accessing the device**

Your blueprint should include udev rules for your debugger. Add a task to install the appropriate rules file:

```yaml
  - name: Install J-Link udev rules
    get_file:
      url: "https://raw.githubusercontent.com/your-org/configs/main/99-jlink.rules"
      sha256: "..."
      dest: /etc/udev/rules.d/99-jlink.rules
    run_command:
      command: udevadm control --reload-rules && udevadm trigger
```

**On Windows (WSL2): device not showing up**

Run this in PowerShell (as administrator):
```powershell
usbipd list
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

Then run `alloy-host usb attach` from WSL.
