# YubiKey BitLocker setup options and commands

Use this alongside the [baseline and recovery instructions](FIELD-NOTES.md). These options apply to Windows BitLocker **data drives**, not preboot unlocking of the operating system drive. Choose a policy before generating a key: changing a key's PIN or touch policy requires a replacement key. Existing data does not need re-encryption when changing certificate protectors.

## Choose an unlock policy

| Choice | PIN policy | Touch policy | What possession of the key permits | Validation here |
| --- | --- | --- | --- | --- |
| PIN only | once | never | Private-key use after PIN verification, which can be cached in a card session | Original Windows-provider setup passed BitLocker unlocking |
| PIN plus touch | once | always | PIN verification per card session and touch for each key operation | Supported policy combination; this exact variant was not tested with BitLocker here |
| PIN for every operation plus touch | always | always | Fresh PIN verification and touch per operation | Hardware key creation and certificate signing passed; BitLocker unlock did not pass in our tests |
| Touch only | never | always | Anyone holding the key can approve use by touching | Windows private-key test and BitLocker unlock event passed |
| Presence only | never | never | Software can use the connected key without PIN or touch consent | Policy available; not tested with BitLocker here and not recommended for protecting an unattended connected key |

Touch detects anyone's finger; it is not fingerprint identification. Touch policy `cached` is another supported variant that can reuse a recent touch for approximately 15 seconds. It is not equivalent to `always` and was not tested here. PIN policy `once` does not promise a PIN prompt on every drive unlock if Windows or the card session caches verification.

The final observed installation uses **touch only**, plus a recovery password. PIN plus touch offers stronger protection against theft of the key, but do not assume a policy combination works in a particular BitLocker/driver version until you test it.

## Prepare Windows and the YubiKey

1. Back up the drive's files. Run Windows PowerShell as administrator. Verify the correct data-drive letter.
2. Install the official Yubico Smart Card Minidriver and Manager CLI. Read the current vendor installation instructions. For winget users, these are the package identifiers:

```powershell
winget install --id Yubico.YubiKeySmartCardMinidriver --exact --source winget
winget install --id Yubico.YubiKeyManagerCLI --exact --source winget
```

3. Inspect the device and existing slots:

```powershell
$ykman = 'C:\Program Files\Yubico\YubiKey Manager CLI\ykman.exe'
& $ykman info
& $ykman piv info
```

The observed device was a YubiKey 5C NFC with firmware 5.8.0. Do not publish the serial number or full diagnostic output without checking it for personal information.

4. Change a known factory PIV PIN using interactive prompts. Keep it even with touch-only unlocking: administrative operations may still require it. Configure management-key protection according to the current Yubico instructions. On the observed device the minidriver had already protected the management key with the PIN and blocked the factory PUK. Do not guess unknown credentials or reset a working PIV application.

```powershell
& $ykman piv access change-pin
# For a device whose current management credentials you control:
# This stores a newly generated management key on the device, protected by PIN.
& $ykman piv access change-management-key --generate --protect
```

Stop on errors. Do not repeat credential attempts blindly. A factory or unchanged PUK and a protected management key are separate concerns; inspect their actual state. Do not use set-retries as a harmless retry-counter adjustment: Yubico documents that it resets the PIN and PUK to defaults.

## Generate one new hardware key

Choose a genuinely unused slot. Slot 9d is intended for key management. Slot 82 was unused and worked in the touch-only test; it is a retired key-management slot and can require Windows enumeration refresh. Never overwrite an occupied slot just to follow an example.

The following checks inspect slot 82. If either a key or certificate already exists, choose another suitable empty slot and update every subsequent command consistently:

```powershell
$slot = '82'
& $ykman piv keys info $slot
& $ykman piv info
```

Proceed only after confirming no key and no certificate in the selected slot. Create a new working directory for public material:

```powershell
$work = Join-Path $env:USERPROFILE 'YubiKey-BitLocker-Public'
if (Test-Path -LiteralPath $work) { throw 'Choose a fresh working directory.' }
New-Item -ItemType Directory -Path $work | Out-Null
$publicKey = Join-Path $work 'public-key.pem'
$certificateFile = Join-Path $work 'certificate.cer'
```

Select **one** policy, then run the generation command once. The example below selects touch-only; use the table for the other values:

```powershell
$pinPolicy = 'never'
$touchPolicy = 'always'
& $ykman piv keys generate --algorithm RSA2048 `
    --pin-policy $pinPolicy --touch-policy $touchPolicy $slot $publicKey
if ($LASTEXITCODE -ne 0) { throw 'Generation failed; inspect state before retrying.' }
& $ykman piv keys info $slot
```

Confirm `Origin: GENERATED` and the chosen PIN/touch policies. This creates the private key on the device and exports only its public key. Do not substitute an import-private-key command if your goal is on-device generation.

## Create and register the certificate

Keep the key visible and use a local interactive terminal. Press Enter to start, then touch the contact when it flashes. Administrative PIN requests are possible even for a key whose unlock policy is PIN NEVER.

```powershell
Read-Host 'Press Enter when ready, then touch the key only when it flashes' | Out-Null
& $ykman piv certificates generate `
    --subject 'CN=BitLocker YubiKey Data Drive' --valid-days 3650 `
    --no-update-chuid $slot $publicKey
if ($LASTEXITCODE -ne 0) {
    throw 'Certificate failed. Reuse this key; do not rerun key generation.'
}
& $ykman piv certificates export --format DER $slot $certificateFile
if ($LASTEXITCODE -ne 0) { throw 'Certificate export failed.' }
$cert = Import-Certificate -FilePath $certificateFile -CertStoreLocation Cert:\CurrentUser\My
$thumbprint = $cert.Thumbprint
```

YubiKey Manager's simple self-signed certificate command does not offer the dedicated BitLocker EKU options used by the Windows-provider baseline. Microsoft permits a certificate without EKU; if EKU is present it must match BitLocker policy. The observed successful touch-only unlock event identified the protector for this simple certificate. Do not claim a later dedicated certificate was essential to that success.

Before using a certificate, Windows must associate it with the matching hardware private key:

```powershell
certutil -csp 'Microsoft Base Smart Card Crypto Provider' -key
certutil -user -repairstore My $thumbprint
$cert = Get-Item "Cert:\CurrentUser\My\$thumbprint"
$cert | Select-Object Subject,Thumbprint,HasPrivateKey
```

Repair may exercise the key, so be ready for touch/PIN prompts. If the key container is missing, stop, unplug/reinsert the YubiKey, and allow the minidriver to refresh. Yubico documents a default refresh window of 300 seconds for keys enrolled outside its minidriver. Inspect the provider/container association before continuing. A success exit code accompanied by an unsuccessful private-key test is not sufficient validation. Do not regenerate keys repeatedly or write arbitrary card objects to work around a refresh delay.

For a dedicated BitLocker EKU certificate generated on Windows, follow the [provider-based baseline](FIELD-NOTES.md#generate-a-dedicated-certificate-on-the-key). To enroll such a certificate against an existing key, New-SelfSignedCertificate supports ExistingKey and Container; those require the actual verified container name for that key. Never copy a container name from another PC. This existing-key certificate variant was exercised during troubleshooting, but is not required by the successful simple-certificate path above.

## Add the unlock method to the drive

On a personally managed PC, permit self-signed certificates as described in the baseline. Keep enterprise certificate-policy decisions with the administrator.

For an **already encrypted drive**, unlock it with a known working method first. Adding a protector to a locked drive fails. Then:

```powershell
$drive = 'D:'
if ((Get-BitLockerVolume $drive).LockStatus -ne 'Unlocked') {
    throw 'Unlock the drive with its current method before adding a protector.'
}
manage-bde -protectors -add $drive -certificate -ct $thumbprint
if ($LASTEXITCODE -ne 0) { throw 'Protector addition failed; do not start an unlock test.' }
```

For a **new unencrypted drive**, use the shared [recovery creation and full-encryption steps](FIELD-NOTES.md#add-protectors-and-back-up-recovery-before-encryption). Create and verify recovery before encryption, and wait for 100% encrypted with Protection On before claiming completion.

## Test the exact key

Close files and applications using the data drive. Check that the recovery file contains the expected password and that an independent secure backup exists. Do not use a force-dismount flag for a routine test.

```powershell
Lock-BitLocker -MountPoint $drive -ErrorAction Stop | Out-Null
Read-Host 'Press Enter when ready to unlock, then watch the key' | Out-Null

# Touch-only or presence-only: no -pin argument.
manage-bde -unlock $drive -certificate -ct $thumbprint
if ($LASTEXITCODE -ne 0) { throw 'Unlock failed; retain the known working protector.' }
```

For the PIN-protected path, use this unlock command instead and enter the PIN at the local prompt:

```powershell
manage-bde -unlock $drive -certificate -ct $thumbprint -pin
```

Do not click a different GUI unlock method during this test; it may select the old key. If nothing flashes and the command immediately fails, stop and inspect the certificate association, protector, policy and error. Touching an idle key can type an OTP into the terminal; it does not approve a PIV operation that already ended.

Verify both the volume and the event identifying which protector unlocked it:

```powershell
Get-BitLockerVolume $drive |
    Select-Object VolumeStatus,ProtectionStatus,LockStatus,AutoUnlockEnabled
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-BitLocker/BitLocker Management'
    Id=782
} -MaxEvents 3 | Select-Object TimeCreated,Message
```

Compare the event's drive and protector ID to the intended protector, not just the Unlocked state. Hardware policy metadata and the matching event together confirmed our touch-only path.

## Keep only the chosen method and recovery

Test recovery from the actual saved copy, preferably with the YubiKey removed. Keep a secure copy outside this PC. Recovery bypasses the YubiKey and must stay private.

After the intended unlock and recovery pass, list protectors and remove only the reviewed alternate certificate protector IDs. Preserve the tested new protector and recovery. These commands are templates; replace only the placeholder for a specific alternate protector after reviewing it:

```powershell
(Get-BitLockerVolume $drive).KeyProtector |
    Select-Object KeyProtectorId,KeyProtectorType
# Remove-BitLockerKeyProtector -MountPoint $drive -KeyProtectorId '{ALTERNATE-PROTECTOR-ID}'
```

Removing a volume protector does not delete a YubiKey private key. Do not erase old card keys without checking whether other drives or files depend on them. For one chosen certificate plus recovery, the expected final volume has exactly one PublicKey protector and one RecoveryPassword protector, with automatic unlock disabled.

## USB touch and NFC are different

USB touch means contacting the key's metal sensor while it is plugged in. NFC uses a compatible contactless smart-card reader; a key having NFC does not give the PC a reader. Our PC exposed only a USB smart-card reader. NFC unlocking, another computer, and fingerprint-based variants were not tested. OTP is not a native BitLocker unlock method. Removing a key never automatically relocks a volume that is already unlocked.

## Sources

- [Yubico PIV policies and generation commands](https://docs.yubico.com/software/yubikey/tools/ykman/PIV_Commands.html)
- [Yubico minidriver defaults and refresh behavior](https://docs.yubico.com/software/yubikey/tools/minidriver/md_config-registry.html)
- [Microsoft certificate protector requirements](https://learn.microsoft.com/en-us/windows/win32/secprov/protectkeywithcertificatethumbprint-win32-encryptablevolume)
- [Microsoft smart-card certificate generation parameters](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate)
- [Microsoft certificate unlock command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-unlock)
