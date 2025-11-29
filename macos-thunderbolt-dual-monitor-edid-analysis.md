# 🛠️ Fixing macOS Multi-Monitor Bandwidth Issues by Editing Custom EDID  
**Case Study & Troubleshooting Log**

This document summarizes the full debugging journey of resolving a macOS external-monitor bandwidth conflict by manually modifying EDID (Extended Display Identification Data).  
It covers the **problem**, **root cause**, **technical analysis**, and **final working solution**, so it can be referenced later.

---

# 1. 📌 Problem Summary

When connecting:

- **Monitor A (Main)** → 4K 240Hz via DisplayPort on a Thunderbolt 4 Multimedia Pro Dock  
- **Monitor B (Secondary)** → 2K 170Hz via HDMI on the same Thunderbolt 4 Multimedia Pro Dock

According to Apple’s official documentation, the M4 Pro should support **two external 4K 144Hz monitors**.  
However, in this setup, both displays were connected through the same Thunderbolt 4 dock, which caused a bandwidth negotiation conflict leading to the failure of detecting the second monitor.

macOS would **fail to detect the second display** or reduce refresh rate due to link bandwidth saturation.

When both were connected:

- The system forced:  
  - **4K 100Hz** + **2K 170Hz**,  
  - or **4K only**, depending on the cable/hub combination.
- macOS refused to enable both high refresh rates at the same time.


This indicated that **the primary display was consuming more DP Alt-Mode bandwidth than expected**, preventing the second display from initializing.

## Additional Hardware Notes

Before purchasing the new Thunderbolt 4 Multimedia Pro Dock, the previous dock used was a Thunderbolt 3 hub.  
Although the TB3 hub provided both a DisplayPort output and a USB‑C DP Alt‑Mode output, it **did not support DSC passthrough over USB‑C**, which prevented high‑refresh‑rate 4K modes from working reliably.  
This limitation was one of the reasons a new dock was required.

Additionally, even when using **two identical Thunderbolt docks**, each connected to a different external display, macOS was still unable to drive both monitors simultaneously at high refresh rates.  
This suggests that the M4 Pro’s **single Thunderbolt port was saturating its available DisplayPort bandwidth**, especially when one monitor attempted to negotiate modes above 4K 144Hz.  
Only after removing the 4K240Hz EDID timing did macOS stop over‑allocating bandwidth, allowing dual‑monitor output to function normally.

---

# 2. 🔍 Root Cause

### ✔ The monitor’s **EDID contained a rogue 4K 240Hz Detailed Timing Block**  
One of the **DisplayID Type I (Detailed Timing)** entries advertised:

- **3840×2160 @ 240Hz**  
- **Pixel Clock ≈ 2332 MHz**

This bandwidth **exceeds macOS DP Alt-Mode limits** (and USB-C dock passthrough), causing:

- macOS to assume the display supports 4K 240Hz
- macOS attempts to reserve the bandwidth for it  
- leaving *no bandwidth* for the second monitor  
- resulting in detection failures or refresh rate drops

### ✔ The problem was NOT the cable, hub, or macOS  
The real issue was **a false 4K240Hz timing inside the EDID**—specifically the Detailed Timing entry with a **2332 MHz pixel clock**, which macOS interpreted as a valid 240Hz mode.  
Because of this, macOS attempted to handshake and allocate bandwidth for 4K240Hz, which exceeded the Thunderbolt dock’s available DP Alt-Mode bandwidth.  
Removing this 2332 MHz timing block prevented macOS from attempting the 240Hz handshake, which immediately restored normal dual‑monitor operation.

---

# 3. 🧪 Troubleshooting Steps

### 3.1 Verified the EDID content  
Using **BetterDisplay** + **AW EDID Editor**, we examined the monitor’s EDID in Base64 format and confirmed:

- DisplayID Block 4 (Timing Type I) contained a **2332MHz pixel clock timing**
- It represented **4K 240Hz**, which the monitor *does* support, but macOS attempted to negotiate this mode through the dock, exceeding the dock’s available DP Alt‑Mode bandwidth.

### 3.2 Confirmed priority behavior  
macOS prioritizes:

1. **Detailed Timing in EDID Base**
2. **DisplayID Timing Blocks**
3. **CEA Extension Timing**
4. Other capability blocks

Thus, the 4K240 entry in **DisplayID Block #4** overrode all other timing info and forced macOS to enable 240Hz capability.

### 3.3 Tested behavior by removing the block  
When **Block 4** (Timing #4) was deleted:

- 240Hz disappeared  
- 144Hz reappeared  
- macOS immediately allowed the second monitor to connect  
- No instability or missing functionality

This confirmed that the **4K240 timing was the sole cause**.

---

# 4. ✅ Final Solution

### ✔ **Remove Timing Block #4** from the DisplayID Detailed Timing list  
In AW EDID Editor:

1. Navigate to  
   **DisplayID Extension → Timing Type I**
2. Select the entry corresponding to  
   `Pixel Clock: 2332 MHz`
3. Delete it
4. Save the modified EDID (`*.bin`)
5. Re-import it into BetterDisplay
6. Enable “Keep connection when EDID changes”
7. Apply & restart macOS display service (or reboot)

### Result:

- **144Hz mode returned**
- **240Hz mode disappeared** (expected)
- macOS no longer over-allocates bandwidth
- Both monitors can now run simultaneously (e.g., 4K144 + 2K170)
- No side-effects (HDR, VRR, colorimetry preserved)

---

# 5. 📈 Final Working EDID Characteristics

After removal of the problematic timing:

- Native modes remain:
  - 4K 144Hz  
  - 4K 120Hz  
  - 4K 100Hz  
  - (No 240Hz—this is correct)
- CEA extension still intact
- DisplayID blocks 1–3 remain unchanged
- Block 5 restored properly
- All chromaticity & HDR metadata untouched

This is the **safest and cleanest EDID patch**.