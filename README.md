# tv
Config for my TV connected to my headless coding agent box

## Requirements

```bash
sudo apt update
sudo apt install tmux kbd
```

## Start automatically at boot

### Set the config
```
cp tv.conf ~/.config/tv.conf
```

### Setup the service:
```
sudo install -m 755 tv-dashboard-start /usr/local/bin/tv-dashboard-start
sudo install -m 755 tv-panel /usr/local/bin/tv-panel
sudo install -m 644 tv-dashboard.service /etc/systemd/system/tv-dashboard.service
sudo systemctl daemon-reload
sudo systemctl restart tv-dashboard.service
```

## Start or reconnect to a session

From an ordinary SSH shell:

```bash
tmux -L default new-session -A -s agent
```

Change to your project directory and launch your chosen agent there. The TV picks up the session automatically. Press **Ctrl+b**, then **d** to detach and leave it running.

## Edit the panels

The default layout keeps the agent on the left, with three compact panels in the right column:

- **CPU:** aggregate utilisation across all cores, a rolling graph and load averages.
- **Memory:** used/total RAM, available RAM, swap usage and a rolling graph. Used RAM is total minus Linux's `MemAvailable`, so reclaimable caches are not counted as unavailable memory.
- **Storage:** usage bars and used/free space for each filesystem, excluding temporary and squashfs mounts.

CPU and memory graphs retain 40 samples, taken every two seconds (about 80 seconds). They use ASCII characters and work on a headless Ubuntu Linux console without a graphical desktop or extra monitoring packages. History starts fresh when a panel restarts. Storage refreshes every 15 seconds.

System warnings and network/uptime remain below the agent. The network panel refreshes every five seconds, showing uptime and each active global-scope IPv4/IPv6 address on its own line, labelled by interface. Loopback/link-local addresses and wide traffic-counter tables are omitted. The commented sections at the end of `tv.conf` provide optional **services**, **Docker containers**, **temperatures**, and **application logs**. Uncomment both lines for a section to enable it, then configure its command. Extra panels reduce the space available to the agent.

The CPU, memory, storage and network commands live in `tv-panel`, which uses Linux `/proc` counters, `df`, `ip` and `uptime`. Reinstall it after editing it:

```bash
sudo install -m 755 tv-panel /usr/local/bin/tv-panel
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

## Troubleshooting

- **Waiting for agent:** start or reconnect to `tmux -L default new-session -A -s agent`.
- **CPU/memory/storage panel keeps retrying:** check its error message and install `tv-panel` using the command above. These panels require Linux.
- **No sessions:** check the service log for configuration errors.
- **Cannot find pane:** quote position targets and check that earlier splits succeeded.
- **Size missing:** use `-l 35%`, not `-p 35`.
- **Console 8 occupied:** the launcher deliberately takes over console 8. Reserve that console for this dashboard.
- **Display goes blank:** the launcher disables Linux console blanking where supported; check the TV's own sleep settings too.

## References

- [Ubuntu openvt manual](https://manpages.ubuntu.com/manpages/jammy/man1/openvt.1.html)
- [Ubuntu tmux manual](https://manpages.ubuntu.com/manpages/noble/man1/tmux.1.html)
- [systemd service documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
