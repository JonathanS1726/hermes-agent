# Windows Hermes profiles via VPS Telegram — manual operations runbook

Status: validated manually in November 2026. This document describes the observed operator setup; it is **not** an unattended deployment and contains **no live credentials**.

## Purpose and boundaries

Send requests from a phone's Telegram chat to a VPS-hosted Hermes coordinator, which dispatches to the chosen **Windows Hermes profile** and its persistent Bot Chat. Project orchestration and delegated specialist agents remain inside their respective profiles. The existing live configuration is **not** checked into this repository.

```text
Telegram phone → VPS Hermes container → Docker bridge gateway
              → VPS loopback/SSH forwards → Windows loopback API
              → Windows default gateway multiplexer → named profile / Bot Chat
```

The VPS Telegram bot and Windows Hermes are separate installations. Each Windows profile retains its own memory, credentials and role. A successful peer message targets the canonical Bot Chat for that profile. Routing to an arbitrary preexisting session ID has **not** been validated and must not be promised.

## Recorded validation

- VPS Hermes Docker container running, with the Telegram bot responding.
- Windows Hermes default-profile gateway running in foreground and serving 11 named profiles through one multiplexer.
- Old Python 3.11 virtual environment caused `pydantic_core` import failures; it was recoverably renamed after verifying it had no consumers. Windows runtime subsequently used managed Python 3.14.7. Do **not** delete the recoverable legacy directory without a separate cleanup decision.
- Windows API loopback-only. Unauthenticated local and forwarded model requests correctly returned HTTP 401; authenticated requests returned HTTP 200.
- The VPS registered the Windows default peer and 11 named profile peers, with independent keys and prechange backups. Do not equate a registered key with a successful runtime turn for every profile.
- End-to-end Telegram natural-language routing was validated for `whatsappsaas` and `ministeriocristiano`; messages reached their Windows Bot Chats. The other specialist peers were registered and keyed, but were not all tested by message.
- No Windows Scheduled Task, legacy VBS/CMD launcher, or unattended SSH auth was present in the final checks. Manual startup is intentional.

Named profiles:

| Project / role | Profile |
| --- | --- |
| WhatsApp SaaS coordinator | `whatsappsaas` |
| WhatsApp implementer | `whatsappcoder` |
| WhatsApp reviewer | `whatsappreviewer` |
| WhatsApp explorer | `whatsappexplorer` |
| WhatsApp researcher | `whatsappresearcher` |
| Ministerio Cristiano coordinator | `ministeriocristiano` |
| Ministerio implementer | `ministeriocoder` |
| Ministerio reviewer | `ministerioreviewer` |
| Ministerio explorer | `ministerioexplorer` |
| Cross-project engineering coordinator | `engineeringorchestration` |
| Engineering orchestration reviewer | `engineeringorchestrationreviewer` |

## Manual startup before leaving the PC

### 1. Windows PowerShell: gateway

```powershell
hermes gateway status
```

If stopped, open **PowerShell window 1** and leave this command running:

```powershell
hermes gateway run
```

The multiplexer serves named profiles. Avoid launching a second gateway if the default one already runs.

### 2. Windows PowerShell: required reverse SSH tunnel

Open **PowerShell window 2**, substituting the trusted VPS SSH hostname/address, and leave it open:

```powershell
ssh -o ExitOnForwardFailure=yes -o ConnectTimeout=10 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -R 127.0.0.1:18642:127.0.0.1:8642 root@<VPS_HOST>
```

VPS loopback TCP 18642 points to Windows loopback TCP 8642 through SSH. Authentication may prompt interactively. The `-N` session is normally silent.

### 3. VPS terminal: container-to-reverse-tunnel bridge

The background bridge should already be running; check for its listener:

```bash
ss -ltn | grep ':18643 '
```

If present, **do not start a duplicate**. If absent, verify the actual Docker gateway first:

```bash
docker network inspect hermes-agent-ofgs_default --format '{{(index .IPAM.Config 0).Gateway}}'
```

Only when the returned gateway is `172.16.1.1`:

```bash
ssh -f -N -o ExitOnForwardFailure=yes -o ConnectTimeout=10 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -L 172.16.1.1:18643:127.0.0.1:18642 root@127.0.0.1
```

The Docker container then reaches the Windows API at `http://172.16.1.1:18643`; per-profile peers use `/p/<profile-name>`. If the bridge address differs or SSH host verification fails, **stop and inspect**, do not silently accept a new identity or substitute addresses.

### 4. Windows PowerShell: optional dashboard tunnel

Only for accessing the VPS-hosted Hermes dashboard from Windows; **not required for Telegram delegation**:

```powershell
ssh -o ExitOnForwardFailure=yes -o ConnectTimeout=10 -N -L 127.0.0.1:18640:127.0.0.1:32768 root@<VPS_HOST>
```

## Validation and daily use

- On Windows, `hermes gateway status` should report a running gateway. `hermes -p whatsappsaas gateway status` should report the default multiplexer.
- On VPS, the peer list should show `[key set]` for the default Windows peer and each named profile; never display key **values**.
- In Telegram, ask the VPS Hermes coordinator: `Ask the Windows profile whatsappsaas to reply PHONE_READY_OK in its Bot Chat, and relay the result here.`
- For another project, replace `whatsappsaas` with `ministeriocristiano` or an explicitly named specialist. Address the coordinator if its established delegation and review workflow should run.
- An ordinary Telegram instruction with a session ID does **not** prove that an existing desktop session was opened. Ask for a read-only session lookup/ownership check first.

## Duration and interruptions

No fixed duration limit: processes can remain active while Windows is **awake, signed in, online**, and the foreground gateway/reverse-tunnel windows remain open; the VPS container and bridge must also be alive. Locking Windows is fine. Sleep, shutdown, logout, window closure, or an SSH disconnect can stop remote access. SSH keepalives detect dead connections but **do not automatically reconnect**. This setup intentionally has no unattended startup. Never change sleep policy, create scheduled tasks, or modify SSH authentication as part of this runbook.

## Security and backup rules

- Keep both APIs loopback-bound; do not publish the Windows API, Docker bridge listener, or dashboard directly to the internet.
- Each profile has its own `API_SERVER_KEY`; VPS peer settings store corresponding secrets. Values reside in protected deployment `.env` files, **never Git**. Do not expose keys in commands, screenshots, URLs, logs, prompts, or chatbot replies.
- Do not check in `.env`, `auth.json`, Telegram bot tokens, Codex OAuth files, SSH keys, recovery archives, private backup paths, saved sessions, or actual external host identifiers. Secrets and existing prechange backups stay on their machines with restricted permissions.
- The prechange snapshots cover the Windows local API, individual profile environments, and VPS peer registration; restoring any snapshot is a separate approval-gated recovery action.
- Unauthorized HTTP 401 was an expected security result; authenticated HTTP 200 proved only that the corresponding API endpoint accepted the credential, not arbitrary project permissions.
- This setup was tested for basic route/auth boundaries. It is **not** a blanket security certification or authorization for code execution or production modifications.

## Project routing and continuation

Each project keeps a separate memory/context. Examples of Telegram addressing:

```text
To Windows profile whatsappsaas, canonical Bot Chat: <task>.
Keep the WhatsApp SaaS delegation/review workflow and existing approvals.
```

```text
To Windows profile ministeriocristiano, canonical Bot Chat: <task>.
Keep the Ministerio Cristiano project boundaries and existing approvals.
```

For an explicit historical session: `Locate session <ID> for profile <name> read-only, verify ownership and whether resume is supported; do not claim to switch sessions until confirmed.`

Phase 114 of AI WhatsApp Automation 6 (cancellation): the latest checkpoint retained an **unapplied draft** migration, uncertain V1-like objects in an isolated PostgreSQL test DB, and an incomplete V2 runtime validation. Preserve the V1/V2 isolated checkpoint and temporary scratch artifacts; inspect provenance and status before advancing. Do not reapply migration 114, run tests that mutate the database, touch n8n/Meta, stage, commit, or push SaaS code without the project-specific approval gate.

## Documentation references

- [Hermes multi-profile gateways](https://hermes-agent.nousresearch.com/docs/user-guide/multi-profile-gateways)
- [Hermes Bot Mode](https://hermes-agent.nousresearch.com/docs/user-guide/bot-mode)
- [Hermes sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions)

_Recorded from observed operator checks and successful Telegram peer-routing messages. Changes to the live Windows/VPS configuration were not part of this documentation commit._
