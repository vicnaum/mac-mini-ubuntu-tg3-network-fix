# Troubleshooting tg3 Network Issues on Mac Mini

(aka MacMini 2014 Ubuntu tg3 Network Fix)

This guide addresses the common network connectivity issues with Broadcom Tigon3 (tg3) network cards on Mac Mini hardware running Linux, particularly Ubuntu. The main symptom is periodic network drops or complete network loss, often accompanied by the error: `tg3_stop_block timed out`.

## Hardware Details

Tested on:

- Mac Mini (2014)
- Network Card: Broadcom NetXtreme BCM57766 Gigabit Ethernet PCIe (rev 01)
- Ubuntu with kernel 6.8.0-52-generic
- Network interface name: enp3s0f0 (renamed from eth0)

## Symptoms

- Periodic network disconnections
- Complete network loss
- Router doesn't see the device
- System logs showing `tg3_stop_block timed out` errors
- Network interface becoming unresponsive
- Zero values in NAPI info blocks
- Errors in logs like:
  ```
  [2662296.422419] tg3 0000:03:00.0: enp3s0f0: Host status block [0x00000000, 0x00000000] (type:0, ver:0)
  [2662296.422440] tg3 0000:03:00.0: enp3s0f0: NAPI info [00000000:00000000 (0000), 00000000:00000000 (0000)]
  [2662296.554259] tg3 0000:03:00.0: tg3_stop_block timed out, ofs=4c00 enable_bit=2
  ```

## Root Causes

### 1. IOMMU-Related Issues

- Modern kernels (6.8+) have IOMMU enabled by default
- Incompatibility between Apple hardware's firmware/BIOS and IOMMU
- DMA buffer mapping/unmapping issues through IOMMU
- Intel VT-d (hardware virtualization) conflicts
- IOMMU feature inconsistencies on Mac Mini hardware:
  ```
  DMAR: IOMMU feature pgsel_inv inconsistent
  DMAR: IOMMU feature sc_support inconsistent
  DMAR: IOMMU feature pass_through inconsistent
  ```

### Understanding IOMMU

IOMMU (Input-Output Memory Management Unit) is a hardware component that manages how devices access system memory. It acts as a security barrier by:

- Translating device-visible virtual addresses to physical addresses
- Protecting system memory from unauthorized DMA (Direct Memory Access) operations
- Providing memory isolation for virtualization
- Acting similar to how MMU (Memory Management Unit) isolates process memory

#### Security Implications of Disabling IOMMU

Disabling IOMMU does come with some security trade-offs:

1. **What You Lose**:

   - Hardware-enforced isolation against DMA attacks
   - Some virtualization security features
   - Protection against misbehaving hardware

2. **Risk Assessment**:

   - For a server with only network exposure (no untrusted physical access):
     - Risk is minimal as IOMMU mainly protects against local DMA attacks
     - Network-based attacks are not affected by IOMMU status
   - Higher risk scenarios (where you might want to keep IOMMU enabled):
     - Systems with untrusted PCI/USB devices
     - Virtualization servers requiring strong isolation
     - Environments with physical security concerns
     - Systems processing sensitive data with strict security requirements

3. **Recommendation**:
   - For a typical home server (like this Mac Mini use case):
     - The stability benefits outweigh the security trade-offs
     - Physical access is controlled
     - No untrusted hardware is connected
     - Network security remains unaffected

## Solutions (In Order of Recommendation)

### 1. Disable IOMMU (Most Common Fix)

This solution bypasses the problematic DMA translation that often causes the tg3 driver to hang. Note that while both options below are possible, our testing showed that complete IOMMU disable works better than pass-through mode on Mac Mini hardware.

```bash
# Edit GRUB configuration
sudo nano /etc/default/grub

# Add to GRUB_CMDLINE_LINUX_DEFAULT:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=off"

# Update GRUB
sudo update-grub

# Reboot
sudo reboot
```

A successful disable should show these messages in logs:

```bash
$ dmesg | grep -i iommu
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-6.8.0-52-generic ... intel_iommu=off
[    0.050185] DMAR: IOMMU disabled
```

Note: We initially tried pass-through mode (`intel_iommu=on iommu=pt`) but it didn't resolve the inconsistency warnings on Mac Mini hardware. Complete disable proved more effective.

## Verifying the Fix

After applying the solution:

1. Immediate verification:

```bash
# Check if IOMMU is properly disabled
dmesg | grep -i iommu

# Check tg3 driver status
dmesg | grep tg3
```

You should see:

- IOMMU disabled message
- Clean tg3 initialization without timeout errors
- Network link up at 1000 Mbps
- No error messages about tg3_stop_block

2. Long-term verification (24-48 hours):

```bash
# Check for any tg3 errors since last boot
journalctl -b | grep tg3

# Check current network interface status
ip link show enp3s0f0
```

Success indicators:

- No `tg3_stop_block timed out` errors in logs
- No network disconnections
- Router consistently sees the device
- Stable network performance

3. Network stability test:

```bash
# Run a long ping test
ping -i 2 8.8.8.8 | while read line; do echo "$(date): $line"; done
```

## Additional Resources

- [Kernel Bugzilla - tg3 Issues](https://bugzilla.kernel.org/show_bug.cgi?id=12877)
- [Ubuntu Forums - Mac Mini Network Issues](https://askubuntu.com/questions/260335/)
- [Proxmox Forum - Network Issues](https://forum.proxmox.com/threads/network-issues-after-updating.150523/)

## Contributing

This guide is based on real-world testing with a Mac Mini 2014 running Ubuntu with kernel 6.8.0-52-generic. If you have additional insights, different hardware configurations, or alternative solutions, please feel free to contribute to this guide.
