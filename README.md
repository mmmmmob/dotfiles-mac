# Mac Dotfiles & App Sync Workflow (chezmoi)

ระบบจัดการ Dotfiles กลางและแยกรายการแอปพลิเคชันระหว่างเครื่องส่วนตัว (`personal`) และเครื่องทำงาน (`work`) โดยใช้ **chezmoi** ควบคู่กับ **Homebrew Bundle**

---

## 1. โครงสร้างไฟล์ใน Repository (`~/.local/share/chezmoi`)

```text
.
├── .chezmoi.toml.tmpl                     # Script ถาม Role ครั้งแรกตอน init
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
│       └── executable_cz-sync-apps        # สคริปต์ดัมป์ Brewfile และ Manual apps
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

### D. ฟังก์ชันใน `~/.zshrc` (คำสั่งซิงก์ด่วน `dot-sync`)

```bash
# Chezmoi auto sync function
dot-sync() {
  echo "==> [1/4] Dumping latest apps & brew packages..."
  cz-sync-apps

  echo "==> [2/4] Pulling remote updates..."
  chezmoi git pull -- --autostash --rebase

  echo "==> [3/4] Committing changes (if any)..."
  local cz_dir
  cz_dir="$(chezmoi source-path)"

  if [[ -n $(git -C "$cz_dir" status --porcelain) ]]; then
    git -C "$cz_dir" add -A
    git -C "$cz_dir" commit -m "sync: $(date '+%Y-%m-%d %H:%M:%S') [$(chezmoi execute-template '{{ .role }}')]"
    git -C "$cz_dir" push
    echo "==> Pushed updates successfully."
  else
    echo "==> Nothing new to commit."
  fi

  echo "==> [4/4] Applying changes locally..."
  chezmoi apply
}
```

---

## 3. Workflow ใช้งานในชีวิตประจำวัน (Existing Machines)

### เมื่อมีการลงแอปใหม่ / อัปเดต Dotfiles

รันคำสั่งเดียวจบ:

```bash
dot-sync
```

คำสั่งนี้จะดัมป์ Brewfile และ manual-apps ประจำ Role นั้น, ดึง commit ล่าสุดแบบ auto-rebase, commit และ push ขึ้น GitHub ให้อัตโนมัติ

### เมื่อสลับไปใช้งานอีกเครื่องหนึ่ง

ต้องการดึงการตั้งค่าหรือ config ที่เพิ่ง push มาลงเครื่องปัจจุบัน:

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

เปิด Terminal บนเครื่องใหม่แล้วรัน:

```bash
# 1. ติดตั้ง Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. ตั้งค่า Homebrew PATH สำหรับ Apple Silicon
eval "$(/opt/homebrew/bin/brew shellenv)"

# 3. ติดตั้ง chezmoi
brew install chezmoi
```

### สเต็ป 2: ดึง Dotfiles และเลือก Role

```bash
chezmoi init --apply <GITHUB_REPO_URL>
```

- ระบบจะถามบทบาทของเครื่อง:

  ```text
  ??? Machine role (personal, work) [personal]:
  ```

- พิมพ์ `personal` หรือ `work` แล้วกด Enter
- chezmoi จะวางไฟล์ Dotfiles, เพิ่มสคริปต์ `cz-sync-apps`, และสั่งรัน `brew bundle` ติดตั้งโปรแกรมของ Role นั้นให้ทันทีแบบอัตโนมัติ

### สเต็ป 3: ตรวจสอบและลงแอปนอก Brew Store

เปิดดูลิสต์แอปที่ต้องโหลด `.dmg` / `.pkg` จากภายนอก:

```bash
cat ~/.config/brew/manual-apps.$(chezmoi execute-template '{{ .role }}').txt
```

---

## 5. สรุปคำสั่งสำคัญ (Cheat Sheet)

| คำสั่ง                | หน้าที่การทำงาน                                                 |
| --------------------- | --------------------------------------------------------------- |
| `dot-sync`            | ดัมป์ข้อมูลแอปของ Role ปัจจุบัน, Pull, Commit และ Push ขึ้น Git |
| `cz-sync-apps`        | ดัมป์ Brewfile และ manual apps ลงใน chezmoi source              |
| `chezmoi update`      | Pull ข้อมูลล่าสุดจาก Git และ Apply ลงเครื่องปัจจุบัน            |
| `chezmoi diff`        | ดูความแตกต่างระหว่างไฟล์ใน Repo กับไฟล์จริงก่อน Apply           |
| `chezmoi edit-config` | แก้ไข Role ของเครื่องปัจจุบัน (`personal` หรือ `work`)          |
| `chezmoi cd`          | เปิดโฟลเดอร์ต้นทางของ chezmoi เพื่อจัดการ Git โดยตรง            |
