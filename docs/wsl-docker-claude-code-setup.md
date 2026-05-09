# WSL + Docker + Claude Code Container Setup

This document records the current setup used on `kona`'s WSL Ubuntu 24.04 machine.

Last updated: 2026-05-09.

The old setup routed WSL, apt, Docker, and containers through a Windows HTTP proxy on `127.0.0.1:7890`. That setup has been replaced. The current setup uses WSL NAT and lets the Windows host's Clash TUN virtual adapter route outbound traffic.

## Current Architecture

```text
Claude Code
  -> Docker container launched by claude-container
  -> Docker default bridge network
  -> Docker daemon in WSL Ubuntu
  -> WSL NAT
  -> Windows host
  -> Clash TUN virtual adapter
  -> Internet
```

SSH access from outside the machine is handled separately:

```text
outside client
  -> Windows host port 2222
  -> WSL Ubuntu port 22
```

Important results:

- WSL is back in NAT mode.
- Windows mirrored networking is disabled.
- WSL does not export `HTTP_PROXY`, `HTTPS_PROXY`, `http_proxy`, or `https_proxy`.
- `apt` does not use an explicit proxy config.
- Docker daemon does not use an explicit proxy config.
- Claude containers use Docker's default bridge network.
- The `claude-container` wrapper does not inject proxy environment variables.
- Claude Code config is persisted outside the temporary container.

## Windows Host Requirements

Use Clash / Mihomo with TUN mode enabled on Windows.

Recommended state:

```text
Clash TUN: enabled
System traffic: routed through TUN
WSL networking mode: NAT
WSL mirrored networking: disabled
WSL autoProxy: disabled
WSL dnsTunneling: disabled
```

The current Windows user-level WSL config is:

```ini
[wsl2]
  autoProxy=false
  dnsTunneling=false
```

On this machine the file is:

```text
/mnt/c/Users/13988/.wslconfig
```

If `.wslconfig` is changed, restart WSL from Windows PowerShell:

```powershell
wsl --shutdown
```

## SSH Access

WSL's OpenSSH server listens on port `22`.

Windows forwards external port `2222` to WSL port `22`.

The WSL SSH daemon config should keep port `22`:

```text
/etc/ssh/sshd_config
```

Relevant line:

```text
Port 22
```

Do not use `http_proxy` for SSH. `http_proxy` is only for HTTP-compatible proxy clients and should not point at port `22`.

## GitHub SSH Access

With Clash TUN and fake-ip routing, `github.com:22` can be routed to a Clash fake IP such as `198.18.x.x` and closed before SSH key exchange. GitHub provides an SSH-over-443 endpoint for this case.

Configure `~/.ssh/config` so all normal GitHub SSH remotes use port `443` automatically:

```sshconfig
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
  AddKeysToAgent yes
```

Add the GitHub SSH-over-443 host key if it is not already present:

```bash
ssh-keygen -F '[ssh.github.com]:443' -f ~/.ssh/known_hosts || \
  printf '%s\n' '[ssh.github.com]:443 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl' >> ~/.ssh/known_hosts
```

Verify:

```bash
ssh -T git@github.com
```

Expected:

```text
Hi <user>! You've successfully authenticated, but GitHub does not provide shell access.
```

After this, ordinary SSH remotes keep working without changing remote URLs:

```bash
git clone git@github.com:owner/repo.git
git fetch
git pull
git push
```

HTTPS remotes already use port `443` and do not need this SSH config.

## WSL System Setup

`/etc/wsl.conf` should enable systemd and set the default user:

```ini
[boot]
systemd=true

[user]
default=kona
```

After changing this file, restart WSL from Windows PowerShell:

```powershell
wsl --shutdown
```

## Shell Environment

Do not set proxy variables in `~/.bashrc` for the normal TUN setup.

These should be absent unless a specific tool needs a temporary proxy:

```bash
HTTP_PROXY
HTTPS_PROXY
ALL_PROXY
NO_PROXY
http_proxy
https_proxy
all_proxy
no_proxy
```

Check:

```bash
printenv | rg -i '^(http|https|all|no)_proxy=|^(HTTP|HTTPS|ALL|NO)_PROXY=' || true
```

Expected: no output.

The `.bashrc` still sets the local Claude image and PATH:

```bash
# Use the locally rebuilt Claude Code image. Rebuild it from
# ~/claude-projects/claude-container when updating Claude Code.
export CLAUDE_IMAGE="kona/claude-container:latest"

if [[ ":$PATH:" != *":$HOME/.local/bin:"* ]]; then
    export PATH="$HOME/.local/bin:$PATH"
fi
```

## Install Docker Engine on WSL Ubuntu 24.04

Install prerequisites:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```

Add Docker's official GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker apt source:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

If an old broken Docker source exists, remove it:

```bash
sudo rm -f /etc/apt/sources.list.d/docker.list
```

Install Docker:

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add the user to the Docker group:

```bash
sudo usermod -aG docker "$USER"
```

Restart WSL once after changing group membership:

```powershell
wsl --shutdown
```

## Docker Proxy State

Docker daemon should not have a proxy drop-in for this setup.

These old files should not exist in their active locations:

```text
/etc/systemd/system/docker.service.d/proxy.conf
/etc/apt/apt.conf.d/95wsl-proxy
```

If they exist, remove or archive them:

```bash
sudo mkdir -p ~/network-cleanup-backups

if [ -f /etc/apt/apt.conf.d/95wsl-proxy ]; then
  sudo mv /etc/apt/apt.conf.d/95wsl-proxy ~/network-cleanup-backups/95wsl-proxy.disabled
fi

if [ -f /etc/systemd/system/docker.service.d/proxy.conf ]; then
  sudo mv /etc/systemd/system/docker.service.d/proxy.conf ~/network-cleanup-backups/docker-proxy.conf.disabled
fi

sudo chown -R "$USER:$USER" ~/network-cleanup-backups
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Verify Docker daemon has no proxy environment:

```bash
systemctl show docker --property=Environment
docker info --format '{{json .HTTPProxy}} {{json .HTTPSProxy}} {{json .NoProxy}}'
```

Expected:

```text
Environment=
"" "" ""
```

Verify Docker can pull and run a container:

```bash
docker run --rm hello-world
```

Expected:

```text
Hello from Docker!
```

## apt Proxy State

`apt` should not use an explicit proxy for this setup.

Verify:

```bash
apt-config dump | rg -i 'Acquire::.*Proxy|127\.0\.0\.1:7890' || true
```

Expected: no output.

Then verify package metadata can be refreshed:

```bash
sudo apt-get update
```

## Install Claude Container

Project used:

```text
https://github.com/KONA-159/claude-container
```

This is a fork of `nezhar/claude-container`. It is a Docker-based launcher for Claude Code. It is not Claude Code itself. It is a wrapper around `docker run`.

Install the wrapper:

```bash
mkdir -p ~/.local/bin ~/.local/share/bash-completion/completions
curl -fsSL https://raw.githubusercontent.com/KONA-159/claude-container/main/bin/claude-container -o ~/.local/bin/claude-container
curl -fsSL https://raw.githubusercontent.com/KONA-159/claude-container/main/completions/claude-container -o ~/.local/share/bash-completion/completions/claude-container
chmod +x ~/.local/bin/claude-container
```

Ensure `~/.local/bin` is in PATH via `.bashrc` as shown above.

Clone the fork so the local image can be rebuilt when Claude Code needs an update:

```bash
mkdir -p ~/claude-projects
git clone git@github.com:KONA-159/claude-container.git ~/claude-projects/claude-container
```

Build the local Claude Code image:

```bash
cd ~/claude-projects/claude-container
docker build --pull --no-cache -t kona/claude-container:latest ./claude-code
```

## Claude Container Wrapper Behavior

The wrapper should not use host networking for normal mode. It should also not inject proxy variables.

It still avoids mounting every project to the same container path. Claude Code stores project state by absolute path. If every project appears as `/workspace`, resume history, trust state, and allowed tools can be mixed across unrelated projects.

This fork maps each host workspace to:

```text
/workspaces/<host-project-directory-name>
```

The normal-mode `docker run` behavior should be:

```bash
WORKSPACE_BASENAME="$(basename "$WORKSPACE_DIR")"
CONTAINER_WORKSPACE_DIR="/workspaces/${WORKSPACE_BASENAME}"

docker run --rm -it \
    -v "$WORKSPACE_DIR:$CONTAINER_WORKSPACE_DIR" \
    -v "$CONFIG_DIR:/claude" \
    -w "$CONTAINER_WORKSPACE_DIR" \
    -e "CLAUDE_CONFIG_DIR=/claude" \
    -e "USER_UID=$(id -u)" \
    -e "USER_GID=$(id -g)" \
    "$IMAGE" \
    $COMMAND
```

Verify the active wrapper has no old network settings:

```bash
rg -n '127\.0\.0\.1:7890|--network host|HTTP_PROXY|HTTPS_PROXY|http_proxy|https_proxy|NO_PROXY|no_proxy' ~/.local/bin/claude-container || true
```

Expected: no output.

## Verify Claude Container

Check versions:

```bash
claude-container --version
```

On this machine, it currently reports:

```text
Claude Container: v1.6.12
Docker: 29.4.3
```

The wrapper version is not the same as Claude Code's real version.

Check environment passed into the container:

```bash
claude-container env
```

Expected: no proxy values such as `HTTP_PROXY=http://127.0.0.1:7890`.

## How to Use Claude Code

Use by project:

```bash
cd /home/kona/projects/project-a
claude-container
```

This starts a temporary Docker container. It is not long-running by default.

The wrapper uses:

```text
docker run --rm -it
```

So when Claude exits, the container is removed.

Data persists because these host directories are mounted:

```text
current project directory -> /workspaces/<project-directory-name>
~/.config/claude-container/config -> /claude
```

View Claude Code help:

```bash
claude-container claude --help
```

Resume sessions:

```bash
claude-container claude --continue
claude-container claude --resume
```

Enter the container shell:

```bash
claude-container --shell
```

## Updating Claude Code

Claude Code may show:

```text
Auto-update failed · Try claude doctor or npm i -g @anthropic-ai/claude-code
```

For a bare-metal install, `claude update` or `npm i -g @anthropic-ai/claude-code` is appropriate. In this Docker-based setup, updating inside a temporary container is not reliable because the container is removed after exit.

Instead, update by rebuilding the local image:

```bash
cd ~/claude-projects/claude-container
git pull
docker build --pull --no-cache -t kona/claude-container:latest ./claude-code
claude-container claude --version
```

To pin a specific version:

```bash
docker build --pull --no-cache \
  --build-arg CLAUDE_CODE_VERSION=<version> \
  -t kona/claude-container:latest \
  ./claude-code
```

## Optional `claude` Alias

If you want `claude` to behave like Claude Code:

```bash
echo "alias claude='claude-container claude'" >> ~/.bashrc
source ~/.bashrc
```

Then:

```bash
claude
claude --help
claude --version
```

will go through the container.

## Config Directory Mapping

Official Claude Code docs mention:

```text
~/.claude/
~/.claude.json
```

In this container setup, the wrapper sets:

```bash
-v "$CONFIG_DIR:/claude"
-e "CLAUDE_CONFIG_DIR=/claude"
```

Default host config directory:

```text
~/.config/claude-container/config
```

So this directory acts as the containerized Claude Code user config root.

Examples:

```text
Official user settings:
~/.claude/settings.json

Current container setup:
~/.config/claude-container/config/settings.json
```

```text
Official global state:
~/.claude.json

Current container setup:
~/.config/claude-container/config/.claude.json
```

Project-level config still belongs inside each project:

```text
project/CLAUDE.md
project/.claude/settings.json
project/.claude/settings.local.json
project/.mcp.json
```

For reliable project-specific behavior, start Claude from the project directory.

## Current Claude Config Snapshot

This section records the current containerized Claude Code user config from:

```text
~/.config/claude-container/config
```

Sensitive values are redacted.

### `settings.json`

Current file path:

```text
~/.config/claude-container/config/settings.json
```

Current effective content, with secrets hidden:

```json
{
  "env": {
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0",
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api",
    "ANTHROPIC_AUTH_TOKEN": "<redacted>",
    "ANTHROPIC_API_KEY": "",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek/deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek/deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek/deepseek-v4-flash"
  }
}
```

When recreating this setup on another machine, replace `<redacted>` with the real token if using OpenRouter.

## Project Switching Recommendation

Do not start Claude from `/` or from a huge directory containing unrelated data.

Recommended:

```bash
cd /home/kona/projects/project-a
claude-container
```

Switch projects by exiting Claude and starting it again:

```bash
cd /home/kona/projects/project-b
claude-container
```

Reason:

- `CLAUDE.md` has project/directory loading behavior.
- `.claude/settings.json`, hooks, MCP, and permission settings are most reliable when Claude starts in that project root.
- `--add-dir` is useful for adding access to extra directories, but it should not be treated as full project-root switching.

## Deprecated Old Proxy Setup

Do not restore the old setup unless Clash TUN is unavailable and you intentionally want a local HTTP proxy path.

Deprecated items:

```text
Windows HTTP proxy at 127.0.0.1:7890
WSL mirrored networking for localhost proxy access
~/.bashrc HTTP_PROXY/HTTPS_PROXY exports
/etc/apt/apt.conf.d/95wsl-proxy
/etc/systemd/system/docker.service.d/proxy.conf
claude-container --network host
claude-container proxy env injection
```

The old setup worked around DNS/proxy pollution before host-level TUN routing was fixed. With TUN enabled, explicit proxy configuration creates unnecessary failure points, especially because `127.0.0.1` inside WSL or inside a Docker container is not the Windows host proxy.

## Quick Health Checks

WSL proxy variables:

```bash
printenv | rg -i '^(http|https|all|no)_proxy=|^(HTTP|HTTPS|ALL|NO)_PROXY=' || true
```

Expected: no output.

apt:

```bash
apt-config dump | rg -i 'Acquire::.*Proxy|127\.0\.0\.1:7890' || true
sudo apt-get update
```

Docker daemon:

```bash
systemctl show docker --property=Environment
docker info --format '{{json .HTTPProxy}} {{json .HTTPSProxy}} {{json .NoProxy}}'
docker run --rm hello-world
```

Expected Docker proxy output:

```text
Environment=
"" "" ""
```

Claude container wrapper:

```bash
rg -n '127\.0\.0\.1:7890|--network host|HTTP_PROXY|HTTPS_PROXY|http_proxy|https_proxy|NO_PROXY|no_proxy' ~/.local/bin/claude-container || true
claude-container --version
```

GitHub SSH:

```bash
ssh -T git@github.com
```

## Known Good Versions on This Machine

```text
Ubuntu: 24.04.4 LTS noble
Kernel: WSL2, 6.6.87.2-microsoft-standard-WSL2
Docker Engine: 29.4.3
Docker Compose: v5.1.3
Claude Container wrapper/image: 1.6.12
WSL network mode: NAT
Windows traffic routing: Clash TUN
SSH forwarding: Windows host port 2222 -> WSL port 22
GitHub SSH: git@github.com remapped to ssh.github.com:443
Explicit WSL/Docker/apt HTTP proxy: disabled
```
