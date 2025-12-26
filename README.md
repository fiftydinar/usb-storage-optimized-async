# usb-storage-optimized-async

Udev rule and systemd service to solve the problem of USB storage devices incomplete data sync during ejecting on Linux.

## Problem

0. You're on a Linux PC
1. You write some data to the USB storage device,  
   for example, you write Windows ISO on it, so you can install it on another PC later
2. You eject the USB storage device once the ISO writing apparently gets finished
3. You plug the USB storage device on another PC to install Windows on it
4. You notice that Windows doesn't want to load and that data on USB storage device is corrupted

This happens often on Linux and is one of the most frustrating problems that users can face.  
This is something that Windows doesn't have an issue with.

## Cause

On Linux, USB storage devices use `async` mount option by default, which enables USB writing cache on RAM.  
This speeds up the operation of writing the data to the USB storage device considerably compared to `sync`.  
It's also more efficient in writing the data, writing less data on the storage compared to `sync`.  
However, this is not tuned properly by default, which makes the system incorrectly report that data is fully written too early,  
while some of the data is still in the USB writing cache.

As a comparison, Windows enables behavior similar to `sync` by default for USB storage devices.

## Solution

### Non-ideal solutions
1. Always manually clicking "Unmount" on USB storage device, waiting for it to finish and then ejecting it.  
   This will always guarantee that all the data will be flushed to the USB storage device,  
   but this is not as user-friendly as simply ejecting the USB drive.
2. Using `sync` as a mount option for USB storage device.  
   This disables the USB writing cache and solves this problem.  
   However, data writing will be much slower and it will increase the tear on USB storage device.
   
### More ideal solution
1. Using this udev rule and systemd service to optimize the `async` mount option, without sacrificing USB storage speed and its write cycles.

## How it works

### Udev rule

1. Launch the `usb-storage-optimized-async-udev` script when USB storage device is detected and supply it with USB device block, USB vendor id and USB model id values
2. It determines the USB port speed of the USB storage device that is plugged into it using `lsusb -t -v` output  
   (systemd's udev has an option to supply `ATTRS{speed}` for this, but I found it to be inaccurate in my testing)
3. Do the calculation of the ideal BDI `max_bytes` value for the USB writing cache on the USB storage device based on:  
  - USB port speed (in MB)  
  - buffer time (0.05)  
  - safety factor (1.3)  
4. Calculation looks like this:  
   ( `USB port speed` / 8 ) * `buffer time` * `safety factor` * 1024 * 1024
5. Enable the data write limit (BDI `strict_limit`) and apply the calculated value to BDI `max_bytes` file

By lowering and limiting the `max_bytes` BDI value, we assure that USB storage device won't overuse the USB writing cache.

### Systemd service
It does the same as the `udev` rule, with the difference that it only applies once on boot,  
as `udev` rule can trigger too early during the system boot when USB storage device is already plugged in for it to work.

## Requirements
- Linux 6.1+ kernel  
  - `max_bytes` BDI value is only available since then.
  - https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-class-bdi
- `/bin/sh` POSIX shell
- `udev`
  - only tested on SystemD's `udev`
- `lsusb`
- `gawk`
- `bc`
- Script autostarter on boot (`init` system like SystemD or something else)
  - for applying the service script fix during later stage of boot, when USB storage device is already plugged in before and during the boot, where `udev` triggers too early to work

## Known quirk
- If 2 or more USB storage devices have the same USB vendor ID and USB model ID, then this fix won't apply for those USB storage devices.  
This doesn't happen often, but it's worth to note.  
`lsusb` output doesn't show the information about the USB storage block device, hence why we can't use anything else as an identifier.

## Credits
To clarify, I am not the one who discovered this, I'm the one who just refined it to work reliably.

Linux Manjaro uses the different and simpler script compared to mine, which I tested and found it to not work reliably,  
however, the maintainer couldn't reproduce the issue.  
https://gitlab.manjaro.org/applications/udev-usb-sync

- Linux Manjaro user Megavolt who discovered this method and explained it well with benchmarks.  
  - https://forum.manjaro.org/t/strict-limit-of-write-cache-0s-sync-time-policy-for-usb-devices-by-default/166934
- Linux Manjaro and `udev-usb-sync` script maintainer Frede H who patiently reviewed and answered my emails regarding the script.
