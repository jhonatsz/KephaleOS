---
type: technology
status: active
created: 2026-09-15
updated: 2026-09-15
aliases:
  - ".NET Framework TLS"
  - "SchUseStrongCrypto"
  - "SystemDefaultTlsVersions"
  - "Windows Server 2016 TLS"
  - "dotnet TLS defaults"
  - "System.Net.WebException Could not create SSL/TLS secure channel"
tags: [dotnet, windows, tls, security, legacy-runtime]
sources:
  - "[[work/incidents/2026-09-15-bullzip-sqs-consumer-stall]]"
maturity: production
---

# .NET Framework TLS defaults & hardening

## What it is

.NET Framework (the pre-Core, Windows-only runtime — 4.x line) has its own
opinions about which TLS versions and cipher suites to offer during outbound
HTTPS handshakes, **separate from what Windows SCHANNEL supports**. Those
opinions are controlled by two registry flags and two environment
conditions. Without hardening, a .NET Framework client on a fully-patched
Windows Server can still fail to reach a modern TLS 1.2+ endpoint — because
.NET, not Windows, is picking the wrong protocols.

This page is the durable checklist for making that go away.

## The two registry flags that matter

Both go under this key (and its 32-bit twin — see next section):

```
HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319
```

| Flag | Value 1 means | Effect |
|---|---|---|
| **`SchUseStrongCrypto`** | Use strong crypto + expand default protocol list | .NET's `SecurityProtocolType` default flips from `Ssl3, Tls` → `Tls, Tls11, Tls12`. Filters weak ciphers (RC4, 3DES, NULL) out of ClientHello. Introduced with .NET 4.6 (2015). |
| **`SystemDefaultTlsVersions`** | Delegate TLS version choice to Windows SCHANNEL | .NET stops hardcoding its own version list and inherits from OS. If Windows later enables TLS 1.3 (via KB), the .NET process picks it up **without any code or registry change**. Introduced with .NET 4.7. |

**Recommended: set both to `1`.** They control different things and stack cleanly.

## The 32-bit / 64-bit trap (this catches people)

On any 64-bit Windows host, .NET Framework processes read the registry
based on their **process bitness**, not on the OS bitness:

| Process | Registry hive read |
|---|---|
| 64-bit .NET Framework | `HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319` |
| **32-bit .NET Framework** | **`HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319`** |

**You must set both flags under both hives** if the host runs a mix of
32-bit and 64-bit .NET Framework processes. Setting only the 64-bit hive
is the classic silent failure mode — 64-bit test scripts and PowerShell
succeed, but a 32-bit production service on the same box fails with:

```
System.Net.WebException: The request was aborted:
Could not create SSL/TLS secure channel.
```

This is what caused [[work/incidents/2026-09-15-bullzip-sqs-consumer-stall]] to
be misdiagnosed as a TLS issue initially — the 64-bit test scripts passed
while the actual 32-bit consumer was still broken.

## What `SchUseStrongCrypto` = 0 (or missing) actually does

Default legacy behavior — .NET Framework offers this in ClientHello:

```
SecurityProtocolType = Ssl3, Tls    (i.e. SSL 3.0 + TLS 1.0 only)
```

Ciphers offered include RC4-MD5, RC4-SHA, DES-CBC-SHA, 3DES-EDE-CBC-SHA,
NULL-MD5, NULL-SHA, and export-grade weak options. Against any server that
requires TLS 1.2 minimum (which is virtually everything modern), handshake
fails immediately with `protocol_version(70)` alert → surfaced to the app as
`Could not create SSL/TLS secure channel`.

**Missing** = same as `0`. Microsoft treats absence as disabled.

## What `SchUseStrongCrypto` = 1 does

Overrides the default to:

```
SecurityProtocolType = Tls, Tls11, Tls12    (TLS 1.0/1.1/1.2)
```

Also strips a hardcoded blocklist of weak ciphers from ClientHello.
This one flag alone fixes the vast majority of "legacy .NET can't talk to
modern server" cases.

**Does NOT:**
- Enable TLS 1.3 (that needs SCHANNEL + .NET 4.8+)
- Disable weak protocols at the OS level (that's SCHANNEL registry)
- Override explicit `ServicePointManager.SecurityProtocol = X` in code

## What `SystemDefaultTlsVersions` = 1 adds

Changes the default `ServicePointManager.SecurityProtocol` from the .NET
hardcoded list to `SecurityProtocolType.SystemDefault` (value 0, a
sentinel). At connection time, .NET queries SCHANNEL for the currently
enabled protocols and uses those.

**Effect over time:** the .NET process auto-inherits TLS 1.3 if Windows
gets it. On Windows Server 2019/2022 this happens naturally. On Windows
Server 2016 it requires KB5017308+ and SCHANNEL registry enablement.

**Requires .NET Framework 4.7+.** Older runtimes silently ignore the flag.

## Version support matrix

| .NET Framework | Honors `SchUseStrongCrypto` | Honors `SystemDefaultTlsVersions` | TLS 1.3 possible |
|---|---|---|---|
| < 4.6 | ❌ | ❌ | ❌ |
| 4.6.x | ✅ | ❌ | ❌ |
| 4.7.x | ✅ | ✅ | ❌ (Windows-dependent) |
| 4.8 | ✅ | ✅ | ✅ (Windows-dependent) |
| 4.8.1 | ✅ | ✅ | ✅ (Windows-dependent) |
| .NET Core / .NET 5+ | Ignored (already TLS-1.2-default) | Ignored | ✅ (always) |

## Verify current state — quick check

Run this in an **elevated** PowerShell on the host:

```powershell
'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319',
'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' | ForEach-Object {
    $p = Get-ItemProperty $_ -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Hive                     = $_
        SchUseStrongCrypto       = if ($null -ne $p.SchUseStrongCrypto)      { $p.SchUseStrongCrypto }      else { 'MISSING (=0)' }
        SystemDefaultTlsVersions = if ($null -ne $p.SystemDefaultTlsVersions) { $p.SystemDefaultTlsVersions } else { 'MISSING (=0)' }
    }
} | Format-Table -AutoSize
```

Both hives should show `1` in both columns. Anything else → fix.

## Apply the fix — the canonical two-liner

Elevated PowerShell:

```powershell
# 64-bit hive
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -Name 'SchUseStrongCrypto'       -Value 1 -Type DWord
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319' -Name 'SystemDefaultTlsVersions' -Value 1 -Type DWord

# 32-bit hive (WOW6432Node)
Set-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -Name 'SchUseStrongCrypto'       -Value 1 -Type DWord
Set-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319' -Name 'SystemDefaultTlsVersions' -Value 1 -Type DWord
```

Or via `reg.exe` (more forgiving on old PowerShell / when the property is
missing):

```
reg add "HKLM\SOFTWARE\Microsoft\.NETFramework\v4.0.30319"          /v SchUseStrongCrypto       /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Microsoft\.NETFramework\v4.0.30319"          /v SystemDefaultTlsVersions /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319" /v SchUseStrongCrypto       /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319" /v SystemDefaultTlsVersions /t REG_DWORD /d 1 /f
```

**Restart the .NET process** to pick up the new values. No reboot needed —
.NET reads the registry once at process start.

## Alternative — code fix (when you can't touch registry)

Add one line at the top of app startup, before any HTTPS call:

```csharp
System.Net.ServicePointManager.SecurityProtocol =
    System.Net.SecurityProtocolType.Tls12
    | System.Net.SecurityProtocolType.Tls13;  // Tls13 constant needs .NET 4.8+
```

For .NET 4.7.x (no `Tls13` constant), use the bitmask value directly:

```csharp
System.Net.ServicePointManager.SecurityProtocol =
    (System.Net.SecurityProtocolType)3072;  // 3072 = Tls12
```

**Registry approach is preferred** — no code change, no redeploy, applies to
every .NET process on the host.

## Check process bitness (when you don't know)

The registry flag has to be set in the hive matching the process's
architecture. To confirm which .exe is which:

```powershell
# For a running process
$p = Get-Process -Name '<processname>' | Select-Object -First 1
$b = [IO.File]::ReadAllBytes($p.Path)[0..4095]
switch ([BitConverter]::ToUInt16($b, [BitConverter]::ToInt32($b, 0x3C) + 4)) {
    0x014c { '32-bit (x86)' }
    0x8664 { '64-bit (x64)' }
    default { 'other' }
}
```

Or just check Task Manager → Details tab → add the **Platform** column.

## Diagnostic script

See `Test-TmTls.ps1` / `Test-SbiqTls.ps1` at
`/Users/jhonatsz/Test-*.ps1` — comprehensive per-TLS-version handshake
tests + registry inspection + application-level HTTPS test. Reusable for any
`https://<hostname>/` endpoint. Change the `$target` variable at the top.

## Common failure signatures → fix

| Symptom | Fix |
|---|---|
| `System.Net.WebException: Could not create SSL/TLS secure channel` from a legacy .NET service | Set both flags in the hive matching the process bitness |
| 64-bit PowerShell tests connect fine but the actual service still fails | The service is 32-bit — set the flags under `WOW6432Node` |
| Handshake works on TLS 1.0 but not 1.2 to a modern server | .NET is defaulting to `Ssl3, Tls` — set `SchUseStrongCrypto=1` |
| `TLS 1.3` doesn't negotiate even after both flags set | Windows doesn't have TLS 1.3 SCHANNEL enabled (WS2016 needs KB5017308+; WS2019 may need enablement) |
| Flags don't take effect on .NET 4.5.x | Runtime doesn't honor them — upgrade to .NET 4.7+ or use the code path |

## Related

- [[aws-elbv2-alb-nlb]] — server-side TLS policy on AWS ALBs; the client
  compatibility table on that page cross-references the SchUseStrongCrypto
  requirement for Windows Server 2016 callers
- [[work/incidents/2026-09-15-bullzip-sqs-consumer-stall]] — the incident
  that surfaced this pattern in CyberSoft's `AWS-DU` host

## Sources

- Microsoft docs: [Transport Layer Security (TLS) best practices with the .NET Framework](https://learn.microsoft.com/dotnet/framework/network-programming/tls)
- KB4019276 (2018) — the original Microsoft guidance on these flags
- 2026-09-15 CyberSoft SafeBoxIQ / TaskManager incident: 32-bit .NET
  Framework 4.7.2 service on Windows Server 2016 (`AWS-DU`) failing to
  reach `tm.msd.cybersoftbpo.com` after ALB tightening to
  `TLS13-1-2-Res-2021-06`. Fixed via `WOW6432Node\...\SchUseStrongCrypto=1`.
