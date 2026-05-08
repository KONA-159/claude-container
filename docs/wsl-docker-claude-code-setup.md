# WSL + Docker + Claude Code Container Setup

This document records the setup used on `kona`'s WSL Ubuntu 24.04 machine, from a fresh WSL environment to a working Docker-based Claude Code workflow.

The goal is to make the same setup reproducible on another Windows + WSL machine.

## Final Architecture

```text
Claude Code
  -> Docker container launched by claude-container
  -> Docker host network
  -> WSL Ubuntu
  -> Windows host proxy at 127.0.0.1:7890
  -> FlClash rule proxy
  -> Internet
```

Important result:

- WSL shell traffic goes through the Windows proxy.
- `apt` goes through the Windows proxy.
- Docker daemon image pulls go through the Windows proxy.
- Claude Code runs inside a Docker container.
- Claude Code config is persisted outside the container.

## Problems We Hit

### 1. apt `NOSPLIT`

`sudo apt update` failed with errors like:

```text
Clearsigned file isn't valid, got 'NOSPLIT'
The repository 'http://archive.ubuntu.com/ubuntu noble InRelease' is no longer signed.
```

This was not an Ubuntu repository problem. It meant WSL direct network access was receiving invalid content, likely due to network/proxy/DNS interference.

Fix: route apt through the working Windows proxy.

### 2. Docker source file malformed

At one point Docker's apt source was written across multiple lines:

```text
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg]
  https://download.docker.com/linux/ubuntu noble stable
```

That produced:

```text
Malformed entry ... docker.list
```

Fix: remove the broken old-style file and use Docker's official `.sources` format.

### 3. Windows proxy only listened on localhost

Initially FlClash listened only on:

```text
127.0.0.1:7890
```

WSL could not reliably reach it. After enabling LAN access / binding to all interfaces, Windows showed:

```powershell
netstat -ano | findstr LISTENING | findstr :7890
```

Expected output:

```text
TCP    0.0.0.0:7890    0.0.0.0:0    LISTENING
TCP    [::]:7890       [::]:0       LISTENING
```

### 4. Docker daemon did not inherit shell proxy

Even when `curl` worked in WSL, Docker image pulls failed:

```text
failed to resolve reference "docker.io/library/hello-world:latest"
dial tcp 157.240...:443: i/o timeout
```

Those IPs were clearly wrong for Docker Hub, indicating DNS/network pollution. The Docker daemon pulls images itself, so shell environment variables were not enough.

Fix: configure the Docker systemd service with proxy environment variables.

### 5. Containers could not use ordinary Docker bridge networking for proxy

Inside containers:

- Direct internet access timed out.
- `host.docker.internal:7890` did not work.
- `--network host` plus `127.0.0.1:7890` worked.

Fix: patch the local `claude-container` wrapper to use `--network host` and pass proxy variables.

## Windows Host Requirements

Use a proxy client such as FlClash / Clash Verge Rev / Mihomo Party.

Required proxy settings:

```text
Allow LAN: enabled
Listen address / bind address: 0.0.0.0
Mixed or HTTP port: 7890
Mode: rule proxy is fine
```

Verify on Windows PowerShell:

```powershell
netstat -ano | findstr LISTENING | findstr :7890
```

Good:

```text
0.0.0.0:7890
[::]:7890
```

Bad:

```text
127.0.0.1:7890
```

Also verify the proxy itself works on Windows:

```powershell
curl.exe -v -x http://127.0.0.1:7890 https://registry-1.docker.io/v2/
```

Expected result includes:

```text
HTTP/1.1 200 Connection established
HTTP/1.1 401 Unauthorized
```

`401 Unauthorized` is a good result for Docker Registry. It means the registry is reachable and asking for auth.

## WSL Proxy Setup

In this environment, WSL uses mirrored/shared networking, so WSL can use:

```text
127.0.0.1:7890
```

Test from WSL:

```bash
curl -I --max-time 10 -x http://127.0.0.1:7890 https://registry-1.docker.io/v2/
```

Expected:

```text
HTTP/1.1 200 Connection established
HTTP/2 401
```

Add this to `~/.bashrc`:

```bash
# Use the Windows host proxy from WSL. This works with WSL mirrored networking
# when the Windows proxy client listens on 0.0.0.0:7890.
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="$HTTP_PROXY"
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export NO_PROXY="localhost,127.0.0.1,::1"
export no_proxy="$NO_PROXY"

# Use the locally rebuilt Claude Code image. Rebuild it from
# ~/claude-projects/claude-container when updating Claude Code.
export CLAUDE_IMAGE="kona/claude-container:latest"

if [[ ":$PATH:" != *":$HOME/.local/bin:"* ]]; then
    export PATH="$HOME/.local/bin:$PATH"
fi
```

Reload:

```bash
source ~/.bashrc
```

Avoid large `NO_PROXY` values such as:

```text
192.168.*,10.*,172.*
```

Those can make tools bypass the proxy unexpectedly.

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

Make sure WSL uses systemd. `/etc/wsl.conf` should contain:

```ini
[boot]
systemd=true

[user]
default=kona
```

From Windows PowerShell, restart WSL if needed:

```powershell
wsl --shutdown
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

Restart WSL once after this:

```powershell
wsl --shutdown
```

## Configure Docker Daemon Proxy

Docker daemon needs its own proxy config.

Create:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Write `/etc/systemd/system/docker.service.d/proxy.conf`:

```bash
sudo tee /etc/systemd/system/docker.service.d/proxy.conf > /dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,::1"
EOF
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Verify:

```bash
sudo systemctl show --property=Environment docker
```

Expected:

```text
Environment=HTTP_PROXY=http://127.0.0.1:7890 HTTPS_PROXY=http://127.0.0.1:7890 NO_PROXY=localhost,127.0.0.1,::1
```

Verify Docker:

```bash
docker run --rm hello-world
```

Expected:

```text
Hello from Docker!
```

If group membership has not taken effect yet, use:

```bash
sudo docker run --rm hello-world
```

## Configure apt Proxy

Create `/etc/apt/apt.conf.d/95wsl-proxy`:

```bash
sudo tee /etc/apt/apt.conf.d/95wsl-proxy > /dev/null <<'EOF'
Acquire::http::Proxy "http://127.0.0.1:7890";
Acquire::https::Proxy "http://127.0.0.1:7890";
EOF
```

Verify:

```bash
sudo apt-get update
```

Expected: no `NOSPLIT`, no Docker TLS handshake errors.

## Install Claude Container

Project used:

```text
https://github.com/KONA-159/claude-container
```

This is a fork of `nezhar/claude-container` with the local WSL adaptations already applied. It is a Docker-based launcher for Claude Code. It is not Claude Code itself. It is a wrapper around `docker run`.

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

## WSL Adaptations Built Into This Fork

In this WSL setup, ordinary Docker bridge networking cannot reach the Windows proxy, so the forked launcher uses host networking in normal mode.

Also avoid mounting every project to the same container path, such as `/workspace`. Claude Code stores project state by absolute path. If every project appears as `/workspace`, resume history, trust state, and allowed tools can be mixed across unrelated projects.

This fork maps each host workspace to:

```text
/workspaces/<host-project-directory-name>
```

The relevant normal-mode `docker run` behavior is:

```bash
WORKSPACE_BASENAME="$(basename "$WORKSPACE_DIR")"
CONTAINER_WORKSPACE_DIR="/workspaces/${WORKSPACE_BASENAME}"

docker run --rm -it \
    --network host \
    -v "$WORKSPACE_DIR:$CONTAINER_WORKSPACE_DIR" \
    -v "$CONFIG_DIR:/claude" \
    -w "$CONTAINER_WORKSPACE_DIR" \
    -e "CLAUDE_CONFIG_DIR=/claude" \
    -e "HTTP_PROXY=${HTTP_PROXY:-http://127.0.0.1:7890}" \
    -e "HTTPS_PROXY=${HTTPS_PROXY:-http://127.0.0.1:7890}" \
    -e "http_proxy=${http_proxy:-http://127.0.0.1:7890}" \
    -e "https_proxy=${https_proxy:-http://127.0.0.1:7890}" \
    -e "NO_PROXY=localhost,127.0.0.1,::1" \
    -e "no_proxy=localhost,127.0.0.1,::1" \
    -e "USER_UID=$(id -u)" \
    -e "USER_GID=$(id -g)" \
    "$IMAGE" \
    $COMMAND
```

## Verify Claude Container

Check versions:

```bash
claude-container --version
```

On this machine, it reported:

```text
Claude Container: v1.6.12
Docker: 29.4.3
```

The wrapper version is not the same as Claude Code's real version.

Check real Claude Code version inside the container:

```bash
claude-container claude --version
```

On this machine:

```text
2.1.133 (Claude Code)
```

Check environment passed into the container:

```bash
claude-container env
```

Expected proxy values:

```text
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
```

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

Command breakdown:

```text
docker build
```

Builds a Docker image from a Dockerfile.

```text
--pull
```

Before building, Docker tries to pull the latest base image. In this project the base image is:

```dockerfile
FROM node:22-alpine
```

```text
--no-cache
```

Forces Docker to rerun every build step instead of reusing cached layers. This matters because the Dockerfile installs `@anthropic-ai/claude-code@latest`; without `--no-cache`, Docker may reuse an old npm install layer.

```text
-t kona/claude-container:latest
```

Tags the newly built image as:

```text
kona/claude-container:latest
```

This is the local image used by the launcher through:

```bash
export CLAUDE_IMAGE="kona/claude-container:latest"
```

```text
./claude-code
```

Uses `./claude-code` as the build context. Docker reads:

```text
./claude-code/Dockerfile
```

and includes files from that directory, such as `entrypoint.sh`.

The fork's `claude-code/Dockerfile` installs:

```dockerfile
ARG CLAUDE_CODE_VERSION=latest
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}
```

Without a build arg, `latest` is resolved at image build time. For example, if npm's latest version is `2.1.133` when the image is built, this image will keep using `2.1.133` until it is rebuilt. It will not update automatically when npm publishes a newer release.

To pin a specific version:

```bash
docker build --pull --no-cache \
  --build-arg CLAUDE_CODE_VERSION=2.1.133 \
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

### `.claude.json`

Current file path:

```text
~/.config/claude-container/config/.claude.json
```

This file is Claude Code's global state/config file in this containerized setup. The equivalent official bare-metal path is usually:

```text
~/.claude.json
```

Current content with identifiers, API metrics, and session values hidden:

```json
{
  "numStartups": 1,
  "tipsHistory": {
    "new-user-warmup": "<redacted>",
    "plan-mode-for-complex-tasks": 1
  },
  "hasCompletedOnboarding": true,
  "firstStartTime": "2026-05-08T14:29:37.325Z",
  "userID": "<redacted>",
  "opusProMigrationComplete": true,
  "sonnet1m45MigrationComplete": true,
  "cachedChromeExtensionInstalled": false,
  "changelogLastFetched": 1778250577946,
  "projects": {
    "/workspace": {
      "allowedTools": [],
      "mcpContextUris": [],
      "mcpServers": {},
      "enabledMcpjsonServers": [],
      "disabledMcpjsonServers": [],
      "hasTrustDialogAccepted": true,
      "projectOnboardingSeenCount": 1,
      "hasClaudeMdExternalIncludesApproved": false,
      "hasClaudeMdExternalIncludesWarningShown": false,
      "lastCost": 0,
      "lastAPIDuration": "<redacted>",
      "lastAPIDurationWithoutRetries": "<redacted>",
      "lastToolDuration": 0,
      "lastDuration": 9910,
      "lastLinesAdded": 0,
      "lastLinesRemoved": 0,
      "lastTotalInputTokens": "<redacted>",
      "lastTotalOutputTokens": "<redacted>",
      "lastTotalCacheCreationInputTokens": "<redacted>",
      "lastTotalCacheReadInputTokens": "<redacted>",
      "lastTotalWebSearchRequests": 0,
      "lastFpsAverage": 2.83,
      "lastFpsLow1Pct": 23.31,
      "lastModelUsage": {},
      "lastSessionId": "<redacted>",
      "lastSessionMetrics": "<redacted>"
    }
  },
  "lastReleaseNotesSeen": "2.1.69",
  "officialMarketplaceAutoInstallAttempted": true,
  "officialMarketplaceAutoInstalled": true
}
```

Note: this snapshot was created before the wrapper was changed to mount projects under `/workspaces/<project-directory-name>`. New sessions should appear under project-specific paths instead of all sharing `/workspace`.

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

## Temporary sudo Access Used During Setup

During setup, passwordless sudo was enabled with:

```bash
sudo sh -c 'printf "%s\n" "kona ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/99-kona-codex && chmod 0440 /etc/sudoers.d/99-kona-codex && visudo -cf /etc/sudoers.d/99-kona-codex'
```

After setup, remove it:

```bash
sudo rm -f /etc/sudoers.d/99-kona-codex
```

## Quick Health Checks

Windows proxy:

```powershell
netstat -ano | findstr LISTENING | findstr :7890
curl.exe -v -x http://127.0.0.1:7890 https://registry-1.docker.io/v2/
```

WSL proxy:

```bash
curl -I --max-time 10 https://registry-1.docker.io/v2/
```

apt:

```bash
sudo apt-get update
```

Docker daemon proxy:

```bash
sudo systemctl show --property=Environment docker
docker run --rm hello-world
```

Claude container:

```bash
claude-container claude --version
claude-container env
```

## Known Good Versions on This Machine

```text
Ubuntu: 24.04.4 LTS noble
Kernel: WSL2, 6.6.87.2-microsoft-standard-WSL2
Docker Engine: 29.4.3
Docker Compose: v5.1.3
Claude Container wrapper/image: 1.6.12
Claude Code inside local container image: 2.1.133
Proxy port: 7890
```
