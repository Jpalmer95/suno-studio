# How to Use Suno Studio

Quick guide for Hermes Agent users who want to generate AI music via Suno.

## What is Suno Studio?

Suno Studio is a Hermes Agent skill that automates music generation on [Suno](https://suno.com) using expert prompt engineering and browser automation. Just describe what you want in natural language, and the agent handles the rest.

## Installation (3 Steps)

### 1. Install the Skill

**Option A: Clone the repo** (recommended)
```bash
cd ~/.hermes/skills/creative/
git clone https://github.com/Jpalmer95/suno-studio.git suno-studio-repo
cp -r suno-studio-repo/creative/suno-studio .
rm -rf suno-studio-repo
```

**Option B: Download just the skill file**
```bash
mkdir -p ~/.hermes/skills/creative/suno-studio
curl -o ~/.hermes/skills/creative/suno-studio/SKILL.md \
  https://raw.githubusercontent.com/Jpalmer95/suno-studio/main/creative/suno-studio/SKILL.md
```

### 2. Set Up Persistent Browser

Suno Studio requires a browser running in CDP (Chrome DevTools Protocol) mode with persistent authentication.

**Create the launch script:**
```bash
mkdir -p ~/.hermes/scripts

cat > ~/.hermes/scripts/launch-browser-cdp.sh << 'EOF'
#!/bin/bash
PORT=9222
DATA_DIR="$HOME/.hermes/chrome-debug"

# Check if already running
if curl -s "http://127.0.0.1:${PORT}/json/version" > /dev/null 2>&1; then
    echo "Browser CDP already running on port ${PORT}"
    exit 0
fi

# Find browser (try Brave first, then Chrome, then Chromium)
BROWSER=""
for cmd in brave-browser google-chrome chromium chromium-browser; do
    if command -v $cmd >/dev/null 2>&1; then
        BROWSER=$cmd
        break
    fi
done

if [ -z "$BROWSER" ]; then
    echo "ERROR: No Chromium-based browser found"
    echo "Install one with: sudo apt install chromium-browser"
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

echo "Launched $BROWSER on CDP port ${PORT} (PID: $!)"
EOF

chmod +x ~/.hermes/scripts/launch-browser-cdp.sh
```

**Configure Hermes:**
```bash
hermes config set browser.cdp_url "http://127.0.0.1:9222"
```

### 3. Log Into Suno (One-Time)

```bash
# Start the CDP browser
~/.hermes/scripts/launch-browser-cdp.sh

# Connect to it from a regular browser
xdg-open http://127.0.0.1:9222  # Linux
# or: open http://127.0.0.1:9222  # macOS
# or: start http://127.0.0.1:9222  # Windows

# In the CDP browser, go to suno.com and log in with Google/Discord/etc.
# Your login persists in ~/.hermes/chrome-debug/ forever
```

## Usage

Start a new Hermes session and ask for music:

```bash
hermes
```

### Basic Examples

```
> Make me a lo-fi beat for studying
> Generate a pop song about summer love
> Create ambient music for meditation
> I need a rock anthem about coding at 3am
```

### Detailed Examples

```
> Make a jazz noir track, smoky saxophone, upright bass, brushed drums,
  70 BPM, minor key, starts sparse, builds to full trio

> Generate an epic orchestral piece, 90 BPM, starts with solo cello,
  layers in strings, builds to full orchestra with brass fanfare,
  male baritone vocalist, cinematic and heroic
```

### DJ Mode

```
> DJ mode: 4-song coffee shop set, acoustic vibes
> DJ set: 5-song workout playlist, starts with warm-up tempo, builds to peak cardio, then cool down
> Make me a 3-song podcast intro/outro set: opener, transition stinger, closer
```

### Advanced Options

```
> Generate a track with weirdness at 80, really experimental
> Make this with style influence at 90, stick closely to the prompt
> Set weirdness low (20) but high style influence (80) for polished results
```

## After Reboot

The CDP browser doesn't survive reboots. After restart, run:

```bash
~/.hermes/scripts/launch-browser-cdp.sh
```

Then use normally. The script is idempotent — safe to run when already active.

**Alternative:** Use Hermes' built-in `/browser connect` command.

**Auto-start on boot (optional):**

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/browser-cdp.service << EOF
[Unit]
Description=Browser CDP for Hermes Agent
After=graphical-session.target

[Service]
Type=simple
ExecStart=${HOME}/.hermes/scripts/launch-browser-cdp.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable browser-cdp
systemctl --user start browser-cdp
sudo loginctl enable-linger $USER  # Start on boot, not just login
```

## How It Works

1. You describe what you want ("lo-fi beats for coding")
2. Hermes loads `suno-studio` + `songwriting-and-ai-music` skills
3. Agent engineers an expert Suno prompt with:
   - Evocative title (≤8 words)
   - Detailed style description (up to 1000 chars, describes dynamic journey)
   - Lyrics with proper [metatags] ([Intro], [Verse], [Chorus], etc.)
   - More Options (weirdness, style influence, exclusions)
4. Browser automation drives suno.com/create:
   - Navigates to the page
   - Enables Custom Mode
   - Fills all fields
   - Clicks Create
   - Polls for completion (30-90 seconds)
   - Extracts audio URLs
5. Returns direct links to your generated music

## Requirements

- **Hermes Agent** installed and configured
- **Chromium-based browser** (Brave, Chrome, or Chromium)
- **Suno account** (free tier: 10 songs/day, Pro: 50-200/day)
- **Linux/macOS/WSL2** (Windows native not tested)

## Troubleshooting

### "Not logged in" error
```bash
~/.hermes/scripts/launch-browser-cdp.sh
xdg-open http://127.0.0.1:9222
# Log into Suno manually once
```

### Browser won't start
```bash
# Check if port 9222 is in use
lsof -i :9222
# Kill the process or change port
```

### Generation times out
Suno can be slow during peak hours. Wait a few minutes and retry.

### CAPTCHA appears
Solve it manually in the CDP browser (http://127.0.0.1:9222), then retry.

## Comparison: Suno vs HeartMuLa

| Feature | Suno (this skill) | HeartMuLa (local) |
|---------|------------------|-------------------|
| **Quality** | State-of-art (v4.5/v5) | Good (3B params) |
| **Auth** | Suno account required | None |
| **Cost** | Free: 10/day, Pro: 50-200/day | Unlimited (local GPU) |
| **Speed** | 30-90 seconds | ~4 minutes |
| **Offline** | No | Yes |

**Recommendation:** Use Suno for quality, HeartMuLa for unlimited generations or when Suno is down.

## More Information

- [Full Documentation](README.md)
- [Setup Guide](SETUP.md)
- [Example Prompts](EXAMPLES.md)
- [Skill File](creative/suno-studio/SKILL.md)

## Contributing

Found a bug or have an idea? Open an issue or PR on [GitHub](https://github.com/Jpalmer95/suno-studio).

## License

MIT — use freely, modify as needed.

## Credits

- Built with [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research
- Prompt engineering from `songwriting-and-ai-music` skill
- Browser automation via Hermes `browser_*` tools
- Music generation by [Suno](https://suno.com)
