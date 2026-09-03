# netserv CLI Commands

`netserv` is a single npm binary (`dist/server/index.js`) that registers four
subcommands through NoArg. Running it with no subcommand prints help.

| Command | Purpose |
| ------- | ------- |
| `web` | Serve a directory over HTTP with a browser UI |
| `ftp` | Serve a directory over FTP |
| `get` | Receive files pushed by a peer |
| `send` | Push files to a peer running `get` |

## web

Starts an Express server and serves the bundled React UI alongside a JSON API.

| Flag | Type | Default | Effect |
| ---- | ---- | ------- | ------ |
| `--password` | string | unset | Enables authentication; without it every route is open |
| `--host` | string | unset | Binds a single address instead of every IPv4 interface |
| `--port` | number | `8000` | Listening port |
| `--qr` | boolean | `true` | Prints a terminal QR code of each URL |
| `--writable`, `-w` | boolean | `false` | Mounts the rename, delete, new-folder, and upload routes |

The positional `Target Dir` argument selects the directory to serve and defaults
to the current directory. It is resolved to an absolute path at startup.

When `--host` is omitted the server iterates the IPv4 addresses of every network
interface and starts one listener per interface, printing the interface name and
URL for each.

Read-only is the default. The mutating routes are not registered at all unless
`--writable` is passed, so they resolve as 404 rather than 403 on a read-only
server.

## ftp

Starts an `ftp-srv` server, retrying once per second if the listen call fails.

| Flag | Type | Default | Effect |
| ---- | ---- | ------- | ------ |
| `--host`, `-h` | string | unset | Binds a single address instead of every IPv4 interface |
| `--port`, `-p` | number | `2221` | Listening port |
| `--username`, `--user` | string | unset | Required credential |
| `--password`, `--pass` | string | unset | Required credential |
| `--root` | string | `.` | Directory exposed to clients |

`--username` and `--password` must be supplied together; providing one without
the other throws at startup. When neither is supplied the server accepts
anonymous logins. If the root directory does not exist, the process exits on the
first login attempt.

## get

Receives an upload from a peer. See the netserv Transfer API for the wire
protocol.

| Flag | Type | Default | Effect |
| ---- | ---- | ------- | ------ |
| `--output`, `-o` | string | `.` | Directory incoming files are written to |
| `--code`, `-c` | number | unset | Pairing code, constrained to 1–9999 |

The port is not fixed. For each IPv4 interface the receiver asks portfinder for
a free port starting at 8080, binds it, and prints the resulting host/port pairs
that the sender needs.

## send

Pushes one or more files to a running `get` receiver.

| Argument | Type | Purpose |
| -------- | ---- | ------- |
| `host` | string | Receiver address |
| `port` | number | Receiver port |
| `target...` | string list | Files or folders to send |

| Flag | Type | Default |
| ---- | ---- | ------- |
| `--code` | number | unset |

The sender assembles a multipart body, measures its exact length, requests a
token for that length, and replays the token on the upload. Upload progress is
rendered with a progress bar; once the bytes are on the wire the sender waits for
the receiver to finish writing before reporting completion.
