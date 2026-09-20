# whisper-dictate

Push-to-talk dictation for Linux, in one bash script. Press a hotkey, speak,
press it again. The transcript is pasted into whatever you were typing in and
left on the clipboard. Nothing runs in the background between uses.

It sends audio to any OpenAI-compatible transcription endpoint, so it works
with a Whisper server on your own machine or LAN, or with OpenAI's API.

## Requirements

| Purpose | Package (Debian/Ubuntu) | Notes |
|---|---|---|
| Recording | `pipewire-bin` (`pw-record`) | `pulseaudio-utils` or `alsa-utils` also work |
| Upload / parse | `curl`, `python3` | almost certainly installed already |
| Clipboard | `wl-clipboard` on Wayland, `xclip` on X11 | |
| Auto-paste | X11: `xdotool` + `x11-utils`. Wayland: `ydotool` (GNOME, KDE) or `wtype` (sway, Hyprland, niri) | optional; set `AUTO_PASTE=0` without it; see [Wayland](#wayland) |
| Notifications | `libnotify-bin` | optional |
| Sound cues | `libcanberra-gtk3-module` or `pulseaudio-utils`, plus `sound-theme-freedesktop` | optional |

On Ubuntu with X11 and GNOME, this covers everything:

```sh
sudo apt install pipewire-bin curl python3 xclip xdotool x11-utils libnotify-bin \
                 libcanberra-gtk3-module sound-theme-freedesktop
```

The script works out at run time whether it is on X11 or Wayland and which
compositor is running, and uses the matching tools. `whisper-dictate check`
prints what it found and what is missing. Wayland needs one or two extra
packages depending on the compositor; see [Wayland](#wayland).

## Install

```sh
mkdir -p ~/.local/bin
cp whisper-dictate ~/.local/bin/
chmod +x ~/.local/bin/whisper-dictate
```

`~/.local/bin` is on the PATH by default on most distributions. Any location
works as long as the hotkey points at the full path.

## Wayland

Wayland has no standard way to inject keystrokes or to ask which window has
focus, so both jobs need a compositor-specific tool. The script picks one
from the session it finds itself in:

| Compositor | Paste keystrokes with | Focused-window lookup |
|---|---|---|
| GNOME | `ydotool`, else `xdotool` through Xwayland | Window Calls extension, else the accessibility bus |
| KDE Plasma | `ydotool`, else `wtype` | `kdotool` |
| sway, Hyprland, niri | `wtype`, else `ydotool` | the compositor's own IPC |

Run `whisper-dictate check` after installing. It shows the session and
compositor it detected, the recorder, clipboard, and paste tools it will use,
why any candidate is unusable, what it thinks the focused window is, and
whether the server answers.

**GNOME.** ydotool drives a virtual keyboard through `/dev/uinput`, so it
needs its daemon running and access to that device:

```sh
sudo apt install ydotool
sudo usermod -aG input "$USER"
systemctl --user enable ydotool
```

Then log out and back in so the group change and the package's udev rule take
effect, and confirm with `whisper-dictate check`. If it still says `ydotoold
is not running`, check `journalctl --user -u ydotool`: a failure to open
`/dev/uinput` after a re-login means the group did not reach your user
session. That happens when something survives the logout (a tmux, herdr, or
similar server) and keeps the old `systemd --user` manager alive; user
services inherit its groups. Reboot, or kill the survivor before logging out.

An alternative that needs no daemon or group is `sudo apt install xdotool`:
Xwayland forwards its synthetic keys to GNOME through libei. GNOME asks once
per login whether to allow it, and the first paste after logging in may be
lost while that dialog is up.

Telling terminals apart from other apps (Ctrl+Shift+V versus Ctrl+V) is the
harder part on GNOME, which does not expose the focused window. GTK, Qt, and
Electron apps, including Ptyxis, GNOME Terminal, and Console, announce their
active window on the accessibility bus, and the script reads that. kitty,
alacritty, and foot do not register there, so they receive Ctrl+V unless the
[Window Calls](https://extensions.gnome.org/extension/4724/window-calls/)
extension is installed; the script uses it whenever it is present. The
blunt alternative is to make the terminal accept Ctrl+V as well; for kitty
that is one line in `kitty.conf`:

```
map ctrl+v paste_from_clipboard
```

It costs readline's quoted-insert and Vim's visual-block key (`ctrl+q` does
the same in Vim). Sending Ctrl+Shift+V everywhere is not an option: browsers
treat it as paste-as-plain-text, which is harmless, but VS Code opens a
Markdown preview, LibreOffice opens Paste Special, JetBrains opens the
clipboard history, and plain GTK and Qt entries ignore it.

**KDE Plasma.** The same ydotool steps as GNOME. Install `kdotool` for the
focused-window lookup.

**sway, Hyprland, niri.** `sudo apt install wtype`. Nothing else to set up;
the focused window comes from `swaymsg`, `hyprctl`, or `niri msg`.

`PASTE_TOOL=ydotool` (or `xdotool`, `wtype`, `none`) in the config overrides
the automatic choice.

## A transcription server

Pick one. The script only needs the URL.

**whisper.cpp** (CPU or GPU, runs anywhere)

```sh
whisper-server -m ggml-large-v3-turbo.bin --port 8080 --inference-path /v1/audio/transcriptions
```

URL: `http://localhost:8080/v1/audio/transcriptions`. This is the script's
default, so nothing to configure.

**faster-whisper** via a server such as Speaches (NVIDIA GPU, fastest self-hosted option)

```sh
docker run --gpus all -p 8000:8000 ghcr.io/speaches-ai/speaches:latest-cuda
```

URL: `http://localhost:8000/v1/audio/transcriptions`. Set `WHISPER_MODEL` to
the model name the server expects (for example `Systran/faster-whisper-large-v3`).

**OpenAI's API** (no local hardware, audio leaves your machine)

URL `https://api.openai.com/v1/audio/transcriptions`, `WHISPER_MODEL=whisper-1`,
and `WHISPER_API_KEY` set to your key.

**A server on another machine** works the same as a local one. Use its hostname
in the URL. Some servers ignore `model`, some require it, and some also serve
their own native endpoint alongside the OpenAI-compatible one. Point the script
at the OpenAI-compatible path.

Confirm the server answers before binding a hotkey:

```sh
curl -sS http://localhost:8080/v1/audio/transcriptions -F file=@some.wav
```

## Configure

```sh
cp whisper-dictate.conf.example ~/.config/whisper-dictate.conf
```

Edit it. The file is plain `KEY=VALUE` lines; quote values that contain spaces.
Every line is optional. The ones worth setting on day one:

- `WHISPER_URL` if your server is not on `localhost:8080`.
- `WHISPER_PROMPT`: names, places, and jargon you say often. Whisper conditions
  on this text and it is the cheapest accuracy improvement available.
- `SOUND_VOLUME` if the cues are too loud or too quiet.

An environment variable overrides the file, so a second hotkey can run, say,
`WHISPER_LANGUAGE=fr whisper-dictate` without a second config.

## Bind a hotkey

The script takes no arguments for normal use. The same key starts and stops.

**GNOME:** Settings, Keyboard, Keyboard Shortcuts, Custom Shortcuts, add one
with the command `/home/you/.local/bin/whisper-dictate` and a key such as
Super+D.

**KDE Plasma:** System Settings, Shortcuts, Custom Shortcuts, New, Global
Shortcut, Command/URL.

**sway** (in `~/.config/sway/config`):

```
bindsym $mod+d exec ~/.local/bin/whisper-dictate
```

**Hyprland** (in `hyprland.conf`):

```
bind = SUPER, D, exec, ~/.local/bin/whisper-dictate
```

**i3**: same line as sway.

Optionally bind a second key to `whisper-dictate cancel` to throw away a
recording without transcribing it.

## Using it

1. Click into a text field.
2. Press the hotkey. A short sound and a "Recording…" notification confirm it.
3. Speak.
4. Press the hotkey again. A second sound means the audio is on its way.
5. The text appears in the field, a third sound plays, and a notification
   shows the first line of what was heard.

Recording stops by itself after `MAX_SECONDS` (default five minutes) and is
transcribed as normal. Recordings under a third of a second are ignored, as are
the stock phrases Whisper produces from silence ("Thank you.", "you").

In terminals the script sends Ctrl+Shift+V instead of Ctrl+V. If an app takes
neither, set `PASTE_MODE=type` to have the text typed in keystroke by
keystroke, or `AUTO_PASTE=0` and paste by hand.

## Troubleshooting

- **Nothing happens on the hotkey.** Run the script from a terminal twice, a
  few seconds apart. Errors print to stderr and also appear as notifications.
- **"could not reach ..."** The server is down or the URL is wrong. Try the
  curl line above.
- **Text lands on the clipboard but is not pasted.** Run `whisper-dictate
  check`; it names the paste tool it chose, or says why none is usable. The
  notification after a dictation also says so. Pasting works only in the
  window that had focus when you pressed stop.
- **Pasted into a terminal as a stray `^V`, or nothing arrived in kitty.** The
  focused-window lookup did not recognise the terminal, so Ctrl+V was sent.
  `whisper-dictate check` shows what it sees as the focused window; on GNOME
  see [Wayland](#wayland) for the Window Calls extension or the kitty
  `map ctrl+v` workaround.
- **Wrong microphone.** The script records from the system default input.
  Change it in your sound settings or with `wpctl set-default <id>`.
- **Cues too loud.** Lower `SOUND_VOLUME`, or set an individual cue to `""`.

Runtime files (the in-progress WAV and a pid file) live in
`$XDG_RUNTIME_DIR/whisper-dictate` and are deleted after each dictation.
