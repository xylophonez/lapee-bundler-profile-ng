# LapEE Bundler Profile NG

This repo contains the working profile JSON for the native P4 AO-paid LapEE bundler profile.

It intentionally tracks only:

- `lapee-bundler-profile-ng.json` - the AO-payment-only native P4 profile.
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

Published `recharging-ledger@1.0` package used by `lapee-bundler-profile-recharging.json`:

- Spec ID: `mFnIcWlmBnyq8NMPQB3tIaqeoQJLNRcyoozEzGzE-a0`
- Implementation ID: `arPd0tHsKdO1ODuypwxkgAtp7i5LQdw8yyQ6v9HBKVw`
- Signer: `vZY2XY1RD9HIfWi8ift-1_DnHLDadZMWrufSh-_rKF0`

## Profile Behavior

Both profiles boot through `lapee-p4-bootstrap@1.0` and configure the paid bundler path through native P4.

The AO-payment-only profile disables remote device loading and settles native P4 charges directly through `ao-payment@1.0`.

- `ao-payment@1.0` imports AO token payments into the local P4 ledger.
- Native P4 charges debit that local ledger when `~bundler@1.0/item` is posted.
- `ao-payment@1.0` auto-withdraws the charged AO quantity to `bundler-beneficiary`.
- `bundler-settlement@1.0` is no longer required for the immediate paid-bundle path.

The recharging profile enables remote loading only for the published `recharging-ledger@1.0` package and sets:

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

For `lapee-bundler-profile-recharging.json`, bake the local bundler/AO-payment/P4 bootstrap devices into the image, then let the profile load only the pinned published `recharging-ledger@1.0` package at runtime.

## Recharging Waterfall Proof

The recharging profile was validated against a QEMU booted no-TME LapEE image on 2026-06-30.

- Profile: `lapee-bundler-profile-recharging.json`
- Source image: `/home/fn/Dev/FWD/os/build/images/lapee-no-tme-p4-recharging-20260630-signed.img`
- Profiled image: `/home/fn/Dev/FWD/os/build/images/lapee-no-tme-p4-recharging-profile-20260630-signed.img`
- Combined device overlay: `/home/fn/Dev/FWD/os/build/hyperbeam-devices-recharging-20260630`
- Preloaded device index: `MOZP68snhAaZzm21hp5VlIQqz_MRZBCBojqmnLYhwnQ`
- Proof artifacts: `/home/fn/Dev/FWD/os/build/qemu-p4-recharging-20260630`

Runtime profile evidence from boot attestation:

- `lapee-profile`: `aopayment-bundler-p4-recharging-waterfall`
- `p4-ledger-device`: `recharging-ledger@1.0`
- `load-remote-devices`: `true`
- `trusted-device-signers`: `vZY2XY1RD9HIfWi8ift-1_DnHLDadZMWrufSh-_rKF0`
- `ao-payment-ledger`: `22gMV7qKPiB9hljfYoTIuVXTf7zr1ZnuAegPU6LhqgA`
- `ao-payment-deposit-address`: `TQjQDXAs5ZcdPV_qKUpxXACqFl_GIRnPZll_1wNt6O0`

QEMU waterfall proof summary:

- Free/recharge item: `QJTO4FiwLSUTrFTv8jAlxLVUana4HACiVZ9mqrVGFbA`
- Paid/fallback item: `TarlhXowf8jz2DmEIDd0M2WK8bsdQZyE6NxcjJxypxo`
- Free quote: `2532316469`
- Recharge before: `3000000000`
- Recharge after free item: `467693510`
- Paid quote: `2532316469`
- Payment message: `tZADcpuuzg7v-UKBH73sVLhhWh6d_dlhGIZLIlDe6Xc`
- Payment slot: `2495981`
- Payment quantity imported: `6064632938`
- Paid bundle status: `200`
- Sender local ledger after ingest: `6064632938`
- Sender local ledger after paid charge: `3532316469`
- Ledger spent by fallback charge: `2532316469`
- Beneficiary AO delta: `2532316469`
