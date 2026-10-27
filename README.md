# Zotero WebDAV server on Termux

Run an [rclone](https://rclone.org/) WebDAV server on an Android device using
[Termux](https://termux.dev/). Zotero can use the server as a WebDAV file
storage provider, with files stored in the `ZoteroWebDAV` directory of the
rclone remote named `gdrive` (typically Google Drive).

The `start-server` script also acquires a Termux wake lock and starts the SSH
daemon. It is designed to be run at device startup, for example by a Termux
boot task.

## Requirements

- Android with Termux installed.
- `rclone` installed in Termux and a working remote named `gdrive`.
- The remote must allow access to `gdrive:ZoteroWebDAV`.
- Termux:Boot, if you want Android to start the server automatically.

Install rclone in Termux if needed:

```sh
pkg install rclone openssh
```

Configure the remote with `rclone config` and confirm that it works, for
example with `rclone lsd gdrive:`. The script does not configure rclone or
create the remote for you.

## Configure WebDAV credentials

Create the credentials file expected by the script:

```sh
mkdir -p "$HOME/.config"
cat > "$HOME/.config/zotero-webdav.env" <<'EOF'
RCLONE_USER=your-webdav-username
RCLONE_PASS=your-webdav-password
EOF
chmod 600 "$HOME/.config/zotero-webdav.env"
```

Use values appropriate for your Zotero WebDAV account. Keep this file private;
it contains the WebDAV password.

## Start the server

Copy `start-server` onto the device, then run it in Termux:

```sh
sh /path/to/start-server
```

The script checks the credentials file, acquires a wake lock, starts `sshd`,
waits 15 seconds for networking, and starts the WebDAV server if one is not
already running. The server listens on port `6969` on all network interfaces
and serves `gdrive:ZoteroWebDAV` with rclone's full VFS cache mode.

In Zotero's file-sync settings, choose **Zotero** as the WebDAV version and
enter the device's reachable IP address and port, for example:

```text
http://192.168.1.25:6969/
```

Replace the example address with the Android device's address on your network,
and enter the same username and password from the credentials file. Confirm
that the device is reachable from the computer or phone running Zotero.

## Start automatically with Termux:Boot

Install and open Termux:Boot once, then place a startup script in its boot
directory. For example, if this repository is cloned at
`$HOME/rclone-termux-sync`:

```sh
mkdir -p "$HOME/.termux/boot"
cat > "$HOME/.termux/boot/start-zotero" <<'EOF'
#!/data/data/com.termux/files/usr/bin/sh
sh "$HOME/rclone-termux-sync/start-server"
EOF
chmod +x "$HOME/.termux/boot/start-zotero"
```

Android battery optimization can stop background processes; allow Termux and
Termux:Boot to run in the background if startup or availability is unreliable.

## Logs and troubleshooting

- Startup output: `~/.local/state/start-zotero.log`
- rclone WebDAV output: `~/.local/state/zotero-webdav.log`
- Check rclone configuration with `rclone listremotes` and verify the target
  path with `rclone lsf gdrive:ZoteroWebDAV`.
- Check that the server is listening with `ss -ltn` and that port `6969` is
  reachable from the Zotero device.
- The script starts `sshd` as well as WebDAV; Termux OpenSSH normally listens
  on port `8022`.

## Network security

The server binds to `0.0.0.0` and uses plain HTTP. Anyone who can reach the
device on port `6969` may be able to connect, so use it only on a trusted
network and avoid exposing the port to the public internet. Use a private
network or VPN when connecting from outside the local network.
