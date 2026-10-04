# Remote Desktop into Ubuntu GNOME with Nobody Logged In

A battle-tested setup for connecting to a Ubuntu GNOME desktop (Wayland) from a phone / Windows App over RDP **even when nobody is logged in and the monitor/TV is off** — including the full pitfall journal.

> **Document date: 2026-10-04.** All bug statuses in this document are as of this date. If GNOME's "Remote Login" session-handoff bug has been fixed by the time you read this, re-evaluate the conclusion in Section 4 and update the revision history.
>
> 中文版见 [README.zh-CN.md](README.zh-CN.md)。
>
> Environment snapshot: **Ubuntu 26.04 LTS / GNOME Shell 50.1 (Wayland) / gnome-remote-desktop 50.0** on an AMD Ryzen 7 7735HS mini PC. The architecture below should transfer to any recent GNOME + gnome-remote-desktop setup.

## 1. Goal

Walk out the door, open the phone, and connect straight back to the home mini PC — **without anyone being logged in and with the display powered off**.

## 2. The Five-Layer Architecture

Each layer solves one specific failure mode. Remove any layer and you fall back to "someone must be at the machine".

### Layer 1: HDMI dummy plug (output)
A dummy plug (4K@60 EDID emulator) in an HDMI port keeps the OS believing a display is attached, so the compositor keeps a logical monitor alive when the TV is off or on another input. With a TV + dummy plug in mirror mode, the TV wakes up with the session.

### Layer 2: GDM auto-login (session)
`/etc/gdm3/custom.conf`:
```ini
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=youruser
```
This guarantees a user session exists after boot. The Desktop Sharing daemon is per-session: no session, no service.

### Layer 3: lock on login (security)
An autostart entry (`~/.config/autostart/lock-on-login.desktop`) runs `sleep 5 && loginctl lock-session` after login. Auto-login's price is "boot = open desktop"; the delayed lock pays it back: the session starts (services come up), then the screen locks 5 seconds later.

### Layer 4: `allow-locked-remote-desktop` extension (admission)
The GNOME extension [allow-locked-remote-desktop](https://github.com/jikamens/allow-locked-remote-desktop) adds `unlock-dialog` to gnome-remote-desktop's `session-modes`. RDP clients connecting while the screen is locked see the unlock dialog instead of being dropped.

Note: the GDM greeter and the unlock dialog look **identical** in a photo. Tell them apart with `loginctl list-sessions`, not by looking.

### Layer 5: Desktop Sharing on port 3390 + passwordless keyring (service)
- User-mode Desktop Sharing listens on **3390** (when Remote Login is also enabled, the system service takes 3389 and the user service moves to 3390). Open the port in your VPN/ACL (we use Tailscale: `tcp:3390`).
- **Set the GNOME login keyring password to EMPTY** (seahorse → Login keyring → Change Password → blank). With auto-login there is no password to unlock the keyring, so gnome-remote-desktop cannot read its stored credentials — clients get `Credentials are not set, denying client`. A blank-password keyring unlocks on any login method.

### Client side
- Microsoft's Windows App has no port field: write `IP:PORT` in the PC-name field (e.g. `100.x.y.z:3390`).
- Alternative client: **aRDP** (open source, FreeRDP-based). It fails fast with a retry button instead of hanging forever on a dead connection (see Pitfall 3).

## 3. Why NOT "Remote Login" (port 3389)

GNOME offers two modes: **Desktop Sharing** (take over the current user desktop) and **Remote Login** (log in from the GDM screen). Remote Login sounds like the correct "nobody logged in" answer, but **as of 2026-10-04 it has show-stopping bugs — do not rely on it**:

1. **Session-handoff black screen**: if a session for your user already exists, logging in through the Remote Login greeter prompts "session already running / force stop". Forcing the takeover usually lands you in a **completely black screen with only a movable cursor**. Logs show the GDM→session handoff breaking, plus leaked zombie `gdm-greeter` sessions piling up.
2. **Unstable credential verification**: the same username/password that works on Desktop Sharing repeatedly fails on Remote Login with `Could not find user in SAM database` / `MIC verification failed`.
3. **Dead clicks on the greeter**: over RDP, clicking the account on the GDM login screen sometimes does nothing; a reconnect "retry lottery" is needed.

**Conclusion (2026-10-04):** GNOME Remote Login is unreliable when a user session already exists. The five-layer architecture deliberately routes around it: auto-login guarantees the session always exists, Desktop Sharing takes over directly, and the broken GDM handoff path is never used.

> If the GDM handoff bug gets fixed, Remote Login becomes the cleaner "unattended + real login screen" answer (no auto-login needed, keyring can have a password again). If you're reading this after that fix, please update the revision history.

## 4. Pitfall Journal (symptom → root cause → fix)

### Pitfall 1: client error 0x104 "cannot find the computer"
- **Not a network problem**: nobody was logged in → no user session → no Desktop Sharing daemon. Auto-login itself was dying: GDM logs `Session never registered, failing`, the session lived ~1 second.
- **Real cause**: a user-level systemd unit had `Wants=graphical-session.target` + `WantedBy=default.target`. At boot it activated the target first; when the real gnome-session started it saw the target already active, concluded "a graphical session is already running!", and exited. It's a race — boots sometimes work, sometimes don't.
- **Fix**: session-scoped units must be *pulled by* the target (`WantedBy=graphical-session.target` + `PartOf=`), never pull it (`Wants=`) themselves.

### Pitfall 2: locked-screen connections rejected with "credentials not set"
- **Root cause**: auto-login types no password → `gkr-pam: couldn't unlock the login keyring` → the keyring stays locked → the RDP server finds no stored credentials (`[RDP] Credentials are not set, denying client`). Manual-login machines never see this, which makes it very confusing.
- **Fix**: blank password on the login keyring (Layer 5).
- **Verification**: verify through the Secret Service protocol — do NOT trust `gnome-keyring-daemon --unlock`, it exits 0 regardless of whether the password is correct. Correct method: call `Lock` then `Unlock` on `org.freedesktop.Secret.Service` (arguments are `ao` arrays, collection path `/org/freedesktop/secrets/collection/login`). If `Unlock` returns `prompt="/"` and the collection lands in the unlocked list, the blank password is proven.

### Pitfall 3: Windows App stuck on "Securing the connection…" forever
- **Root cause**: server log shows `BIO_read retries exceeded` → `ERRCONNECT_CONNECT_TRANSPORT_FAILED`. TCP connected but the client never delivered a single byte — the transport is already dead; that screen will spin until you kill it. Usually happens when the phone's radio just woke up or the VPN path switched (relay ↔ direct): the handshake's first flight falls into the gap.
- **Fix**: nothing to fix server-side. **Kill and retry immediately** — logs show a retry 1 second after a failure connecting instantly. Waiting costs ~30 s of timeout; retrying costs 1 s. aRDP fails fast with a retry button and feels much better here.

### Pitfall 4: "where do I set the port in Windows App?"
Write `IP:PORT` in the PC-name field. There is no separate port field.

### Pitfall 5: TV shows "no signal" after lock
Normal: the screensaver powers off all outputs via DPMS (dummy plug included). RDP streams from the compositor's internal pipeline, independent of physical outputs. Connecting remotely shows the unlock dialog; unlocking lights the TV back up.

## 5. Acceptance Checklist

1. Reboot, touch nothing → lands on the desktop, locks 5 s later (TV black is normal);
2. Connect from the phone to `IP:3390` → unlock dialog → password → desktop;
3. TV lights up in mirror mode; session services come up automatically;
4. Repeat step 2 while locked — still connects (extension admits it).

## 6. Revision History

- **2026-10-04**: first version. Ubuntu 26.04 LTS / GNOME Shell 50.1 / gnome-remote-desktop 50.0. Remote Login handoff bug status: **unfixed**.
