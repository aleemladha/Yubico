# Keyfield

An independent, interactive field guide to YubiKey smart-card unlocking for Windows BitLocker data drives.

Choose an unlock policy, review its tradeoffs, and generate step-by-step PowerShell commands for your data drive. The page includes command copying, a verification checklist, security explanations, and downloadable documentation. Everything runs locally in the browser; it never reads your device or asks for PINs, recovery passwords, or private keys.

## Read the guides

- [Setup options and commands](SETUP-OPTIONS.md)
- [Detailed field notes and the original PIN-protected baseline](FIELD-NOTES.md)
- Open `index.html` in a browser to use the interactive guide offline.

The site is designed for GitHub Pages from the root of the `main` branch. No build step, dependency installation, or third-party script is required.

## Validation

The website was tested in Chromium at desktop, tablet, and mobile widths. Tests covered whole-row policy selection, keyboard selection, matching generated commands, the empty-slot confirmation gate, clipboard copying, configuration settings, checklist persistence/reset, downloads, and page overflow. All 29 generated PowerShell variants passed syntax checks without execution.

On the observed Windows 11 Enterprise and YubiKey 5C NFC configuration, original PIN-only unlocking, touch-only unlocking, full XTS-AES 256 encryption, and recovery from the saved backup passed. PIN ALWAYS plus touch ALWAYS did not pass BitLocker unlocking. Other combinations retain their untested labels. Browser tests do not validate hardware behavior on a visitor's PC.

## Security boundaries

These instructions apply to data drives, not preboot unlocking of the operating system drive. Back up your files before changing encryption. Never overwrite occupied PIV slots or remove a working protector before testing its replacement and recovery. Touch-only allows anyone holding the key to approve use. Recovery passwords bypass the key and must be protected independently. Removing a key does not relock an unlocked drive.

This is a community draft with independently written instructions and links to vendor documentation. It is not affiliated with OpenAI, Microsoft, or Yubico. Its visual style takes inspiration from OpenAI's Defense Factory page.
