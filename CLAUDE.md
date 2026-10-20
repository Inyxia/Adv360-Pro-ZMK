# CLAUDE.md

Kinesis Advantage360 Pro firmware (ZMK), Inyxia fork. Working branch: `personal`.
Session learnings and debugging history live in `LEARNINGS.md`. Keep this file to stable process facts.

## Building

- `make left` builds the left half, `make all` builds both halves. Both use Docker (or podman if present) and write to `firmware/` as `<UTC timestamp>-<commit>-<left|right>.uf2`.
- The Makefile runs `docker run -it`, which fails when there is no TTY (for example, when run from a script). Run the same steps without `-it`.
- `bin/get_version.sh` rewrites `config/version.dtsi`, and the Makefile restores it with `git checkout` afterwards. Don't commit that file's changes by accident.
- The image is built from `Dockerfile`, which `COPY`s `bin/build.sh` and runs it as the entrypoint. `bin/*.sh` must be executable in git (mode 755). On `personal` they were stored without the execute bit, which made the container fail with `permission denied`. Fix the mode in git, not just in a copy.
- The first image build is slow (roughly 8 to 10 minutes), mostly `west update`. Container git fetches from GitHub can stall with the default network. `docker build --network=host` fixed that. Use it for both `docker build` and `docker run`.
- `config/west.yml` pins the ZMK base. `personal` uses `refil/zmk` `adv360-z3.5`. `V3.0` uses `adv360-z3.2-3`. Don't mix firmware from the two bases on one keyboard.

## Flashing

- Enter bootloader mode with a double-tap of the reset button, or the `&bootloader` key on the `sys` layer. A `ADV360PRO` drive mounts, and `lsusb` loses that half's `Adv360 Pro` / `Adv360 Pro rt` entry.
- Copy one `.uf2` to the drive. The half reboots and the drive unmounts.
- Each half needs its own image: `-left` for "Adv360 Pro", `-right` for "Adv360 Pro rt".
- Do not flash `settings-reset.uf2` as a settings reset. Its base address is `0x026000`, which is the start of `code_partition`, so it replaces the running firmware with a settings-erase program. It was used once, and the keyboard only worked again after reflashing the normal firmware.
- Backups of the working images, with checksums, are in `/home/kenton-seward/firmware-backups/`. Keep a copy of any build you flash there, named by commit.

## Bluetooth

- The left half is the BLE central (`ZMK_SPLIT_ROLE_CENTRAL`) and the only host-facing device. Pair the "Adv360 Pro" device. The right half connects to it over the split link.
- BLE privacy is off. Passkey entry is off (`CONFIG_ZMK_BLE_PASSKEY_ENTRY=n`), so pairing is Just Works and nothing needs typing on the keyboard.
- Profiles are selected with `&bt BT_SEL 0`–`4`. `&bt BT_CLR` on the `personal` sys layer, next to `BT_SEL 4`, clears the bond for the current profile. On `V3.0` it's on the mod layer.
- Pairing on Linux: `bluetoothctl`, then `power on`, `agent on`, `default-agent`, `scan on`, `pair <addr>`, `trust <addr>`, `connect <addr>`. `trust` may need a second run before it sticks.
- `AuthenticationFailed` after a connect means stored keys disagree. Clear the keyboard's bonds (or the host's, with `bluetoothctl remove <addr>`) and pair again.
- Watch the host side with `journalctl -u bluetooth -f`. Filter out the `Endpoint registered` noise.

## Safety

- Don't commit or push changes unless asked.
- `macro_ver` (sys layer, `config/version.dtsi`) types a version stamp: the UTC build date, the first four characters of the branch name, and the short commit hash. `bin/get_version.sh` regenerates it on every build. It isn't a secret, so it can be committed and shared.
