# Bluetooth Audio Fixes for Wyoming Satellite

This document describes the fixes implemented to resolve issues with Bluetooth audio devices in Wyoming Satellite.

## Issues Fixed

### 1. Voice Command Misinterpretation
**Problem:** When using `--awake-wav` or `--done-wav` options, the satellite would not correctly understand voice commands after wake word detection.

**Cause:** The microphone was being muted during awake.wav playback, causing audio loss with Bluetooth devices. Bluetooth audio streams cannot be properly paused and resumed like regular audio devices.

**Solution:** Added automatic detection of Bluetooth devices and disabled microphone muting when Bluetooth is detected.

### 2. Prolonged Listening State
**Problem:** The LED indicator would remain on for an extended period after finishing speaking.

**Cause:** The satellite could get stuck in a listening state, especially with Bluetooth devices that may have buffering issues.

**Solution:** Implemented a configurable listening timeout that automatically stops listening after a set period and returns to wake word detection.

### 3. Satellite Getting Stuck After Wake Word
**Problem:** The satellite would get stuck after only one wake word detection and wouldn't respond to subsequent wake words.

**Cause:** The wake word service wasn't properly resetting after detection, and Bluetooth audio buffering could cause issues with the wake word detection pipeline.

**Solution:**
- Added a listening timeout to prevent indefinite listening states
- Added a small delay for Bluetooth devices to reset properly between conversations
- Maintained continuous audio stream to wake service for reliable detection

## New Command-Line Options

### `--mic-bluetooth-no-mute`
Automatically disables microphone muting for Bluetooth devices. This prevents audio loss during awake.wav playback.

Example:
```bash
--mic-bluetooth-no-mute
```

### `--wake-listening-timeout`
Sets the maximum seconds to listen for a voice command after wake word detection (default: 15 seconds).

Example:
```bash
--wake-listening-timeout 10
```

## Updated Service Configuration

Here's an updated systemd service configuration that includes the fixes:

```ini
[Unit]
Description=Wyoming Satellite (Bluetooth Fixed)
After=network-online.target pipewire-pulse.service bluetooth.target wyoming-porcupine3.service
Wants=network-online.target wyoming-porcupine3.service

[Service]
Type=simple
Restart=always
RestartSec=2
ExecStartPre=/usr/bin/pactl set-card-profile bluez_card.70_4F_08_02_D9_7E headset-head-unit
ExecStart=/opt/wyoming/wyoming-satellite/.venv/bin/python -m wyoming_satellite \
  --name "Salon Uydu" \
  --uri tcp://0.0.0.0:10700 \
  --mic-command "/usr/bin/parec --device=bluez_input.70_4F_08_02_D9_7E.0 --format=s16le --rate=16000 --channels=1" \
  --snd-command "/usr/local/bin/pacat-wrapper.sh" \
  --wake-uri tcp://0.0.0.0:10400 \
  --wake-word-name hey-home \
  --wake-refractory-seconds 2 \
  --wake-listening-timeout 10 \
  --mic-bluetooth-no-mute \
  --awake-wav /opt/wyoming/sounds/listening.wav \
  --done-wav /opt/wyoming/sounds/done.wav \
  --debug

[Install]
WantedBy=default.target
```

## How the Fixes Work

### Bluetooth Detection
The system now automatically detects Bluetooth devices by looking for "bluez_" in the microphone command. When detected, it disables microphone muting to prevent audio loss.

### Listening Timeout
After wake word detection, the satellite will listen for a maximum of the configured timeout period. If no speech-to-text result is received within this time, it automatically returns to wake word detection mode.

### Wake Service Reset
The wake word service maintains a continuous audio stream for proper detection:
1. Wake service continues to receive audio throughout the conversation
2. A small delay is added for Bluetooth devices after conversation ends
3. The detection pipeline is properly reinitialized with _send_wake_detect()

## Testing the Fixes

1. Restart the Wyoming Satellite service:
   ```bash
   sudo systemctl restart wyoming-satellite
   ```

2. Test wake word detection multiple times to ensure it doesn't get stuck

3. Verify that voice commands are correctly recognized after the awake sound plays

4. Check that the listening indicator turns off promptly after speaking

## Troubleshooting

If you still experience issues:

1. **Increase the listening timeout** if commands are being cut off:
   ```bash
   --wake-listening-timeout 20
   ```

2. **Disable awake/done sounds entirely** if they still cause issues:
   Remove the `--awake-wav` and `--done-wav` options

3. **Check Bluetooth connection stability**:
   ```bash
   bluetoothctl info 70:4F:08:02:D9:7E
   ```

4. **Monitor the debug logs** for any errors:
   ```bash
   journalctl -u wyoming-satellite -f
   ```

## Additional Notes

- The fixes are designed to be backward compatible with non-Bluetooth devices
- The automatic Bluetooth detection can be overridden with the `--mic-bluetooth-no-mute` flag
- The listening timeout prevents the satellite from getting stuck indefinitely
- The wake word refractory period still applies to prevent multiple rapid detections
