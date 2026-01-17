# Intel NUC WinRing0 Vulnerable Driver - Permanent Removal Guide

Windows Defender flags `OpenHardwareMonitorLib.sys` (WinRing0 driver) as **VulnerableDriver:WinNT/Winring0** due to [CVE-2020-14979](https://nvd.nist.gov/vuln/detail/CVE-2020-14979). This driver is bundled with Intel's NUC Software Studio service and gets reinstalled via Windows Update.

This repository provides a permanent solution.

## The Problem

1. Intel's `performancedriverextension.inf` installs `NucSoftwareStudioService`
2. The service deploys `OpenHardwareMonitorLib.sys` (WinRing0 kernel driver)
3. Windows Defender detects it as a vulnerable driver (CVE-2020-14979)
4. Windows Update `Intel Corporation - Extension - 1.0.0.38` keeps reinstalling it
5. Standard removal methods don't prevent reinstallation

## Root Cause

The **Intel NUC Performance Driver** (`ACPI\INTC1036`) is an ACPI device that exposes NUC-specific features:
- Fan control (software-adjustable curves)
- LED control (RGB/power LED customization)
- Thermal monitoring
- Power management

The vulnerable extension hooks into this base driver. Windows Update sees the device and offers driver updates for it.

## Solutions (Choose One)

### Option A: Permanently Disable the Device (Recommended)

This disables the ACPI device entirely via Group Policy, preventing any drivers from loading.

**Trade-offs:**
- Lose software fan/LED control (BIOS defaults still work)
- Most secure - eliminates the attack surface entirely

Run `scripts/disable-device-gpo.ps1` as Administrator, or manually:

```powershell
# Run as Administrator
.\scripts\disable-device-gpo.ps1
```

### Option B: Block Driver Updates Only

Keeps the base driver but blocks all Windows Update driver installations for this device.

**Trade-offs:**
- Device remains functional with current base driver
- Won't receive future driver updates (security or otherwise)

Run `scripts/block-driver-updates.ps1` as Administrator.

### Option C: Manual Cleanup + Hide Update

Less permanent - requires re-hiding if Intel releases new versions.

Run `scripts/remove-and-hide.ps1` as Administrator.

## Quick Diagnosis

Check your current status:

```powershell
.\scripts\diagnose.ps1
```

## File Locations Reference

| Component | Path |
|-----------|------|
| Service executable | `C:\WINDOWS\System32\DriverStore\FileRepository\performancedriverextension.inf_amd64_*\Service\NucSoftwareStudioService.exe` |
| Vulnerable driver | `...\Service\OpenHardwareMonitorLib.sys` |
| Base driver inf | `oem*.inf` (PerformanceDriver) |
| Extension inf | `performancedriverextension.inf` |

## Verification

After applying any solution, verify:

```powershell
# Check service is gone
sc.exe query NucSoftwareStudioService

# Check no pending Intel extension updates
# (Run scripts/diagnose.ps1 for full status)
```

## Reverting Changes

### To re-enable the device:
```powershell
# Remove the Group Policy block
Remove-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeviceInstall\Restrictions" -Recurse -Force

# Re-enable the device
$device = Get-PnpDevice | Where-Object { $_.InstanceId -like '*INTC1036*' }
Enable-PnpDevice -InstanceId $device.InstanceId -Confirm:$false
```

### To unblock driver updates:
```powershell
Remove-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeviceInstall\Restrictions" -Recurse -Force
```

## References

- [CVE-2020-14979 - WinRing0 Privilege Escalation](https://nvd.nist.gov/vuln/detail/CVE-2020-14979)
- [Microsoft Vulnerable Driver Blocklist](https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/microsoft-recommended-driver-block-rules)

## Contributing

If you have additional findings or improvements, please open an issue or PR.

## License

MIT License - Use at your own risk. Always verify scripts before running as Administrator.
