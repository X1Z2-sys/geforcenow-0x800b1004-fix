# Fix: GeForce NOW error 0x800B1004 caused by oversize root certificates in the Windows certificate store

## Symptoms

- GeForce NOW (Windows app, tested on 2.0.90.126) starts a game session, then the stream dies about half a second later with **error 0x800B1004**.
- The app logs show the session was set up successfully, then terminated immediately: `extendedCode 2148208644`, `reason 51`, `streamStopReason 3`, and all stream stats at `-1` (no stream data was ever received).
- The browser player (play.geforcenow.com) works fine on the same PC.
- Other PCs on the same network run the native app fine.
- Reinstalling the app, toggling the firewall, changing DNS, and disabling VPN software do not help.

If that matches, this guide is probably your fix.

## Root cause

When a stream starts, the GeForce NOW streamer loads the Windows **Local Machine root certificate store** into its TLS library and writes each certificate into a **fixed 4096-byte buffer**.

A handful of **post-quantum pilot/test root certificates** (ML-DSA) are much larger than normal roots: roughly **7,569 to 7,664 bytes** each, versus 1,000 to 2,700 bytes for ordinary roots. Each oversize certificate overflows the buffer and fails to load. After the last one fails, the whole certificate load aborts, the streamer's TLS handshake to the streaming server fails certificate verification, and the session dies instantly as error 0x800B1004.

In the confirmed case there were exactly **7** oversize certificates in the store, and the streamer log showed exactly **7** buffer errors before the failure. Removing those 7 certificates fixed the app completely.

These are test certificates. They are not used by real websites, which is why nothing else on the PC was affected.

## How to reproduce

You can reproduce the error on any Windows PC where GeForce NOW currently works:

1. Get a root certificate larger than 4096 bytes (DER encoded). The post-quantum pilot roots listed below work; they are publicly distributed test certificates from the CAs' pilot programs.
2. Install it into the Local Machine store, from an elevated command prompt:
   ```
   certutil -addstore -f AuthRoot <file>.cer
   ```
3. Fully close GeForce NOW (including the tray icon) and reopen it.
4. Launch any game. The session sets up, then dies in about half a second with error **0x800B1004**. The `geronimo.log` (path in Step 1) shows `WriteX509CertPem: error certSize=4096`, then `Failed to load certificate from stream`, then `certificate verify failed` on the streaming connection.
5. Remove the certificate (elevated) and relaunch:
   ```
   certutil -delstore AuthRoot <thumbprint>
   ```
   Streaming works again.

That install, fail, remove, works cycle is the proof of cause. In the confirmed case, 7 oversize certificates were installed and the log showed exactly 7 buffer errors before the failure; the certificate load appears to abort once it hits the oversize entries, so a single one should be enough to trigger it.

## Step 1: Confirm it in the streamer log

After a failed launch, open this file:

```
%LOCALAPPDATA%\NVIDIA Corporation\GeForceNOW\geronimo.log
```

Search for these lines, in this order:

```
WriteX509CertPem: error certSize=4096
Failed to load certificate from stream
certificate verify failed
```

If you see them, you have this bug. The number of `WriteX509CertPem` lines tells you how many oversize certificates are in your store.

## Step 2: Find the oversize certificates

Open PowerShell and run:

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
  ForEach-Object { [PSCustomObject]@{ Subject = $_.Subject; Thumbprint = $_.Thumbprint; SizeBytes = $_.RawData.Length } } |
  Where-Object { $_.SizeBytes -gt 4096 } |
  Sort-Object SizeBytes -Descending |
  Format-Table -AutoSize
```

Known examples (all post-quantum **PILOT / TEST / DEV** roots, all expiring in 2027):

| Subject | Size (bytes) |
|---|---|
| Sectigo Pilot Server Authentication Root M27 | 7,664 |
| SSL.com TLS ML Root CA 2026 - Pilot | 7,646 |
| IdenTrust Pilot Root TLS ML-DSA CA 1 | 7,628 |
| DigiCert PQC TLS PILOT MLDSA87 Root CA | 7,615 |
| HARICA TLS ML Root CA 2026 - Pilot | 7,591 |
| ComSign Dev Root PQ CA | 7,579 |
| UniTrust Global TLS MLDSA Root R1 TEST | 7,569 |

Your list may differ. What matters is the pattern: certificates larger than 4096 bytes whose names say Pilot, Test, or Dev.

**Safety check before removing anything:** only remove a certificate if it is clearly a pilot/test/dev certificate like the ones above. If an oversize certificate in your store is a production root from a CA you rely on, do not remove it. Send NVIDIA support your `geronimo.log` lines instead and ask about the 4096-byte certificate buffer.

## Step 3: Back up the certificates first

Do not skip this. The backup makes the whole fix reversible in one command per certificate.

```powershell
$dir = "$env:USERPROFILE\gfn-cert-backup"
New-Item -ItemType Directory -Force $dir | Out-Null
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object { $_.RawData.Length -gt 4096 } |
  ForEach-Object { Export-Certificate -Cert $_ -FilePath "$dir\$($_.Thumbprint).cer" }
Get-ChildItem $dir
```

## Step 4: Remove the oversize certificates

The certificates physically live in the **AuthRoot** store (the `Root` store is a merged view). Run PowerShell **as Administrator**:

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object { $_.RawData.Length -gt 4096 } |
  ForEach-Object { certutil -delstore AuthRoot $_.Thumbprint }
```

## Step 5: Verify

1. Re-run the Step 2 command. It should return **nothing** (zero certificates over 4096 bytes).
2. Fully close GeForce NOW (including the tray icon) and reopen it.
3. Launch a game. It should stream normally.
4. Optional confirmation: check `geronimo.log` again. There should be no `WriteX509CertPem` errors, and the stream stats should show real frame counts instead of `-1` values.

## How to undo

Restore any backed-up certificate, from an elevated command prompt:

```
certutil -addstore -f AuthRoot "%USERPROFILE%\gfn-cert-backup\<THUMBPRINT>.cer"
```

Repeat for each certificate you removed.

## Notes

- The real bug is the fixed 4096-byte buffer in the streamer. This guide is a workaround until NVIDIA ships a fix; if a future app update enlarges the buffer, the certificates can be restored with no downside.
- The total number of certificates in your store does not matter. Only individual certificates larger than 4096 bytes trigger the failure.
- Where the pilot certificates came from is unknown; they appear to be distributed with post-quantum TLS pilot programs. Simply having them installed breaks nothing else, because normal Windows TLS does not use a 4096-byte per-certificate buffer.
- On the first successful launch after the fix, Windows may show a firewall permission prompt for the streamer. That is expected; the streamer never got far enough to ask before. Allow it.
