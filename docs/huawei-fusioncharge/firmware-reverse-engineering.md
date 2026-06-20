# Firmware reverse-engineering — Huawei FusionCharge AC OCPP client

> Static analysis explaining *why* the FusionCharge OCPP client never reconnects after a network interruption. Read-only RE for interoperability and repair; **no proprietary firmware is redistributed** — only findings, symbol/address references, and short excerpts for commentary. Not affiliated with Huawei.

## Package layout

The FusionCharge OTA package (app `FusionCharge V100R023C10SPC220`, BSP `V300R024C10SPC327`) unpacks to:

- `smu_app.tar.xz` → the main application tree, including:
  - `bin_arm/bin/app_arm` — main supervisory application (12 MB, **fully stripped** — only ~740 dynamic symbols, almost all library imports).
  - `apploadbin/ocpp_plugin` — **the OCPP client**, a standalone plugin binary.
  - `so_bin/libwebsockets.so.19` — the WebSocket library the OCPP client uses.
- `nor_flash_mnt/kp/keeper` — process supervisor for `app_arm` (statically linked).
- Per-component signing material (see [Code signing](#code-signing-why-you-cannot-just-reflash)).

`ocpp_plugin.sh` shows the OCPP client runs as the **unprivileged** user `prorunacc` with only `CAP_NET_BIND_SERVICE`. It is **not** the safety controller (contactor / CP-PWM / RCD live elsewhere, reachable via Modbus). Losing OCPP does not affect charging — consistent with observed behaviour.

## Tooling

- Unpack: `tar` / `xz`.
- Disassembly: LLVM `objdump --triple=thumbv7-linux-gnueabihf`, and **radare2 6.1.6** (`e asm.bits=16; af; pdc`). Ghidra 12.1.2 also available.
- The binary is **ARM 32-bit, Thumb-2, PIE, C++**, stripped of its symbol table **but the `.dynsym` is intact**, so exported method addresses are available. Mapping symbols (`$a/$t`) were stripped, so Thumb must be forced (`asm.bits=16`); auto-detection decodes it as ARM and produces garbage.

## The OCPP client structure

RTTI/`.dynsym` names reveal a clean C++ design (Huawei DOPRA/VOS middleware):

- `OcppLinkMgr` / `OcppProtoLinkMgr` — connection state machine
- `CRetryProtocolCmdList`, `…RetryCmd` — retry framework (used for Start/Stop/MeterValues)
- `HeartBeatProtocolCmd` — heartbeats
- `OcppProtocolAlarmManager`, `OcppAlarmReportProtocolCmd` — alarms (drive the fault LED via `app_arm`)
- Offline support is present: `OfflineStart/Stop`, `AllowOfflineTxForUnknownId`, `LocalAuthorizeOffline`, `trans_offline_data_db` — i.e. losing the CSMS is **not** supposed to stop charging.

## Root cause: no dead-peer detection at any layer

### 1. No libwebsockets keepalive / validity ping

The only libwebsockets symbols imported are:

```
lws_create_context   lws_client_connect_via_info   lws_service
lws_callback_on_writable   lws_cancel_service   lws_set_timeout
```

There is **none** of `lws_retry_*`, `lws_validity_*`, or `lws_sul_*` — i.e. no connection-validity policy. lws will not proactively PING an idle link nor hang up a silently dead one.

`OcppLinkMgr::SetPingInterval(unsigned short)` (`0x14a090`) does **not** arm an lws ping — it just stores the value to an object field:

```
; OcppLinkMgr::SetPingInterval — stores arg to this+0xF6, returns
strh   r2, [r3, #0xF6]
bx     lr
```

### 2. The FSM is event-driven on a *stored* status

`OcppLinkMgr::ProcLinkMgrTimerEvent` (`0x148fe4`) ticks and calls, in order, `ProcLinkFsm → ProcUnableLinkEvent → ProcDisconnLinkEvent → ProcTrafficStatOnTimer`. `ProcLinkFsm` (`0x149dd0`) simply switches on `GetConnectStatus()` — a **stored** state variable that is only updated by the libwebsockets event callback (`OcppEventCallBack` `0x1477e8`). No independent liveness probe exists, so the FSM only knows the link is down if lws delivered a disconnect event.

### 3. Reconnection is gated on an external "connect" signal

`OcppLinkMgr::ProcConnSigNotifyMsg` (`0x147fe0`) → `IsStartConnect()` → `RestartConnect()`. The client (re)connects only when handed a connection signal from `app_arm`'s registration/connection manager. The idle-state handler `ProcIdleState` (`0x149ea8`) only *reports* state; it does not self-open a socket. When the FSM gives up, `ProcDisconnLinkEvent` (`0x14959c`) merely counts retries and calls `SendLinkRestartToNetManager(true)` — it **asks `app_arm`** to restart the link:

```
; OcppLinkMgr::ProcDisconnLinkEvent (excerpt, Thumb)
ldrh.w r3, [r7, #0x14a]      ; retry counter
adds   r3, #1
...
cmp.w  r3, #0x708            ; threshold (1800)
bls    .keep_waiting
...
bl     SendLinkRestartToNetManager   ; ask app_arm to restart the link
```

After a Wi-Fi blip the FSM parks here waiting for a connect-signal that, in practice, only re-fires on a **cold boot**.

### 4. Heartbeats are fire-and-forget

`HeartBeatProtocolCmd::Request` (`0x12a680`) builds the Heartbeat and sends it via `SendCallMsg` → `lws_write`; it never tracks whether a `Heartbeat.conf` comes back. It is a `CAsSyncProtocolCmd`, **not** a `CRetryProtocolCmd` with a `TimeoutProcess` (that retry/timeout machinery is wired only to Start/Stop/MeterValues). A missed heartbeat ack triggers nothing, so heartbeats provide no liveness either.

### The immediate "red ring" path (separate, lesser issue)

On a *clean* disconnect, `OcppLinkMgr::ProcLinkStateChange` (`0x148908`) → `ReportLinkStatus` (`0x14900c`) → `ReportLinkStatusToNetworkMgr` (`0x149194`) reports link-down to `app_arm` **synchronously, with no debounce**, and `app_arm` raises the "Failed to Communicate with OCPP System" alarm / fault ring. (The alarm/LED decision itself lives in `app_arm`, which is stripped and not analysed in depth.)

## Code signing (why you cannot just reflash)

Every component is integrity-protected and the device enforces it:

- `<component>_filelist.txt` pins a `SHA256-Digest` per file with `ForceVerify=1`.
- `<component>_filelist.cms` is a detached **CMS / PKCS#7 signedData** over that manifest, using **RSA-PSS**, chained to Huawei's `CN=RSA-PSS Integrity Root CA 1 → RSA-PSS Integrity CA 1`.
- `config.txt` sets `SecureBoot=1`.

So modifying any byte breaks the SHA-256 → CMS verification fails → the component is rejected. A working patch would require Huawei's private key. This is why all workarounds here are **CSMS-side**, never on-device.

## Conclusion

The OCPP client trusts its TCP/WebSocket connection indefinitely with **no application- or transport-level keepalive, no heartbeat-ack timeout, and no autonomous reconnect** — it only establishes OCPP on a cold boot. Any transient network loss therefore parks it until a power-cycle. The fix belongs in the firmware (add a validity ping / heartbeat-ack timeout / autonomous reconnect); externally, the best you can do is detect it and prevent the network blip. See **[README.md](README.md)** for the practical workaround and test results.
