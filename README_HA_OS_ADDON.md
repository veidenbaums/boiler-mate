## 📦 Running boiler-mate on Home Assistant OS (Raspberry Pi 5, Native)

> **This section describes running boiler-mate natively on Home Assistant OS**
> (for example on a Raspberry Pi 5 / ARM64), without Docker.

### Why native?
- Raspberry Pi 5 is **ARM64**
- Existing Docker images are typically **amd64**
- Native build is simpler, faster, and fully supported

### Requirements

- Home Assistant OS
- Raspberry Pi 5 (ARM64 / aarch64)
- Mosquitto MQTT broker (HA add-on)
- Terminal & SSH add-on enabled
- NBE boiler reachable over TCP

### Enable SSH Access

In Home Assistant:

Settings → Add-ons → Terminal & SSH

- Enable **SSH server**
- Add your SSH public key
- Start the add-on

Connect:

```bash
ssh user@<HOME_ASSISTANT_IP>
```

### Clone boiler-mate

All files must be under `/config` (persistent storage):

```bash
cd /config
git clone https://github.com/veidenbaums/boiler-mate.git
cd boiler-mate
```

### Install Go (build dependency)

Home Assistant OS does not include Go by default.

```bash
apk add --no-cache go
go version
```

### Build boiler-mate (ARM64)

```bash
go build -o boiler-mate ./cmd/boiler-mate
```

### Run boiler-mate (foreground test)

```bash
./boiler-mate \
  --log-level info \
  --controller tcp://<SERIAL>:<PASSWORD>@<BOILER_IP>:8483 \
  --mqtt mqtt://<MQTT_USER>:<MQTT_PASSWORD>@127.0.0.1:1883
```

If successful:
- Boiler connects
- MQTT connects
- Home Assistant discovery messages are published

### Verify in Home Assistant

**MQTT traffic**

Settings → Devices & Services → MQTT → Listen to topic

Topic:

```
nbe/<SERIAL>/#
```

**Device discovery**

Settings → Devices & Services → MQTT → NBE Boiler (<SERIAL>)

## 🔁 Start boiler-mate Automatically After Home Assistant Restart

Home Assistant OS does not provide a native service manager for custom binaries.
The recommended way to auto-start `boiler-mate` is via a **Home Assistant automation**.

---

### 1. Create a Start Script

```bash
nano /config/boiler-mate-start.sh
```

```sh
#!/bin/sh
cd /config/boiler-mate

./boiler-mate \
  --log-level info \
  --controller tcp://<SERIAL>:<PASSWORD>@<BOILER_IP>:8483 \
  --mqtt mqtt://<MQTT_USER>:<MQTT_PASSWORD>@127.0.0.1:1883
```

```bash
chmod +x /config/boiler-mate-start.sh
```

---

### 2. Register a shell_command in Home Assistant

Add to `configuration.yaml`:

```yaml
shell_command:
  start_boiler_mate: "/config/boiler-mate-start.sh"
```

Restart Home Assistant.

---

### 3. Create Startup Automation

- Trigger: **Home Assistant started**
- Action: `shell_command.start_boiler_mate`

```yaml
alias: Start boiler-mate on HA startup
description: ""
triggers:
  - event: start
    trigger: homeassistant
actions:
  - delay: "00:01:00"
  - action: shell_command.start_boiler_mate
mode: single
```

---

### 4. Verify

Restart Home Assistant and check MQTT traffic:
```
nbe/<SERIAL>/#
```

---

### Notes

- Use only ONE startup method
- Do NOT use cron
- Stop boiler-mate before rebuilding:
  ```bash
  pkill -9 -f boiler-mate
  ```


### Debug mode

```bash
./boiler-mate \
  --log-level debug \
  --controller tcp://<SERIAL>:<PASSWORD>@<BOILER_IP>:8483 \
  --mqtt mqtt://<MQTT_USER>:<MQTT_PASSWORD>@127.0.0.1:1883
```

### Rebuilding after code changes

```bash
cd /config/boiler-mate
go build -o boiler-mate ./cmd/boiler-mate
pkill -9 -f boiler-mate
/config/boiler-mate-start.sh
```

> ⚠️ Home Assistant runs **only the compiled binary**.
> Source changes have no effect until rebuilt.

### Stopping boiler-mate

```bash
pkill -9 -f boiler-mate
```

Verify:

```bash
ps -ef | grep boiler-mate | grep -v grep
```

No output = stopped.

### Home Assistant MQTT Discovery Notes

- MQTT discovery entities persist in Home Assistant
- Changing entity `Key` values creates new entities
- Old entities are not removed automatically

**Cleanup during development**

Settings → Devices & Services → MQTT → NBE Boiler → Delete device

Then restart boiler-mate to rediscover cleanly.

### Raspberry Pi 5 Notes

- Architecture: **ARM64 (aarch64)**
- Docker images for `amd64` will not work
- Native build is the recommended approach
