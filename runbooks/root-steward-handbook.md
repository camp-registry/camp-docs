# camp root signing: steward instructions

You hold one of four hardware keys behind camp's root. Signing takes three of the four. What you do: one setup call (about an hour), one 15-minute re-sign a year, and be reachable for rare emergencies. This page is the whole procedure, commands included. Assumes a YubiKey 5 series key that has not been used for PIV before.

## Before the call: prepare your computer (10 minutes)

**Check the key first.** The procedure needs the PIV application, which every YubiKey 5 series key has (5 NFC, 5C, 5C NFC, 5Ci, 5 Nano, 5C Nano, and the FIPS variants) and the YubiKey 4 series had. The blue "Security Key by Yubico" models, Google Titan keys and other FIDO-only keys do not have it and cannot be used here, whatever their form factor. If `yubico-piv-tool -a status` below fails to find a PIV application, that is the cause; tell David and get a 5-series key before the call.

Install two command-line tools. Nothing secret is ever stored on the computer; any laptop is fine.

- **macOS:** `brew install yubico-piv-tool opensc` (Homebrew prints a caveat that "the OpenSSH PKCS11 smartcard integration will not work"; ignore it, we do not use OpenSSH for this)
- **Windows (not used by the current stewards, kept for reference):** install "Yubico PIV Tool" from https://developers.yubico.com/yubico-piv-tool/Releases/ and "OpenSC" from https://github.com/OpenSC/OpenSC/releases (both are standard installers). Commands below run in PowerShell.
- **Linux (Debian, Ubuntu 22.04/24.04):** `sudo apt install yubico-piv-tool ykcs11 opensc` (the `ykcs11` package is the PKCS#11 module; without it the sign step has nothing to load)

Plug the key in and check it is seen:

```
yubico-piv-tool -a status
```

Set your PIN now (6 to 8 digits; the factory PIN is `123456`). Choose one you will still know in a year; after three wrong tries the slot locks:

```
yubico-piv-tool -a change-pin
```

Note where the PKCS#11 module landed; you need the path on the call:

- macOS (Apple silicon): `/opt/homebrew/lib/libykcs11.dylib` (Intel: `/usr/local/lib/libykcs11.dylib`)
- Windows: `C:\Program Files\Yubico\Yubico PIV Tool\bin\libykcs11.dll`
- Linux: `/usr/lib/x86_64-linux-gnu/libykcs11.so`

Confirm the tools talk to each other (it should list the key with no error; the slot may be empty, that is fine):

```
pkcs11-tool --module <libykcs11 path> --list-slots
```

Tell David which operating system you used so nothing surprises anyone on the call. Stop here: step 1 (generating the key) is done together on the call. If you have already run it, no harm done; you will run it again on the call, which replaces the key, and the PEM from that run is the one you paste.

## On the call (about an hour, video, with chat for pasting)

**1. Generate your key.** Creates a P-256 key inside slot 9c (digital signature). The private half never leaves the chip. It asks for the PIN on every signature and a touch of the key.

```
yubico-piv-tool -a generate -s 9c -A ECCP256 --pin-policy=always --touch-policy=cached -o camp-root-<github-username>.pem
```

Use your GitHub username in the file name (for example `camp-root-davidpesce.pem`); it is how the registry names people everywhere else. Paste the contents of the file (a public key, not secret) into the call chat. David collects the four.

If you look in YubiKey Manager or Yubico Authenticator afterwards, slot 9c still shows as empty. That is expected: those apps list certificates, and we never load one; the key is in the slot. `ykman piv info` lists the key on current firmware, and the self-check in step 4 works on any file if you want proof.

**2. Receive the root payload.** David publishes the same two files for everyone and pastes two `curl -O` lines into the chat; run them in the folder you will sign from. `root.json` is readable: it lists the four public keys, the threshold of 3, the registry's online keys, the expiry. `root-payload.bin` is the exact bytes to sign. Open `root.json` and check your own public key is in it. (Neither file is secret; step 3 is what proves you received them intact.)

**3. Check the bytes together.** Compute the fingerprint of the payload and read it aloud; David reads his. They must match exactly, for everyone, before anyone signs. This step is the job.

- macOS: `shasum -a 256 root-payload.bin`
- Linux: `sha256sum root-payload.bin`
- Windows: `Get-FileHash root-payload.bin -Algorithm SHA256`

**4. Sign.** Run this from the folder where you saved `root-payload.bin` (or give full paths for the two files; "Cannot open root-payload.bin" means you are in a different folder). It asks for your PIN twice, once to log in and once for the signature itself, same PIN both times, then the key blinks for a touch.

```
pkcs11-tool --module <libykcs11 path> --login --sign --mechanism ECDSA-SHA256 --id 2 --signature-format openssl --input-file root-payload.bin --output-file camp-root-<github-username>.sig
```

Optional self-check before sending (macOS/Linux, or Windows with Git installed):

```
openssl dgst -sha256 -verify camp-root-<github-username>.pem -signature camp-root-<github-username>.sig root-payload.bin
```

The chat cannot carry files, so paste the signature as text: macOS `base64 -i camp-root-<github-username>.sig`, Linux `base64 -w0 camp-root-<github-username>.sig`, and paste the output. Signatures are not secret, and David's assembly step verifies each one against your public key, so a garbled paste is caught, not accepted.

**5. Done when David confirms.** He attaches the signatures, the verifier reports at least three valid, and he publishes the record (date, names, key fingerprints, root fingerprint). Keep the key somewhere safe and boring, not on your daily keyring. Keep the `.pem` file; it is public and handy for later self-checks.

## The annual re-sign (15 minutes, about eleven months from now)

Steps 2 to 5 again with a new `root.json` and `root-payload.bin`. Usually the only change is the expiry date; David says so and the file shows it. Three of four suffice, so tell him if you cannot make it.

## If something happens

- **Lost, broken, PIN forgotten:** tell David. Your key alone signs nothing. You generate a new key (step 1) and the others sign a root that swaps in your new public key.
- **Stepping down, or a fifth steward joining:** same procedure, one key removed or added, threshold stays 3.
- **A registry online key is compromised:** an unscheduled re-sign; David explains the change, you verify the file shows exactly that, sign.
- **Someone rushing you or asking you to sign a file you have not hashed:** decline. There is always time for step 3.

## What your signature means

"I confirm this exact file." Three things make it true: the fingerprint you computed matches what David read out; your public key is in the file; the changes described are the only changes there. Nothing else is expected of you.
