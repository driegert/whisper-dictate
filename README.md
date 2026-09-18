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
| Clipboard | `xclip` on X11, `wl-clipboard` on Wayland | |
| Auto-paste | `xdotool` + `x11-utils` on X11, `ydotool` on Wayland | optional; set `AUTO_PASTE=0` without it |
| Notifications | `libnotify-bin` | optional |
| Sound cues | `libcanberra-gtk3-module` or `pulseaudio-utils`, plus `sound-theme-freedesktop` | optional |

On Ubuntu with X11 and GNOME, this covers everything:

```sh
sudo apt install pipewire-bin curl python3 xclip xdotool x11-utils libnotify-bin \
                 libcanberra-gtk3-module sound-theme-freedesktop
```

Check which session type you have with `echo $XDG_SESSION_TYPE`. On Wayland,
`ydotool` needs its daemon running and permission on `/dev/uinput`; see its
documentation. Everything else works the same.

## Install

```sh
mkdir -p ~/.local/bin
cp whisper-dictate ~/.local/bin/
chmod +x ~/.local/bin/whisper-dictate
```

`~/.local/bin` is on the PATH by default on most distributions. Any location
works as long as the hotkey points at the full path.

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
neither, set `PASTE_MODE=type` to have the text typed in keystroke by keystroke
(X11 only), or `AUTO_PASTE=0` and paste by hand.

## Troubleshooting

- **Nothing happens on the hotkey.** Run the script from a terminal twice, a
  few seconds apart. Errors print to stderr and also appear as notifications.
- **"could not reach ..."** The server is down or the URL is wrong. Try the
  curl line above.
- **Text lands on the clipboard but is not pasted.** On X11, check that
  `xdotool` and `xprop` are installed. On Wayland, check `ydotool` and its
  daemon. Pasting works only in the window that had focus when you pressed stop.
- **Wrong microphone.** The script records from the system default input.
  Change it in your sound settings or with `wpctl set-default <id>`.
- **Cues too loud.** Lower `SOUND_VOLUME`, or set an individual cue to `""`.

Runtime files (the in-progress WAV and a pid file) live in
`$XDG_RUNTIME_DIR/whisper-dictate` and are deleted after each dictation.
