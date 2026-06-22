# Assetto Corsa & Content Manager Linux/Proton Setup Guide

This document describes the exact steps, paths, and configurations required to successfully run **Assetto Corsa** (AppID `244210`) with **Content Manager** on Linux under a Steam / Proton environment. This serves as a reference for future installations or troubleshooting.

---

## 1. System Directory Structure

On this system, the directories are configured as follows:
*   **Linux Steam Path:** `/home/user/.local/share/Steam` (symlinked at `~/.steam/root`)
*   **Assetto Corsa Game Directory (on sda):** `/mnt/sda/SteamLibrary/steamapps/common/assettocorsa`
*   **Assetto Corsa Proton Prefix (pfx):** `/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx`
*   **Active Compatibility Tool:** `GE-Proton10-34` (located at `/home/user/.local/share/Steam/compatibilitytools.d/GE-Proton10-34`)
*   **Content Manager Path:** `/home/user/Documents/content-manager/Content Manager.exe`

---

## 2. Core Prefix Fixes

Running Content Manager and Assetto Corsa successfully requires editing the game's specific Proton prefix (`/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx`), **not** the default Wine prefix (`~/.wine`).

### A. MSVCR100_CLR0400.dll Fix (Fixes .NET Initialization Crashes)
The dummy `msvcr100_clr0400.dll` files shipped by default in Wine contain version resources but lack actual exports (like `_initterm_e`), causing Content Manager to crash immediately on startup.
1. Extract the native Microsoft binaries directly from a .NET 4.0 installer.
2. Copy the **64-bit DLL** to:
   `/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx/drive_c/windows/system32/msvcr100_clr0400.dll`
3. Copy the **32-bit DLL** to:
   `/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx/drive_c/windows/syswow64/msvcr100_clr0400.dll`
4. Register the DLL override as `native` inside the Proton prefix. Run using the Proton runtime:
   ```bash
   STEAM_COMPAT_CLIENT_INSTALL_PATH="/home/user/.local/share/Steam" STEAM_COMPAT_DATA_PATH="/mnt/sda/SteamLibrary/steamapps/compatdata/244210" "/home/user/.local/share/Steam/compatibilitytools.d/GE-Proton10-34/proton" run reg add "HKCU\Software\Wine\DllOverrides" /v msvcr100_clr0400 /t REG_SZ /d native /f
   ```

### B. Link Steam Config (Allows Content Manager to Log In)
Content Manager looks for `loginusers.vdf` in the prefix's simulated Steam folder to auto-detect the Steam profile.
1. Create the target config directory:
   ```bash
   mkdir -p "/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx/drive_c/Program Files (x86)/Steam/config"
   ```
2. Symlink the Linux host's active `loginusers.vdf` into it:
   ```bash
   ln -sf "/home/user/.steam/root/config/loginusers.vdf" "/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx/drive_c/Program Files (x86)/Steam/config/loginusers.vdf"
   ```

### C. Create `steam_appid.txt` (Fixes "Steam init failed" Game Crash)
When the game is launched directly by Content Manager outside of the Steam launcher interface, the Steamworks API inside `acs.exe` needs to know the AppID to initialize. Without this, the game crashes instantly with `ERROR: Steam init failed` in the logs.
*   Create a text file containing only `244210` at:
    `/mnt/sda/SteamLibrary/steamapps/common/assettocorsa/steam_appid.txt`

### D. Missing Log Directories (Fixes AppData Writing Exceptions)
Proton runs processes under the virtual Windows user profile name **`steamuser`**. Ensure directories exist so that writing configs/logs doesn't throw `DirectoryNotFoundException`:
*   **CSP/Tweak directory:** `/mnt/sda/SteamLibrary/steamapps/common/assettocorsa/extension/internal/`
*   **Log directory:** `/mnt/sda/SteamLibrary/steamapps/compatdata/244210/pfx/drive_c/users/steamuser/Documents/Assetto Corsa/logs/`

---

## 3. Content Manager Configurations

Once Content Manager launches:
1.  Navigate to **Settings** -> **Content Manager** -> **Drive**.
2.  Set the **Game Starter** to **`Naive`** (this is critical: other starters attempt to hook into a Windows Steam client process that does not exist in the Linux container).
3.  Ensure **Force 32-bit mode** is **unchecked** (disabled) so that the game runs in 64-bit mode (required for Custom Shaders Patch/mods).

---

## 4. Launching Content Manager

### Environment Variables
When launching Content Manager, you **must** supply `SteamAppId=244210` and `STEAM_COMPAT_APP_ID=244210` so that Proton passes the correct Steam initialization context down to the spawned game process (`acs.exe`). 

### Launch Command
```bash
env STEAM_COMPAT_CLIENT_INSTALL_PATH="/home/user/.local/share/Steam" \
    STEAM_COMPAT_DATA_PATH="/mnt/sda/SteamLibrary/steamapps/compatdata/244210" \
    SteamAppId=244210 \
    STEAM_COMPAT_APP_ID=244210 \
    "/home/user/.local/share/Steam/compatibilitytools.d/GE-Proton10-34/proton" run "/home/user/Documents/content-manager/Content Manager.exe"
```

### Desktop Entry (`.desktop` file)
Save this as `/home/user/Desktop/Content Manager.desktop` and make it executable (`chmod +x`):
```ini
[Desktop Entry]
Name=Content Manager
Comment=Assetto Corsa launcher and mod manager
Exec=env STEAM_COMPAT_CLIENT_INSTALL_PATH="/home/user/.local/share/Steam" STEAM_COMPAT_DATA_PATH="/mnt/sda/SteamLibrary/steamapps/compatdata/244210" SteamAppId=244210 STEAM_COMPAT_APP_ID=244210 "/home/user/.local/share/Steam/compatibilitytools.d/GE-Proton10-34/proton" run "/home/user/Documents/content-manager/Content Manager.exe"
Icon=steam_icon_244210
Terminal=false
Type=Application
Categories=Game;
```
