# LapEE Bundler Profile NG

This repo contains the working profile JSON for the native P4 AO-paid LapEE bundler profile.

It intentionally tracks only:

- `lapee-bundler-profile-ng.json` - the AO-payment-only native P4 profile.
- `lapee-bundler-profile-paid.json` - the AO-payment-only native P4 profile using the current published remote device packages.
- `lapee-bundler-profile-recharging.json` - the recharging-ledger waterfall profile.
- `README.md` - the manifest for the profile, source repos, and proof run.

The device source lives in separate repos. Keeping this repo profile-only avoids duplicating Erlang device code and makes the profile artifact easy to review, pin, and publish.

## Device Source

- `ao-payment@1.0`: https://github.com/xylophonez/aopayment-device
- `arweave-byte-pricing@1.0`: https://github.com/xylophonez/arweave-byte-pricing-device
- `bundler-settlement@1.0`: https://github.com/xylophonez/bundler-settlement-device
- `lapee-bundler-gc@1.0`: https://github.com/xylophonez/lapee-bundler-gc-device
- `lapee-p4-bootstrap@1.0`: https://github.com/xylophonez/lapee-p4-bootstrap-device
- `pricing-router@1.0`: https://github.com/xylophonez/pricing-router-device
- `simple-oracle@1.0`: https://github.com/xylophonez/simple-oracle-device
- `recharging-ledger@1.0`: https://github.com/permaweb/recharging-ledger

The combined development mirror is:

- https://github.com/xylophonez/lapee-bundler-devices-mirror/tree/feat/p4-compat

Published patched `recharging-ledger@1.0` package used by `lapee-bundler-profile-recharging.json`:

- Source: https://github.com/permaweb/recharging-ledger/pull/1
- Source commit: `63e0612bb4b8a9daeba00832588d5d4d3a4f0aa8`
- Spec ID: `OnCecZpD33DV7kuf-WNw1hTNHAqb6u4IEKQ3Tc3Q6Eg`
- Implementation ID: `MmzTVPVlgH0UPLAZW09JUH_dTWDwxgdqJgPZa8NTtlA`
- Signer: `aYDOU6kEcE3lK7aA-gTmUKHTbLlQnZZXWpv1_i_Uq1U`

Published updated `lapee-p4-bootstrap@1.0` package used by `lapee-bundler-profile-recharging.json`:

- Spec ID: `RSQlLsmPZwwk7mxZuwMziGBFTMSTJT9OHm1xlq6MzrQ`
- Implementation ID: `0ER8jwnxByvfp0IXfgI1U_nypaq9x_Y0-NGbB68Ii0Y`
- Signer: `aYDOU6kEcE3lK7aA-gTmUKHTbLlQnZZXWpv1_i_Uq1U`

Published updated `ao-payment@1.0` package used by `lapee-bundler-profile-recharging.json`:

- Spec ID: `7eAMYQ9PG8esiP-XIz1UoP78aI1htNgdtHqSbNm94X4`
- Implementation ID: `EJwu47Q0MK2DMjohTRX4kNYc7QLiXZmKq3s8QLTgTX4`
- Signer: `aYDOU6kEcE3lK7aA-gTmUKHTbLlQnZZXWpv1_i_Uq1U`
- Pre-push validation: `/home/fn/Dev/rani/aopayment-device/scripts/pre-push-test.sh`

## Profile Behavior

Both profiles boot through `lapee-p4-bootstrap@1.0` and configure the paid bundler path through native P4.

The baked-local AO-payment-only profile disables remote device loading and settles native P4 charges directly through `ao-payment@1.0`.

The remote AO-payment-only profile, `lapee-bundler-profile-paid.json`, uses the same published package pins as the recharging profile, but does not set `p4-ledger-device` to `recharging-ledger@1.0` and does not include any `recharging-ledger-*` settings. It is the paid-only profile for base LapEE when the current device set should be loaded from the network without sponsoring recharge balance.

- `ao-payment@1.0` imports AO token payments into the local P4 ledger.
- Native P4 charges debit that local ledger when `~bundler@1.0/item` is posted.
- `ao-payment@1.0` auto-withdraws the charged AO quantity to `bundler-beneficiary`.
- `bundler-settlement@1.0` is no longer required for the immediate paid-bundle path.

The recharging profile is explicit for base LapEE: all custom devices are resolved through the first `name-resolvers` entry and pinned in `trusted-devices`. It sets:

- `p4-ledger-device`: `recharging-ledger@1.0`
- `recharging-ledger-max`: `3000000000`
- `recharging-ledger-recharge`: `1000`
- `recharging-ledger-period`: `1`
- `recharging-ledger-fallback.device`: `ao-payment@1.0`
- `p4-commit-charge`: left unset, so P4's default signed charge remains enabled.

Runtime behavior:

- A sender spends recharge units first.
- If a charge exceeds the sender's remaining recharge balance, `recharging-ledger@1.0` falls through to `ao-payment@1.0`.
- The fallback is all-or-nothing: a charge of `N` with less than `N` recharge remaining charges the full `N` through AO payment.

## Proven Build

The JSON in this repo was copied from the local profile used in the QEMU proof run:

- Source path: `/home/fn/Dev/FWD/os/build/local-profiles/aopayment-bundler-p4-native-baked-local-20260629.json`
- Profiled image: `/home/fn/Dev/FWD/os/build/images/lapee-no-tme-aopayment-p4-autowithdraw-profile-20260629-signed.img`
- Preloaded device index: `AHEfwbPh0hq-20u5JNdicAEfjg60UwpbDCI7zsqRh24`
- Proof artifacts: `/home/fn/Dev/FWD/os/build/qemu-p4-flow-20260629-autowithdraw-profile`

QEMU proof summary:

- Payment message: `euwfJwDYGw5V0WzfG1NMF82zG7fOYJQWFUJwk1fHXTQ`
- Payment slot: `2495513`
- Bundle item: `ZNq0StxWO26ifWzc26Gfed_pou4_mtj8Ik_MunQBozM`
- Quote/payment quantity: `2509673921`
- Bundle status: `200`
- Beneficiary AO delta: `2509673921`
- Sender local ledger after charge: `0`
- Deposit AO after auto-withdraw: `0`

## Applying The Profile

From the LapEE OS repo, apply this profile to a signed image with:

```sh
scripts/apply-profile-to-image.sh \
  /path/to/lapee-bundler-profile-ng.json \
  build/images/lapee-no-tme-aopayment-p4-autowithdraw-20260629-signed.img \
  build/images/lapee-no-tme-aopayment-p4-autowithdraw-profile-20260629-signed.img
```

For `lapee-bundler-profile-ng.json`, the device source should be baked into the image before profile application. That AO-payment-only profile does not fetch remote device code at runtime.

For `lapee-bundler-profile-recharging.json`, start from base LapEE and let the profile load the pinned published device packages. The profile explicitly resolves:

- `ao-payment@1.0`
- `arweave-byte-pricing@1.0`
- `bundler-settlement@1.0`
- `lapee-bundler-gc@1.0`
- `lapee-p4-bootstrap@1.0`
- `pricing-router@1.0`
- `simple-oracle@1.0`
- `recharging-ledger@1.0`

## Recharging Waterfall Proof

The current recharging profile was validated against a base no-TME LapEE image on 2026-06-30. This proof starts from the base image, injects only `lapee-bundler-profile-recharging.json`, remote-loads the pinned published packages, and posts real bundle items through `~bundler@1.0/tx`.

- Profile: `lapee-bundler-profile-recharging.json`
- Source image: `/home/fn/Dev/FWD/os/build/images/lapee-no-tme-source-20260629.img`
- Profiled image: `/home/fn/Dev/FWD/os/build/images/lapee-no-tme-source-recharging-pr1-profile-20260630.img`
- Proof artifacts: `/home/fn/Dev/FWD/os/build/qemu-p4-recharging-pr1-20260630`

Runtime profile evidence from `/~meta@1.0/info`:

- `lapee-profile`: `aopayment-bundler-p4-recharging-waterfall`
- `p4-ledger-device`: `recharging-ledger@1.0`
- `load-remote-devices`: `true`
- `ao-payment-ledger`: `FbFrDZc5ZTN3UCY3r_cD1w9JlCYzmQeJCGNKwCjhwOQ`
- `ao-payment-deposit-address`: `j8npaq8hRCZP21cBtMk-_JrchUbkWeB_ksaaYQxGJiw`
- `~p4@1.0/balance?target=<sender>` after the first item returned `465701843`.

QEMU waterfall proof summary:

- Free/recharge item: `JEBPzl8vRouENi5xLfliVgcLxE2DYuLzgJqXy3O924k`
- Paid/fallback item: `dkACLtoynbbbU13Yen_Ni4pTHpYfuoRIgvMV851MTYE`
- Free quote: `2534398694`
- Recharge before: `3000000000`
- Recharge after free item: `465611254`
- Paid quote: `2534398694`
- Payment message: `hcn2wxUkFU7zM7hsOip6POqMaeWtcL5XhQ6J57BZtks`
- Payment slot: `2496041`
- Payment quantity imported: `6068797388`
- Paid bundle status: `200`
- Sender local ledger after ingest: `6068797388`
- Sender local ledger after paid charge: `3534398694`
- Ledger spent by fallback charge: `2534398694`
- Beneficiary AO delta: `2534398694`

Earlier custom-overlay proof artifacts are retained at `/home/fn/Dev/FWD/os/build/qemu-p4-recharging-20260630`; they validated runtime behavior before the profile was corrected to explicitly pin every custom device for base LapEE.
