# localcomm

A zero-config command-line tool for finding and messaging other devices on your local network, written in async Rust.

Run `localcomm serve` on one machine and it advertises itself over mDNS (the same mechanism AirDrop and Chromecast use for discovery). Any other machine on the LAN can then find it with `localcomm discover` and send it messages over TCP. You don't need IP addresses, a central server, or any configuration.

> **Status:** early development. Discovery, the message server, and one-off messages work today. File transfer is implemented but not yet wired into the CLI. Chat, device listing, and the TUI are planned. See [Roadmap](#roadmap).

## Features

- **mDNS service discovery.** Servers register as `_localcomm._tcp.local.` with TXT properties (device name, version), and clients browse for them with a configurable timeout.
- **Async TCP transport.** Built on Tokio. The server accepts many concurrent connections, spawning one task per connection and funnelling parsed messages into a single handler over an `mpsc` channel.
- **Simple wire protocol.** Messages are newline-delimited JSON with a UUID, type, sender, recipient, timestamp, and free-form metadata, so they're easy to debug with `nc` or `jq`.
- **Streaming file transfer (in progress).** Files are sent in 64 KB chunks with a SHA-256 checksum computed on both ends.

## Quick start

### Requirements

- Rust **1.85+** (the crate uses the 2024 edition)
- Two or more machines on the same LAN, with multicast allowed (mDNS uses UDP port 5353)

### Build

```bash
git clone https://github.com/bineetsinghchadha/localcomm.git
cd localcomm
cargo build --release
```

The binary is written to `target/release/localcomm`.

### Try it

On machine A, start a server:

```bash
localcomm serve --port 8080 --name laptop
```

```
=== localComm Server ===
Device Name: laptop
Port: 8080
Server listening on port 8080
Advertising via mDNS

Waiting for msgs...
```

On machine B, find it:

```bash
localcomm discover --timeout 5
```

```
Found 1 device(s):

1. laptop._localcomm._tcp.local.
Hostname: laptop.local.
Address: 192.168.1.42:8080
 Properties:
  name: laptop
  version: 1.0.0
```

Then send it a message:

```bash
localcomm send --to 192.168.1.42:8080 --message "hello from B"
```

Machine A prints:

```
[14:03:27] machine-b: hello from B
```

## Commands

| Command | Description | Status |
|---|---|---|
| `serve [-p PORT] [-n NAME]` | Listen for messages and advertise over mDNS. Port defaults to 8080 and name defaults to the hostname. | ✅ Working |
| `discover [-t SECONDS]` | Browse the LAN for localcomm servers. Timeout defaults to 5 seconds. | ✅ Working |
| `send -t HOST:PORT -m TEXT` | Send a single text message. | ✅ Working |
| `transfer send -f FILE -t HOST:PORT` | Stream a file to a receiver. | 🚧 Implemented, not yet wired into the CLI |
| `transfer receive [-p PORT] [-o DIR]` | Receive a file. Port defaults to 9090 and output directory to the current one. | 🚧 Implemented, not yet wired into the CLI |
| `chat HOST:PORT` | Interactive two-way chat. | 📋 Planned |
| `list` | Show previously discovered devices. | 📋 Planned |
| `tui` | Full-screen terminal UI. | 📋 Planned |

Run `localcomm --help` or `localcomm <command> --help` for all options. Set `RUST_LOG=debug` for verbose logging via `tracing`.

## How it works

```
 Machine A                                   Machine B
┌───────────────────────┐                   ┌───────────────────────┐
│ localcomm serve       │                   │ localcomm discover    │
│                       │  mDNS announce    │                       │
│  Discovery ──────────────────────────────▶│  Discovery (browse)   │
│  (_localcomm._tcp)    │  UDP 5353         │                       │
│                       │                   │ localcomm send        │
│  TcpTransport         │  JSON + "\n"      │                       │
│   ├ task per conn  ◀─────────────────────────  TcpTransport       │
│   └ mpsc ──▶ handler  │  TCP 8080         │                       │
└───────────────────────┘                   └───────────────────────┘
```

### Protocol

Each message is one JSON object followed by a newline:

```json
{
  "id": "5f0c3a8e-2d1b-4c55-9a39-8e1f6b1d7c20",
  "type": "text",
  "from": "machine-b",
  "to": "192.168.1.42:8080",
  "content": "hello from B",
  "timestamp": 1790479705,
  "metadata": {}
}
```

The supported `type` values are `text`, `file`, `filechunk`, `filecomplete`, `heartbeat`, `ack`, and `error`. The server prints `text` messages, prints the filename from `metadata.filename` for `file` messages, and ignores `heartbeat` messages.

The file transfer path uses a separate raw stream: a header (`<filename>\n<size>\n`), then the file bytes in 64 KB chunks. Both sides compute a SHA-256 checksum as the bytes stream through.

## Project layout

```
src/
├── main.rs            # Entry point: tracing setup, command dispatch
├── cli.rs             # clap command and argument definitions
├── commands/          # One module per subcommand
│   ├── serve.rs       # TCP listener, mDNS advertising, message loop
│   ├── discover.rs    # mDNS browse and pretty-print
│   ├── send.rs        # One-shot text message
│   ├── transfer.rs    # Chunked file send/receive with SHA-256
│   └── chat.rs, list.rs, tui.rs   # Stubs
├── discovery/
│   └── mdns.rs        # Service registration and browsing (mdns-sd)
├── protocol/
│   └── message.rs     # Message and MessageType, JSON (de)serialization
├── transport/
│   ├── tcp.rs         # Listener, connect, framed send/receive, receiver task
│   └── connection.rs  # Per-connection send/receive wrapper
└── ui/
    └── tui.rs         # Placeholder for the terminal UI
```

## Roadmap

- [ ] Wire `transfer` into `main.rs` and re-enable the `indicatif` progress bars
- [ ] Send the checksum to the receiver and verify it after transfer
- [ ] Interactive `chat` mode with concurrent read/write over one connection
- [ ] Persist discovered devices for `list`
- [ ] Heartbeats and `ack` messages for delivery confirmation
- [ ] Send and transfer by device name, resolved through mDNS instead of `host:port`
- [ ] Terminal UI
- [ ] Encryption (for example, a Noise handshake or TLS with self-signed certificates)
- [ ] Integration tests with two in-process peers

## Built with

[Tokio](https://tokio.rs) · [mdns-sd](https://crates.io/crates/mdns-sd) · [clap](https://crates.io/crates/clap) · [serde](https://serde.rs) / serde_json · [sha2](https://crates.io/crates/sha2) · [uuid](https://crates.io/crates/uuid) · [chrono](https://crates.io/crates/chrono) · [tracing](https://crates.io/crates/tracing) · [anyhow](https://crates.io/crates/anyhow)

## Security note

Traffic is currently **unencrypted and unauthenticated**. Any device on the LAN can discover a running server and send it messages or files. Use it only on networks you trust.
