# Pico HSM — schmidtw fork (Waveshare RP2350)

> **This is a fork of [polhenarejos/pico-hsm](https://github.com/polhenarejos/pico-hsm).**
> Branch `waveshare-pico2350/v6.6` = upstream tag `v6.6` plus board-specific
> changes for a Waveshare RP2350 board with a WS2812 LED on GP16.
> `master` mirrors upstream; build from the branch, not from `master`.
>
> Devices built from this tree report USB manufacturer `schmidtw/pico-hsm`.
> Upstream's documentation follows below and still applies.

## Target hardware

| | |
|---|---|
| Board | Waveshare RP2350 (`PICO_BOARD=pico2`) |
| Flash | 2 MB and 4 MB variants both work — no build change needed |
| Status LED | WS2812 on GP16, RGB channel order |
| USB ID | `20A0:4230` (Nitrokey HSM) |

The USB ID is deliberate. `libccid` resolves readers against the **stock**
`ifd-ccid.bundle/Contents/Info.plist` only — it ignores the bundle it was
loaded from — so a supplementary CCID bundle can never register a new
VID/PID. Any other ID means patching `pcsc-lite-ccid` on every host.

## Prerequisites

Fedora ships everything needed; no third-party ARM toolchain required.

```sh
sudo dnf install -y arm-none-eabi-gcc-cs arm-none-eabi-gcc-cs-c++ \
                    arm-none-eabi-newlib arm-none-eabi-binutils-cs \
                    cmake make git python3
```

The Pico SDK is a separate checkout and is **not** vendored here. Only the
`tinyusb` and `mbedtls` submodules are needed — a full recursive clone pulls
far more than this build uses:

```sh
git clone -b 2.2.0 --depth 1 https://github.com/raspberrypi/pico-sdk.git
git -C pico-sdk submodule update --init --depth 1 lib/tinyusb lib/mbedtls
```

Pin `2.2.0`; that is the version this branch is built and tested against.

`cmake` also fetches `picotool` and re-checks-out `mbedtls` (pinned to
`v3.6.6` by the SDK) from GitHub at **configure** time, so the first
configure needs network access.

## Build

```sh
git clone --recursive -b waveshare-pico2350/v6.6 \
    https://github.com/schmidtw/pico-hsm.git
cd pico-hsm

# PICO_SDK_PATH must point at the pico-sdk checkout from above.
PICO_SDK_PATH=../pico-sdk cmake -S . -B build \
    -DPICO_BOARD=pico2 \
    -DVIDPID=NitroHSM \
    -DWS2812_PIN=16 \
    -DLED_ORDER=RGB \
    -DLED_BRIGHTNESS=2

make -C build -j$(nproc)
```

| Flag | Purpose | Required |
|---|---|---|
| `PICO_BOARD=pico2` | RP2350 target; wrong value yields an incompatible UF2 family | yes |
| `VIDPID=NitroHSM` | `20A0:4230` — the only ID Fedora's stock CCID driver recognises | **yes** |
| `WS2812_PIN=16` | Selects the WS2812 driver and its GPIO. Omitted, the firmware drives GP25, which is not wired on this board | for the LED |
| `LED_ORDER=RGB` | This board's LED is RGB-ordered; omitted, red and green appear swapped | cosmetic |
| `LED_BRIGHTNESS=2` | 0–15 | cosmetic |

`PICO_FLASH_SIZE_BYTES` is **not** needed. `low_flash.c` clamps the
filesystem to the JEDEC-detected chip size, so one build serves both the
2 MB and 4 MB boards.

**Never pass `-DCMAKE_C_FLAGS`.** It replaces the Pico SDK toolchain's
architecture flags and the build fails deep inside the SDK with a
misleading spin-lock error. Use `target_compile_definitions` instead.

Confirm the options reached the compiler:

```sh
tr ' ' '\n' < build/CMakeFiles/pico_hsm.dir/flags.make \
    | grep -E 'WS2812|LED_ORDER|LED_BRIGHTNESS'
```

## Flash

Hold **BOOTSEL** while plugging the board in, then:

```sh
cp build/pico_hsm.uf2 /run/media/$USER/RP2350/ && sync
```

The board reboots itself. A copy error at the end is normal — the device
disconnects mid-write by design. Verify:

```sh
lsusb | grep 20a0:4230
opensc-tool -l            # "Nitrokey Nitrokey HSM ... "
```

A UF2 only rewrites the program area; the filesystem partition survives
reflashing, so **a reflashed device still holds its previous keys and PINs.**

## Provisioning — required before first use

`sc-hsm-tool --initialize` alone is **not sufficient** on a device that has
been used before. `cmd_initialize.c` regenerates the device authentication
key only when the old MKEK cannot be recovered:

```c
if (ret_mkek != PICOKEY_OK || !file_has_data(fdkey)) { ...regenerate... }
```

Initialising with an unchanged PIN recovers the old MKEK and deliberately
preserves the existing device key. If that key was written by different
firmware it cannot be parsed, and **every key generation afterwards fails
with `CKR_GENERAL_ERROR` / SW `6400`**, no matter how often you reinitialise.

Break MKEK recovery by initialising twice, with different PINs the first
time — both the user PIN and the SO-PIN must differ, since recovering
either one preserves the old key:

```sh
# Pass 1 - throwaway credentials. Both must differ from your real ones:
# load_mkek() tries the user PIN and then the SO-PIN, and recovering
# either one preserves the old device key.
sc-hsm-tool --initialize --so-pin 00112233445566AA --pin 999111 --dkek-shares 1

# Pass 2 - your real credentials. The throwaway MKEK is now unrecoverable,
# so the device key is regenerated again and bound to these.
sc-hsm-tool --initialize --so-pin "$SOPIN" --pin "$USERPIN" --dkek-shares 1
```

SO-PIN is exactly 16 hex characters; the user PIN is 6–15 characters.
Power-cycle the board afterwards — `--initialize` rewrites the filesystem
and the device may not respond until it is replugged.

Then verify, because this is the step that silently fails:

```sh
pkcs11-tool --login --pin "$USERPIN" --keypairgen \
    --key-type EC:secp384r1 --id 05 --label test
```

Success means provisioning worked. `CKR_GENERAL_ERROR` means the device key
is still the old one — check that both PINs in pass 1 really differed from
those in pass 2. Delete the test key when done.

The same two passes can be driven from `pypicohsm`
(`h.initialize(pin=..., sopin=..., dkek_shares=1, no_dev_cert=True)`),
which is the path this procedure was verified against; `sc-hsm-tool` issues
the same INITIALIZE APDU.

## Python tooling

`pypicohsm` and `pypicokey` were removed from PyPI (upstream issue #126).
Mirrors exist under the [LibreKeys](https://github.com/librekeys) org
(`pypicohsm`, `pypicokey-mirror`, `pycvc`). Install with
`--system-site-packages` to reuse a distro `pyscard`, since building it
needs `pcsc-lite-devel`.

`PicoHSM()` defaults to PIN `648219` — always pass `PicoHSM(pin=...)`, or
constructing it burns a user-PIN retry.

Note that `pypicohsm`'s `initialize()` POSTs the device public key to
`https://www.picokeys.com/pico/pico-hsm/cvc/` to obtain a signed
certificate chain. **That endpoint returns 404.** Pass `no_dev_cert=True`
to skip it; the firmware writes a self-signed chain itself and validates
no issuer, so nothing is lost. Use `pycvc` if you want a chain you control.

## Changes from upstream

| Commit | Change |
|---|---|
| `cmake` | `WS2812_PIN`, `LED_ORDER`, `LED_BRIGHTNESS` build options |
| `pico-keys-sdk` | build-time LED brightness and RGB colour-order defaults |
| `hsm` | PKCS#15 token manufacturer reports `schmidtw/pico-hsm` |
| `pico-keys-sdk` | USB manufacturer string reports `schmidtw/pico-hsm` |

The product string stays `Pico Key`: interface descriptors are composed as
`<product> <interface>`, so changing it would rename the CCID interface and
pcscd's reader name.

---

# Pico HSM
This project aims to transform a Raspberry Pi Pico or ESP32 microcontroller into a Hardware Security Module (HSM). The modified Pico or ESP32 board will be capable of generating and storing private keys, performing AES encryption or decryption, and signing data without exposing the private key. Specifically, the private key remains securely on the board and cannot be retrieved since it is encrypted within the flash memory.

## Capabilities
### > Key generation and encrypted storage
Private and secret keys are secured using a master AES 256 key (MKEK). The MKEK is encrypted with a hashed and salted version of the PIN.
**No private/secret keys, DKEK or PIN are stored in plain text ever. Never.**

### > RSA Key Generation (1024 to 4096 Bits)
RSA key generation is supported for 1024, 2048, 3072, and 4096 bits. Private keys never leave the device.

### > ECDSA Key Generation (192 to 521 Bits)
ECDSA key generation supports various curves from 192 to 521 bits.

### > ECC Curves
Supported ECC curves include secp192r1, secp256r1, secp384r1, secp521r1, brainpoolP256r1, brainpoolP384r1, brainpoolP512r1, secp192k1 (insecure), secp256k1, Curve25519, and Curve448.

### > SHA Digests
ECDSA and RSA signatures can be combined with SHA-1, SHA-224, SHA-256, SHA-384, and SHA-512 digests.

### > Multiple RSA Signature Algorithms
Supported RSA signature algorithms include RSA-PSS, RSA-PKCS, and raw RSA signatures.

### > ECDSA Signatures
ECDSA signatures can be raw or pre-hashed.

### > ECDH Key Derivation
Supports the ECDH algorithm for calculating shared secrets.

### > EC Private Key Derivation
Allows ECDSA key derivation.

### > RSA Decryption
Supports RSA-OEP and RSA-X.509 decryption.

### > AES Key Generation
Supports AES key generation with keys of 128, 192, and 256 bits.

### > AES-CBC Encryption/Decryption
Performs AES-CBC encryption and decryption.

### > Advanced AES Modes
Supports AES encryption and decryption in ECB, CBC, CFB, OFB, XTS, CTR, GCM, and CCM modes, with customizable IV/nonce and additional authenticated data (AAD).[^4]

### > AES Key Generation (128, 192, 256, 512 Bits)
Supports AES key generation up to 512 bits, useful for AES XTS where two 256-bit keys are concatenated.

### > CMAC
Supports AES-CMAC authentication.[^1]

### > AES Secret Key Derivation
Supports AES secret key derivation.[^1]

### > PIN Authorization
Private and secret keys require prior PIN authentication. Supports alphanumeric PINs.

### > PKCS11 Compliant Interface
Interfacing with the PKCS11 standard is supported.

### > Hardware Random Number Generator (HRNG)
Contains an HRNG designed for maximum entropy.

### > Device Key Encryption Key (DKEK) Shares
Supports importing DKEK shares to wrap, unwrap, and encrypt keys.

### > DKEK n-of-m Threshold Scheme
Supports an n-of-m threshold scheme to prevent outages when a DKEK custodian is unavailable.

### > USB/CCID Support
Full USB CCID stack for communication with the host via OpenSC and PCSC, allowing the use of frontend applications like OpenSSL via the PKCS11 module.

### > Extended APDU Support
Supports extended APDU packets, allowing up to 65535 bytes.

### > CV Certificates
Handles CVC certificates and requests to minimize internal certificate storage.

### > Attestation
Each generated key is attached to a certificate signed by an external PKI, ensuring the key was generated by the specific device.

### > Import External Keys and Certificates
Allows importing private keys and certificates via WKY or PKCS#12 files.[^2][^3]

### > Transport PIN
Allows a transport PIN for provisioning, ensuring the device has not been tampered with during transportation.[^2]

### > Press-to-Confirm Button
Uses the BOOTSEL button to confirm operations with private/secret keys, providing a 15-second window to confirm the operation to protect against unauthorized use.

### > Store and Retrieve Binary Data
Allows the storage of arbitrary binary data files.

### > Real-Time Clock (RTC)
Includes an RTC with external date and time setting and retrieval.

### > Secure Messaging
Supports secure channels to encrypt data packets between the host and device, preventing man-in-the-middle attacks.

### > Session PIN
A specific session PIN can be set during session opening to avoid systematic PIN usage.

### > PKI CVCert Remote Issuing for Secure Messaging
Secure channel messages are secured with a certificate issued by an external PKI.

### > Multiple Key Domains
Supports separate key domains protected by independent DKEKs, allowing different keys in different domains.

### > Key Usage Counter
Tracks and limits the usage of private/secret keys, disabling keys once their usage counter reaches zero.

### > Public Key Authentication (PKA)
Supports PKA for enhanced security, requiring a secondary device for authentication using a challenge-response mechanism.

### > Secure Lock
Adds an extra layer of security by locking the Pico HSM to a specific computer using a private key.

### > ChaCha20-Poly1305
Supports the ChaCha20-Poly1305 encryption algorithm for secure data encryption.[^4]

### > X25519 and X448
Supports DH X25519 and X448 for key agreement, though these cannot be used for signing.

### > Key Derivation Functions
Supports HKDF, PBKDF2, and X963-KDF for symmetric key derivation.

### > HMAC
Supports HMAC generation with SHA digest algorithms.

### > CMAC
Supports CMAC with AES for keys of 128, 192, and 256 bits.

### > XKEK
Supports an advanced key sharing scheme (XKEK) for securely wrapping and unwrapping keys within authorized domains.

### > Master Key Encryption Key (MKEK)
Uses an MKEK to securely store all keys, encrypted with an ephemeral key derived from the hashed PIN.

### > Hierarchical Deterministic Key Generation
Supports BIP32 for asymmetric key derivation and SLIP10 for symmetric key derivation, enabling crypto wallet deployment with infinite key generation. Supports NIST 256 and Koblitz 256 curves for master key generation.[^4]

### > One Time Programming (OTP) Storage
The OTP securely stores the MKEK (Master Key Encryption Key) and Device Key permanently, making it inaccessible from external interfaces. This ensures that the key is protected against unauthorized access and tampering.

### > Secure Boot
Secure Boot ensures only authenticated firmware can run on the device, verifying each firmware’s digital signature to block unauthorized code.

### > Secure Lock
Secure Lock restricts the device to the manufacturer’s firmware only, locking out debug access and preventing any further boot key installations.

### > Rescue Interface
 A built-in rescue interface allows recovery of the device if it becomes unresponsive or undetectable. This feature provides a way to restore the device to operational status without compromising security.

### > LED Customization
 The LED can be customized to reflect device status and user preferences, offering flexible color and brightness options for an enhanced user experience.

[^1]: PKCS11 modules (`pkcs11-tool` and `sc-tool`) do not support CMAC and key derivation. It must be processed through raw APDU command (`opensc-tool -s`).
[^2]: Available via SCS3 tool. See [SCS3](/doc/scs3.md "SCS3") for more information.
[^3]: Imports are available only if the Pico HSM is previously initialized with a DKEK and DKEK shares are available during the import process.
[^4]: Available by using PicoHSM python tool.

### > ESP32-S3 support
Pico HSM also supports ESP32-S3 boards, which add secure storage, flash encryption and secure boot.

### > Dynamic VID/PID
Supports setting VID & PID on-the-fly. U

### > Rescue Pico HSM
Pico HSM Tool implements a new CCID stack to rescue the Pico HSM in case it has wrong VID/PID values and it is not recognized by the OS.

## Security considerations
All secret keys (both asymmetric and symmetric) are encrypted and stored in the flash memory. The MKEK, a 256-bit AES key, is used to protect these private and secret keys. Keys are held in RAM only during signature and decryption operations, and are loaded and cleared each time to avoid potential security vulnerabilities.

The MKEK itself is encrypted using a doubly salted and hashed PIN, and the PIN is hashed in memory during sessions. This ensures that the PIN is never stored in plaintext, neither in flash memory nor in RAM. However, if no secure channel is used, the PIN may be transmitted in plaintext from the host to the HSM.

DKEKs are used during the export and import of private/secret keys and are part of a Key Domain. A Key Domain is a set of secret/private keys that share the same DKEK. These are also shared by the custodians and are not specific to Pico HSM. Therefore, if a key does not belong to a Key Domain (and thus lacks a DKEK), it cannot be exported.

In the event that the Pico is stolen, the private and secret key contents cannot be accessed without the PIN, even if the flash memory is dumped.

### RP2350 and ESP32-S3
RP2350 and ESP32-S3 microcontrollers are equipped with advanced security features, including Secure Boot and Secure Lock, ensuring that firmware integrity and authenticity are tightly controlled. Both devices support the storage of the Master Key Encryption Key (MKEK) in an OTP (One-Time Programmable) memory region, making it permanently inaccessible for external access or tampering. This secure, non-volatile region guarantees that critical security keys are embedded into the hardware, preventing unauthorized access and supporting robust defenses against code injection or firmware modification. Together, Secure Boot and Secure Lock enforce firmware authentication, while the MKEK in OTP memory solidifies the foundation for secure operations.

## Download
**If you own an ESP32-S3 board, go to [ESP32 Flasher](https://www.picokeys.com/esp32-flasher/) for flashing your Pico HSM.**

If you own a Raspberry Pico (RP2040 or RP2350), go to [Download page](https://www.picokeys.com/getting-started/). If your board is mounted with the RP2040, then select Pico. If your board is mounted with the RP2350 or RP2354, select Pico2.

UF2 files are shiped with a VID/PID granted by RaspberryPi (2E8A:10FD). If you plan to use it with OpenSC or similar tools, you should modify Info.plist of CCID driver to add these VID/PID or use the [PicoKey App](https://www.picokeys.com/picokeyapp/ "PicoKey App").

You can use whatever VID/PID for internal purposes, but remember that you are not authorized to distribute the binary with a VID/PID that you do not own.

Note that the [PicoKey App](https://www.picokeys.com/picokeyapp/ "PicoKey App") is the most recommended.

## Build for Raspberry Pico
Before building, ensure you have installed the toolchain for the Pico and the Pico SDK is properly located in your drive.

```
git clone https://github.com/polhenarejos/pico-hsm
cd pico-hsm
git submodule update --init --recursive
mkdir build
cd build
PICO_SDK_PATH=/path/to/pico-sdk cmake .. -DPICO_BOARD=board_type -DUSB_VID=0x1234 -DUSB_PID=0x5678
make
```
Note that `PICO_BOARD`, `USB_VID` and `USB_PID` are optional. If not provided, `pico` board and VID/PID `FEFF:FCFD` will be used.

Additionally, you can pass the `VIDPID=value` parameter to build the firmware with a known VID/PID. The supported values are:

- `NitroHSM`
- `NitroFIDO2`
- `NitroStart`
- `NitroPro`
- `Nitro3`
- `Yubikey5`
- `YubikeyNeo`
- `YubiHSM`
- `Gnuk`
- `GnuPG`

After running `make`, the binary file `pico_hsm.uf2` will be generated. To load this onto your Pico board:

1. Put the Pico board into loading mode by holding the `BOOTSEL` button while plugging it in.
2. Copy the `pico_hsm.uf2` file to the new USB mass storage device that appears.
3. Once the file is copied, the Pico mass storage device will automatically disconnect, and the Pico board will reset with the new firmware.
4. A blinking LED will indicate that the device is ready to work.

### Docker
Independent from your Linux distribution or when using another OS that supports Docker, you could build a specific pico-hsm version in a Linux container.

```
sudo docker build \
    --build-arg VERSION_PICO_SDK=2.0.0 \
    --build-arg VERSION_MAJOR=5 \
    --build-arg VERSION_MINOR=0 \
    --build-arg PICO_BOARD=waveshare_rp2040_zero \
    --build-arg USB_VID=0xfeff \
    --build-arg USB_PID=0xfcfd \
    -t pico-hsm-builder .

sudo docker run \
    --name mybuild \
    -it pico-hsm-builder \
    ls -l /home/builduser/pico-hsm/build_release/pico_hsm.uf2

sudo docker cp mybuild:/home/builduser/pico-hsm/build_release/pico_hsm.uf2 .

sudo docker rm mybuild
```

## Usage
The firmware uploaded to the Pico contains a reader and a virtual smart card, similar to having a physical reader with an inserted SIM card. We recommend using [OpenSC](http://github.com/opensc/opensc/ "OpenSC") to communicate with the reader. If OpenSC is not installed, you can download and build it or install the binaries for your system.

To ensure that the Pico is detected as an HSM, use the following command:
```sh
opensc-tool -an
```
It should return a text similar to:
```sh
Using reader with a card: Free Software Initiative of Japan Gnuk
3b:fe:18:00:00:81:31:fe:45:80:31:81:54:48:53:4d:31:73:80:21:40:81:07:fa
SmartCard-HSM
```
The name of the reader may vary if you modified the VID/PID.

For further details and operations, refer to the following documentation:

- Initialization and Asymmetric Operations [doc/usage.md](/doc/usage.md)
- Signing and Verification Operations [doc/sign-verify.md](/doc/sign-verify.md)
- Asymmetric Encryption and Decryption [doc/asymmetric-ciphering.md](/doc/asymmetric-ciphering.md)
- Backup, Restore, and DKEK Share Management [doc/backup-and-restore.md](/doc/backup-and-restore.md)
- AES Key Generation, Encryption, and Decryption [doc/aes.md](/doc/aes.md)
- 4096 Bits RSA Support [doc/scs3.md](/doc/scs3.md)
- Storing and Retrieving Arbitrary Data [doc/store_data.md](/doc/store_data.md)
- Extra Options (e.g., set/get real datetime, enable/disable press-to-confirm button [doc/extra_command.md](/doc/extra_command.md)
- Public Key Authentication [doc/public_key_authentication.md](/doc/public_key_authentication.md)

## Operation time
### Keypair generation
Generating EC keys is almost instant. RSA keypair generation takes some time, specially for `3072` and `4096` bits.

| RSA key length (bits) | Average time (seconds) |
| :---: | :---: |
| 1024 | 16 |
| 2048 | 124 |
| 3072 | 600 |
| 4096 | ~1000 |

### Signature and decrypt
| RSA key length (bits) | Average time (seconds) |
| :---: | :---: |
| 1024 | 1 |
| 2048 | 3 |
| 3072 | 7 |
| 4096 | 15 |

## Press-to-confirm button
The Raspberry Pico includes a BOOTSEL button used for loading firmware initially. Once the Pico HSM firmware is running, this button can be repurposed for additional functionalities. Specifically, the Pico HSM utilizes this button to confirm private and secret operations, a feature that is optional but highly recommended for enhanced security.

When enabled, each time a private or secret key operation is initiated, the Pico HSM enters a waiting state where it awaits user confirmation by pressing the BOOTSEL button. During this waiting period, the Pico HSM's LED remains mostly illuminated but blinks off briefly every second, signaling to the user to press the button for confirmation. If no action is taken, the Pico HSM will continue to wait indefinitely. This operation mode includes periodic timeout commands sent to the host to prevent the session from timing out prematurely.

This feature adds an additional layer of security by requiring physical user intervention for sensitive operations such as signing or decrypting data. It mitigates risks associated with unauthorized applications or scripts using the Pico HSM without user awareness. However, it is not recommended for server environments or other automated settings where physical access to press the button may not be practical.

For more details on configuring and using this feature, refer to the [doc/extra_command.md](/doc/extra_command.md) document.

## Led blink
Pico HSM uses the led to indicate the current status. Four states are available:

### Press to confirm
The Led is almost on all the time. It goes off for 100 miliseconds every second.

![Press to confirm](https://user-images.githubusercontent.com/55573252/162008917-6a730eac-396c-44cc-890e-802294be30a3.gif)

### Idle mode
In idle mode, the Pico HSM goes to sleep. It waits for a command and it is awaken by the driver. The Led is almost off all the time. It goes on for 500 milliseconds every second.

![Idle mode](https://user-images.githubusercontent.com/55573252/162008980-d5a5caad-072e-400c-98e3-2c606b4b2af9.gif)

### Active mode
In active mode, the Pico HSM is awaken and ready to receive a command. It blinks four times in a second.

![Active](https://user-images.githubusercontent.com/55573252/162008997-1ea8cd7e-5384-4893-9dcb-b473153fc375.gif)

### Processing
While processing, the Pico HSM is busy and cannot receive additional commands until the current is processed. In this state, the Led blinks 20 times in a second.

![Processing](https://user-images.githubusercontent.com/55573252/162009007-df45111e-2473-4a92-97c5-15c3cd19babd.gif)

## Driver

The Pico HSM uses either the `sc-hsm` driver from [OpenSC](https://github.com/OpenSC/OpenSC/) or the `sc-hsm-embedded` driver from [CardContact](https://github.com/CardContact/sc-hsm-embedded/) to interface with external applications. These drivers employ the standardized PKCS#11 interface, making it compatible with various cryptographic engines that support PKCS#11, such as OpenSSL, P11 library, or pkcs11-tool.

Internally, the Pico HSM organizes and manages its data using the PKCS#15 structure, which includes elements like PINs, private keys, and certificates. Commands can be issued to interact with these stored elements using tools such as `pkcs15-tool`. For example, `pkcs15-tool -D` lists all elements stored within the Pico HSM.

Communication with the Pico HSM follows the same protocols and methods used with other smart cards, such as OpenPGP cards or similar devices.

For advanced usage scenarios, refer to the documentation and examples provided. Additionally, the Pico HSM supports the SCS3 tool for more sophisticated operations and includes features like multiple key domains. For detailed information on SCS3 usage, refer to [SCS3 documentation](/doc/scs3.md).

## License and Commercial Use

This project is available under two editions:

**Community Edition (FOSS)**
- Released under the GNU Affero General Public License v3 (AGPLv3).
- You are free to study, modify, and run the code, including for internal evaluation.
- If you distribute modified binaries/firmware, OR if you run a modified version of this project as a network-accessible service, you must provide the corresponding source code to the users of that binary or service, as required by AGPLv3.
- No warranty. No SLA. No guaranteed support.

**Enterprise / Commercial Edition**
- Proprietary license for organizations that want to:
  - run this in production with multiple users/devices,
  - integrate it into their own product/appliance,
  - enforce corporate policies (PIN policy, admin/user roles, revocation),
  - deploy it as an internal virtualized / cloud-style service,
  - and *not* be required to publish derivative source code.
- Base package includes:
  - commercial license (no AGPLv3 disclosure obligation for your modifications / integration)
  - onboarding call
  - access to officially signed builds
- Optional / on-demand enterprise components that can be added case-by-case:
  - ability to operate in multi-user / multi-device environments
  - device inventory, traceability and secure revocation/offboarding
  - custom attestation, per-organization device identity / anti-cloning
  - virtualization / internal "HSM or auth backend" service for multiple teams or tenants
  - post-quantum (PQC) key material handling and secure PQC credential storage
  - hierarchical deterministic key derivation (HD wallet–style key trees for per-user / per-tenant keys, firmware signing trees, etc.)
  - cryptographically signed audit trail / tamper-evident logging
  - dual-control / two-person approval for high-risk operations
  - secure key escrow / disaster recovery strategy
  - release-signing / supply-chain hardening toolchain
  - policy-locked hardened mode ("FIPS-style profile")
  - priority security-response SLA
  - white-label demo / pre-sales bundle

Typical licensing models:
- Internal use (single legal entity, including internal private cloud / virtualized deployments).
- OEM / Redistribution / Service (ship in your product OR offer it as a service to third parties).

These options are scoped and priced individually depending on which components you actually need.

For commercial licensing and enterprise features, email pol@henarejos.me
Subject: `ENTERPRISE LICENSE <your company name>`

See `ENTERPRISE.md` for details.

## Credits
Pico HSM uses the following libraries or portion of code:
- mbedTLS for cryptographic operations.
- TinyUSB for low level USB procedures.
