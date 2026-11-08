## Hyprland login workaround (greetd + Noctalia greeter)
Are you tired of these messages in your Hyprland CrashReports when logging in through greetd?
```
DEBUG from aquamarine ]: drm: Enumerated device /sys/devices/pci0000:00/0000:00:03.1/0000:07:00.0/0000:08:00.0/0000:09:00.0/drm/card1
ERR from aquamarine ]: [libseat] [libseat/backend/logind.c:121] Could not take device: Device or resource busy
ERR from aquamarine ]: libseat: Couldn't open device at /dev/dri/card1
ERR from aquamarine ]: drm: Skipping device /sys/devices/pci0000:00/0000:00:03.1/0000:07:00.0/0000:08:00.0/0000:09:00.0/drm/card1, not a KMS device
DEBUG from aquamarine ]: drm: Explicit device list /dev/dri/card1
ERR from aquamarine ]: drm: Explicit device /dev/dri/card1 not found
ERR from aquamarine ]: drm: Found no gpus to use, cannot continue
ERR from aquamarine ]: DRM Backend failed
DEBUG from aquamarine ]: Starting the Aquamarine backend!
DEBUG from aquamarine ]: Starting the Wayland backend!
ERR from aquamarine ]: Wayland backend cannot start: wl_display_connect failed (is a wayland compositor running?)
ERR from aquamarine ]: Requested backend (wayland) could not start, enabling fallbacks
ERR from aquamarine ]: Implementation wayland failed, erasing.
CRIT from aquamarine ]: Cannot open backend: no allocator available
```
### Problem and Explanation
Logging in through greetd with the Noctalia greeter failed on the first attempt. Hyprland crashed on startup (`bad_optional_access` in `initServer`) and greetd returned to the greeter. The second login worked, and logging in from a TTY was never affected.

Hyprland's log showed:

```
[libseat] Could not take device: Device or resource busy
libseat: Couldn't open device at /dev/dri/card1
```

The Noctalia greeter runs its own wlroots compositor as the `greeter` user, which holds `/dev/dri/card1`. Hyprland was starting before logind had finished releasing the GPU from the greeter session, so it couldn't take the device.

### Fix: `hypr-session` wrapper
`/usr/local/bin/hypr-session` waits for all `greeter`-user processes to exit (up to about 3 seconds), pauses briefly for logind to finish cleanup, then launches Hyprland through uwsm.
```
#!/bin/sh
# wait up to ~3s for greeter processes to disappear, then let logind settle
for i in $(seq 30); do
    pgrep -u greeter >/dev/null || break
    sleep 0.1
done
sleep 0.3
exec uwsm start hyprland.desktop
```

### Setup
    1. Copy hyprland-delayed.desktop to /usr/share/wayland-sessions/
```
 sudo rsync -aP hyprland-delayed.desktop /usr/share/wayland-sessions/
```
    2. Copy hypr-session to /usr/local/bin/
```
 sudo rsync -aP hypr-session /usr/local/bin/
```
    3. Copy greetd to /etc/pam.d/
```
 sudo rsync -aP greetd /etc/pam.d/
```
    4. Make hypr-session executable
```
 sudo chmod +x /usr/local/bin/hypr-session
```
### Notes
Tested on Arch-Linux with Greetd & Noctalia-Greeter
- The wrapper only fixes the GPU handoff race. It doesn't touch PAM, so keyring unlocking (gnome-keyring) is configured separately in `/etc/pam.d/greetd`.
- If the session entry lives in `/usr/share`, a package update may remove it. `/usr/local/share/wayland-sessions/` is safer if your greeter reads it.
- Ideally the greeter would release the GPU fully before greetd starts the session, which is worth reporting upstream to the Noctalia greeter project.
