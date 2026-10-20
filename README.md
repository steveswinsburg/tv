# tv
Config for my TV connected to my headless coding agent box

## Requirements

```bash
sudo apt update
sudo apt install tmux htop kbd
```

## Start automatically at boot

Set `~/.config/tv.conf`

Set the launcher: `/usr/local/bin/tv-dashboard-start`

```bash
#!/bin/bash
set -eu

as_steve() {
    /usr/sbin/runuser -u steve -- /usr/bin/env -u TMUX HOME=/home/steve TERM=linux TZ=Australia/Sydney "$@"
}

# Start with an empty config and a temporary session, then load the layout.
# This also makes config errors visible in the service log.
if ! as_steve /usr/bin/tmux -L tv has-session -t '=tv' 2>/dev/null; then
    if ! as_steve /usr/bin/tmux -L tv list-sessions >/dev/null 2>&1; then
        as_steve /usr/bin/tmux -L tv -f /dev/null \
            new-session -d -s __tv_startup -x 240 -y 67
    fi
    as_steve /usr/bin/tmux -L tv source-file /home/steve/.config/tv.conf
    as_steve /usr/bin/tmux -L tv has-session -t '=tv'
    as_steve /usr/bin/tmux -L tv kill-session -t '=__tv_startup' 2>/dev/null || true
fi

# Replace any previous TV viewer before taking over the dedicated console.
as_steve /usr/bin/tmux -L tv detach-client -s '=tv' 2>/dev/null || true

# -w keeps the launcher running while the viewer is attached.
exec /usr/bin/openvt -f -c 8 -s -w -- \
    /usr/sbin/runuser -u steve -- /usr/bin/env -u TMUX \
    HOME=/home/steve TERM=linux TZ=Australia/Sydney \
    /bin/bash -c 'setterm --blank 0 --powersave off --powerdown 0 2>/dev/null || true; exec /usr/bin/tmux -L tv attach-session -t tv'
EOF

sudo chmod 755 /usr/local/bin/tv-dashboard-start
```

Create the boot service:

```bash
sudo tee /etc/systemd/system/tv-dashboard.service >/dev/null <<'EOF'
[Unit]
Description=TV tmux dashboard on console 8
After=systemd-user-sessions.service getty@tty8.service
Conflicts=getty@tty8.service

[Service]
Type=simple
Environment=TZ=Australia/Sydney
WorkingDirectory=/home/steve
ExecStart=/usr/local/bin/tv-dashboard-start
Restart=always
RestartSec=10
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
EOF
```

Hand the current TV display over to the service. This rebuilds the TV panels; the agent's default tmux server stays separate.

```bash
sudo systemctl stop tv-dashboard.service

tmux -L tv kill-server 2>/dev/null || true
sudo pkill -t tty8 -x htop || true

sudo systemctl daemon-reload
sudo systemctl enable --now tv-dashboard.service
sudo systemctl status tv-dashboard.service --no-pager
```

The dashboard will now start at boot and reconnect if its viewer exits. A passwordless local login is unnecessary: the service opens console 8 and runs the dashboard as `steve`. The service uses administrative privileges to open the console.

The agent panel waits until the `coding` session exists. It does not automatically launch an agent or restore running processes after reboot.

## Start or reconnect to the agent

From an ordinary SSH shell:

```bash
tmux -L default new-session -A -s coding
```

Change to your project directory and launch your chosen agent there. The TV picks up the session automatically. Press **Ctrl+b**, then **d** to detach and leave it running.

## Edit the panels

```bash
nano ~/.config/tv.conf
```

Each `new-session` or `split-window` line contains its panel's command. Keep the retry loop around commands that may exit or be unavailable. Keep position targets quoted, for example `-t 'tv:dashboard.{top-left}'`, and use percentage sizes such as `-l 35%`.

Because this config creates sessions and panes, do not repeatedly source it into the existing dashboard. After editing, rebuild it:

```bash
sudo systemctl stop tv-dashboard.service
tmux -L tv kill-server 2>/dev/null || true
sudo systemctl start tv-dashboard.service
```

This stops and recreates all TV panel commands. Keep your actual coding agent in the separate default server, so rebuilding the dashboard does not stop it.

## Daily commands

```bash
# Check service status
sudo systemctl status tv-dashboard.service --no-pager

# Recent startup errors and logs
sudo journalctl -u tv-dashboard.service -b -n 80 --no-pager

# Follow service logs
sudo journalctl -u tv-dashboard.service -f

# Show console 8 on the TV
sudo chvt 8

# List TV panels
tmux -L tv list-panes -a

# List agent sessions
tmux -L default list-sessions

# View the TV layout over SSH, read-only, without affecting its size
tmux -L tv attach-session -r -t tv

# Refresh the TV status bar
tmux -L tv refresh-client -S

# Stop the dashboard
sudo systemctl stop tv-dashboard.service

# Start it again
sudo systemctl start tv-dashboard.service

# Disable automatic startup and stop it
sudo systemctl disable --now tv-dashboard.service
```

Viewing the TV layout over SSH uses the same selected TV window and pane. Detach with **Ctrl+b**, then **d**. Use a fresh SSH shell for administrative commands.

## Appearance

Apply continuous green separators immediately:

```bash
tmux -L tv set -g pane-border-style 'fg=green'
tmux -L tv set -g pane-active-border-style 'fg=green'
```

Keep these lines in `~/.config/tv.conf` to preserve the appearance:

```tmux
set -g pane-border-style "fg=green"
set -g pane-active-border-style "fg=green"
set -g status-interval 5
```

Tmux draws shared panel separators, rather than a complete outside frame.

## Timezone

Check the machine's clock:

```bash
timedatectl
```

If needed, set Sydney time:

```bash
sudo timedatectl set-timezone Australia/Sydney
```

The launcher explicitly starts the TV server with `TZ=Australia/Sydney`. If an existing server still displays UTC, rebuild the dashboard using the commands under “Edit the panels”; setting a tmux environment variable alone may not update the server process's timezone.

## Troubleshooting

- **Waiting for coding:** start or reconnect to `tmux -L default new-session -A -s coding`.
- **No sessions:** check the service log for configuration errors.
- **Cannot find pane:** quote position targets and check that earlier splits succeeded.
- **Size missing:** use `-l 35%`, not `-p 35`.
- **Console 8 occupied:** the launcher deliberately takes over console 8. Reserve that console for this dashboard.
- **Display goes blank:** the launcher disables Linux console blanking where supported; check the TV's own sleep settings too.

## References

- [Ubuntu openvt manual](https://manpages.ubuntu.com/manpages/jammy/man1/openvt.1.html)
- [Ubuntu tmux manual](https://manpages.ubuntu.com/manpages/noble/man1/tmux.1.html)
- [systemd service documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)

