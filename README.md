# Mac Dotfiles & App Sync Workflow (chezmoi)

ระบบจัดการ Dotfiles กลางและแยกรายการแอปพลิเคชันระหว่างเครื่องส่วนตัว (`personal`) และเครื่องทำงาน (`work`) โดยใช้ **chezmoi** ควบคู่กับ **Homebrew Bundle** พร้อมระบบ Auto-sync อัตโนมัติด้วย **macOS launchd**

---

## 1. โครงสร้างไฟล์ใน Repository (`~/.local/share/chezmoi`)

```text
.
├── .chezmoi.toml.tmpl                     # Template ถาม Role ครั้งแรกตอน chezmoi init
├── dot_gitconfig                          # Git config กลางที่ใช้ร่วมกัน
├── dot_zshrc                              # Zsh config กลางที่ใช้ร่วมกัน
├── dot_config/
│   └── brew/
│       ├── Brewfile.personal              # Homebrew packages ของเครื่อง Personal
│       ├── Brewfile.work                  # Homebrew packages ของเครื่อง Work
│       ├── manual-apps.personal.txt       # ลิสต์แอปนอก Brew ของ Personal
│       └── manual-apps.work.txt           # ลิสต์แอปนอก Brew ของ Work
├── dot_local/
│   └── bin/
│       ├── executable_cz-sync-apps        # สคริปต์ดัมป์ Brewfile และ Manual apps
│       └── executable_dot-sync            # สคริปต์หลักรัน Dump + Git Sync + Apply
├── Library/
│   └── LaunchAgents/
│       └── com.user.dotsync.plist         # Service ตั้งเวลาทำงานอัตโนมัติด้วย launchd
└── run_onchange_darwin-install-packages.sh.tmpl # Auto-run brew bundle เมื่อ Brewfile เปลี่ยน
```

---

## 2. ไฟล์สคริปต์และ Config หลัก (Source Files)

### A. `.chezmoi.toml.tmpl` (ถาม Role ตอน `init`)

```toml
{{- $role := promptChoiceOnce . "role" "Machine role" (list "personal" "work") -}}
[data]
    role = {{ $role | quote }}
```

### B. `run_onchange_darwin-install-packages.sh.tmpl` (Auto-install เมื่อไฟล์เปลี่ยน)

```bash
{{- $brewfile := joinPath .chezmoi.sourceDir "dot_config/brew" (printf "Brewfile.%s" .role) -}}
{{- if stat $brewfile -}}
#!/usr/bin/env bash
set -euo pipefail

# Brewfile hash: {{ include (printf "dot_config/brew/Brewfile.%s" .role) | sha256sum }}

echo "==> Syncing Homebrew packages for role: {{ .role }}..."
brew bundle --file="${HOME}/.config/brew/Brewfile.{{ .role }}"
{{- end -}}
```

### C. `dot_local/bin/executable_cz-sync-apps` (สคริปต์ดัมป์แอป)

```bash
#!/usr/bin/env bash
set -euo pipefail

ROLE=$(chezmoi execute-template '{{ .role }}')
BREW_DIR="$(chezmoi source-path)/dot_config/brew"

if [[ -z "$ROLE" ]]; then
  echo "Error: Machine role not found."
  exit 1
fi

mkdir -p "$BREW_DIR"

echo "==> Updating Brewfile.${ROLE}..."
brew bundle dump --force --file="${BREW_DIR}/Brewfile.${ROLE}"

echo "==> Updating manual-apps.${ROLE}.txt..."
comm -23 \
  <(ls -1 /Applications ~/Applications 2>/dev/null | grep '\.app$' | sed 's/\.app$//' | tr '[:upper:]' '[:lower:]' | sort -u) \
  <(brew list --cask 2>/dev/null | tr '[:upper:]' '[:lower:]' | sort -u) \
  > "${BREW_DIR}/manual-apps.${ROLE}.txt"

echo "==> Sync completed for ${ROLE}. Check changes with: chezmoi cd && git status"
```

### D. `dot_local/bin/executable_dot-sync` (สคริปต์สั่ง Sync เต็มระบบ)

```bash
#!/usr/bin/env bash
set -euo pipefail

# ระบุ PATH ให้ครอบคลุมการรันผ่าน launchd
export PATH="/opt/homebrew/bin:/usr/local/bin:${HOME}/.local/bin:/usr/bin:/bin:/usr/sbin:/sbin"

CZ_DIR="$(chezmoi source-path)"

echo "==> [$(date '+%Y-%m-%d %H:%M:%S')] Starting dot-sync..."

# 1. ดัมป์ลิสต์แอปและ brew packages ล่าสุด
cz-sync-apps

# 2. Pull remote ล่าสุดแบบ auto-stash rebase
chezmoi git pull -- --autostash --rebase

# 3. Commit & Push ถ้ามีไฟล์เปลี่ยนแปลง
if [[ -n $(git -C "$CZ_DIR" status --porcelain) ]]; then
  ROLE=$(chezmoi execute-template '{{ .role }}')
  git -C "$CZ_DIR" add -A
  git -C "$CZ_DIR" commit -m "sync: $(date '+%Y-%m-%d %H:%M:%S') [${ROLE}] (auto)"
  git -C "$CZ_DIR" push
  echo "==> Pushed updates successfully."
else
  echo "==> Nothing new to commit."
fi

# 4. Apply configuration ล่าสุดลงเครื่อง
chezmoi apply
```

### E. `Library/LaunchAgents/com.user.dotsync.plist` (launchd Auto-Sync)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "[http://www.apple.com/DTDs/PropertyList-1.0.dtd](http://www.apple.com/DTDs/PropertyList-1.0.dtd)">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.dotsync</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>exec "${HOME}/.local/bin/dot-sync"</string>
    </array>

    <!-- รันอัตโนมัติทุกวันเวลา 19:00 น. -->
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>19</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>

    <key>StandardOutPath</key>
    <string>/tmp/dotsync.stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/dotsync.stderr.log</string>
</dict>
</plist>
```

---

## 3. Workflow ใช้งานในชีวิตประจำวัน (Existing Machines)

### การทำงานแบบอัตโนมัติ (Automated)

- ระบบ **launchd** จะสั่งรัน `dot-sync` ทุกวันเวลา **19:00 น.** (หรือทันทีที่ปลุกเครื่องหากเครื่องหลับอยู่)
- ตรวจสอบผลการทำงานย้อนหลังได้ที่:
  ```bash
  cat /tmp/dotsync.stdout.log
  cat /tmp/dotsync.stderr.log
  ```

### เมื่อลงแอปใหม่แล้วต้องการ Sync ทันที (Manual Trigger)

พิมพ์คำสั่งนี้ใน Terminal เพื่อดัมป์และ Push ขึ้น Git ทันที:

```bash
dot-sync
```

### เมื่อสลับไปใช้งานอีกเครื่องหนึ่ง

ดึงการตั้งค่าหรือ config ที่เพิ่ง push มาลงเครื่องปัจจุบัน:

```bash
chezmoi update
```

- chezmoi จะ pull โค้ดล่าสุด
- อัปเดต Dotfiles (`.zshrc`, `.gitconfig`)
- รัน `brew bundle` ติดตั้งเฉพาะแพ็กเกจของเครื่องนี้ (ถ้า `Brewfile.<role>` ของเครื่องนี้มีการเปลี่ยนแปลง)

### เมื่อแก้ไข Dotfiles กลาง (`~/.zshrc`, `~/.gitconfig`)

- **วิธีแก้ผ่าน chezmoi:**
  ```bash
  chezmoi edit ~/.zshrc
  chezmoi apply
  chezmoi cd && git commit -am "Update zshrc" && git push origin main && exit
  ```
- **วิธีแก้ไฟล์ในเครื่องตรงๆ:**
  ```bash
  chezmoi re-add ~/.zshrc
  chezmoi cd && git commit -am "Update zshrc" && git push origin main && exit
  ```

---

## 4. Workflow เซ็ตอัปเครื่องใหม่แกะกล่อง (New Mac Setup)

### สเต็ป 1: ติดตั้ง Homebrew & chezmoi

```bash
# 1. ติดตั้ง Homebrew
/bin/bash -c "$(curl -fsSL [https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh](https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh))"

# 2. ตั้งค่า Homebrew PATH สำหรับ Apple Silicon
eval "$(/opt/homebrew/bin/brew shellenv)"

# 3. ติดตั้ง chezmoi
brew install chezmoi
```

### สเต็ป 2: ดึง Dotfiles และเลือก Role

```bash
chezmoi init --apply <GITHUB_REPO_URL>
```

- พิมพ์เลือกบทบาทของเครื่องนี้: `personal` หรือ `work` แล้วกด Enter
- chezmoi จะวางไฟล์ Dotfiles, Binary สคริปต์, Plist และสั่งรัน `brew bundle` ติดตั้งโปรแกรมของ Role นั้นให้อัตโนมัติ

### สเต็ป 3: เปิดใช้งาน launchd Auto-Sync

โหลด Service เข้าสู่ระบบของ macOS:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.dotsync.plist
```

_(ทดสอบยิง service ทำงานทันทีด้วย: `launchctl kickstart -k gui/$(id -u)/com.user.dotsync`)_

### สเต็ป 4: ตรวจสอบและลงแอปนอก Brew Store

เปิดดูแอปที่ต้องติดตั้งแบบ Manual:

```bash
cat ~/.config/brew/manual-apps.$(chezmoi execute-template '{{ .role }}').txt
```

---

## 5. สรุปคำสั่งสำคัญ (Cheat Sheet)

| คำสั่ง                                                 | หน้าที่การทำงาน                                             |
| :----------------------------------------------------- | :---------------------------------------------------------- |
| `dot-sync`                                             | รัน Dump apps + Pull + Push Git + Apply ครบวงจร             |
| `cz-sync-apps`                                         | ดัมป์ Brewfile และ manual-apps ลง chezmoi source อย่างเดียว |
| `chezmoi update`                                       | Pull ข้อมูลล่าสุดจาก Git + Apply ลงเครื่องปัจจุบัน          |
| `chezmoi diff`                                         | ดูความต่างระหว่างไฟล์ใน Repo กับไฟล์จริงบนเครื่องก่อน apply |
| `chezmoi edit-config`                                  | แก้ไข Role ของเครื่องปัจจุบัน (`personal` หรือ `work`)      |
| `chezmoi cd`                                           | สลับเข้าสู่โฟลเดอร์ต้นทางของ chezmoi เพื่อจัดการ Git โดยตรง |
| `launchctl kickstart -k gui/$(id -u)/com.user.dotsync` | สั่งให้ launchd ทำงานทันทีโดยไม่ต้องรอเวลา                  |

_(ทดสอบยิง service ทำงานทันทีด้วย: `launchctl kickstart -k gui/$(id -u)/com.user.dotsync`)_

### สเต็ป 4: ตรวจสอบและลงแอปนอก Brew Store

เปิดดูแอปที่ต้องติดตั้งแบบ Manual:

```bash
cat ~/.config/brew/manual-apps.$(chezmoi execute-template '{{ .role }}').txt
```

---

## 5. สรุปคำสั่งสำคัญ (Cheat Sheet)

| คำสั่ง                                                 | หน้าที่การทำงาน                                             |
| :----------------------------------------------------- | :---------------------------------------------------------- |
| `dot-sync`                                             | รัน Dump apps + Pull + Push Git + Apply ครบวงจร             |
| `cz-sync-apps`                                         | ดัมป์ Brewfile และ manual-apps ลง chezmoi source อย่างเดียว |
| `chezmoi update`                                       | Pull ข้อมูลล่าสุดจาก Git + Apply ลงเครื่องปัจจุบัน          |
| `chezmoi diff`                                         | ดูความต่างระหว่างไฟล์ใน Repo กับไฟล์จริงบนเครื่องก่อน apply |
| `chezmoi edit-config`                                  | แก้ไข Role ของเครื่องปัจจุบัน (`personal` หรือ `work`)      |
| `chezmoi cd`                                           | สลับเข้าสู่โฟลเดอร์ต้นทางของ chezmoi เพื่อจัดการ Git โดยตรง |
| `launchctl kickstart -k gui/$(id -u)/com.user.dotsync` | สั่งให้ launchd ทำงานทันทีโดยไม่ต้องรอเวลา                  |
