## Hyprland login workaround (greetd + Noctalia greeter)

### Problem
Logging in through greetd with the Noctalia greeter failed on the first attempt. Hyprland crashed on startup (`bad_optional_access` in `initServer`) and greetd returned to the greeter. The second login worked, and logging in from a TTY was never affected.

Hyprland's log showed:

```
[libseat] Could not take device: Device or resource busy
libseat: Couldn't open device at /dev/dri/card1
```

The Noctalia greeter runs its own wlroots compositor as the `greeter` user, which holds `/dev/dri/card1`. Hyprland was starting before logind had finished releasing the GPU from the greeter session, so it couldn't take the device.

### Fix: `hypr-session` wrapper
`/usr/local/bin/hypr-session` waits for all `greeter`-user processes to exit (up to about 3 seconds), pauses briefly for logind to finish cleanup, then launches Hyprland through uwsm.

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
