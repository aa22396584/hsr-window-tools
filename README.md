# HSR Window Tools / WindowPatcher v6

**Development, Issues & Pull Requests:**  
https://github.com/aa22396584/hsr-window-tools

**Mirrors:**  
[GitLab](https://gitlab.com/aa22396584/hsr-window-tools) ·
[Codeberg](https://codeberg.org/ImL1s/hsr-window-tools)


> **Why this GitHub home?** Public development moved here from [`ImL1s/hsr-window-tools`](https://github.com/ImL1s/hsr-window-tools) because that GitHub account is currently restricted (anonymous visitors get 404 on the profile and many assets). This is the same project. Please open Issues and Pull Requests here.

[![CI](https://github.com/aa22396584/hsr-window-tools/actions/workflows/ci.yml/badge.svg)](https://github.com/aa22396584/hsr-window-tools/actions/workflows/ci.yml)
[![Release](https://github.com/aa22396584/hsr-window-tools/actions/workflows/release.yml/badge.svg)](https://github.com/aa22396584/hsr-window-tools/actions/workflows/release.yml)
[![Latest Release](https://img.shields.io/github/v/release/aa22396584/hsr-window-tools?label=release)](https://github.com/aa22396584/hsr-window-tools/releases/latest)

## Screenshots

<table>
  <tr>
    <td align="center"><strong>HSR window with thickframe (resizable + maximizable)</strong></td>
    <td align="center"><strong>Settings window — target management UI</strong></td>
  </tr>
  <tr>
    <td><img src="docs/screenshots/01-hsr-thickframe.png" alt="HSR thickframe demo" width="500"/></td>
    <td><img src="docs/screenshots/03-settings-window.png" alt="Settings window" width="500"/></td>
  </tr>
  <tr>
    <td>StarRail.exe window after patching: standard Windows title bar with min/max/close buttons, draggable border, snap-able by FancyZones / Win+arrow. (UID redacted)</td>
    <td>WPF Fluent UI tray app: target list (StarRail / Calculator) with style indicators, primary actions on right (red Delete = destructive), activity feed at bottom.</td>
  </tr>
</table>

[繁體中文](#繁體中文) | [English](#english)

---

## English

Windows PowerShell tooling that makes Honkai: Star Rail and similar Unity game windows resizable, maximizable, and compatible with Windows Snap / PowerToys FancyZones.

### What it does

- Detects target game processes and adds `WS_THICKFRAME + WS_MAXIMIZEBOX` to their windows.
- Makes the window draggable by border, maximizable, and Snap/FancyZones-friendly.
- Runs as a WPF tray app, with legacy one-shot patch scripts still available.
- Requires administrator privileges when patching elevated game windows; UAC behavior depends on local policy.

### Quick start

```powershell
# Start tray mode
.\WindowPatcher\WindowPatcher.bat

# (Optional) Auto-start on Windows logon
.\Install-Autostart.bat      # creates HIGHEST scheduled task
.\Uninstall-Autostart.bat    # removes it

# (Optional) Pin to Start Menu + Desktop (admin-elevated shortcuts)
.\Install-StartMenu.bat      # creates Start Menu + Desktop .lnk with admin flag
.\Uninstall-StartMenu.bat    # removes them
# Then: press Win → search "WindowPatcher" → right-click → 釘選到「開始」畫面 / 釘選到工作列
# (Win11 22H2+ blocks programmatic taskbar pinning, so the last pin is a one-click manual step)

# CLI smoke checks
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --help
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --status
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --scan-now

# Legacy one-shot patch for StarRail.exe
.\APPLY-PATCH.bat
```

### Persistence & auto-apply
- Config (targets, FPS profile, scan interval) is persisted to `%LOCALAPPDATA%\WindowPatcher\config.json` and loaded on every start.
- The tray app runs a `DispatcherTimer` (default 2s) that detects game processes and automatically applies window style + FPS patches without any manual click.
- Install-Autostart.bat registers a `onlogon HIGHEST` scheduled task so the tray relaunches itself after every Windows login.
- Install-StartMenu.bat creates a Start Menu + Desktop `.lnk` (with the `.lnk` admin-elevation flag set, byte `0x15 |= 0x20`) so users can launch WindowPatcher from `Win + search` or the desktop with one UAC prompt. Best-effort attempts to pin to the taskbar; Win11 22H2+ usually blocks this, so the script prints a one-step manual fallback (right-click → 釘選到工作列).

### Localization (i18n)

5 locales supported: **zh-TW** (canonical) · **en** · **zh-CN** · **ja** · **ko**.

- Resource files: `WindowPatcher\i18n\<locale>\strings.psd1` (PowerShell standard `Import-LocalizedData` format)
- Switch via tray right-click → **Language / 語言** ▸ Auto / 繁體中文 / English / 简体中文 / 日本語 / 한국어 → restart prompt
- `config.json.language = 'auto'` (default): detect `$PSCulture` → exact match → parent-locale match (e.g. `zh-HK` → `zh-CN`, `en-AU` → `en`) → fallback `en`
- Adding a 6th locale: add `WindowPatcher\i18n\<locale>\strings.psd1` (75 keys, same set as `zh-TW`), append to `$script:SupportedLocales` + tray menu `$langOpts`. Pester contract test enforces key-parity across all locales.

### FPS note (2026-05 真實測試確認)

End-to-end test on a live StarRail (PID 35188) verified: HSR **still uses** the single
`GraphicsSettings_Model_h2986158309` JSON binary key (DJB2 hash 5/5 matches).
The key was previously absent only because the user had never opened the in-game
graphics settings panel. After clicking any setting + ESC, HSR writes the full
JSON blob and our wizard correctly diffs + patches FPS to 120.

**Zero-to-hero flow (verified, no need to close HSR)**:
1. Start tray (`WindowPatcher\WindowPatcher.bat`) — tray auto-elevates and watches game processes
2. Start HSR — tray polling injects `WS_THICKFRAME + WS_MAXIMIZEBOX` within 2s
3. Right-click tray icon → **FPS Probe Wizard** → 確定 to baseline
4. In HSR: ESC → Settings → Graphics → toggle FPS → ESC out of settings panel (don't close the game)
5. Wizard polls registry every 2s, detects FPS binary change → auto-prompts "patch to 120?"
6. Click Yes → registry now `{"FPS":120,...}` + wizard auto-enables persistent guard
7. **Persistent guard**: any later HSR settings change that overwrites FPS gets re-patched within 2s by main DispatcherTimer
8. Restart HSR — game reads `{"FPS":120}` and runs at 120 FPS

FPS tooling is disabled by default for HSR (`fpsProfile = "none"`, `fpsTarget = 0`)
to avoid noisy "key not found" while baseline isn't set. Activate via tray menu or
`WindowPatcher\RegProbe.ps1` when you actually want to flip the bit.

### Key files

| Path | Purpose |
| --- | --- |
| `WindowPatcher/WindowPatcher-WPF.ps1` | Main WPF tray app and CLI entry point |
| `WindowPatcher/WindowPatcher.bat` | Tray launcher |
| `WindowPatcher/reload-admin.ps1` | Restart tray app elevated |
| `HSR-Patch.ps1` | Legacy one-shot StarRail window patcher |
| `APPLY-PATCH*.bat` | Legacy wrapper scripts, optionally resizing the window |
| `HsrWatcher.ps1` | Legacy polling watcher |
| `Install-Watcher.bat` / `Uninstall-Watcher.bat` | Legacy scheduled-task watcher management |
| `WindowPatcher/RegProbe.ps1` | Registry diff helper for FPS key discovery |
| `WindowPatcher/FPS-Wizard.ps1` | CLI FPS probe wizard wrapper |

### Verification snapshot

The project has been smoke-tested against a live StarRail window:

- `--help`, `--list`, `--status`, `--scan-now`, `--apply StarRail`, and `--diagnose` returned successfully.
- Live StarRail window style contained both `THICKFRAME=True` and `MAXIMIZEBOX=True`.
- Temporary resize/restore checks confirmed the patched window can be resized and restored.
- `APPLY-PATCH*.bat` wrappers were tested with automatic restore.
- All tracked PowerShell scripts passed parser checks.

### Risks and limitations

- This tool manipulates another process's window style via Win32 APIs and may require elevation.
- The launchers use `ExecutionPolicy Bypass` for local unsigned scripts; inspect the source and only run copies from this repository.
- Restarting the game window restores the original window style; the patch is not permanent.
- FPS registry changes are a separate risk surface from window style patching and are not enabled by default for HSR.
- The legacy scheduled-task watcher runs from the local script path with highest privileges; install it only from a trusted, non-shared folder.
- Use at your own risk; game publisher policies may change.

### Uninstall

```powershell
# Stop tray process
Get-CimInstance Win32_Process -Filter "Name = 'pwsh.exe'" |
  Where-Object { $_.CommandLine -like '*WindowPatcher-WPF.ps1*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

# Remove user config/logs
Remove-Item "$env:LOCALAPPDATA\WindowPatcher" -Recurse -Force
```

---

## 繁體中文

這是一組 Windows PowerShell 工具，用來讓《崩壞：星穹鐵道》和類似 Unity 遊戲視窗可以拖曳邊框、最大化，並能被 Windows Snap / PowerToys FancyZones 正常抓取。

### 核心功能

- 偵測目標遊戲 process，為視窗補上 `WS_THICKFRAME + WS_MAXIMIZEBOX`。
- 讓原本鎖死大小的視窗可以拖曳邊框、最大化、Snap、FancyZones。
- 主要介面是 WPF 托盤程式；舊版一次性 patch 腳本仍保留。
- 修補高權限遊戲視窗時需要管理員權限；是否能完全 zero-touch 取決於本機 UAC 政策。

### 快速開始

```powershell
# 啟動托盤模式
.\WindowPatcher\WindowPatcher.bat

# (可選) 開機自動啟動
.\Install-Autostart.bat       # 建 HIGHEST schtasks
.\Uninstall-Autostart.bat     # 移除

# (可選) 釘到開始選單 + 桌面捷徑 (內建 admin 提權 flag)
.\Install-StartMenu.bat       # 建 Start Menu + Desktop .lnk
.\Uninstall-StartMenu.bat     # 移除
# 然後: Win 鍵 → 搜「WindowPatcher」→ 右鍵 → 釘選到「開始」畫面 / 釘選到工作列
# (Win11 22H2+ 擋了程式化釘選工作列,所以最後這一下是手動)

# CLI 基本檢查
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --help
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --status
pwsh -NoProfile -ExecutionPolicy Bypass -File .\WindowPatcher\WindowPatcher-WPF.ps1 --scan-now

# 舊版 StarRail.exe 一次性 patch
.\APPLY-PATCH.bat
```

### FPS 說明 (2026-05 真實測試確認)

對活 HSR (PID 35188) 端到端測試確認:HSR **仍使用** 單一 `GraphicsSettings_Model_h2986158309` JSON binary key (DJB2 hash 演算法已驗證 5/5 命中)。先前看不到此 key 純粹是因為使用者**從未開過遊戲內畫面設定**,只要進設定隨便動一個選項 + ESC,HSR 就會把完整 JSON blob (含 `"FPS":N`) 寫入 registry,wizard 立刻能 diff + patch 為 120。

**從零到 120 FPS 完整流程 (不需關 HSR,已驗證 E2E)**:
1. 啟動 tray (`WindowPatcher\WindowPatcher.bat`) — 自動提權 + watch 遊戲 process
2. 啟動 HSR — tray 約 2 秒內注入 `WS_THICKFRAME + WS_MAXIMIZEBOX`
3. 右下角托盤 → 右鍵 → **FPS 探查精靈** → 確定建基線
4. HSR 內: ESC → 設定 → 畫面 → 切換 FPS → ESC 關設定面板 (**不必關遊戲**)
5. Wizard 偵測 FPS binary 變化 → 跳「是否 patch 到 120?」對話框 → 點「是」
6. **第二個 dialog 問是否啟動「持續守護」** (opt-in, 預設不啟動):
   - **是** → 之後 HSR 動其他設定若把 FPS 寫回,工具 2 秒內 re-patch 回 120
   - **否** → 只 patch 這一次, 後續手動調整不被覆寫
7. 隨時可從 tray 右鍵選單 **「持續守護: 啟動/停止」** 切換
8. 重啟 HSR — 遊戲讀 `{"FPS":120}` → 跑 120 FPS

**如何停止持續守護** (若想手動調 HSR FPS 不被覆寫):
- 右下角托盤右鍵 → **「持續守護: 停止」** (config 寫回 fpsTarget=0)
- 或開「管理目標」→ 編輯 StarRail target → fpsProfile = "none"

FPS tweak 預設不啟用 (`fpsProfile = "none"`, `fpsTarget = 0`) 是為了避免「找不到 key」噪音 — 透過托盤選單的 **FPS 探查精靈** 或 `WindowPatcher\RegProbe.ps1` 主動開啟即可。

### 主要檔案

| 路徑 | 用途 |
| --- | --- |
| `WindowPatcher/WindowPatcher-WPF.ps1` | 主程式，WPF 托盤 app 與 CLI 入口 |
| `WindowPatcher/WindowPatcher.bat` | 托盤啟動器 |
| `WindowPatcher/reload-admin.ps1` | 重新啟動托盤並提權 |
| `HSR-Patch.ps1` | 舊版 StarRail 一次性視窗修補 |
| `APPLY-PATCH*.bat` | 舊版 wrapper，可選擇 resize |
| `HsrWatcher.ps1` | 舊版 polling watcher |
| `Install-Watcher.bat` / `Uninstall-Watcher.bat` | 舊版排程 watcher 安裝/移除 |
| `Install-Autostart.bat` / `Uninstall-Autostart.bat` | 開機自啟動 (schtasks onlogon HIGHEST) |
| `Install-StartMenu.bat` / `Uninstall-StartMenu.bat` | Start Menu + 桌面捷徑 (`.lnk` byte 0x15 admin flag,Win 搜尋即見) |
| `WindowPatcher/i18n/<locale>/strings.psd1` | 多語系字串檔 (zh-TW canonical / en / zh-CN / ja / ko,75 keys 對齊) |

### 多語系 (i18n)

支援 5 種語言: **zh-TW** (canonical) · **en** · **zh-CN** · **ja** · **ko**。

- 切換方式: tray 右鍵 → **語言 / Language** ▸ 選擇 → 提示重啟生效
- `config.json.language = 'auto'` (預設): 偵測 `$PSCulture` → 完全 match → 父語系 match (`zh-HK` → `zh-CN`、`en-AU` → `en`) → fallback `en`
- 加第 6 種語系: 新增 `WindowPatcher\i18n\<locale>\strings.psd1` (75 keys,跟 zh-TW canonical 同集合),加到 `$script:SupportedLocales` + tray `$langOpts`。Pester contract test 強制 key parity
| `WindowPatcher/RegProbe.ps1` | FPS key 探查用 registry diff helper |
| `WindowPatcher/FPS-Wizard.ps1` | CLI 版 FPS 探查精靈 wrapper |

### 實測摘要

已對 live StarRail 視窗做過安全 smoke test：

- `--help`、`--list`、`--status`、`--scan-now`、`--apply StarRail`、`--diagnose` 都能正常返回。
- live StarRail 視窗 style 已包含 `THICKFRAME=True` 與 `MAXIMIZEBOX=True`。
- 實際 resize / restore 測試確認視窗可被調整大小並還原。
- `APPLY-PATCH*.bat` wrapper 已測試，並在測試後自動還原視窗大小。
- 所有已追蹤的 PowerShell 腳本通過 parser check。

### 風險與限制

- 本工具透過 Win32 API 操作其他 process 的視窗 style，可能需要管理員權限。
- 啟動器會使用 `ExecutionPolicy Bypass` 執行本機未簽章腳本；請先檢查來源，只執行本 repo 的可信副本。
- 視窗 style 修補不是永久修改；重啟遊戲視窗後會回到原狀。
- FPS registry 改寫和視窗 style 修補是不同風險面，且 HSR 預設不啟用 FPS patch。
- 舊版排程 watcher 會從本機腳本路徑以最高權限執行；只建議從可信、非共用資料夾安裝。
- 遊戲廠商政策可能改變，請自行承擔使用風險。

### 卸載

```powershell
# 停止托盤 process
Get-CimInstance Win32_Process -Filter "Name = 'pwsh.exe'" |
  Where-Object { $_.CommandLine -like '*WindowPatcher-WPF.ps1*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

# 移除使用者設定與 log
Remove-Item "$env:LOCALAPPDATA\WindowPatcher" -Recurse -Force
```

---

## Support / 支持

If this project saved you some time, you can [buy me a coffee](https://buymeacoffee.com/iml1s).

如果這個專案幫你省了點時間，可以請我喝杯咖啡。
