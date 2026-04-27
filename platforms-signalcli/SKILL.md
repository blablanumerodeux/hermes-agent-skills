---
name: signalcli
description: Configure signal-cli as a Signal gateway for Hermes. Covers daemon modes, SSE envelopes, Note to Self limitations, and RPC interface quirks.
---

# signal-cli Gateway Setup for Hermes

## Overview
signal-cli can run as a local Signal gateway, allowing Hermes to send/receive Signal messages via its HTTP daemon or TCP daemon interface.

## Installation
```bash
# Install signal-cli (example for x86_64 Linux)
wget https://github.com/Asugak/Worker/releases/download/v0.13.19/signal-cli-linux-x86_64-0.13.19.tar.gz
tar -xzf signal-cli-linux-x86_64-0.13.19.tar.gz
sudo mv signal-cli-0.13.19/bin/signal-cli /usr/local/bin/
signal-cli --version  # confirm 0.13.19
```

## Register/Link Account
```bash
# Register with a number (via SMS or voice)
signal-cli -a +1234567890 register

# Verify (SMS code)
signal-cli -a +1234567890 verify 123456

# OR link as secondary device (avoids SMS verification)
signal-cli link -n "Hermes Gateway"  # outputs QR code URL like "signal://linkdevice?..."
# Scan with Signal app → Devices → Link new device
```

## Daemon Modes

### HTTP Daemon (SSE for receiving)
```bash
signal-cli -a +1234567890 daemon --http 127.0.0.1:8080 --no-receive-stdout
```
- SSE stream at: `http://127.0.0.1:8080/api/v1/events?account=%2B1234567890`
- **Receives** messages as SSE events
- HTTP RPC at: `http://127.0.0.1:8080/api/v1/rpc` (POST JSON-RPC)
- **Limitation**: Note to Self messages arrive with empty envelope (see below)

### TCP Daemon (raw socket JSON-RPC)
```bash
signal-cli -a +1234567890 daemon --tcp 127.0.0.1:7583
```
- Raw socket JSON-RPC on port 7583 — **not HTTP**, curl cannot test it directly
- Use `nc` or a socket client to interact
- Useful as fallback when HTTP SSE doesn't deliver message content

### Both HTTP + TCP simultaneously
```bash
signal-cli -a +1234567890 daemon --http 127.0.0.1:8080 --tcp 127.0.0.1:7583 --no-receive-stdout
```

## Envelope Fields Reference

### Regular incoming message (SMS from another user)
```json
{
  "envelope": {
    "source": "+1234567890",
    "sourceNumber": "+1234567890",
    "sourceUuid": "...",
    "sourceName": "Contact Name",
    "sourceDevice": 1,
    "timestamp": 1776450107580,
    "dataMessage": { "message": "Hello", "timestamp": 1776450107580 },
    "syncMessage": null
  }
}
```

### Note to Self message (sent from same gateway number)
```json
{
  "envelope": {
    "source": "+1234567890",
    "sourceNumber": "+1234567890",
    "sourceUuid": "...",
    "sourceName": "Your Name",
    "sourceDevice": 1,
    "timestamp": 1776450107580,
    "syncMessage": {},
    "dataMessage": null
  }
}
```
**Critical**: `syncMessage: {}` with `dataMessage: null` means Note to Self. The message text is NOT embedded in the SSE envelope — this is a fundamental signal-cli HTTP SSE limitation.

## Known Limitations

### 1. Note to Self has no message content in SSE
signal-cli's HTTP daemon SSE mode does not embed Note to Self message text in the envelope. `syncMessage: {}` arrives but `dataMessage` is `null`. The message body is never sent through the SSE stream in this mode.

**Workarounds**:
- Use a **different number** to send to the gateway (regular SMS works fine)
- Link as a **secondary device** (Note to Self may arrive differently)
- Use **D-Bus interface** (`--dbus` flag) which may handle this differently

### 2. `receive` RPC does not accept timestamp
```bash
# This DOES NOT WORK - receive doesn't accept timestamp
curl -X POST http://127.0.0.1:8080/api/v1/rpc -d '{"jsonrpc":"2.0","id":1,"method":"receive","params":{"timestamp":1776450107580}}'

# Only works with maxMessages and/or timeout
curl -X POST http://127.0.0.1:8080/api/v1/rpc -d '{"jsonrpc":"2.0","id":1,"method":"receive","params":{"maxMessages":1,"timeout":1}}'
```

### 3. `receive` RPC conflicts with active SSE stream
If the SSE event stream is active, calling `receive` RPC returns empty `[]`. They cannot coexist.

### 4. TCP daemon is raw socket, not HTTP
```bash
# This returns HTTP/0.9 garbage — TCP daemon is NOT HTTP
curl -s http://127.0.0.1:7583

# Use netcat for TCP JSON-RPC
echo '{"jsonrpc":"2.0","id":1,"method":"receive","params":{"maxMessages":1,"timeout":1}}' | nc -q1 127.0.0.1 7583
```

## Hermes Configuration (.env)
```bash
SIGNAL_ALLOWED_USERS=+1234567890        # allowed senders (your number)
SIGNAL_ACCOUNT=+1234567890              # gateway account number
SIGNAL_HTTP_URL=http://127.0.0.1:8080   # signal-cli HTTP daemon URL
SIGNAL_SSE_PATH=/api/v1/events          # SSE endpoint path
```

## Verifying Setup
```bash
# Check daemon is running
ps aux | grep signal-cli

# Test SSE stream
curl -s -N "http://127.0.0.1:8080/api/v1/events?account=%2B1234567890" --max-time 5

# Send a test message (via RPC)
curl -s -X POST "http://127.0.0.1:8080/api/v1/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"send","params":{"account":"+1234567890","recipient":["+0987654321"],"message":"test"}}'
```

## Troubleshooting

### SSE connected but no response
1. Check if message is Note to Self — if `syncMessage: {}` and `dataMessage: null`, the text won't be available
2. Try sending from a different number instead of Note to Self
3. Verify `SIGNAL_ALLOWED_USERS` includes your number in `.env`
4. Check Hermes logs: `tail -50 /root/.hermes/logs/agent.log`

### Daemon won't start
```bash
# Kill existing daemon
pkill -f signal-cli
# Restart fresh
signal-cli -a +1234567890 daemon --http 127.0.0.1:8080 --no-receive-stdout
```
