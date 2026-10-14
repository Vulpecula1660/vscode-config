# 全新 WSL 開發環境設定

## 1. 安裝 WSL 與 Ubuntu

以系統管理員身分開啟 PowerShell：

```powershell
wsl --install -d Ubuntu-24.04
```

明確指定 `Ubuntu-24.04`，不要用 `-d Ubuntu`：後者會隨時間指向最新的 LTS（可用 `wsl --list --online` 查看），本文件的版本說明以 24.04 為準。

若要求重新開機，重開後啟動 Ubuntu 並依畫面建立 Linux 使用者名稱與密碼。接著回到 PowerShell：

```powershell
wsl --update
wsl --list --verbose          # 確認 Ubuntu-24.04 的 VERSION 是 2
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
  starship \
  gh
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

Starship 與 eza 的圖示需要 Nerd Font。字型只裝在 Windows 端：WSL 的畫面由 Windows Terminal 繪製，WSL 內不需要安裝字型。

1. 下載 [Meslo.zip](https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Meslo.zip)，想換其他字型可到 <https://www.nerdfonts.com/font-downloads> 預覽挑選。
2. 解壓縮後全選 `.ttf` → 右鍵 →「安裝」。
3. 重開 Windows Terminal，到 **設定 → Ubuntu 24.04 → 外觀 → 字型** 選 `MesloLGM Nerd Font`。
4. VS Code 的整合終端機需另外在使用者設定中指定：

   ```json
   "terminal.integrated.fontFamily": "'MesloLGM Nerd Font', monospace"
   ```

在 WSL 執行以下命令，依序出現 Apple logo、powerline 三角形、資料夾圖示就代表生效；顯示 `▯` 多半是終端機沒重開，或選到別的字型：

```bash
echo -e "\uf179 \ue0b0 \uf07c"
```

## 9. 設定 Git

```bash
git config --global user.name "你的名字"
git config --global user.email "你的信箱"
git config --global init.defaultBranch main
git config --global pull.ff only              # 或 pull.rebase true，依團隊習慣
git config --global fetch.prune true
git config --global push.autoSetupRemote true
git config --global core.editor "code --wait"
```

## 10. 設定 SSH Key 與 GitHub

```bash
ssh-keygen -t ed25519 -C "你的信箱"
eval "$(keychain --eval --quiet id_ed25519)"
```

存放路徑直接 Enter 使用預設的 `~/.ssh/id_ed25519`。設了 passphrase 後，keychain 每次重開 WSL 只會問一次，之後所有 shell 共用同一個 agent（第 6 節的 `.zshrc` 已含此設定）。

### 登入 GitHub 並上傳公鑰

用 GitHub CLI 登入，可一併上傳公鑰：

```bash
gh auth login
```

依序選擇 `GitHub.com` → `SSH` → 上傳 `~/.ssh/id_ed25519.pub` → `Login with a web browser`。瀏覽器沒有自動開啟時，手動到 <https://github.com/login/device> 輸入畫面上的一次性代碼。

也可以手動上傳：把公鑰送進 Windows 剪貼簿，貼到 GitHub 的 **Settings → SSH and GPG keys → New SSH key**，Key type 選 `Authentication Key`：

```bash
clip.exe < ~/.ssh/id_ed25519.pub
```

不要用 `cat` 顯示後再用滑鼠複製：第 6 節把 `cat` 設成 `bat` 的別名，輸出會帶行號與框線。要在終端機顯示時用 `command cat ~/.ssh/id_ed25519.pub`。

### 測試連線

```bash
ssh -T git@github.com
```

第一次會詢問是否信任 GitHub 的 host key，輸入 `yes`。出現 `Hi <使用者名稱>! You've successfully authenticated...` 即成功，後面接著斷線是正常的（GitHub 不提供 shell）。之後 clone 要用 SSH 網址 `git@github.com:owner/repo.git` 才會走這把金鑰。

### 用 SSH Key 簽署 commit（選用）

同一把金鑰也能用來簽署 commit，GitHub 上會顯示 `Verified`：

```bash
git config --global gpg.format ssh
git config --global user.signingkey "$HOME/.ssh/id_ed25519.pub"
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

再把公鑰以**簽署金鑰**的身分加到 GitHub（與上面的 Authentication Key 是分開的兩筆）：

```bash
gh auth refresh -h github.com -s admin:ssh_signing_key
gh ssh-key add ~/.ssh/id_ed25519.pub --type signing --title "WSL signing key"
```

或手動在 **New SSH key** 頁面再加一次同一把公鑰，Key type 改選 `Signing Key`。

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
```

需要 pnpm 時執行 `npm install -g pnpm`。

### Python

Ubuntu 24.04 已內建 `python3`（3.12），搭配 `uv` 就足夠，不需要 `brew install python`——多裝一套只會讓 `python3` 指向哪一個取決於 PATH 順序。

```bash
uv python install 3.13   # 需要其他版本時，uv 會下載獨立的 CPython
uv init                  # 在專案目錄建立 pyproject.toml
uv add <package>         # 加入依賴，並自動建立 .venv
```

不想用專案管理（沒有 `pyproject.toml`）時，改用 `uv venv` 建立 `.venv`，再用 `uv pip install <package>` 安裝。

## 12. 安裝 Docker Engine

直接在 WSL 內安裝 Docker Engine，不使用 Docker Desktop。若 Windows 已安裝 Docker Desktop，先在其 **Settings > Resources > WSL Integration** 關閉 Ubuntu-24.04 的整合，或直接解除安裝；兩套並存時 `docker` 連到哪個 daemon 不確定，映像與 volume 也會散落在兩處。

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

- 不要用 Homebrew 安裝 `docker`：brew 的 formula 只有 CLI，會和 apt 安裝的 CLI 互相蓋掉。
- 映像、容器與 volume 都存在 Ubuntu 的 `ext4.vhdx`，會讓該檔案持續變大。定期 `docker system prune`（加 `--volumes` 會一併刪除未使用的 volume 與其資料）。

## 13. 驗證安裝

```bash
brew doctor
command -v zsh brew git gh rg fd bat eza fzf zoxide starship fnm node python3 go uv docker code
git --version
node --version
python3 --version
go version
docker version
gh auth status
ssh -T git@github.com
date
locale
```

## 14. 選用的 WSL 資源設定

Windows 端的 `%UserProfile%\.wslconfig` 可限制所有 WSL 2 發行版使用的 CPU 與記憶體。預設上限是 Windows 記憶體的 50% 與全部邏輯處理器，沒有適合所有電腦的固定值，建議從 Windows 的 **WSL Settings** 依硬體與工作負載調整。手動編輯時，以下數值只是範例：

```ini
[wsl2]
memory=8GB
processors=4
```

修改 `.wslconfig` 後要 `wsl --shutdown` 再重新啟動。除非願意自行補回 VS Code 等 Windows 路徑，否則保留 `/etc/wsl.conf` 的 `appendWindowsPath=true` 預設值。
