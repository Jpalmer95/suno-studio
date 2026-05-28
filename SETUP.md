# Setup Guide — Suno Studio

Complete installation and configuration guide for the Suno Studio skill.

## Prerequisites

- **Hermes Agent** installed and configured
- **Linux/macOS** (Windows WSL2 works)
- **Chromium-based browser** (Brave recommended, Chrome/Chromium also work)
- **Suno account** (free or paid)

## Installation Steps

### Step 1: Install the Skill

**From Hermes Skill Hub:**
```bash
hermes skills install suno-studio
```

**Manual installation:**
```bash
mkdir -p ~/.hermes/skills/creative/suno-studio
curl -o ~/.hermes/skills/creative/suno-studio/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/suno-studio/main/SKILL.md
```

Verify it's installed:
```bash
hermes skills list | grep suno-studio
```

### Step 2: Install a Chromium-Based Browser

**Ubuntu/Debian:**
```bash
# Brave (recommended)
sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] \
  https://brave-browser-apt-release.s3.brave.com/ stable main" | \
  sudo tee /etc/apt/sources.list.d/brave-browser-release.list
sudo apt update
sudo apt install brave-browser

# Or Chromium
sudo apt install chromium-browser

# Or Google Chrome
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | \
  sudo gpg --dearmor -o /usr/share/keyrings/google-chrome.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome.gpg] \
  http://dl.google.com/linux/chrome/deb/ stable main" | \
  sudo tee /etc/apt/sources.list.d/google-chrome.list
sudo apt update
sudo apt install google-chrome-stable
```

**macOS:**
- Download Brave from https://brave.com/
- Or Chrome from https://www.google.com/chrome/

### Step 3: Create the CDP Launch Script

Create `~/.hermes/scripts/launch-brave-cdp.sh`:

```bash
mkdir -p ~/.hermes/scripts
cat > ~/.hermes/scripts/launch-brave-cdp.sh << 'EOF'
#!/bin/bash
# Launch Chromium-based browser in CDP mode with persistent profile
# This keeps Suno (and all other) logins alive across sessions

PORT=9222
DATA_DIR="$HOME/.hermes/chrome-debug"

# Check if already running
if curl -s "http://127.0.0.1:${PORT}/json/version" > /dev/null 2>&1; then
    echo "Browser CDP already running on port ${PORT}"
    curl -s "http://127.0.0.1:${PORT}/json/version" | head -4
    exit 0
fi

# Find browser (try Brave first, then Chrome, then Chromium)
BROWSER=""
for cmd in brave-browser google-chrome chromium-browser chromium; do
    if command -v $cmd >/dev/null 2>&1; then
        BROWSER=$cmd
        break
    fi
done

if [ -z "$BROWSER" ]; then
    echo "ERROR: No Chromium-based browser found"
    echo "Install one with: sudo apt install chromium-browser (or brave-browser)"
    exit 1
fi

# Launch with CDP
mkdir -p "$DATA_DIR"
nohup $BROWSER \
    --remote-debugging-port=${PORT} \
    --user-data-dir="${DATA_DIR}" \
    --no-first-run \
    --no-default-browser-check \
    --disable-background-timer-throttling \
    --disable-backgrounding-occluded-windows \
    --disable-renderer-backgrounding \
    > /dev/null 2>&1 &

BROWSER_PID=$!
echo "Launched $BROWSER CDP (PID: ${BROWSER_PID}) on port ${PORT}"
echo "User data dir: ${DATA_DIR}"

# Wait for it to be ready
for i in {1..10}; do
    if curl -s "http://127.0.0.1:${PORT}/json/version" > /dev/null 2>&1; then
        echo "Browser CDP is ready!"
        exit 0
    fi
    sleep 1
done

echo "WARNING: Browser may not have started properly"
exit 1
EOF

chmod +x ~/.hermes/scripts/launch-brave-cdp.sh
```

### Step 4: Configure Hermes

Tell Hermes to use the CDP browser:

```bash
hermes config set browser.cdp_url "http://127.0.0.1:9222"
```

Verify the config:
```bash
grep cdp_url ~/.hermes/config.yaml
# Should show: cdp_url: http://127.0.0.1:9222
```

### Step 5: Log Into Suno (One-Time)

This is a manual step to authenticate Suno. Your login persists permanently in the CDP browser profile.

```bash
# Start the CDP browser
~/.hermes/scripts/launch-brave-cdp.sh

# Connect to it from a regular browser
# Open: http://127.0.0.1:9222
# This shows the CDP browser's tabs

# In the CDP browser, navigate to: https://suno.com
# Click "Log in" and complete Google/Discord/etc. authentication
# Once logged in, close the connection tab (not the CDP browser itself)
```

Your Suno login now persists in `~/.hermes/chrome-debug/` and will survive reboots.

### Step 6: Test It

Start a new Hermes session:

```bash
hermes
```

Try a simple request:
```
> Make me a lo-fi beat for studying
```

The agent should:
1. Load the suno-studio skill
2. Engineer a detailed Suno prompt
3. Navigate to suno.com/create (you'll see it in the CDP browser window)
4. Fill in Custom Mode fields
5. Click Create
6. Wait 30-90 seconds
7. Return audio URLs and Suno links

Success looks like:
```
✅ Generated: "Midnight Study Session"
   Duration: 3:24
   Audio: https://cdn1.suno.ai/abc123.mp3
   Suno: https://suno.com/song/abc123-def456
```

### Step 7: Auto-Start on Boot (Optional)

Make the CDP browser start automatically on system boot:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/brave-cdp.service << EOF
[Unit]
Description=Brave Browser CDP for Hermes Agent
After=graphical-session.target

[Service]
Type=simple
ExecStart=${HOME}/.hermes/scripts/launch-brave-cdp.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable brave-cdp
systemctl --user start brave-cdp

# Start on boot even without login
sudo loginctl enable-linger $USER
```

Now Brave CDP launches automatically every boot. Verify:
```bash
systemctl --user status brave-cdp
```

## Post-Reboot Checklist

If you didn't set up auto-start, do this after rebooting:

```bash
# 1. Start the CDP browser
~/.hermes/scripts/launch-brave-cdp.sh

# 2. Verify it's running
curl -s http://127.0.0.1:9222/json/version | jq .Browser

# 3. Use normally
hermes
> Generate a synthwave track for coding
```

## Verification Checklist

- [ ] Skill installed (`hermes skills list | grep suno-studio`)
- [ ] Chromium-based browser installed (`which brave-browser`)
- [ ] Launch script created (`~/.hermes/scripts/launch-brave-cdp.sh`)
- [ ] Hermes config updated (`hermes config | grep cdp_url`)
- [ ] CDP browser running (`curl http://127.0.0.1:9222/json/version`)
- [ ] Logged into Suno (manually in CDP browser once)
- [ ] Test generation successful

## Troubleshooting Installation

### "No Chromium-based browser found"
Install Brave, Chrome, or Chromium:
```bash
sudo apt install brave-browser
# or
sudo apt install chromium-browser
```

### Port 9222 already in use
Another process is using the CDP port. Either:
```bash
# Kill the existing process
lsof -i :9222
kill <PID>

# Or use a different port
PORT=9223 ~/.hermes/scripts/launch-brave-cdp.sh
hermes config set browser.cdp_url "http://127.0.0.1:9223"
```

### Hermes can't connect to browser
Verify the CDP browser is actually running:
```bash
curl -s http://127.0.0.1:9222/json/version
# Should return JSON with browser version info

# If not, start it:
~/.hermes/scripts/launch-brave-cdp.sh
```

Check Hermes config:
```bash
hermes config | grep browser
# Should show: cdp_url: http://127.0.0.1:9222
```

### Skill not found
The skill didn't install properly. Reinstall:
```bash
# Remove old
rm -rf ~/.hermes/skills/creative/suno-studio

# Reinstall
hermes skills install suno-studio
# or manual:
mkdir -p ~/.hermes/skills/creative/suno-studio
curl -o ~/.hermes/skills/creative/suno-studio/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/suno-studio/main/SKILL.md
```

Then reload skills in your Hermes session:
```
/reload-skills
```

### "Not logged in" when trying to generate
Your CDP browser isn't authenticated with Suno. Log in manually once:
```bash
~/.hermes/scripts/launch-brave-cdp.sh
# Open http://127.0.0.1:9222 in a regular browser
# Navigate to suno.com and log in with Google/Discord/etc.
```

## Updating the Skill

**From Hermes Skill Hub:**
```bash
hermes skills check
hermes skills update
```

**Manual:**
```bash
curl -o ~/.hermes/skills/creative/suno-studio/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/suno-studio/main/SKILL.md
```

Then reload in your session:
```
/reload-skills
```

## Uninstalling

```bash
# Remove skill
rm -rf ~/.hermes/skills/creative/suno-studio

# Remove browser profile (optional, deletes saved logins)
rm -rf ~/.hermes/chrome-debug

# Remove launch script (optional)
rm ~/.hermes/scripts/launch-brave-cdp.sh

# Remove systemd service (if configured)
systemctl --user disable brave-cdp
rm ~/.config/systemd/user/brave-cdp.service

# Remove Hermes config
hermes config set browser.cdp_url ""

# Reload skills
hermes  # then /reload-skills in session
```

## Next Steps

- Read the [full documentation](README.md) for advanced usage
- Try [example prompts](EXAMPLES.md) to see what's possible
- Check out the [songwriting skill](../../creative/songwriting-and-ai-music/) for deeper prompt engineering knowledge
