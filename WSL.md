# 全新 WSL 開發環境設定

適用 Windows 11，或 Windows 10 2004（組建 19041）以上。

## 1. 安裝 WSL 與 Ubuntu

以系統管理員身分開啟 PowerShell：

```powershell
wsl --install -d Ubuntu
```

若要求重新開機，重開後啟動 Ubuntu 並依畫面建立 Linux 使用者名稱與密碼。接著回到 PowerShell：

```powershell
wsl --update
wsl --list --verbose          # 確認 Ubuntu 的 VERSION 是 2
```

## 2. 更新 Ubuntu 並安裝基礎工具

以下改在 Ubuntu 中執行：

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  build-essential \
  procps \
  curl \
  file \
  git \
  zsh \
  ca-certificates \
  locales \
  unzip \
  keychain
sudo apt autoremove -y
```

建立空的 `.zshrc` 避免首次啟動 zsh 進入設定精靈，再把預設 shell 改為 zsh：

```bash
touch ~/.zshrc
chsh -s "$(command -v zsh)"
exec zsh
```

`chsh` 會要求輸入 Linux 使用者密碼；因 PAM 設定失敗時改用 `sudo chsh -s "$(command -v zsh)" "$USER"`。確認結果：

```bash
ps -p $$ -o comm=                      # 目前 shell
getent passwd "$USER" | cut -d: -f7    # 預設 shell
```

## 3. 安裝 Homebrew

```bash
NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

## 4. 安裝常用工具

```bash
brew install -y \
  wget \
  tree \
  htop \
  jq \
  fzf \
  ripgrep \
  fd \
  bat \
  eza \
  zoxide \
  starship
```

## 5. 安裝 zsh 外掛

```bash
brew install -y \
  zsh-autosuggestions \
  zsh-syntax-highlighting \
  zsh-completions \
  zsh-history-substring-search
```

## 6. 設定 `.zshrc`

全新環境可直接貼上以下命令，一次寫入 `~/.zshrc`：

```bash
cat > ~/.zshrc <<'EOF'
# Homebrew
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

# PATH（uv tool install、pipx 放 ~/.local/bin；go install 放 ~/go/bin）
export PATH="$HOME/.local/bin:$HOME/go/bin:$PATH"

# Editor
export EDITOR="${EDITOR:-vim}"
export VISUAL="$EDITOR"

# History
HISTFILE="$HOME/.zsh_history"
HISTSIZE=100000
SAVEHIST=100000
setopt SHARE_HISTORY
setopt HIST_IGNORE_ALL_DUPS
setopt HIST_IGNORE_SPACE

# Completion
fpath=("$HOMEBREW_PREFIX/share/zsh-completions" $fpath)
autoload -Uz compinit
compinit

# Completion 行為
zmodload -i zsh/complist
setopt AUTO_MENU              # 連按第二次 Tab 進入選單
setopt COMPLETE_IN_WORD       # 游標在字中間也能補
setopt ALWAYS_TO_END
unsetopt MENU_COMPLETE        # 不要自動選中第一個
eval "$(dircolors -b)"
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{[:lower:][:upper:]}={[:upper:][:lower:]}' 'r:|=*' 'l:|=* r:|=*'
zstyle ':completion:*' special-dirs true
zstyle ':completion:*' list-colors "${(s.:.)LS_COLORS}"
zstyle ':completion:*' use-cache yes
zstyle ':completion:*' cache-path "$HOME/.cache/zsh/compcache"
zstyle ':completion:*:cd:*' tag-order local-directories directory-stack path-directories

# SSH agent
if [[ -f "$HOME/.ssh/id_ed25519" ]]; then
  eval "$(keychain --eval --quiet id_ed25519)"
fi

# Plugins
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=8"
source "$HOMEBREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh"

# Tool integrations
source <(fzf --zsh)
eval "$(zoxide init zsh)"
eval "$(starship init zsh)"

# Node（fnm 於第 11 節安裝，--use-on-cd 會依 .nvmrc / .node-version 自動切換）
if command -v fnm >/dev/null; then
  eval "$(fnm env --use-on-cd --shell zsh)"
fi

# Aliases（可依喜好增減）
alias ls='eza --group-directories-first'
alias ll='eza -lh --git --group-directories-first'
alias la='eza -lah --git --group-directories-first'
alias cat='bat --paging=never'

# history-substring-search 必須載入在 syntax-highlighting 之後
source "$HOMEBREW_PREFIX/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh"
source "$HOMEBREW_PREFIX/share/zsh-history-substring-search/zsh-history-substring-search.zsh"
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down
# 部分終端在 application mode 送出的是 ^[OA / ^[OB，一併綁定
bindkey '^[OA' history-substring-search-up
bindkey '^[OB' history-substring-search-down
EOF
```

重新載入：`exec zsh`

若 `compinit` 回報 completion 目錄權限不安全，用 `autoload -Uz compaudit; compaudit` 列出目錄，確認由自己或 root 擁有後，只對這些目錄執行 `chmod go-w <目錄>`，不要遞迴修改整個 Homebrew prefix。之後新工具的 completion 沒出現時，用 `rm -f ~/.zcompdump && exec zsh` 重建快取。

## 7. 設定 Starship

```bash
mkdir -p ~/.config
starship preset catppuccin-powerline -o ~/.config/starship.toml
exec zsh
```

這個 preset 使用 Nerd Font 圖示，字型在下一節安裝；此時提示字元出現 `▯` 是正常的。

## 8. 安裝 Nerd Font

Windows Terminal 內附的 `Cascadia Code` / `Cascadia Mono` 不是 Nerd Font 版本，缺少 Starship、eza 用到的圖示。

**字型只裝在 Windows 端**：WSL 的輸出由 Windows 上的終端機程式繪製，WSL 內不需要安裝任何字型，`fc-list` 沒有輸出是正常的。

### 下載

以 Meslo 為例。換其他字型只要改檔名（`CascadiaCode.zip`、`JetBrainsMono.zip`、`FiraCode.zip`、`Hack.zip`）：

```
最新版：https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Meslo.zip
鎖版本：https://github.com/ryanoasis/nerd-fonts/releases/download/v3.5.0/Meslo.zip
```

要看預覽圖挑字型用 <https://www.nerdfonts.com/font-downloads>，要看版本與 changelog 用 <https://github.com/ryanoasis/nerd-fonts/releases>。

### 安裝（Windows 端，三選一）

**方式 A：scoop（推薦，可重現、免 UAC、可升級）**

未安裝 scoop 時，先在一般（非系統管理員）PowerShell 執行：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```

```powershell
scoop bucket add nerd-fonts
scoop install nerd-fonts/Meslo-NF          # 圖示保持原始寬度
# scoop install nerd-fonts/Meslo-NF-Mono   # 強制單寬，終端機對齊最穩
# scoop install nerd-fonts/Meslo-NF-Propo  # 比例寬
```

裝在使用者層（`%LOCALAPPDATA%\Microsoft\Windows\Fonts` 與 `HKCU`），日後用 `scoop update nerd-fonts/Meslo-NF` 升級。

**方式 B：手動**

解壓 `Meslo.zip`，全選 `.ttf` → 右鍵 →「安裝」（使用者層）或「為所有使用者安裝」（系統層，需 UAC）。

**方式 C：PowerShell（無互動，適合自動化）**

```powershell
$tmp = "$env:TEMP\nf"
New-Item -ItemType Directory -Force $tmp | Out-Null
Invoke-WebRequest "https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Meslo.zip" -OutFile "$tmp\Meslo.zip"
Expand-Archive "$tmp\Meslo.zip" "$tmp\Meslo" -Force

$dest = "$env:LOCALAPPDATA\Microsoft\Windows\Fonts"
$reg  = "HKCU:\Software\Microsoft\Windows NT\CurrentVersion\Fonts"
New-Item -ItemType Directory -Force $dest | Out-Null

Get-ChildItem "$tmp\Meslo\*.ttf" | ForEach-Object {
    Copy-Item $_.FullName $dest -Force
    New-ItemProperty -Path $reg -Name "$($_.BaseName) (TrueType)" -Value "$dest\$($_.Name)" -PropertyType String -Force | Out-Null
}
```

三種方式都不需要系統管理員權限（方式 B 選使用者層時）。安裝後要重開 Windows Terminal 與 VS Code，字型清單才會出現。

### 設定與驗證

Windows Terminal：**設定 → Ubuntu profile → 外觀 → 字型 → `MesloLGM Nerd Font`**，或直接編輯 `settings.json`（位於 `%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json`）：

```json
"font": { "face": "MesloLGM Nerd Font" }
```

VS Code 的整合終端機需另外在使用者設定中指定：

```json
"terminal.integrated.fontFamily": "'MesloLGM Nerd Font', monospace"
```

在 WSL 執行以下命令，依序出現 Apple logo、powerline 三角形、資料夾圖示就代表生效；顯示 `▯` 多半是終端機沒重開，或 profile 選到別的字型：

```bash
echo -e "\uf179 \ue0b0 \uf07c"
```

## 9. 設定 Git

```bash
git config --global user.name "你的名字"
git config --global user.email "你的信箱"
git config --global core.autocrlf input
git config --global push.autoSetupRemote true
git config --global core.editor "code --wait"
```

## 10. 設定 SSH Key 與 GitHub

```bash
ssh-keygen -t ed25519 -C "你的信箱"
eval "$(keychain --eval --quiet id_ed25519)"
```

存放路徑直接 Enter 使用預設的 `~/.ssh/id_ed25519`。設了 passphrase 後，keychain 每次重開 WSL 只會問一次，之後所有 shell 共用同一個 agent（第 6 節的 `.zshrc` 已含此設定）。

取得公鑰，貼到 GitHub 的 **Settings → SSH and GPG keys → New SSH key**，Key type 選 `Authentication Key`：

```bash
cat ~/.ssh/id_ed25519.pub          # 或 clip.exe < ~/.ssh/id_ed25519.pub 送進 Windows 剪貼簿
```

測試連線：

```bash
ssh -T git@github.com
```

第一次會詢問是否信任 GitHub 的 host key，輸入 `yes`。出現 `Hi <使用者名稱>! You've successfully authenticated...` 即成功，後面接著斷線是正常的（GitHub 不提供 shell）。之後 clone 要用 SSH 網址 `git@github.com:owner/repo.git` 才會走這把金鑰。

## 11. 安裝程式語言

```bash
brew install -y fnm go uv
```

不要對同一套 runtime 混用兩種安裝來源。

### Node

```bash
exec zsh
fnm install --lts
fnm use lts-latest
fnm default lts-latest
node --version
corepack enable
```

### Python

Ubuntu 已內建 `python3`（24.04 為 3.12），搭配 `uv` 就足夠，不需要 `brew install python`——多裝一套只會讓 `python3` 指向哪一個取決於 PATH 順序。

```bash
uv python install 3.12   # 需要特定版本時，uv 會下載獨立的 CPython
uv venv                  # 在專案目錄建立 .venv
uv add <package>         # 或 uv pip install <package>
```

## 12. 安裝 Docker

Docker Desktop 與 WSL 內的 Docker Engine 只能擇一。兩套並存時 `docker` 連到哪個 daemon 不確定，映像與 volume 也會散落在兩處。

### 方式 A：Docker Desktop

在 Windows 安裝後，於 **Settings > Resources > WSL Integration** 啟用 Ubuntu。Docker Desktop 會把 `docker`、`docker compose` 注入該發行版的 PATH，WSL 內不需要再用 apt 安裝；映像存在自己的 `docker-desktop-data` VHD，不會撐大 Ubuntu 的磁碟。選這個方式就跳過方式 B。

### 方式 B：在 WSL 內安裝 Docker Engine

先移除可能衝突的自帶套件：

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y "$pkg"
done
```

加入 Docker 官方 apt repository：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

安裝 Engine、CLI 與外掛：

```bash
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

啟動 daemon 並把自己加入 `docker` 群組，之後執行 `docker` 就不需要 `sudo`：

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

群組變更要重新登入才生效：回到 PowerShell 執行 `wsl --shutdown` 再重開，或用 `newgrp docker` 讓目前這個 shell 立即生效。

`systemctl` 需要 systemd，Ubuntu for WSL 預設已啟用。若 `systemctl is-system-running` 回報 `offline`，改用 `sudo service docker start`，或在 `/etc/wsl.conf` 加入以下設定後執行 `wsl --shutdown`：

```ini
[boot]
systemd=true
```

### 驗證

```bash
docker version              # 只顯示 Client 沒有 Server 就是連不到 daemon
docker run --rm hello-world
docker compose version
docker buildx version
```

### 注意事項

- 不要用 Homebrew 安裝 `docker`：brew 的 formula 只有 CLI，會和 apt 或 Docker Desktop 的 CLI 互相蓋掉。
- 方式 B 的映像、容器與 volume 都存在 Ubuntu 的 `ext4.vhdx`，會讓該檔案持續變大。定期 `docker system prune`（加 `--volumes` 會一併刪除未使用的 volume 與其資料），並搭配第 15 節的 sparse VHD 讓 Windows 端回收空間。

## 13. 驗證安裝

```bash
brew doctor
command -v zsh brew git rg fd bat eza fzf zoxide starship fnm node python3 go uv docker
git --version
node --version
python3 --version
go version
docker version
ssh -T git@github.com
date
locale
```

## 14. 備份 WSL

先建立輕量的 Homebrew 套件清單：

```bash
brew bundle dump --file="$HOME/Brewfile" --force
```

必須寫 `"$HOME/..."`：`--file=~/Brewfile` 的 `~` 在 zsh 與 bash 都不會展開（`--file` 不是合法的 identifier），會被當成相對路徑而失敗。

建議把 `Brewfile`、`.zshrc`、`.config/starship.toml` 放進自己的 dotfiles repository，不要提交 SSH 私鑰或其他憑證；重建時執行 `brew bundle --file="$HOME/Brewfile"`。Brewfile 只記錄套件名稱不鎖版本，還原時會裝當時可用的版本。

需要完整備份時從 PowerShell 匯出，路徑可依實際磁碟調整（沒有 D 槽就改用 `C:\WSL-Backup`）：

```powershell
New-Item -ItemType Directory -Force D:\WSL-Backup
wsl --shutdown
wsl --export Ubuntu D:\WSL-Backup\ubuntu.tar
```

還原時使用新的發行版名稱與安裝目錄，避免覆蓋現有 Ubuntu：

```powershell
New-Item -ItemType Directory -Force D:\WSL\Ubuntu-Restored
wsl --import Ubuntu-Restored D:\WSL\Ubuntu-Restored D:\WSL-Backup\ubuntu.tar --version 2
wsl --distribution Ubuntu-Restored --user root
```

匯入的發行版可能預設以 root 登入。在剛開啟的 shell 中編輯 `/etc/wsl.conf`，保留既有設定並加入以下內容（名稱換成原本的 Linux 使用者），再回 PowerShell 執行 `wsl --terminate Ubuntu-Restored` 與 `wsl --distribution Ubuntu-Restored`：

```ini
[user]
default=你的Linux使用者名稱
```

較新的 WSL 也支援 `wsl --export Ubuntu <file>.vhdx --vhd`：VHDX 適合快速完整搬移，tar 較通用且能用 `wsl --import` 指定新的安裝位置。

## 15. 選用的 WSL 資源設定

Windows 端的 `%UserProfile%\.wslconfig` 可限制所有 WSL 2 發行版使用的 CPU 與記憶體。預設上限是 Windows 記憶體的 50% 與全部邏輯處理器，沒有適合所有電腦的固定值，建議從 Windows 的 **WSL Settings** 依硬體與工作負載調整。手動編輯時，以下數值只是範例：

```ini
[wsl2]
memory=8GB
processors=4

[experimental]
sparseVhd=true
```

`sparseVhd=true` 只對之後新建立的 VHD 生效。既有發行版先執行 `wsl --update`，再從 PowerShell 啟用：

```powershell
wsl --shutdown
wsl --manage Ubuntu --set-sparse true
```

這讓既有 VHD 能在釋放 Linux 檔案空間後自動回收 Windows 磁碟空間，不是執行當下立即壓縮。若 `wsl --manage` 不支援 `--set-sparse`，以 `wsl --help` 確認目前版本的選項。修改 `.wslconfig` 後同樣要 `wsl --shutdown` 再重新啟動。除非願意自行補回 VS Code 等 Windows 路徑，否則保留 `/etc/wsl.conf` 的 `appendWindowsPath=true` 預設值。
 
