# Screen-Flickering-Intel-NUC7i5
Fixing 4K 60Hz Screen Flickering on Intel NUC7 Series (Linux Mint).

This repository/guide documents the step-by-step process used to resolve a specific issue where a 7th-generation Intel NUC (NUC7i5BNK, i5-7260U) experiences random screen blinking and signal drops.
This occurs when the pc is connected to an external 4K display (e.g., a 65" TV) at 4K 60Hz under Linux Mint.

I believe this solution could be applicable to other NUC models as well and is worth a shot!

## The Symptoms
- The display operates perfectly at 1080p 60Hz or 4K 30Hz.
- When set to 4K 60Hz, the screen randomly blinks black, especially when playing videos in full-screen mode.
- System logs (`dmesg`) output: `[drm] *ERROR* CPU pipe A FIFO underrun`
- `xrandr` lists the connection as `DP-1` rather than HDMI, indicating the system is routing through the internal LSPCon (DisplayPort to HDMI) bridge chip.

## The Solution
Fixing this requires a combination of OS-level configuration, a BIOS update, and a hardware-level firmware flash of the LSPCon chip.

---

### Step 1: Linux Mint Software Adjustments
The `FIFO underrun` error occurs when the Intel integrated graphics driver aggressively drops power states, starving the system of the memory bandwidth required for 4K 60Hz.
<img width="863" height="471" alt="pip error" src="https://github.com/user-attachments/assets/5daddcc5-0329-4f17-b114-942e2fd27b3d" />


1. **Disable Full-Screen Compositing:**
   - Open Cinnamon Settings -> **General**.
   - Toggle **ON**: `Disable compositing for full-screen windows`.

2. **Disable Display C-States via GRUB:**
   - Open a terminal and edit your GRUB configuration:
     ```bash
     sudo nano /etc/default/grub
     ```
   - Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and append `i915.enable_dc=0`. It should look like this:
     ```text
     GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_dc=0"
     ```
   - Save and close the editor: Press **Ctrl+O**, then **Enter** to save the file, followed by **Ctrl+X** to exit nano.
   - Update GRUB to apply the changes:
     ```bash
     sudo update-grub
     ```
   - Reboot the machine.

---

### Step 2: BIOS Update
Update your NUC's BIOS to ensure underlying hardware stability.

1. Download the latest `.bio` file for your NUC model, from here: https://www.asus.com/support/download-center/  
For me it was from here: https://www.asus.com/supportonly/nuc7i5bnk/helpdesk_bios
<img width="1553" height="379" alt="bios" src="https://github.com/user-attachments/assets/964326f2-f270-48cc-a2ac-f557d1140504" />


2. Copy the `.bio` file directly to the root of a FAT32-formatted USB flash drive.
3. Restart the NUC and repeatedly press **F7** during boot to enter the BIOS Flash Update tool.
4. Select the `.bio` file on the USB drive and confirm. Do not remove power while it flashes.

---

### Step 3: LSPCon Firmware Update (The tricky part)
The NUC7 series routes the CPU's DisplayPort signal through an onboard **MegaChips LSPCon** chip to output HDMI 2.0. This chip's factory firmware (v1.72) has known bugs at 4K 60Hz. You must update it to v1.77 (or latest). 
This is done using **BobcatAuxISPTool.exe** program that's available from Asus's support page: https://www.asus.com/supportonly/nuc7i5bnk/helpdesk_bios/
<img width="1567" height="217" alt="asus driver" src="https://github.com/user-attachments/assets/4cd228f2-40cd-4d66-b008-b931b0213911" />


**Important:** The Intel HDMI Firmware Update Tool only runs on Windows. You will need to boot into a Windows To Go USB or a temporary Windows installation.

#### The Windows 11 Troubleshooting Ordeal
The tool recommends to use Windows 10, but its doable from Windows 11 as well (which I had access to).
And if you are using a barebones/temporary Windows 11 installation to perform this flash, you will likely run into a few roadblocks:

1. **Missing DLLs:** A fresh Windows 11 To Go drive might lack specific Visual C++ redistributable packages required by the flasher. If the `BobcatAuxISPTool.exe` throws a missing DLL error upon launch, you will need to manually download and install the required Microsoft Visual C++ dependencies. This can be looked up from here: https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170
  <img width="873" height="475" alt="dll drivers" src="https://github.com/user-attachments/assets/a1681ab6-4c9e-4927-bb31-019ad38afa61" />   
  
`I had to install both x64 and x86 versions as installing only x64 did not help the program run.`

2. **"No Device Found" Error:** The firmware tool (`BobcatAuxISPTool.exe`) must communicate with the LSPCon chip via the display pipeline. **If the proper Intel Iris Graphics driver is missing, the tool will fail to detect the chip entirely.**
   - **The Fix:** Download and install the official Intel Iris Graphics driver for your CPU (e.g., i5-7260U) for Windows, from Intel's website: https://www.intel.com/content/www/us/en/download/776137/intel-7th-10th-gen-processor-graphics-windows.html.
    <img width="948" height="636" alt="iris driver" src="https://github.com/user-attachments/assets/5843ba94-14b0-4319-b64e-5fe2bb3e5691" />


   
   - Once the graphics driver is installed and the NUC is outputting to the TV using the Intel driver (rather than the Microsoft Basic Display Adapter), restart the firmware tool to refresh and rescan the status.

#### Flashing the Chip
1. Run the HDMI Firmware Update Tool.
<img width="684" height="514" alt="exe run" src="https://github.com/user-attachments/assets/90ff6e15-7ffd-437d-a7a5-bc841201ee2e" />

2. Ensure the tool successfully detects `Intel (LS-PCON): 0x0D` in the Graphics Adapter dropdown.
3. Click **Get FW Version** button to collect current version details. Leave rest of the fields unchanged.
4. Click **Start**. The tool will erase the old flash (v1.72) and write the new firmware (v1.77).
5. Wait for the `ISP completed!` success prompt. 
<img width="664" height="485" alt="isp complete" src="https://github.com/user-attachments/assets/26a5dbd9-f3ba-4204-9362-8abe615fc2bc" />


---

### Conclusion
Once the LSPCon is updated to v1.77 and the GRUB kernel parameter is active, shut down the NUC, remove the Windows USB/SSD, and boot back into Linux Mint. 

Open Display Settings, set your resolution to `3840 x 2160` at `60.00 Hz`, and enjoy flicker-free 4K playback.
I did verify that there isn't any sort of random blinking or momentary flickering on my TV.

Good luck!
