# LEARNINGS.md

Dated notes from debugging and firmware work. Add an entry each time something non-obvious is found. Process facts belong in `CLAUDE.md`.

## 2026-10-19 — Bluetooth pairing never completed

- Symptom: the keyboard showed up as "Adv360 Pro" in Bluetooth scans, but never paired or connected.
- Finding: the device advertised as HID with no bond, so the firmware was alive. The failure was in pairing.
- Finding: the first `bluetoothctl pair` produced `AuthenticationFailed` after a short connect. That means stored keys disagree, which is usually a stale bond.
- Fix: the bond data was cleared by flashing the reset image onto the left half. Flashing the `personal` firmware back restored the keyboard. After that, pairing succeeded on the first try.
- Lesson: check whether an "AuthenticationFailed" is a stale-bond problem before touching firmware.

## 2026-10-19 — `settings-reset.uf2` is firmware, not a settings wipe

- I first described it as a settings-only erase. That was wrong. It is a UF2 image based at `0x026000`, which is the start of `code_partition` in `config/boards/arm/adv360/adv360.dtsi`.
- Result: copying it replaced the running firmware, and the half dropped off USB and Bluetooth until its normal firmware was reflashed.
- Lesson: check a UF2's base address against the partition map before flashing.

## 2026-10-19 — Bluetooth and profile keys differ between branches

- `V3.0` binds `&bt BT_CLR` on the mod layer. `personal` didn't bind it, and binds `&bootloader` on the `sys` layer instead. As of this entry, `personal` binds `&bt BT_CLR` on the sys layer next to `BT_SEL 4`.
- `personal` builds on `adv360-z3.5`. `V3.0` builds on `adv360-z3.2-3`. The reset file in the repo is from `V3.0`.
- Lesson: when a fix depends on a keymap binding or a reset file, check the branch it came from.

## 2026-10-20 — Building `personal` from a clean export

- `git archive origin/personal` gave a tree whose `bin/*.sh` had lost the execute bit, so the image's entrypoint failed with `permission denied`. The fix needed an image rebuild, because `COPY` captures the mode.
- First `docker build` stalled for more than ten minutes at a `git` fetch of `lvgl`, then `hal_nordic`. The build finished with `--network=host`.
- `docker run -it` in the Makefile fails without a TTY. Running without `-it` works.
- Lesson: run the build without a TTY, use `--network=host`, and check the execute bits in git.

## 2026-10-20 — Final state

- Both halves run `personal` at `553197c` (`adv360-z3.5`). Left is the central, paired and trusted. Bluetooth works with the USB cable unplugged. Right connects over the split link.
- Backups of both flashed images, with checksums, are in `/home/kenton-seward/firmware-backups/adv360-pro-personal-553197c/`.

## 2026-10-19 — `macro_ver` is a build stamp, not a secret

- `bin/get_version.sh` generates `macro_ver` on every build. It types the UTC build date, the first four characters of the branch name, and the short commit hash, followed by Return.
- Lesson: check the generator script before treating a generated value as sensitive.

## 2026-10-19 — `BT_SEL` row on `personal` had no profile number

- The fifth `&bt BT_SEL` on the `personal` sys layer had no argument. `V3.0` has `&bt BT_SEL 4` in the same slot.
- Fix: changed it to `&bt BT_SEL 4`, and added `&bt BT_CLR` after it.
- Lesson: when a keymap binding looks incomplete, compare the same slot on the other branch.

## Open items

- `personal` stores `bin/build.sh` and `bin/get_version.sh` without the execute bit. Fix it in git.

## 2026-10-19 — Layer switch from qwerty and colemak gets stuck

- Symptom: from qwerty (or colemak), `fn+1` / `fn+3` do nothing. Pressing the top-right key (`&to DEF`, which goes to base) unsticks it. Then `fn+3` works from base, and `fn+1` stops working after that.
- Finding: holding FN in qwerty types `1`, so the fn layer isn't active there. The same `&mo FN` works from base.
- Finding: releasing FN fully before pressing the layout key still fails, so this isn't a held-key overlap.
- Ruled out: `&tog` for the layout keys. It would toggle the current layout off on a second press.
- Source: `zmk_keymap_layer_to` clears every layer, then activates the target. The refil fork's `adv360-z3.5` uses the same logic.
- Status: not fixed. Workaround: press the top-right key to return to base before switching layouts.

## 2026-10-20 — Bumped ZMK base to adv360-z3.5-2 on `personal-z35-2`

- Change: `config/west.yml` pins `refil/zmk` at `adv360-z3.5-2` (was `adv360-z3.5`). The keymap is unchanged.
- Build: both halves compile (`e8e3cab`). Images are backed up in `firmware-backups/adv360-pro-personal-z35-2-e8e3cab/`.
- Flash: the left half first, then the right. The right's old firmware couldn't talk to the new left, so its red light flashed until the right was reflashed. Once both were on the new build, typing worked on both halves, including over Bluetooth.
- Right bootloader: the `&bootloader` key still does nothing. The reset button works.
- `BT_CLR` is still unverified. The profile 4 test showed no visible change because profile 4 had no bond. The next test had to stop because the host shows only one `Adv360 Pro` entry, so I couldn't tell which profile the connection was on.
- Lesson: before testing a clear, check that the host lists a separate entry for the profile you're testing.
