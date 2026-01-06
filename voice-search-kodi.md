# Voice Search for Kodi with Arctic Fuse 2

A complete guide to implementing voice-controlled search for Kodi using Home Assistant and the Unfolded Circle Remote 3.

## Overview

This integration enables voice search for your Kodi library using:
- **Unfolded Circle Remote 3** - Hardware remote with built-in microphone
- **Home Assistant Assist** - Voice pipeline with Whisper STT
- **Custom Kodi Addon** - Bridge to open custom skin windows
- **Arctic Fuse 2** - Native search UI (window 11185)

### Architecture

```
UC Remote 3 Mic → Home Assistant Assist (Whisper STT) → Custom Sentence → REST Command → Kodi JSON-RPC → script.openwindow → Arctic Fuse 2 Search
```

## Prerequisites

- Kodi with Arctic Fuse 2 skin installed
- Home Assistant instance
- Unfolded Circle Remote 3 (firmware 2.8.1+ for voice support)
- Kodi JSON-RPC enabled (Settings → Services → Control → Allow remote control via HTTP)

## Part 1: Kodi Helper Addon

Arctic Fuse 2 uses custom window IDs that aren't accessible via standard JSON-RPC. This small addon bridges that gap.

### Installation

SSH into your Kodi device (CoreELEC/LibreELEC) and create the addon:

```bash
mkdir -p /storage/.kodi/addons/script.openwindow

cat > /storage/.kodi/addons/script.openwindow/addon.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<addon id="script.openwindow" name="Open Window" version="1.0.0" provider-name="local">
  <extension point="xbmc.python.script" library="default.py"/>
  <extension point="xbmc.addon.metadata">
    <summary>Open custom windows via JSON-RPC</summary>
  </extension>
</addon>
EOF

cat > /storage/.kodi/addons/script.openwindow/default.py << 'EOF'
import sys
import xbmc

window_id = None
search_term = None

# Handle both space-separated args and &-separated params
all_params = '&'.join(sys.argv[1:]).split('&')

for arg in all_params:
    if '=' in arg:
        key, value = arg.split('=', 1)
        if key == 'window':
            window_id = value
        elif key == 'search':
            search_term = value

if search_term:
    xbmc.executebuiltin(f'SetProperty(CustomSearchTerm,{search_term},Home)')

if window_id:
    xbmc.executebuiltin(f'ActivateWindow({window_id})')
EOF
```

### Testing the Addon

Restart Kodi to load the addon:

```bash
systemctl restart kodi
```

Test via curl:

```bash
# Open search window
curl -X POST \
  -u kodi:kodi \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "method": "Addons.ExecuteAddon", "params": {"addonid": "script.openwindow", "params": "window=11185"}, "id": 1}' \
  http://<KODI_IP>:8080/jsonrpc

# Open search with pre-filled query
curl -X POST \
  -u kodi:kodi \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "method": "Addons.ExecuteAddon", "params": {"addonid": "script.openwindow", "params": "window=11185&search=Breaking Bad"}, "id": 1}' \
  http://<KODI_IP>:8080/jsonrpc
```

## Part 2: Home Assistant Configuration

### Whisper STT (Docker)

Add to your `docker-compose.yml`:

```yaml
whisper:
  image: rhasspy/wyoming-whisper
  container_name: whisper
  restart: unless-stopped
  ports:
    - "10300:10300"
  volumes:
    - ./whisper-data:/data
  command: --model tiny-int8 --language en
```

Start the container:

```bash
docker-compose up -d whisper
```

### Wyoming Integration

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **Wyoming Protocol**
3. Enter host: `<DOCKER_HOST_IP>` and port: `10300`

### Voice Assistant Pipeline

1. Go to **Settings → Voice Assistants → Add Assistant**
2. Configure:
   - **Name:** Kodi Voice
   - **Language:** English
   - **Conversation agent:** Home Assistant
   - **Speech-to-text:** whisper (Wyoming)
   - **Text-to-speech:** None (or Piper if desired)

### REST Command

Add to `configuration.yaml`:

```yaml
rest_command:
  kodi_af2_search:
    url: "http://<KODI_IP>:8080/jsonrpc"
    method: post
    username: "kodi"
    password: "kodi"
    headers:
      content-type: "application/json"
    payload: '{"jsonrpc": "2.0", "method": "Addons.ExecuteAddon", "params": {"addonid": "script.openwindow", "params": "window=11185&search={{ text }}"}, "id": 1}'
```

### Automation

Add to `automations.yaml`:

```yaml
- id: kodi_voice_search
  alias: "Kodi Voice Search"
  trigger:
    - platform: conversation
      command:
        - "(search|type|find) {query} [on kodi]"
        - "kodi (search|type) {query}"
  action:
    - service: rest_command.kodi_af2_search
      data:
        text: "{{ trigger.slots.query }}"
```

### Reload Configuration

Go to **Developer Tools → YAML** and reload:
- REST Commands
- Automations

### Testing in Home Assistant

1. Go to **Developer Tools → Actions**
2. Select `rest_command.kodi_af2_search`
3. Switch to YAML mode and enter:
   ```yaml
   text: "Breaking Bad"
   ```
4. Click **Perform Action**

## Part 3: Unfolded Circle Remote 3 Setup

### Home Assistant Integration

1. Open UC Remote web configurator (http://<remote-ip>)
2. Go to **Integrations → Add Integration → Home Assistant**
3. Configure:
   - **WebSocket URL:** `ws://<HA_IP>:8123/api/websocket`
   - **Access Token:** Long-lived token from HA (Profile → Security → Create Token)

### Add Voice Assistant Entity

1. In UC configurator, go to the Home Assistant integration
2. Add entity: **Voice Assistant (assist)**
3. Select the voice pipeline created earlier (Kodi Voice)

### Configure Voice Button

1. Go to **Remotes & Docks → Remote → Buttons**
2. Assign the microphone button to the Voice Assistant entity

## Usage

Press and hold the voice button on the UC Remote 3 and say:

- "Search Breaking Bad on Kodi"
- "Find Stranger Things on Kodi"
- "Kodi search The Office"

The Arctic Fuse 2 native search window will open with your query pre-filled.

## Troubleshooting

### Finding Your Skin's Search Window ID

If you're using a different skin, find the correct window ID:

1. Manually open the search window in Kodi
2. Run this command:

```bash
curl -X POST \
  -u kodi:kodi \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "method": "GUI.GetProperties", "params": {"properties": ["currentwindow"]}, "id": 1}' \
  http://<KODI_IP>:8080/jsonrpc
```

3. Note the `id` value in the response
4. Update the `window=` parameter in your REST command

### JSON-RPC Not Responding

Ensure Kodi has remote control enabled:
- **Settings → Services → Control → Allow remote control via HTTP**
- Note the port (default 8080) and credentials

### Voice Not Recognized

- Check Whisper container is running: `docker logs whisper`
- Verify Wyoming integration is connected in HA
- Test STT directly in **Developer Tools → Actions** with `assist_pipeline.run`

### Addon Not Found

If you get "addon not found" error:

```bash
# Restart Kodi to reload addons
systemctl restart kodi
```

## Window IDs Reference

| Skin | Search Window ID |
|------|------------------|
| Arctic Fuse 2 | 11185 |
| Arctic Fuse 2 (Discover Hub) | 11105 |

## Alternative: TMDbHelper Search

If you prefer TMDbHelper's search instead of the native skin search:

```yaml
rest_command:
  kodi_tmdb_search:
    url: "http://<KODI_IP>:8080/jsonrpc"
    method: post
    username: "kodi"
    password: "kodi"
    headers:
      content-type: "application/json"
    payload: '{"jsonrpc": "2.0", "method": "Addons.ExecuteAddon", "params": {"addonid": "plugin.video.themoviedb.helper", "params": {"info": "search", "tmdb_type": "multi", "query": "{{ text }}"}}, "id": 1}'
```

## License

MIT License - Feel free to use and modify as needed.

## Credits

- [Arctic Fuse 2](https://github.com/jurialmunkey/skin.arctic.fuse.2) by jurialmunkey
- [TMDbHelper](https://github.com/jurialmunkey/plugin.video.themoviedb.helper) by jurialmunkey
- [Wyoming Whisper](https://github.com/rhasspy/wyoming-whisper) by Rhasspy
