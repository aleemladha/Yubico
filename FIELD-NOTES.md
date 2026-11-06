# YubiKey smart card unlocking for BitLocker data drives

**Community draft — full encryption, original PIN-based unlocking, recovery from the saved backup, and touch-only hardware-key unlocking have completed. The final installation retains only the tested touch-only certificate protector and recovery password, with automatic unlock disabled. The combined tutorial has not been rerun on a second fresh drive.**

See [Setup options and commands](SETUP-OPTIONS.md) to choose PIN-only, PIN plus touch, touch-only, or presence-only policies. This README contains the original PIN-protected baseline and shared encryption/recovery steps. Do not apply that baseline again to an already configured drive.

Validation observed on the installation: 100% encrypted, Protection On, automatic unlock disabled; a normal lock succeeded, followed by an interactive certificate unlock command with `-pin`, after which the drive reported Unlocked. The PIN was entered locally and was not collected in the guide. A command without `-pin` had failed to unlock, so the working interactive command is:

```powershell
# Substitute your certificate thumbprint. Enter the PIN only at the local prompt.
manage-bde -unlock D: -certificate -ct YOUR_CERTIFICATE_THUMBPRINT -pin
```

The original PIN-only unlock was verified from the unlocked volume state and subsequently from the BitLocker event log. Recovery was later tested by locking the volume and successfully unlocking with the saved recovery password, without displaying that password. Cross-PC compatibility has not been tested.

## Touch only test results

A separate RSA-2048 key was generated directly on the YubiKey in previously empty slot 82 with PIN policy NEVER and touch policy ALWAYS. Windows' direct RSA encryption/decryption test passed, and a subsequent successful BitLocker unlock was confirmed in event 782 against a protector using this slot 82 key. The hardware metadata independently reports GENERATED, PIN NEVER, and touch ALWAYS.

This is an optional possession-based configuration, not the PIN-protected baseline below. Anyone holding the YubiKey can approve use by touching it. While another protector remains, Windows may select that other method; an unlocked drive alone is not evidence that touch was required.

Practical findings from this test:

- Windows needed time to refresh certificates created outside its minidriver. Initially it could not associate the certificate with the key; later container enumeration and repair succeeded. Repeated key generation is not a remedy for this refresh delay.
- A missing key-history object was investigated, but no key-history change was applied; Windows eventually enumerated slot 82 anyway. Do not present that proposed change as a required fix.
- Certificate generation with a touch policy can time out before a user sees instructions. A visible local terminal that waits for Enter before starting makes the timing clear.
- Touch the contact only while a PIV operation is waiting and the key flashes. A touch at an idle terminal can instead type Yubico OTP text; that text is not a PIV PIN and is not a command.
- Issuing a dedicated BitLocker certificate against an existing hardware key does not create another private key. The successful unlock event selected the earlier protector for the same slot 82 key, so this result does not prove the dedicated certificate was necessary to make touch-only work.
- PIN ALWAYS plus touch ALWAYS in slot 9d was generated successfully, but BitLocker unlocking with it did not pass. Manual GUI unlocks used the original PIN-only protector. That combination remains unvalidated here.
- A local recovery file had become empty before one test. It was recreated from the existing recovery protector while the volume was unlocked, checked, and successfully used for recovery. A prior file-creation success is not a substitute for checking the actual backup before locking a drive.

After testing, the user selected touch-only. All alternate certificate protectors were removed from the data volume, retaining the exact protector identified in the successful touch-only unlock event and the existing verified recovery protector. The final drive has exactly two protectors: PublicKey and RecoveryPassword. Other keys and certificates on the YubiKey were not deleted. Anyone with the recovery password can still unlock independently of touch.

Use a YubiKey with PIV support to unlock a BitLocker data drive in Windows. The private key stays on the YubiKey; Windows uses its certificate to protect the drive key. This covers secondary and removable data drives, not preboot unlocking of the Windows system drive. A recovery password remains an independent way to unlock the drive.

## Requirements

- Windows with BitLocker management, administrator access, and a backed-up data drive. Examples use D:; verify your drive letter first.
- A YubiKey with PIV support and the official YubiKey Smart Card Minidriver. FIDO-only Security Keys cannot follow this procedure.
- A secure place outside the PC for the BitLocker recovery password.

The setup commands below succeeded on Windows 11 Enterprise build 26200, YubiKey firmware 5.8.0, Smart Card Minidriver 5.0.4.273, and Yubico PIV Tool 2.7.3. This is one tested configuration, not a compatibility guarantee for every release. See the pending validation checklist below.

## Security defaults and deliberate choices

This procedure makes explicit security choices; Windows and a factory YubiKey do not automatically configure all of them. The setup creates a PIN-protected hardware key, but does not claim the factory PIN, touch policy, or administrative secrets are secure without configuration.

| Setting | Behavior and choice |
| --- | --- |
| Private key generation | Generated through the smart-card provider on the YubiKey; no software private-key file is generated or imported by this procedure |
| Exportability | Requests NonExportable; hardware retains the key. This is not a guarantee against every possible hardware or firmware attack. |
| Slot PIN defaults | 9a and 9d require a PIN, normally once per card session; 9e does not require one by default |
| Factory PIN | Publicly known; change it locally before relying on protection |
| Physical touch | Not required by the certificate-generation command below; PIN protection and touch are separate policies |
| Fingerprint | A standard YubiKey contact detects anyone's touch, not identity; this PIV setup does not configure fingerprint authentication |
| Automatic unlock | Keep disabled so a locked data drive needs an explicit unlock method |
| Recovery | The recovery password bypasses the need for the YubiKey and PIN; store it as carefully as any other unlock credential |
| Encryption | Explicitly selects XTS-AES 256 and entire-volume encryption for this Windows data drive |
| Self-signed certificates | Explicit machine-wide policy exception; this does not turn the certificate into a publicly trusted identity |

A stolen key alone should not satisfy a PIN-protected private-key operation, provided the PIN is not cached in an accessible session. Malware on an unlocked PC can read accessible files without extracting a private key. Removing the YubiKey does not revoke access to an already unlocked volume.

## Inspect the drive and key

Run PowerShell as administrator:

```powershell
Get-Volume -DriveLetter D
Get-BitLockerVolume -MountPoint D: |
    Select-Object MountPoint,VolumeStatus,ProtectionStatus,EncryptionPercentage,AutoUnlockEnabled
```

Inspect existing PIV certificates using Yubico's tools before generating anything. For the default PIV Tool installation:

```powershell
& 'C:\Program Files\Yubico\Yubico PIV Tool\bin\yubico-piv-tool.exe' -a status
```

This procedure was exercised on a YubiKey whose PIV certificate slots were empty. On a key with existing credentials, stop and plan slot allocation before proceeding. Never reset PIV or overwrite a slot containing a private key needed for existing encrypted data.

## Secure the PIV application

The PIV PIN is separate from the Windows sign-in PIN and the FIDO2 PIN. Change a factory PIN before relying on the key. Yubico's documented factory PIV PIN is `123456`; do not try it on a key with an unknown PIN history. Repeated wrong entries can block the PIN.

Use an interactive local terminal, not chat or a transcript that captures secrets:

```powershell
& 'C:\Program Files\Yubico\Yubico PIV Tool\bin\yubico-piv-tool.exe' -a change-pin
```

Enter the existing PIN and a new 6–8 digit PIN at the prompts. Keep it in a secure password manager. Do not supply the new PIN as a command-line argument.

Also configure a nondefault PUK and protect the management key according to the current Yubico minidriver instructions. Check their current state rather than assuming they are still factory defaults: the minidriver may change management-key handling and block a default PUK. Losing these administrative credentials can require a destructive PIV reset. A blocked PUK does not itself stop a working PIN from being used.

## Generate a dedicated certificate on the key

On an otherwise empty PIV application, this command generated an RSA key on the YubiKey and installed a dedicated self-signed BitLocker certificate:

```powershell
$params = @{
    Type              = 'Custom'
    Subject           = 'CN=BitLocker Data Drive YubiKey'
    FriendlyName      = 'BitLocker Data Drive YubiKey'
    CertStoreLocation = 'Cert:\CurrentUser\My'
    Provider          = 'Microsoft Base Smart Card Crypto Provider'
    KeyAlgorithm      = 'RSA'
    KeyLength         = 2048
    KeySpec           = 'KeyExchange'
    KeyExportPolicy   = 'NonExportable'
    KeyUsage          = @('KeyEncipherment','DataEncipherment')
    HashAlgorithm     = 'SHA256'
    NotAfter          = (Get-Date).AddYears(10)
    TextExtension     = @('2.5.29.37={text}1.3.6.1.4.1.311.67.1.1')
}
$cert = New-SelfSignedCertificate @params
$cert | Select-Object Subject,Thumbprint,HasPrivateKey
```

Complete Windows smart-card prompts locally. Record the certificate thumbprint for administration; it is not a private key.

Inspect PIV status again. In the observed setup, the minidriver allocated slot **9a**. Do not assume every system chooses the same slot. Confirm a PIN-requiring slot and policy before proceeding. Slots 9a and 9d require a PIN by default; slot **9e**, Card Authentication, does not. PIN entry may be cached within a card session. A physical-touch requirement is a separate policy and is not established by this command.

The dedicated BitLocker enhanced key usage is `1.3.6.1.4.1.311.67.1.1`. This avoids repurposing an EFS certificate and changing the machine's BitLocker certificate OID to EFS.

### How generation happens on the YubiKey

`New-SelfSignedCertificate` asks `Microsoft Base Smart Card Crypto Provider` to create the RSA key. That provider uses the installed smart-card minidriver to request generation on the YubiKey. The public key is available to Windows for the certificate; private-key operations are delegated to the card. Windows stores the certificate and a provider association rather than an exportable software private key.

`KeySpec = KeyExchange` and the key-encipherment usage make the RSA key suitable for this encryption use. `KeyExportPolicy = NonExportable` requests a non-exportable key. The provider selection is essential: the same export-policy setting on a software provider would not establish that a key was generated on hardware.

Read-only checks after generation:

```powershell
$cert | Select-Object Subject,Thumbprint,HasPrivateKey
certutil -user -store My $cert.Thumbprint
& 'C:\Program Files\Yubico\Yubico PIV Tool\bin\yubico-piv-tool.exe' -a status
```

Verify the provider association and the matching certificate on the card. `HasPrivateKey = True` means Windows has an association with a usable private key; it does not mean the private key is stored in the certificate file. Slot status alone is also not cryptographic proof of hardware generation, because slots can contain imported keys. We observed the provider-based generation flow and resulting card contents; we did not perform a separate key-attestation verification.

A `.cer` export contains a public certificate, not the private key. It can help with administration, but cannot replace a lost YubiKey or the recovery password. Never publish a PFX, PIN, PUK, management key, or recovery password.

### Optional physical touch

PIN plus touch is a separate setup variant. Yubico requires the touch policy to be selected when generating or importing the key. For example, YubiKey Manager CLI supports `--pin-policy` and `--touch-policy` during key generation. Those options are not parameters of the Windows certificate command used here.

Do not overwrite the current working slot to add touch. Generate a replacement key in an unused suitable slot, issue a matching BitLocker certificate, add its protector, test it and recovery, and only then consider retiring the old protector. This replacement procedure has not been performed here. Touch only would prove physical possession, not knowledge of a PIN or a fingerprint. OTP is a separate YubiKey function and is not a BitLocker unlock method.

## Permit the self-signed certificate

For a personally managed PC, allow self-signed BitLocker certificates:

```powershell
$policy = 'HKLM:\SOFTWARE\Policies\Microsoft\FVE'
New-Item -Path $policy -Force | Out-Null
New-ItemProperty -Path $policy -Name SelfSignedCertificates `
    -PropertyType DWord -Value 1 -Force | Out-Null
```

This is a machine-wide policy change. On a managed work PC, use your administrator's certificate and policy process instead. Record the old value before changing it. The certificate must still have the appropriate usage and key capabilities; allowing self-signed certificates does not waive all certificate checks.

## Add protectors and back up recovery before encryption

Use the certificate thumbprint returned above:

```powershell
manage-bde -protectors -add D: -Certificate -ct $cert.Thumbprint
if ($LASTEXITCODE -ne 0) { throw 'Certificate protector failed; stop here.' }
```

Then open **Control Panel → System and Security → BitLocker Drive Encryption** and configure D:. Select smart-card unlocking if the wizard offers the choice. Save the recovery password outside D: and copy it to a secure location outside this PC before starting encryption. A backup on C: is only a temporary local copy. Keep the recovery password out of public logs, screenshots, and issue reports.

For an existing drive, choose **Encrypt entire drive** and software encryption appropriate to the computers that will use it. The observed setup used XTS-AES 256 and full-volume encryption. Check all command and wizard errors before continuing. Do not remove an existing working protector until its replacement and recovery method have been tested.

The exact wizard sequence above still requires a separate clean-install walkthrough. The observed installation used Windows' Win32_EncryptableVolume API instead, as detailed next. Choose one path; do not add duplicate protectors by following both.

### Raw PowerShell procedure used for the installation

These are the underlying operations, with generic names and additional error checks. They are for a new, fully decrypted data drive with no existing protectors, in the same elevated PowerShell session as `$cert`. They are not a script to rerun against a drive already configured by the preceding instructions. Each WMI call returns a status code; PowerShell's ErrorAction alone does not check that code.

First, confirm the target, obtain the volume object, and add the certificate protector:

```powershell
$ErrorActionPreference = 'Stop'
$drive = 'D:'
$state = Get-BitLockerVolume -MountPoint $drive
if ($state.VolumeType -ne 'Data' -or
    $state.VolumeStatus -ne 'FullyDecrypted' -or
    @($state.KeyProtector).Count -gt 0) {
    throw 'Stop: expected a fully decrypted data drive with no protectors.'
}
if (-not $cert.HasPrivateKey) { throw 'Certificate private-key association missing.' }

function Assert-BitLockerResult($result, $operation) {
    if ($result.ReturnValue -ne 0) {
        throw ('{0} failed: 0x{1:X8}' -f $operation, [uint32]$result.ReturnValue)
    }
}
$volume = Get-CimInstance `
    -Namespace root/CIMV2/Security/MicrosoftVolumeEncryption `
    -ClassName Win32_EncryptableVolume -Filter "DriveLetter='$drive'"
if (-not $volume) { throw 'Target volume not found.' }
$smartCard = Invoke-CimMethod -InputObject $volume `
    -MethodName ProtectKeyWithCertificateThumbprint `
    -Arguments @{ CertThumbprint=$cert.Thumbprint; FriendlyName='YubiKey data drive' }
Assert-BitLockerResult $smartCard 'Add certificate protector'
```

Next, create a restricted recovery folder on C:. This local folder is temporary storage, not an independent disaster-recovery backup. The permissions below allow only the current user, SYSTEM, and local Administrators. Use a new folder to avoid changing access to existing documents.

```powershell
$backupDir = Join-Path $env:USERPROFILE 'BitLocker-Community-Recovery'
if (Test-Path -LiteralPath $backupDir) {
    throw 'Choose a fresh backup directory; do not overwrite an existing backup.'
}
New-Item -ItemType Directory -Path $backupDir | Out-Null
$acl = Get-Acl -LiteralPath $backupDir
$acl.SetAccessRuleProtection($true, $false)
$currentSid = [System.Security.Principal.WindowsIdentity]::GetCurrent().User
$allowedSids = @(
    $currentSid
    [System.Security.Principal.SecurityIdentifier]::new('S-1-5-18')
    [System.Security.Principal.SecurityIdentifier]::new('S-1-5-32-544')
)
foreach ($sid in $allowedSids) {
    $rule = [System.Security.AccessControl.FileSystemAccessRule]::new(
        $sid, 'FullControl', 'ContainerInherit,ObjectInherit', 'None', 'Allow')
    $acl.AddAccessRule($rule)
}
Set-Acl -LiteralPath $backupDir -AclObject $acl
```

Create and save a recovery password without printing it into the terminal or a shared transcript:

```powershell
$recovery = Invoke-CimMethod -InputObject $volume `
    -MethodName ProtectKeyWithNumericalPassword `
    -Arguments @{ FriendlyName='Data drive recovery' }
Assert-BitLockerResult $recovery 'Add recovery protector'
$secret = Invoke-CimMethod -InputObject $volume `
    -MethodName GetKeyProtectorNumericalPassword `
    -Arguments @{ VolumeKeyProtectorID=$recovery.VolumeKeyProtectorID }
Assert-BitLockerResult $secret 'Retrieve recovery password'
$backupFile = Join-Path $backupDir 'BitLocker-Data-Recovery.txt'
@(
    "Drive: $drive"
    "Protector ID: $($recovery.VolumeKeyProtectorID)"
    "Recovery password: $($secret.NumericalPassword)"
) | Set-Content -LiteralPath $backupFile -Encoding UTF8
if (-not (Get-Content -LiteralPath $backupFile -Raw).Contains($secret.NumericalPassword)) {
    throw 'Recovery file verification failed; do not start encryption.'
}
Export-Certificate -Cert $cert `
    -FilePath (Join-Path $backupDir 'YubiKey-BitLocker-Public.cer') | Out-Null
Write-Host "Recovery backup saved and verified at $backupFile"
Remove-Variable secret
```

Copy the recovery password to an independent secure destination and verify that copy before continuing. The password is an alternative unlock credential; it is not encrypted inside the text file. Clearing a PowerShell variable is not a guarantee that all copies have been erased from process memory.

Finally, start full software encryption and enable protectors:

```powershell
$started = Invoke-CimMethod -InputObject $volume -MethodName Encrypt `
    -Arguments @{ EncryptionMethod=[uint32]7; EncryptionFlags=[uint32]0 }
Assert-BitLockerResult $started 'Start encryption'
$enabled = Invoke-CimMethod -InputObject $volume -MethodName EnableKeyProtectors
Assert-BitLockerResult $enabled 'Enable protectors'
Get-BitLockerVolume -MountPoint $drive |
    Select-Object MountPoint,VolumeStatus,ProtectionStatus,EncryptionMethod,
        EncryptionPercentage,AutoUnlockEnabled,
        @{Name='ProtectorTypes';Expression={$_.KeyProtector.KeyProtectorType -join ', '}}
```

Method 7 selects XTS-AES 256; flags 0 selects full-volume conversion. Expected protectors are PublicKey and RecoveryPassword. Automatic unlock should be False. If it is unexpectedly enabled, investigate the existing configuration rather than assuming the YubiKey is required.

If any step fails, stop and inspect the existing protectors and backup. Do not delete keys, reset PIV, remove working protectors, or blindly rerun the whole procedure. The successful method calls above were used on the observed installation; the combined publication example with extra guards has been syntax-checked, not rerun against a second fresh drive.

Keep automatic unlocking disabled if you want the key to be needed after the data volume locks. Removing a YubiKey does **not** relock an already unlocked volume.

## Verify encryption and unlocking

```powershell
manage-bde -status D:
```

Wait for **Fully Encrypted**, **100%**, and **Protection On**. During partial encryption, Windows can report Protection Off even when protectors exist. That status is not proof of completed protection. If it remains Off after completion, investigate before relying on the setup.

After saving work and closing applications using the drive:

1. Confirm the recovery backup is readable and stored outside this PC.
2. Restart Windows or lock the data drive without forcing a dismount. Do not interrupt active file writes.
3. Confirm the volume is locked and cannot be opened without supplying an unlock method.
4. Insert the YubiKey, open the drive, and use the smart-card option with the new PIV PIN. Remove and reinsert the key between tests to avoid mistaking PIN caching for a missing PIN requirement.
5. Lock the drive again and test the saved recovery password through the Windows recovery UI, without exposing it in shell history.
6. If another PC must unlock this drive, install the required driver there and test that path too.

Do not publish an end-to-end success claim until these tests pass. Do not repeatedly guess a PIN to test failure behavior.

## Common problems

| Symptom | Check |
| --- | --- |
| Self-signed certificate rejected, error 0x80310086 | SelfSignedCertificates policy and any centrally managed policy |
| No suitable certificate | PIV support, driver, private-key association, key encipherment usage, BitLocker OID, certificate validity |
| Provider error 0x80090017 | The exact Smart Card KSP plus KeyExchange command failed on the observed system; the Base Smart Card Crypto Provider command above succeeded. This does not establish that KSP is universally unsupported. |
| No new PIN prompt | Cached PIN, an already unlocked volume, automatic unlock, or a slot configured without a PIN |
| Key removed but files still accessible | Expected while the volume remains unlocked; save work and lock it or shut down |
| Lost or reset YubiKey | Use the separately stored BitLocker recovery password; a public certificate export alone cannot recover the private key |

## Changes from older community instructions

Some older instructions import a locally created EFS key into Card Authentication and change BitLocker policy to accept the EFS OID. This approach instead generates a dedicated key on the smart card, uses the BitLocker OID, verifies the selected slot, and explicitly includes recovery and unlocking tests. Importing a software-generated key can leave other copies of that key on the PC or in backups.

The [xmelsky gist](https://gist.github.com/xmelsky/cbd12fe448d6e8d193f5735ae2cb214c) prompted this review. This guide uses independently written instructions and does not reproduce its text.

## Sources

- [Microsoft BitLocker FAQ](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/faq)
- [Microsoft New-SelfSignedCertificate](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate)
- [Microsoft certificate protector command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-protectors)
- [Microsoft unlock command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-unlock)
- [Microsoft protection status semantics](https://learn.microsoft.com/en-us/windows/win32/secprov/getprotectionstatus-win32-encryptablevolume)
- [Microsoft certificate protector API](https://learn.microsoft.com/en-us/windows/win32/secprov/protectkeywithcertificatethumbprint-win32-encryptablevolume)
- [Microsoft encryption API and flags](https://learn.microsoft.com/en-us/windows/win32/secprov/encrypt-win32-encryptablevolume)
- [Yubico Manager PIV commands](https://docs.yubico.com/software/yubikey/tools/ykman/PIV_Commands.html)
- [Yubico PIV slots and policies](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-apps-piv.html)
- [Yubico minidriver guide](https://docs.yubico.com/software/yubikey/tools/minidriver/)
- [Yubico PIV Tool](https://developers.yubico.com/yubico-piv-tool/)
