# VoxType + ydotool — Voice Typing on Ubuntu

Hold a key, speak, release — text appears at your cursor. Uses [VoxType](https://voxtype.io) for speech recognition (via Whisper) and [ydotool](https://github.com/ReimuNotMoe/ydotool) to type the result into any application.

## Prerequisites

- Ubuntu (tested on 24.04)
- A working microphone
- `curl` installed (`sudo apt install curl`)
- systemd user sessions enabled (default on Ubuntu desktop)

## Setup

### 1. Install ydotool (prebuilt binaries)

```bash
sudo curl -L -o /usr/local/bin/ydotool \
  https://github.com/ReimuNotMoe/ydotool/releases/download/v1.0.4/ydotool-release-ubuntu-latest

sudo curl -L -o /usr/local/bin/ydotoold \
  https://github.com/ReimuNotMoe/ydotool/releases/download/v1.0.4/ydotoold-release-ubuntu-latest

sudo chmod +x /usr/local/bin/ydotool /usr/local/bin/ydotoold
```

### 2. Configure uinput permissions

```bash
# Add yourself to the input group
sudo usermod -aG input $USER

# Create udev rule for uinput access
sudo tee /etc/udev/rules.d/80-uinput.rules << 'EOF'
KERNEL=="uinput", GROUP="input", MODE="0660"
EOF

# Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger /dev/uinput
```

### 3. Create ydotool user service

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/ydotool.service << 'EOF'
[Unit]
Description=ydotool daemon (user)

[Service]
ExecStart=/usr/local/bin/ydotoold
Restart=always

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable ydotool
```

### 4. Install VoxType

Download the latest `.deb` from https://github.com/peteonrails/voxtype/releases

```bash
curl -L -o /tmp/voxtype_latest.deb \
  https://github.com/peteonrails/voxtype/releases/download/v0.6.2/voxtype_0.6.2-1_amd64.deb
sudo apt install /tmp/voxtype_latest.deb
```

### 5. Download the Whisper model

```bash
voxtype setup --download --model small.en
```

Available models: `tiny.en`, `base.en`, `small.en`, `medium.en`, `large-v3`, `large-v3-turbo`. Larger models are more accurate but slower and use more memory.

### 6. Configure VoxType

```bash
mkdir -p ~/.config/voxtype

cat > ~/.config/voxtype/config.toml << 'EOF'
state_file = "auto"

[hotkey]
# Key to hold for push-to-talk
# Common choices: SCROLLLOCK, PAUSE, RIGHTALT, F13-F24
# Use `evtest` to find key names for your keyboard
key = "RIGHTALT"
modifiers = []

[audio]
device = "default"
sample_rate = 16000
max_duration_secs = 60

[whisper]
model = "small.en"
language = "en"
translate = false

[output]
mode = "type"
fallback_to_clipboard = true
type_delay_ms = 0

[output.notification]
on_recording_start = false
on_recording_stop = false
on_transcription = false
EOF
```

### 7. Create VoxType user service

```bash
cat > ~/.config/systemd/user/voxtype.service << 'EOF'
[Unit]
Description=VoxType voice typing
After=ydotool.service

[Service]
ExecStart=/usr/bin/voxtype
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable voxtype
```

### 8. Reboot and verify

```bash
sudo reboot
```

After reboot:

```bash
systemctl --user status ydotool
systemctl --user status voxtype
ydotool type "hello world"
```

Both services should show `active (running)`. The `ydotool type` command should type "hello world" at your cursor.

### 9. Enable GPU acceleration (recommended)

See [Performance Optimization](#performance-optimization) below. This significantly speeds up transcription.

```bash
sudo voxtype setup gpu --enable
systemctl --user restart voxtype
```

## Usage

**Hold Right Alt** to record, **release** to transcribe. Text appears at your cursor position in any application.

### Changing the hotkey

Edit `~/.config/voxtype/config.toml` and change the `key` value:

```toml
key = "SCROLLLOCK"   # or "PAUSE", "F13", etc.
```

Use `evtest` to find key names for your keyboard (`sudo apt install evtest`).

### Restarting services

```bash
systemctl --user restart voxtype
systemctl --user restart ydotool
```

## Performance Optimization

### GPU acceleration (Vulkan) — recommended

Enables the Intel Iris Xe GPU for inference. This made the biggest difference — transcription dropped from ~3.4s to ~1.0s for the same audio on `small.en`.

```bash
sudo voxtype setup gpu --enable
systemctl --user restart voxtype
```

Requires `mesa-vulkan-drivers` and `libvulkan1` (pre-installed on Ubuntu 24.04).

Verify it's active:

```bash
journalctl --user -u voxtype -n 15 --no-pager | grep -i vulkan
# Should show: whisper_backend_init_gpu: using Vulkan0 backend
```

### Parakeet engine — not recommended

Tested and was slower than Whisper + Vulkan on this hardware (Intel i7-1370P, Iris Xe). To try it anyway:

```bash
voxtype setup parakeet --enable    # enable
voxtype setup parakeet --disable   # revert to Whisper
systemctl --user restart voxtype
```

### Whisper model trade-offs

Tested on Intel i7-1370P with Vulkan GPU acceleration:

| Model        | Size        | Memory      | Speed (CPU) | Speed (Vulkan) | Accuracy             |
| ------------ | ----------- | ----------- | ----------- | -------------- | -------------------- |
| tiny.en      | ~75 MB      | Low         | Fastest     | —              | Lower                |
| base.en      | ~142 MB     | Low         | Fast        | —              | Good                 |
| **small.en** | **~466 MB** | **~480 MB** | **~3.4s**   | **~1.0s**      | **Better (current)** |
| medium.en    | ~1.5 GB     | High        | Slow        | —              | High                 |

Change model in `~/.config/voxtype/config.toml` and download with:

```bash
voxtype setup --download --model <model-name>
systemctl --user restart voxtype
```

## Troubleshooting

| Problem                                  | Fix                                                                                                                                    |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `ydotool type` does nothing              | Check `systemctl --user status ydotool` — the daemon must be running                                                                   |
| Permission denied on `/dev/uinput`       | Verify you're in the `input` group: `groups $USER`. Log out and back in after `usermod`.                                               |
| VoxType not typing (but transcribing)    | ydotoold may not be running. Check with `systemctl --user status ydotool`                                                              |
| Wrong microphone                         | Change `device` in `config.toml` or use `arecord -l` to list devices                                                                   |
| Transcription quality is poor            | Try `model = "small.en"` or `model = "medium.en"` in config (larger = slower but more accurate)                                        |
| GPU enabled but logs show `no GPU found` | Your user may not be in the `render` group. Check with `groups $USER`. Fix: `sudo usermod -aG render $USER`, then log out and back in. |

## Updating VoxType

```bash
# Check current version
voxtype --version

# Download the latest .deb from GitHub releases
# Replace the version number with the latest from:
# https://github.com/peteonrails/voxtype/releases
curl -L -o /tmp/voxtype_latest.deb \
  https://github.com/peteonrails/voxtype/releases/download/v0.6.2/voxtype_0.6.2-1_amd64.deb

# Stop, upgrade, restart
systemctl --user stop voxtype
sudo apt install /tmp/voxtype_latest.deb
systemctl --user start voxtype

# Re-enable GPU acceleration if needed
sudo voxtype setup gpu --enable
systemctl --user restart voxtype

# Verify
voxtype --version
systemctl --user status voxtype
```

Your config (`~/.config/voxtype/config.toml`) and downloaded models (`~/.local/share/voxtype/models/`) are preserved across upgrades.

## Uninstall

> **Note:** These steps have not been tested yet. Review before running.

```bash
systemctl --user disable --now voxtype ydotool
rm ~/.config/systemd/user/voxtype.service ~/.config/systemd/user/ydotool.service
sudo apt remove voxtype
sudo rm /usr/local/bin/ydotool /usr/local/bin/ydotoold
sudo rm /etc/udev/rules.d/80-uinput.rules
rm -rf ~/.config/voxtype
rm -rf ~/.local/share/voxtype
```
