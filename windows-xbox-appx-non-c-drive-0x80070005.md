# Windows Xbox/AppX Cannot Install Store Apps To Non-C Drive

## TL;DR

If Xbox App, Microsoft Store, Windows Settings, and `Add-AppxVolume` all fail when setting a non-C drive as an app/game install target, do not only chase the `WindowsApps` folder.

In this case, the real root cause was broken permissions on:

```text
<drive>:\System Volume Information
```

The affected drives had permissions only for the interactive user:

```text
G:\System Volume Information PORT\a9804:(F)
```

They were missing the required system account:

```text
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

Because AppX, Store, Xbox, Windows Update, and volume services need SYSTEM access to volume metadata, Windows refused to register the drive as an AppX volume.

The fix for the target drive was:

```powershell
icacls "G:\System Volume Information" /grant "SYSTEM:(OI)(CI)(F)"
```

After that, retry:

```powershell
Add-AppxVolume -Path "G:\WindowsApps"
Get-AppxVolume
```

## Symptoms

Observed failures:

- Xbox App could only install to C: reliably.
- Choosing a non-C drive, such as G:, failed with Xbox error:

```text
0x80070002
```

- Windows Settings > System > Storage > Where new content is saved failed when changing "New apps will save to" away from C::

```text
We couldn't set your default save location.
The error code is 0x80070005.
```

- Administrator PowerShell failed:

```powershell
Add-AppxVolume -Path G:
Add-AppxVolume -Path "G:\WindowsApps"
```

with:

```text
Access is denied. (Exception from HRESULT: 0x80070005 (E_ACCESSDENIED))
FullyQualifiedErrorId : DeploymentError,Microsoft.Windows.Appx.PackageManager.Commands.AddAppxVolumeCommand
```

- `Get-AppxVolume` only showed C::

```text
PackageStorePath
----------------
C:\Program Files\WindowsApps
```

## Misleading Clues

Several things looked suspicious but were not the final root cause:

- Stale Xbox game root paths, such as `F:\XboxGames`.
- `G:\.GamingRoot` pointing to `MS` instead of `XboxGames`.
- Empty or manually-created `WindowsApps` folders.
- Failed `.xvs` placeholder folders under `G:\XboxGames`.
- Xbox App cache and WebView state.
- DISM hanging at `62.3%`.

Some of these were real cleanup items, but they did not explain why Windows Settings and `Add-AppxVolume` both returned `0x80070005`.

## Important Path Details

Xbox App drive selection is influenced by:

```text
<drive>:\.GamingRoot
```

This file is a small hidden binary file that stores the game root folder name.

Example content for `G:\XboxGames`:

```text
RGBX....X.b.o.x.G.a.m.e.s...
```

If `.GamingRoot` points to `MS`, Xbox App will show:

```text
G:\MS
```

If it points to `XboxGames`, Xbox App will show:

```text
G:\XboxGames
```

Fixing `.GamingRoot` can fix the UI path, but it does not fix AppX volume registration if `System Volume Information` permissions are broken.

## Correct AppX Volume Command

Use the full package store path, not just the drive letter:

```powershell
Add-AppxVolume -Path "G:\WindowsApps"
Get-AppxVolume
```

Using only this may still start an operation, but the intended AppX package store path is:

```text
<drive>:\WindowsApps
```

## Diagnosis Checklist

Run these in an elevated PowerShell or Command Prompt when possible.

### 1. Check AppX volumes

```powershell
Get-AppxVolume | Select-Object Name,PackageStorePath,IsOffline,IsSystemVolume
```

If only C: appears, non-C Store/Xbox installs are not registered.

### 2. Check drive format and type

```powershell
[System.IO.DriveInfo]::GetDrives() |
  Where-Object { $_.Name -in @("C:\","D:\","E:\","F:\","G:\") } |
  Select-Object Name,DriveType,DriveFormat,IsReady,AvailableFreeSpace,TotalSize,VolumeLabel
```

Expected for Xbox/AppX game installs:

```text
DriveType   Fixed
DriveFormat NTFS
IsReady     True
```

### 3. Check root ACL

```powershell
icacls "G:\"
```

Root ACL differences can matter, but in this case root ACL was not the main blocker.

### 4. Check System Volume Information ACL

This was the decisive check:

```powershell
icacls "G:\System Volume Information"
```

Broken result:

```text
G:\System Volume Information PORT\a9804:(F)
                             PORT\a9804:(OI)(IO)(F)
                             PORT\a9804:(CI)(F)
```

Missing:

```text
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

Good enough fix:

```powershell
icacls "G:\System Volume Information" /grant "SYSTEM:(OI)(CI)(F)"
```

Then verify:

```powershell
icacls "G:\System Volume Information"
```

Expected to include:

```text
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

## Fix Procedure

### 1. Close Xbox App

Close Xbox App and stop related user processes if needed:

```powershell
Get-Process XboxPcApp,XboxPcAppFT,XboxPcTray,XboxGameBarWidgets -ErrorAction SilentlyContinue |
  Stop-Process -Force
```

### 2. Fix the target drive metadata permission

For G::

```powershell
icacls "G:\System Volume Information" /grant "SYSTEM:(OI)(CI)(F)"
```

For multiple affected drives:

```powershell
foreach ($d in "D","E","F","G") {
  icacls "$d`:\System Volume Information" /grant "SYSTEM:(OI)(CI)(F)"
}
```

### 3. Ensure the AppX store path exists

```powershell
if (-not (Test-Path "G:\WindowsApps")) {
  New-Item -ItemType Directory -Path "G:\WindowsApps" | Out-Null
}
attrib +h "G:\WindowsApps"
```

### 4. Register the AppX volume

Run in Administrator PowerShell:

```powershell
Add-AppxVolume -Path "G:\WindowsApps"
Get-AppxVolume
```

Success means `Get-AppxVolume` shows `G:\WindowsApps`.

### 5. Configure Xbox game root

Make sure:

```text
G:\XboxGames
```

exists, and Xbox App shows `G:\XboxGames`.

If Xbox App still shows an old path like `G:\MS`, check:

```powershell
Format-Hex -LiteralPath "G:\.GamingRoot"
```

If needed, recreate `.GamingRoot` to point to `XboxGames`:

```powershell
$root = "G:\.GamingRoot"
attrib -h $root
$bytes = New-Object System.Collections.Generic.List[byte]
$bytes.AddRange([byte[]](0x52,0x47,0x42,0x58,0x01,0x00,0x00,0x00))
$bytes.AddRange([Text.Encoding]::Unicode.GetBytes("XboxGames"))
$bytes.AddRange([byte[]](0x00,0x00))
[IO.File]::WriteAllBytes($root, $bytes.ToArray())
attrib +h $root
```

### 6. Retry Xbox install

Cancel the old failed queue item. Do not use "Restart install" on an old broken session.

Start a fresh install and choose:

```text
G:\XboxGames
```

## Cleanup Notes

Failed Xbox attempts may leave tiny `.xvs` placeholder folders:

```text
G:\XboxGames\<GUID>\<GUID>.xvs
```

If the install queue is cancelled and Xbox is closed, these can be renamed out of the way instead of deleted:

```powershell
Get-ChildItem -LiteralPath "G:\XboxGames" -Force -Directory |
  Where-Object { $_.Name -match "^[0-9A-Fa-f-]{36}$" } |
  ForEach-Object {
    Move-Item -LiteralPath $_.FullName -Destination ("G:\XboxGames\{0}.failed-20260515" -f $_.Name)
  }
```

## What Not To Do

Avoid these as first-line fixes:

- Do not recursively reset the whole drive ACL.
- Do not delete `WindowsApps` or `System Volume Information` blindly.
- Do not keep clicking "Restart install" in Xbox App after a failed session.
- Do not assume this is only an Xbox cache issue if Windows Settings also returns `0x80070005`.
- Do not run multiple DISM sessions at the same time.

## Why This Broke Multiple Drives

All non-C data drives had the same bad pattern:

```text
<drive>:\System Volume Information PORT\a9804:(F)
```

That suggests the drives were previously modified by a permission tool, ownership takeover, migration, backup/restore workflow, or manual ACL reset that replaced the normal system ACL with a user-owned ACL.

The key point is that Windows services do not run as the interactive user. They need `SYSTEM` access.

## Final Root Cause

The non-C drives were not inherently unreadable and were not the wrong filesystem.

They were:

```text
Fixed + NTFS + ready
```

The real blocker was:

```text
Missing SYSTEM full-control permission on <drive>:\System Volume Information
```

That caused Windows AppX deployment and Store default-save-location changes to fail with:

```text
0x80070005 E_ACCESSDENIED
```

Xbox App then surfaced secondary errors such as:

```text
0x80070002
```

because it could create the game root placeholder but could not complete the underlying AppX/Gaming Services install setup.

