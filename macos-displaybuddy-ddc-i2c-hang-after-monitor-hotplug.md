# macOS DisplayBuddy Hangs After External Monitor Hot-Plug

## TL;DR

DisplayBuddy became unresponsive whenever an external monitor was connected or reconnected. Updating and reinstalling the app did not fix it.

The process was blocked inside macOS IOKit:

```text
IOAVServiceWriteI2C
  IOConnectCallMethod
    io_connect_method
      mach_msg2_trap
```

The per-display **Software Control** setting was necessary but not sufficient. DisplayBuddy still had its global DDC value-reading feature enabled, so it attempted a DDC Get-VCP transaction during startup and display reconfiguration. A DDC read begins by writing the request over I2C, which explains why the stack stopped at `IOAVServiceWriteI2C`.

The stable configuration was:

- Set every affected external display to `software` control mode.
- Disable unused external-monitor volume controls.
- Disable both current and legacy forms of DisplayBuddy's global DDC read preference:

```bash
defaults write com.sids.DisplayBuddy com.displaybuddy.ddcReadValuesEnabled -bool false
defaults write com.sids.DisplayBuddy com_displaybuddy_ddcReadValuesEnabled -bool false
```

After this change, DisplayBuddy launched with both monitors connected, remained responsive, adjusted software brightness and contrast, and no longer entered any `IOAVServiceReadI2C` or `IOAVServiceWriteI2C` call during process sampling.

## Case Environment

Verified on 2026-07-21:

- Apple Silicon Mac with M4 Pro
- macOS 26.5.2 (`25F84`)
- DisplayBuddy 3.9.1
- Main display: ASUS `XG32UCWMG`
- Secondary display: Acer `XV272U V`

The same I2C failure had also appeared while DisplayBuddy 3.0.1 was installed, so the updater and app version were not the underlying cause.

## Symptoms

- DisplayBuddy showed **No Response** after connecting or reconnecting an external display.
- Clicking the in-app updater could appear to freeze.
- Reinstalling or updating DisplayBuddy did not prevent recurrence.
- DisplayBuddy's CLI timed out:

```text
Error: Timed out waiting for response from DisplayBuddy.
```

- The affected process entered the macOS `U` state and sometimes could not be terminated until the monitor was disconnected:

```text
PID    STATE  COMMAND
17134  U      /Applications/DisplayBuddy.app/Contents/MacOS/DisplayBuddy
```

- Changing brightness or contrast from the monitor's physical OSD could be immediately overwritten after closing the OSD. DisplayBuddy was writing its saved hardware values back through DDC.
- macOS continued to detect the monitors and render video normally. The failure was in the monitor-control path, not the display signal itself.

## Misleading Clues

### The app updater was not the root cause

The first visible freeze occurred around an update, but both the old and updated app versions blocked in the same I2C functions. Reinstalling only replaced the application bundle; it did not remove the DDC trigger in preferences or change the monitor's I2C behavior.

### Per-display Software Control alone was not enough

Setting `XV272U V` to Software Control allowed DisplayBuddy to work while that was the only external display connected.

The problem returned when `XG32UCWMG` was connected. Even after every saved XG32 record was changed to `software`, DisplayBuddy still hung during startup. This proved that another DDC path was running before or outside the per-display control-mode decision.

### `IOAVServiceWriteI2C` did not necessarily mean "restore a value"

A DDC/CI Get-VCP read is a request-response transaction:

1. Write the Get-VCP request over I2C.
2. Read the monitor's response over I2C.

If step 1 blocks, a read operation appears in the stack as `IOAVServiceWriteI2C`. This is why disabling DisplayBuddy's global DDC **read** preference removed a hang observed in a **write** function.

## Evidence and Root-Cause Isolation

### 1. Confirm that macOS still sees the displays

```bash
system_profiler SPDisplaysDataType -json
```

Both external displays reported:

```text
spdisplays_online = spdisplays_yes
```

That separated the healthy video path from the failing DDC control path.

### 2. Check process and CLI responsiveness

```bash
ps -axo pid,state,etime,command | rg '[/]Applications/DisplayBuddy.app'
/Applications/DisplayBuddy.app/Contents/MacOS/displaybuddy-cli list --json --timeout 3
```

Failure pattern:

- Process state: `U`
- CLI result: timeout

### 3. Sample the blocked process

Replace `<PID>` with the DisplayBuddy PID:

```bash
sample <PID> 3 1 -file /tmp/displaybuddy-sample.txt
rg -n -C 12 'IOAVService(Read|Write)I2C' /tmp/displaybuddy-sample.txt
```

During hot-plug, the main thread was blocked from the display-reconfiguration callback:

```text
com.apple.main-thread
  displayConfigFinalizedProc
    DisplayBuddy
      IOAVServiceWriteI2C
        IOConnectCallMethod
          io_connect_method
            mach_msg2_trap
```

During a fresh application launch with both monitors already connected, the main thread reached the same I2C function before launch completed.

This explains the full-app freeze: a synchronous monitor I/O call with no effective timeout was running on the main thread.

### 4. Inspect the DDC preferences

```bash
defaults read com.sids.DisplayBuddy com.displaybuddy.ddcReadValuesEnabled
defaults read com.sids.DisplayBuddy com_displaybuddy_ddcReadValuesEnabled
```

Both keys were enabled:

```text
1
1
```

DisplayBuddy carried both dotted and underscore-prefixed preference names because of a previous defaults migration. Changing only one form risked leaving the other active.

## Final Fix

### 1. Stop DisplayBuddy

Quit DisplayBuddy normally if possible. If the process is already stuck in `U`, send `TERM` and wait briefly:

```bash
kill -TERM <PID>
ps -p <PID> -o pid=,state=,etime=,command=
```

If the process cannot leave the kernel call, disconnect the monitor that triggered the DDC transaction. In this case, the pending process exited after the I2C path was released.

### 2. Back up the preference domain

Do this before editing preferences:

```bash
defaults export com.sids.DisplayBuddy "$HOME/Desktop/DisplayBuddy-preferences-backup.plist"
```

### 3. Set affected displays to Software Control

Use DisplayBuddy's UI or CLI while the app is responsive:

```bash
/Applications/DisplayBuddy.app/Contents/MacOS/displaybuddy-cli set \
  --uuid <DISPLAY-UUID> \
  --control-mode software \
  --json
```

Confirm the result:

```bash
/Applications/DisplayBuddy.app/Contents/MacOS/displaybuddy-cli list --json
```

Expected for each affected monitor:

```json
{
  "controlMode": "software",
  "type": "external"
}
```

Display UUIDs can change with ports, adapters, docks, or display topology. Do not copy UUIDs from an old incident blindly. In this case, DisplayBuddy had accumulated five saved UUID records for the XG32 across different connection paths, so all matching XG32 records were normalized to `software`.

### 4. Remove unused hardware-only controls

Software Control can implement brightness and contrast without DDC, but it cannot provide true hardware volume, input switching, or monitor power control.

Because this setup only needed occasional brightness and contrast changes, XG32 volume control was disabled:

```json
{
  "preferredControlMode": "software",
  "showsBrightnessControl": true,
  "showsContrastControl": true,
  "showsVolumeControl": false
}
```

### 5. Disable global DDC reads

With DisplayBuddy stopped:

```bash
defaults write com.sids.DisplayBuddy com.displaybuddy.ddcReadValuesEnabled -bool false
defaults write com.sids.DisplayBuddy com_displaybuddy_ddcReadValuesEnabled -bool false
```

Verify both values:

```bash
defaults read com.sids.DisplayBuddy com.displaybuddy.ddcReadValuesEnabled
defaults read com.sids.DisplayBuddy com_displaybuddy_ddcReadValuesEnabled
```

Expected:

```text
0
0
```

### 6. Relaunch and verify

```bash
open -a DisplayBuddy
/Applications/DisplayBuddy.app/Contents/MacOS/displaybuddy-cli list --json --timeout 10
ps -axo pid,state,etime,command | rg '[/]Applications/DisplayBuddy.app/Contents/MacOS/DisplayBuddy'
```

Healthy result:

- Both monitors report `controlMode: software`.
- The process state is `S`, not `U`.
- The CLI responds immediately.

Then sample the live process again:

```bash
sample <PID> 3 1 -file /tmp/displaybuddy-after-fix.txt
rg -n 'IOAVService(Read|Write)I2C' /tmp/displaybuddy-after-fix.txt
```

Expected: no matches.

## Functional Verification

The final test changed each software value by one point and restored it:

```text
XG32 brightness: 100 -> 99 -> 100
XG32 contrast:   70  -> 69 -> 70
XV272 brightness: 100 -> 99 -> 100
XV272 contrast:   70  -> 69 -> 70
```

The values read back correctly, the process remained in state `S`, and a three-second sample contained no I2C calls.

## Trade-Offs

This configuration intentionally removes hardware DDC from DisplayBuddy.

Still available:

- Software brightness
- Software contrast
- Keyboard shortcuts for those software controls

No longer available through DisplayBuddy:

- True monitor backlight changes
- Hardware monitor volume
- Input-source switching
- Monitor power commands
- Reading the monitor's current hardware VCP values

The monitor's physical OSD remains available, and DisplayBuddy no longer writes old hardware values back after the OSD closes.

## Rollback

To restore the complete saved preference domain:

```bash
defaults import com.sids.DisplayBuddy "$HOME/Desktop/DisplayBuddy-preferences-backup.plist"
open -a DisplayBuddy
```

Or re-enable only global DDC reads:

```bash
defaults write com.sids.DisplayBuddy com.displaybuddy.ddcReadValuesEnabled -bool true
defaults write com.sids.DisplayBuddy com_displaybuddy_ddcReadValuesEnabled -bool true
```

Re-enabling DDC may reproduce the original uninterruptible I2C hang on this monitor topology.

## Final Root Cause

The failure was not caused by the updater, installation integrity, display detection, or macOS losing the video signal.

The root cause was:

> DisplayBuddy 3.9.1 synchronously attempted a global DDC Get-VCP transaction during startup and display reconfiguration. On this dual-monitor topology, the macOS IOKit call blocked indefinitely in `IOAVServiceWriteI2C`, and DisplayBuddy performed that call on its main thread.

Per-display Software Control did not fully disable this global probe. Disabling both forms of `ddcReadValuesEnabled` removed the remaining I2C path and produced the stable behavior required for this setup.
