# Suno Studio — AI Music Generation for Hermes Agent

Generate songs on [Suno](https://suno.com) using natural language prompts through Hermes Agent's browser automation. No API keys needed — just a persistent browser session logged into Suno.

## What It Does

Transform vague music requests into expertly-crafted Suno prompts with proper `[metatags]`, dynamic arcs, and detailed style descriptions — then automates the Suno web UI to generate and retrieve your songs.

### Single Song Mode
```
You: "Make me a lo-fi beat for late night coding"

Agent: Engineers a full Suno prompt with:
- Title: "Midnight Compiler"  
- Style: "lo-fi hip hop, mellow jazz piano, vinyl crackle, 75 BPM, 
         starts sparse with muted keys, layers in dusty drums..."
- Lyrics: [Intro] [Verse] [Chorus] structure with instrumental tags
- Generates on suno.com/create
- Returns audio URLs + Suno links
```

### DJ Mode
```
You: "DJ mode: 4-song set starting chill, building to peak, then wind down"

Agent: Plans energy arc:
  Song 1: Ambient intro (energy 2-3)
  Song 2: Groove warm-up (energy 5-6)  
  Song 3: Peak energy (energy 9-10)
  Song 4: Cool down outro (energy 3-4)
  
Generates all 4 sequentially, returns the full setlist with links
```

## Features

- **Expert prompt engineering** — Uses [metatags], dynamic journey descriptions, vocal personas, and all Suno best practices
- **Persistent browser auth** — Log into Suno once, use forever (via CDP browser)
- **Custom Mode automation** — Fills Title, Style, Lyrics, and More Options fields
- **DJ queue planning** — Energy arcs, BPM matching, key transitions
- **Retrieve & download** — Extracts audio URLs and Suno links from completed generations
- **Fallback to HeartMuLa** — Open-source local alternative when Suno is down

## Requirements

1. **Hermes Agent** with browser toolset enabled
2. **Chromium-based browser** (Brave, Chrome, Chromium) with CDP support
3. **Suno account** (free tier works, Pro recommended for more generations)

## Quick Start

### 1. Install the Skill

**Option A: From Hermes Skill Hub** (recommended)
```bash
hermes skills install suno-studio
```

**Option B: Manual installation**
```bash
# Create skills directory
mkdir -p ~/.hermes/skills/creative/suno-studio

# Download the skill file
curl -o ~/.hermes/skills/creative/suno-studio/SKILL.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/suno-studio/main/SKILL.md
```

### 2. Set Up Persistent Browser

The skill requires a browser running in CDP (Chrome DevTools Protocol) mode with a persistent profile. This keeps your Suno login alive across sessions.

**Create the launch script:**
```bash
mkdir -p ~/.hermes/scripts
cat > ~/.hermes/scripts/launch-brave-cdp.sh << 'EOF'
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
for cmd in brave-browser google-chrome chromium-browser chromium; do
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
    > /dev/null 2>&1 &

echo "Launched $BROWSER on CDP port ${PORT} (PID: $!)"

# Wait for ready
for i in {1..10}; do
    if curl -s "http://127.0.0.1:${PORT}/json/version" > /dev/null 2>&1; then
        echo "Browser CDP is ready!"
        exit 0
    fi
    sleep 1
done

echo "WARNING: Browser may not have started properly"
EOF

chmod +x ~/.hermes/scripts/launch-brave-cdp.sh
```

**Configure Hermes to use it:**
```bash
hermes config set browser.cdp_url "http://127.0.0.1:9222"
```

### 3. Log Into Suno (One-Time)

```bash
# Start the CDP browser
~/.hermes/scripts/launch-brave-cdp.sh

# Open the CDP browser in a regular browser tab
# Navigate to: http://127.0.0.1:9222
# This connects you to the CDP instance

# In the CDP browser, go to suno.com and log in with Google/Discord/etc.
# Your login persists in ~/.hermes/chrome-debug/ forever
```

### 4. Use It

Start a new Hermes session and ask for music:
```
> Make me a synthwave track for a midnight drive through neon cities
> Generate a country song about small town sunsets
> DJ mode: 5-song coffee shop set, acoustic vibes
> Create something epic and orchestral, like a movie trailer
```

## Usage Examples

### Basic Prompts
```
"Make me lo-fi beats for studying"
"Generate a pop song about summer love"
"Create ambient music for meditation"
"I need a rock anthem about coding at 3am"
```

### Detailed Prompts
```
"Make a jazz noir track, smoky saxophone, upright bass, brushed drums,
70 BPM, minor key, starts sparse, builds to full trio"

"Generate an epic orchestral piece, 90 BPM, starts with solo cello,
layers in strings, builds to full orchestra with brass fanfare,
male baritone vocalist, cinematic and heroic"
```

### DJ Mode
```
"DJ mode: 4-song workout playlist, starts with warm-up tempo,
builds to peak cardio, then cool down"

"DJ set: 6 songs for a dinner party, start with chill jazz,
progress to upbeat funk, end with smooth soul"

"Make me a 3-song podcast intro/outro set: opener, transition stinger, closer"
```

### Advanced Options
```
"Generate a track with weirdness at 80, really experimental"
"Make this with style influence at 90, stick closely to the prompt"
"Set weirdness low (20) but high style influence (80) for polished results"
```

## Post-Reboot Setup

After rebooting your machine, just run the launch script once:

```bash
~/.hermes/scripts/launch-brave-cdp.sh
```

Then use normally. The script is idempotent — running it when the browser is already active just confirms it's ready.

**Alternative:** Use Hermes' built-in `/browser connect` command in any session.

**Auto-start on boot (optional):** Create a systemd service:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/brave-cdp.service << 'EOF'
[Unit]
Description=Brave Browser CDP for Hermes Agent
After=graphical-session.target

[Service]
Type=simple
ExecStart=/home/jonathan/.hermes/scripts/launch-brave-cdp.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable brave-cdp
systemctl --user start brave-cdp
sudo loginctl enable-linger $USER  # Start on boot, not just login
```

## How It Works

```
1. Load suno-studio + songwriting-and-ai-music skills
2. Parse your intent → extract concept, mood, genre, specifics
3. Engineer expert Suno prompt:
   - Title (evocative, ≤8 words)
   - Style field (up to 1000 chars, describes dynamic journey)
   - Lyrics with [metatags] ([Intro], [Verse], [Chorus], etc.)
   - More Options (weirdness, style influence, exclusions)
4. Browser automation:
   - Navigate to suno.com/create
   - Enable Custom Mode
   - Fill all fields
   - Click Create
   - Poll for completion (30-90 seconds)
   - Extract audio URLs
5. Return results with direct links
```

## Troubleshooting

### "Not logged in" error
The CDP browser isn't logged into Suno. Solution:
```bash
~/.hermes/scripts/launch-brave-cdp.sh
# Then open http://127.0.0.1:9222 in a regular browser
# Navigate to suno.com and log in manually once
```

### Browser won't start
Check if port 9222 is already in use:
```bash
lsof -i :9222
# If something's using it, kill the process or use a different port
```

### Generation times out
Suno can be slow during peak hours. The skill polls for up to 3 minutes. If it times out:
- Check suno.com manually for status
- Try again later
- Use HeartMuLa as fallback (local, no rate limits)

### CAPTCHA appears
Suno sometimes triggers hCaptcha. The browser tools attempt to solve it, but if stuck:
- Solve it manually in the CDP browser (http://127.0.0.1:9222)
- Then retry the generation

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Hermes Agent Session                           │
│  (CLI, Telegram, Discord, etc.)                 │
└─────────┬───────────────────────────────────────┘
          │
          │ Loads skills:
          │ - suno-studio
          │ - songwriting-and-ai-music
          ▼
┌─────────────────────────────────────────────────┐
│  Skill Intelligence Layer                       │
│  - Parse user intent                            │
│  - Engineer Suno prompt with [metatags]         │
│  - Plan DJ queue (if DJ mode)                   │
└─────────┬───────────────────────────────────────┘
          │
          │ Uses browser_* tools:
          │ - browser_navigate
          │ - browser_click
          │ - browser_type
          │ - browser_snapshot
          ▼
┌─────────────────────────────────────────────────┐
│  Browser Automation Layer                       │
│  - CDP protocol on port 9222                    │
│  - Persistent profile: ~/.hermes/chrome-debug/  │
└─────────┬───────────────────────────────────────┘
          │
          │ CDP connection
          ▼
┌─────────────────────────────────────────────────┐
│  Brave CDP Browser                              │
│  - Logged into suno.com (persistent auth)       │
│  - Drives web UI programmatically               │
└─────────────────────────────────────────────────┘
```

## Configuration

All settings in `~/.hermes/config.yaml`:

```yaml
browser:
  cdp_url: http://127.0.0.1:9222  # Persistent browser via CDP
```

Environment variables (in `~/.hermes/.env`):

```bash
# Optional: Override CDP URL
BROWSER_CDP_URL=http://127.0.0.1:9222
```

## Comparison: Suno vs HeartMuLa

| Feature | Suno (via this skill) | HeartMuLa (local) |
|---------|----------------------|-------------------|
| **Quality** | State-of-art (v4.5/v5) | Good (3B params) |
| **Auth** | Suno account required | None |
| **Cost** | Free tier: 10 songs/day<br>Pro: 50-200/day | Unlimited (local GPU) |
| **Speed** | 30-90 seconds | ~4 minutes |
| **Lyrics** | Excellent | Good |
| **Control** | Full (Custom Mode) | Lyrics + tags only |
| **Offline** | No | Yes (RTX 4070 Ti+) |
| **Setup** | CDP browser + login | GPU + model download |

**Recommendation:** Use Suno for quality, HeartMuLa for unlimited generations or when Suno is down.

## Contributing

Found a bug or have an idea? Open an issue or PR on GitHub.

## License

MIT — use freely, modify as needed.

## Credits

- Built with [Hermes Agent](https://github.com/NousResearch/hermes-agent)
- Prompt engineering from `songwriting-and-ai-music` skill
- Browser automation via Hermes `browser_*` tools
- Music generation by [Suno](https://suno.com)
