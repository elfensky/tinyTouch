# Stand-alone PIN mode: a tinyTouch reader for Windows

Status: approach approved by Andrei on 2026-11-16. Builder check done; Andrei's answers are in. Awaiting approval.

## Intent

A tinyTouch reader on the HYTE Windows PC must answer every Windows Hello PIN prompt with one touch. Upstream supports
macOS hosts only. Its model keeps the password in the host's Keychain and needs a helper process on the host
(`macos/tinytouch_helper.py`). Windows has no helper.

Andrei's direction (2026-11-16): new behaviour goes into the firmware, and the host side stays as small as possible.
The HYTE signs in with a Microsoft account and a Windows Hello PIN. That PIN never leaves the device, and Windows Hello
has built-in brute-force protection[^ms]. So a board may store that PIN: a leaked PIN is useful only with the HYTE in
hand.

## Goals

- Reader 2 types the Windows Hello PIN after a finger match, with no software on Windows.
- It works in every Windows Hello PIN prompt: sign-in after a restart, the lock screen, admin prompts, passkey
  (WebAuthn) prompts and Windows Security credential prompts.
- A touch on a sleeping HYTE wakes it and then types the PIN (Andrei, 2026-11-16).
- The digits are correct on every keyboard layout, the Belgian AZERTY layout included.
- No USB serial command returns the stored PIN.

## Non-goals

- A Windows helper app, or any software on Windows.
- Windows Hello biometric integration (a WBF sensor driver).
- Storing a Mac password or any account password on a board.
- Per-finger texts or per-finger Enter (a future improvement in the vault project note).
- Letters or symbols in the PIN: version 1 accepts digits only.
- Flash encryption and secure boot: a separate, later step (Andrei, 2026-11-16).
- Stand-alone use on macOS or Linux, and an upstream merge (both possible later).

## Anti-goals

- The board types the PIN without a finger match.
- Any command or log line that returns or shows the stored PIN.
- Any change to helper-mode behaviour while the flag is off. Reader 1 keeps the upstream firmware.
- A new mode value. Older firmware rejects an unknown mode and resets the whole config
  (`device_config.c:73`, `:110-138`).

## Constraints

Code facts at upstream `274250f` (v0.1.35). Paths under `firmware/tiny_touch_unified/main/` unless shown.

- **Build:** ESP-IDF 5.3.x is enforced (`firmware/tiny_touch_unified/CMakeLists.txt:36-40`). Build with
  `idf.py -C firmware/tiny_touch_unified build`. The factory image comes from `release/assemble-release.py`.
- **Signing:** signed apps without secure boot (`sdkconfig.defaults`). The running app accepts an OTA image only when
  the image carries the same key. So fork firmware goes on by USB (BOOT button held while it connects) once per board.
  After that, `tinytouch update --file` installs fork images. A USB flash of the upstream factory image restores the
  upstream line. Two OTA slots and app rollback protect against a bad update (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`).
- **Config storage:** NVS namespace `tt6`, blob `config`, version 6 (`device_config.c:11-26`). Optional settings live in
  the options blob `custom` (`device_config.c:37`).
- **Touch path:** `touch_hid_task` (`touch_pin_hid.c:526-639`) calls `handle_fingerprint_match` (`:492`). In HID mode
  that calls `request_and_type_password` (`:394-474`), which asks the helper and types through
  `type_ascii_in_session` (`:159-180`). That function maps bytes through the US table `ascii_to_keycode` and presses
  Enter when `submit_enter` is on (`:170`).
- **Mode:** `SET MODE HID` needs no registered host (`config_console.c` `set_mode`). Removing the last host forces PIV
  (`device_config.c:193`).
- **Authorization:** config writes need `AUTH` (`config_console.c:108-112`), a 120 s window. With no enrolled finger,
  `AUTH` passes without a touch (`:131`).
- **Capability detection:** the CLI accepts protocol 6 only (`macos/cli.py:49`). New features are announced as `STATUS`
  flags, as `custom_config=1` is (`config_console.c:157`). The `STATUS` buffer holds 768 bytes (`:148`).
- **Keyboard LEDs:** the HID descriptor is `TUD_HID_REPORT_DESC_KEYBOARD()` (`usb_descriptors.c:30`), which includes
  the LED output report. `tud_hid_set_report_cb` ignores it today (`touch_pin_hid.c:729`).
- **Tests:** a host-side C harness compiles `tests/host/*.c` from Python unittest (`tests/test_latency_firmware.py:17-33`,
  `tests/test_led_firmware.py:30-55`). `touch_pin_hid.c` and `config_console.c` are cut into sections by exact text
  anchors (`tests/test_latency_firmware.py:36-56`). Those anchors must stay intact.

## Approaches considered

| Approach | Outcome |
|---|---|
| **Stand-alone text mode in fork firmware** | **Chosen.** No host software, works at sign-in after a restart, fits the firmware-first direction. |
| Windows helper (a port of the Mac helper, as open upstream PR #104 does for Linux) | Rejected by Andrei: host software, and no typing before the first sign-in after a restart. |
| One reader for the Mac and Windows | Rejected: needs host detection or a per-finger choice. Reader 2 is dedicated instead. |
| PIV smart card | Dropped on 2026-11-16: on macOS the PIN dialog needs a click before a touch can type (spike in the vault project note). |

## Design

### Settings and commands

- **Flag:** `standalone`, 0 or 1, default 0, in its own NVS key in namespace `tt6`, as `piv_touch` and `piv_delay_ms`
  are (`device_config.c:124-134`). Not in the options blob: both firmware lines load that blob only when its size and
  version match exactly (`device_config.c:116-118`), so a new field would reset the LED settings at each switch
  between the upstream and fork lines. Command: `SET STANDALONE <0|1>`, behind `AUTH`, in `set_value`.
- **PIN:** 4 to 32 ASCII digits, in its own NVS key in namespace `tt6`. Commands, both behind `AUTH`:
  - `STANDALONE PIN <digits>`: validates the digits and the length, refuses when no finger is enrolled (because `AUTH`
    then passes without a touch), stores the PIN, replies `OK STANDALONE PIN`.
  - `STANDALONE CLEAR`: deletes the PIN.
- **No read path:** no command, `STATUS` field or log line contains the PIN.
- **STATUS:** adds `standalone=<0|1> standalone_pin=<0|1>`. The second field shows only whether a PIN is stored.
- **Factory reset** (`RESET FACTORY`) deletes the PIN.
- **Zero fingers:** when the last enrolled finger is deleted (`FINGER CLEAR`, deleting the last group, or a cleared
  sensor), the board deletes the PIN (Andrei, 2026-11-16). With no finger enrolled, `AUTH` passes without a touch
  (`config_console.c:131`), so anyone could enroll a finger and have the PIN typed. After re-enrolling, run
  `standalone pin` again.

### Touch path

- In `handle_fingerprint_match`, in HID mode: when `standalone` is 1 and a PIN is stored, type the PIN and return.
  Send no event to a helper. When `standalone` is 1 and no PIN is stored, type nothing and log `standalone_no_pin`.
  In PIV mode the flag has no effect.
- **Digits on the number pad:** type each digit as `HID_KEY_KEYPAD_0` … `HID_KEY_KEYPAD_9`. Number-pad keys give the
  same digit on every layout. The top-row keys do not: on AZERTY they type `&é"'(` without Shift.
- **NumLock:** store the last LED byte from `tud_hid_set_report_cb` (bit 0 = NumLock). If NumLock is off, press
  NumLock before the digits and again after them, so the user's state comes back.
- **Enter:** after the digits, press Enter only when `submit_enter` is 1. Reader 2 is expected to run with
  `submit_enter` 0, because Windows sign-in accepts a correct PIN without Enter (to confirm in step 0 of the checklist).
- **Same safety as today:** check the whole PIN before the first key, never press Enter after a partial PIN, and
  release keys on failure, as `type_ascii_in_session` does (`touch_pin_hid.c:159-180`).
- **Log events:** `standalone_typed`, `standalone_no_pin`. The value field never holds a digit of the PIN.
- **LEDs:** reuse the existing authentication colours.

### Wake from sleep

- The configuration descriptor already declares remote wake-up (`usb_descriptors.c:52`, `:92`). The firmware never
  calls `tud_remote_wakeup()` today.
- On a finger match while the USB link is suspended, and the host has enabled remote wake-up: call
  `tud_remote_wakeup()`, wait for the resume, then wait a settle time for the lock screen, then type. A new setting
  holds the settle time (`SET STANDALONE_WAKE_MS <n>`, default to be set from checklist step 0).
- While the link is suspended, `wait_hid_ready` gives up after 2 s (`touch_pin_hid.c:114-123`), so the wake path must
  run before typing starts, not inside it.
- On every resume the firmware forces a USB reconnect, a macOS workaround (`usb_ccid.c:105-112`). With `standalone` on,
  skip it: a reconnect during the wait would end the HID session and lose the PIN.
- Windows side, no software: Device Manager allows the keyboard to wake the computer, and the BIOS allows USB wake
  from sleep. Both go in the checklist.

### CLI (macOS, used to prepare reader 2)

The new commands run from the fork checkout, never as an installed CLI: `uv run macos/cli.py standalone …`. The Mac
keeps the signed upstream CLI and helper for reader 1. Its Keychain items are bound to the upstream signing identity
(`release/macos-signing.md`), so a fork build with another signature in its place would break reader 1. Firmware
updates of reader 2 use the existing `tinytouch update --file <image>` (`macos/cli.py:2296-2312`).

- `tinytouch standalone pin`: asks for the PIN twice with hidden input, checks 4–32 digits, unlocks with a touch,
  sends `STANDALONE PIN`, then confirms `standalone_pin=1` in `STATUS`. The PIN never appears in output or logs.
- `tinytouch standalone on` and `tinytouch standalone off`: send `SET MODE HID` when needed, then `SET STANDALONE`.
- `tinytouch standalone clear`: sends `STANDALONE CLEAR`.
- Each command checks for the `standalone=` field in `STATUS` first. Firmware without it gets a clear error.

### Preparing reader 2

1. Build the fork firmware with the fork signing key. Flash the factory image by USB, with BOOT held while it connects.
2. Enroll fingers: `tinytouch enroll <n>`.
3. From the fork checkout: `standalone pin`, `standalone on`. Then `tinytouch piv-touch on`, so Windows never sees a
   smart card: in HID mode the card is visible while `piv_touch` is off (`usb_ccid.c:78`), and it answers every command
   with `69 85` (`usb_ccid.c:187-190`). Set `submit_enter` per step 0 of the checklist.
4. Move reader 2 to the HYTE.

### Signing key and releases

- Generate the fork signing key once, with the same scheme as `sdkconfig.defaults` (RSA). Keep it in 1Password and
  never in Git. The file path is `firmware/tiny_touch_unified/secure_boot_signing_key.pem`, which upstream already
  ignores in Git.
- Give the fork firmware a version that shows the fork, so `STATUS` tells the two lines apart.
- Later updates of reader 2 run `tinytouch update --file <fork image>` from the Mac.

### Tests

- Host C tests (`tests/host/`): PIN validation, digit-to-keypad mapping, the NumLock decision, the stand-alone branch
  of `handle_fingerprint_match` (with stubs, as `touch_polling_test.c` does), command parsing with `AUTH`, and the
  absence of any read path.
- Python tests for the new CLI commands, in the style of `tests/test_interactive_cli.py`.
- All existing tests stay green (464 at `274250f`), and the text anchors stay intact.

## Windows acceptance checklist

0. **Before any code, with the keyboard:** on the lock-screen curtain (before the PIN field shows), press one
   number-pad digit: note whether it starts the PIN field or only lifts the curtain. Then in each prompt, type the PIN
   on the number pad without clicking. Note
   whether it lands in the field, whether it submits without Enter, and whether an extra Enter after a submitted PIN
   does any harm. Prompts: sign-in after a restart, lock screen (Win+L), an admin prompt, a passkey prompt in a
   browser, a Windows Security credential prompt.
   Rule (Andrei, 2026-11-16): sign-in and the lock screen decide `submit_enter`. Other prompts may need a manual
   Enter or click, and the project goes on.
1. After flashing reader 2: the same prompts with one touch each.
1a. Sleep: put the HYTE to sleep, touch once, and check that it wakes and signs in. Check Device Manager ("Allow this
    device to wake the computer" on the tinyTouch keyboard) and the BIOS USB wake setting first.
2. NumLock off at sign-in: the PIN still lands, and NumLock is off again afterwards.
3. An unknown finger types nothing.
4. A touch while a text editor has focus types the PIN there. This is expected and accepted: the PIN is useless away
   from the HYTE.

## Security

- **Compromised PC:** each touch types the PIN through the PC, where a keylogger can capture it. The PIN works only on
  the HYTE, behind Windows Hello brute-force protection.
- **Stolen reader:** the PIN can be read from the unencrypted flash, or typed by faking a match on the sensor wire. It
  is useful only with the HYTE in hand. Flash encryption, secure boot and an epoxy fill are the later hardening step.
- **Setup:** the PIN travels once, in plain text, over USB from the Mac to the board.

## Left to the builder

- Exact NVS key names, command spelling, CLI wording and log event names, within the rules above.
- The settle-time default after a wake, from checklist step 0 and 1a.
- What to do when no LED report has arrived since boot. Choose and document one rule: either type number-pad digits
  without touching NumLock, or ask the host for its LED state first.
- The fork version string, and how the fork's release assets are built.

[^ms]: [Windows Hello for Business overview](https://learn.microsoft.com/en-us/windows/security/identity-protection/hello-for-business/hello-why-pin-is-better-than-password) · [🔖 2026-11-16](https://linkding.lav.ren/bookmarks?details=6688)
