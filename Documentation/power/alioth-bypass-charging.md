# Experimental alioth bypass charging

Target: Kawakaze0372/android_kernel_xiaomi_sm8250, seventeen source at
63cf7b7b8ff935d06baa527280f07140fe4e3d74, with the existing pinned SukiSU/SUSFS
build integration. This change has not been validated on hardware.

## Interface

Root-only writes to /sys/class/power_supply/battery/device/bypass_charging:

- 0: release this feature's votes; normal ROM charging policy resumes.
- 1: request battery charging inhibition while retaining the USB input path.

Only alioth builds expose this attribute. It defaults to 0, is not persistent,
and resets on normal USB detach and driver shutdown. Values other than 0/1
are rejected. Enabling without an online USB source returns ENODEV. Reading
1 reports the feature's CHG_DISABLE vote, not proof of zero battery current.

The existing input_suspend and battery_charging_enabled semantics are unchanged.
PixelOS charge limits can still suspend the input independently. Disable ACC
and temporarily disable PixelOS's charge limit while testing this feature.

## Implementation

A dedicated ALIOTH_BYPASS_VOTER inhibits SMB5 battery charging, parallel charging,
and FCC. It never removes other clients' thermal, JEITA, input-suspend, or
voltage votes. BQ2597x pumps are explicitly stopped and reject re-enable attempts
while that dedicated vote is active. The PD policy loop stays out of its pump
state machine and restores its own buck-input path while bypass is requested.

The charging controller still connects the battery to the system power path.
This is battery charging inhibition, not physical battery isolation. Input
power deficits can still cause battery discharge. No hardware claim is made
until current measurements confirm it.

## Build

.github/workflows/alioth-bypass-build.yml builds the pushed commit from this
fork, retains the existing pinned SukiSU/SUSFS patches, and uploads Image,
kernel.config, and an AnyKernel3 ZIP marked BYPASS-UNTESTED. It is restricted
to this fork and the codex/alioth-bypass-charging branch. Existing workflows
and the seventeen branch are not changed.

## Device validation after a successful build

Keep a known-good boot image and recovery path before testing the new kernel.
First verify normal charging and that the new attribute reads 0. Keep wireless
ADB connected so the USB port can remain on the charger. Do not enable the
feature while using OTG or an input-suspend policy.

1. Record USB online, battery current_now, voltage_now, charge_counter, capacity,
   and temp for at least one minute with normal charging. Establish current
   polarity empirically; this device currently reports charging as negative.
2. Write 1 to the new attribute. Confirm USB remains online, BQ2597x charging
   is disabled, and battery current settles near zero. Observe the charge
   counter over several minutes; do not rely solely on Charging status or
   integer capacity.
3. Repeat with a modest real workload. If battery current becomes persistently
   positive, the battery is supporting the load and the full bypass target is
   not met under those conditions.
4. Write 0 and confirm normal charging resumes. Repeat enable/disable twice.
5. Unplug, verify the attribute returns 0, then replug and confirm normal
   charging. Verify the feature is off after reboot.

Enabling or disabling the feature must not disable temperature protections.
A successful CI build proves compilation only, not successful bypass operation.
