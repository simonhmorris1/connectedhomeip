# BLE-only Commissioning Proxy: build and certification tests

How to build a BLE-only Commissioning Proxy (CP) from this repository, run it on
a Raspberry Pi, and run the certification tests `TC_COMPRO_2_1` to
`TC_COMPRO_2_9` against it with th-cli. All nine passed with these commands on
2026-10-26.

```
+--------------+           +------------------+        +-----------------------+
| Test Harness | Ethernet  | Commissioning    |  BLE   | End Device (ED)       |
| (TH)         |---------->| Proxy (CP)       |------->| Raspberry Pi          |
| th-cli       |           | Raspberry Pi     |        | chip-all-clusters-app |
|              |           | all-devices-app, |        |                       |
|              |           | endpoint 5       |        |                       |
+--------------+           +------------------+        +-----------------------+
       |                                                           |
       +-------- Wi-Fi access point, used in TC_COMPRO_2_4 --------+
       |                                                           |
       +--------- serial port, optional, to control the ED --------+
```

The Test Harness (TH) commissions the End Device (ED) through the CP, which
passes the commissioning messages to the ED over BLE.

## What you need

-   **Two Raspberry Pi 4s**, one for the CP and one for the ED.
-   **A TH** installed as in the
    [Matter TH User Guide](https://github.com/project-chip/certification-tool/blob/main/docs/Matter_TH_User_Guide/Matter_TH_User_Guide.adoc),
    from branch `v2.16-beta2+summer2026` or later. Earlier branches do not have
    the Commissioning Proxy tests.
-   **A Linux build host** with Docker and about 10 GB of free disk.
-   **A Wi-Fi access point** on the same Ethernet network as the TH, without
    client isolation: `TC_COMPRO_2_4` ends with the TH talking to the ED over
    it.

Replace `<cp-ip>`, `<ed-ip>`, `<th-ip>`, `<ssid>`, `<password>` and
`<project-id>` with your own values. The SSID and password may not contain
spaces.

## 1. Prepare both Pis

Install Ubuntu Server 24.04.5 (64-bit) with Raspberry Pi Imager and log in as
`ubuntu`. BLE is unreliable on newer kernels
([certification-tool #1123](https://github.com/project-chip/certification-tool/issues/1123)),
so install and boot kernel 6.8.0-1048:

```bash
sudo apt-get install -y linux-image-6.8.0-1048-raspi linux-modules-6.8.0-1048-raspi && sudo flash-kernel --force 6.8.0-1048-raspi && sudo cp /boot/vmlinuz-6.8.0-1048-raspi /boot/firmware/vmlinuz1048 && sudo cp /boot/initrd.img-6.8.0-1048-raspi /boot/firmware/initrd1048 && sudo cp --update=none /boot/firmware/config.txt /boot/firmware/config.txt.orig && sudo sed -i -e 's/^kernel=vmlinuz$/kernel=vmlinuz1048/' -e 's/^initramfs initrd\.img followkernel$/initramfs initrd1048 followkernel/' /boot/firmware/config.txt && sudo reboot
```

After the reboot:

```bash
uname -r                 # must print 6.8.0-1048-raspi
sudo apt-get install -y bluez && sudo systemctl enable --now bluetooth
bluetoothctl list        # must print one "Controller ..." line
mkdir -p ~/apps
```

From the build host, `ssh-copy-id ubuntu@<cp-ip>` and
`ssh-copy-id ubuntu@<ed-ip>`.

## 2. Build both apps (on the build host)

```bash
git clone https://github.com/project-chip/connectedhomeip.git && cd connectedhomeip
scripts/checkout_submodules.py --shallow --platform linux
docker pull ghcr.io/project-chip/chip-build-crosscompile:215
mkdir -p ~/.cache/chip-pw-env-crosscompile

# Runs a command in CI's arm64 cross-compile image, with the toolchain cached in ~/.cache
cross() {
  docker run --rm -it --user "$(id -u):$(id -g)" -e HOME=/tmp \
    -e PW_ENVIRONMENT_ROOT=/pw_env -v "$HOME/.cache/chip-pw-env-crosscompile:/pw_env" \
    -v "$PWD:/workdir" -w /workdir ghcr.io/project-chip/chip-build-crosscompile:215 \
    bash -c "git config --global --add safe.directory '*' && $1"
}

cross 'source scripts/bootstrap.sh'     # once, about 5.5 GB

# CP: all-devices-app without Wi-Fi PAF
cross 'source scripts/activate.sh &&
  export PKG_CONFIG_PATH="$SYSROOT_AARCH64/lib/aarch64-linux-gnu/pkgconfig" &&
  gn gen --root=examples/all-devices-app/posix out/cp \
    --args="chip_device_config_enable_wifipaf=false is_clang=true target_cpu=\"arm64\" sysroot=\"$SYSROOT_AARCH64\"" &&
  ninja -C out/cp all-devices-app'

# ED: chip-all-clusters-app
cross 'source scripts/activate.sh &&
  ./scripts/build/build_examples.py --target linux-arm64-all-clusters-clang build'
```

Once [PR #74981](https://github.com/project-chip/connectedhomeip/pull/74981) is
merged, the CP can also be built with `build_examples.py` and target
`linux-arm64-all-devices-clang-no-wifipaf`. Before that it fails with
`Include not allowed`. The image tag (`215`) is the one CI used on 2026-10-26;
the current tag is in `.github/workflows/examples-linux-arm.yaml`.

## 3. Copy the apps to the Pis

Stop the CP app and the ED app first if they are running. Otherwise `scp` fails
with "Text file busy". The `pkill` lines are harmless when nothing is running.

```bash
file out/cp/all-devices-app out/linux-arm64-all-clusters-clang/chip-all-clusters-app   # both "ARM aarch64"
ssh ubuntu@<cp-ip> pkill -x all-devices-app
ssh ubuntu@<ed-ip> pkill -x chip-all-cluste
scp out/cp/all-devices-app ubuntu@<cp-ip>:apps/
scp out/linux-arm64-all-clusters-clang/chip-all-clusters-app ubuntu@<ed-ip>:apps/
```

Then unplug the ED's Ethernet, so a pass can only come through the CP. From then
on, control the ED directly with a keyboard and screen, or over its serial port
from the TH (see step 4).

## 4. Start the CP and the ED

On the CP, **before every test** (stop the previous run with Ctrl-C):

```bash
rm -f /tmp/chip_* && ~/apps/all-devices-app --device commissioning-proxy:5 \
    --discriminator 3842 --ble-controller 0 2>&1 | tee /tmp/cp_test.log
```

`rm -f /tmp/chip_*` makes every test commission the CP from scratch. The CP is
ready when its log shows `BLE advertisement started successfully`. The
discriminator must not be used by any other Matter device on your network.

Do not start the ED in advance. The tests that use it tell you when to start and
stop it, with a prompt (see the table in step 6), and some tests ask more than
once. Before you start such a test, have a terminal ready on the ED (or a second
TH terminal, if you use the serial port).

-   **Prompt asks for the End Device to be commissionable**: start the ED app.
-   **Prompt asks for it to be not commissionable, stopped or powered off**:
    stop the ED app.

With a keyboard and screen on the ED, start it with this command and stop it
with Ctrl-C:

```bash
rm -f /tmp/chip_* && ~/apps/chip-all-clusters-app --discriminator 3841 --wifi --ble-controller 0
```

### Reset the ED's Wi-Fi

`TC_COMPRO_2_4` asks for this before it commissions the ED. The test gives the
ED its Wi-Fi network through the CP, so the ED must start with no saved network.
The ED app controls `wpa_supplicant` over D-Bus, so `wpa_supplicant` must run
with `-u`. These commands delete the ED's saved Wi-Fi networks and restart
`wpa_supplicant` with none. Run them on the ED, replacing `wlan0` if its Wi-Fi
interface has another name (`ls /sys/class/net`):

```bash
W=wlan0
sudo killall wpa_supplicant; sleep 1
printf 'ctrl_interface=DIR=/run/wpa_supplicant\nupdate_config=1\n' | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
sudo wpa_supplicant -u -s -B -i $W -c /etc/wpa_supplicant/wpa_supplicant.conf
# Check the ED is not connected to an AP: COMPLETED means a saved network is still in use
sudo wpa_cli -i $W status | grep wpa_state     # must not say COMPLETED
```

### Only when using the ED's serial port

Run these on the TH instead. They use the TH's serial console script, which sits
next to `compro_support.py`; that file's "Standalone-serial ED" section
describes the wiring and login set-up.

```bash
S=~/certification-tool/backend/test_collections/matter/sdk_tests/sdk_checkout/python_testing/scripts/sdk/support_modules/serial_console.py

# When a prompt asks for the End Device to be commissionable:
python3 $S --port /dev/ttyUSB0 run "rm -f /tmp/chip_*; nohup ~/apps/chip-all-clusters-app --discriminator 3841 --wifi --ble-controller 0 > /tmp/ed_app.log 2>&1 < /dev/null & sleep 3; pgrep -a -x chip-all-cluste"

# When a prompt asks for it to be stopped, not commissionable or powered off:
python3 $S --port /dev/ttyUSB0 run "pkill -x chip-all-cluste; sleep 2; pgrep -x chip-all-cluste || echo ed-stopped"

# When TC_COMPRO_2_4 asks to clear the ED's Wi-Fi (replace wlan0 if needed):
python3 $S --port /dev/ttyUSB0 run "W=wlan0; sudo killall wpa_supplicant; sleep 1; printf 'ctrl_interface=DIR=/run/wpa_supplicant\nupdate_config=1\n' | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf >/dev/null; sudo wpa_supplicant -u -s -B -i \$W -c /etc/wpa_supplicant/wpa_supplicant.conf; sleep 2; sudo wpa_cli -i \$W status | grep wpa_state"
```

Set `S` once per TH terminal. Keep `sleep 3; pgrep ...` after the `&`, or the
script waits 30 seconds and reports a timeout.

## 5. Set up th-cli (on the TH, once)

```bash
cd ~/certification-tool/cli/scripts
~/.local/bin/th-cli available-tests | grep -o 'TC_COMPRO_2_[0-9]' | sort -u     # must list 2_1 to 2_9
```

Write the PICS (the features the CP supports). A BLE-only CP has Background Scan
(`F01`) but not the Wi-Fi Network Interface feature (`F00`) or its WiFiBand
attribute (`A0007`):

```bash
mkdir -p ~/pics_cp_ble_only && cd ~/pics_cp_ble_only && python3 - <<'EOF'
yes = "S A0000 A0001 A0002 A0003 A0004 A0005 A0006 F01 C00.Rsp C01.Tx C02.Rsp C03.Rsp C04.Tx C05.Rsp C06.Rsp C07.Rsp C08.Tx".split()
no = "F00 A0007".split()
items = "".join(f"<picsItem><itemNumber>COMPRO.{i if i == 'S' else 'S.' + i}</itemNumber>"
                f"<support>{'true' if i in yes else 'false'}</support></picsItem>\n" for i in yes + no)
open("Commissioning Proxy Cluster Test Plan.xml", "w").write(
    f"<clusterPICS><name>Commissioning Proxy Cluster Test Plan</name>\n{items}</clusterPICS>\n")
EOF
```

Write the settings (the th-cli default with the CP's discriminator and your
access point), then create the project. The ID it prints is your `<project-id>`:

```bash
cd ~/certification-tool/cli/scripts && python3 - '<ssid>' '<password>' <<'EOF'
import json, sys
c = json.load(open("/home/ubuntu/certification-tool/cli/th_cli/default_config.json"))
c["dut_config"]["discriminator"] = "3842"
c["network"]["wifi"]["ssid"], c["network"]["wifi"]["password"] = sys.argv[1], sys.argv[2]
json.dump(c, open("/home/ubuntu/cp-ble-project.json", "w"), indent=1)
EOF
~/.local/bin/th-cli project create --name cp-ble-only --config ~/cp-ble-project.json -p ~/pics_cp_ble_only
```

## 6. Run the tests

On the TH, in `~/certification-tool/cli/scripts`. Run the tests one at a time,
in any order. Before each one, restart the CP (step 4). Each command below is
complete: copy it as it is, replacing only `<project-id>` (and, for 2_4,
`<ssid>` and `<password>`).

Tests without the ED:

```bash
~/.local/bin/th-cli run-tests -t TC_COMPRO_2_1 --project-id <project-id> -- --endpoint 5
~/.local/bin/th-cli run-tests -t TC_COMPRO_2_5 --project-id <project-id> -- --endpoint 5
~/.local/bin/th-cli run-tests -t TC_COMPRO_2_9 --project-id <project-id> -- --endpoint 5
```

Tests with the ED (start and stop it when the prompts ask, step 4):

```bash
~/.local/bin/th-cli run-tests -t TC_COMPRO_2_2 --project-id <project-id> -- --endpoint 5 \
  --string-arg ed_transport:ble --int-arg "ed_discriminator:3841 ed_passcode:20202021"

~/.local/bin/th-cli run-tests -t TC_COMPRO_2_3 --project-id <project-id> -- --endpoint 5 \
  --string-arg ed_transport:ble --int-arg "ed_discriminator:3841 ed_passcode:20202021"

~/.local/bin/th-cli run-tests -t TC_COMPRO_2_6 --project-id <project-id> -- --endpoint 5 \
  --string-arg ed_transport:ble --int-arg "ed_discriminator:3841 ed_passcode:20202021"

~/.local/bin/th-cli run-tests -t TC_COMPRO_2_7 --project-id <project-id> -- --endpoint 5 \
  --string-arg ed_transport:ble --int-arg "ed_discriminator:3841 ed_passcode:20202021"

~/.local/bin/th-cli run-tests -t TC_COMPRO_2_8 --project-id <project-id> -- --endpoint 5 \
  --string-arg ed_transport:ble --int-arg "ed_discriminator:3841 ed_passcode:20202021"

~/.local/bin/th-cli run-tests -t TC_COMPRO_2_4 --project-id <project-id> -- --endpoint 5 \
  --string-arg "ed_transport:ble wifi_ssid:<ssid> wifi_password:<password>" \
  --int-arg "ed_discriminator:3841 ed_passcode:20202021"
```

Type each command on one line or keep the `\` line breaks as shown.
`--string-arg` and `--int-arg` may each appear only once, so all the values for
one flag go in one quoted string.

th-cli stops at prompts. Answer a numbered prompt (`1: YES  2: NO`) by typing
the number and pressing Enter; for any other prompt, do what it asks and press
Enter. Each prompt fails after 60 seconds unless you add `--prompt-timeout 180`
before the `--`.

| The prompt says                                                                                                                | Do this                                                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reuse the previous commissioning information?                                                                                  | `2` (NO)                                                                                                                                                                                                                                                                                      |
| Make sure the DUT is in Commissioning Mode                                                                                     | `1` (DONE)                                                                                                                                                                                                                                                                                    |
| Make / Ensure the End Device is commissionable (any transport named)                                                           | Start the ED (step 4). The CP uses BLE whatever the prompt says                                                                                                                                                                                                                               |
| The End Device is NOT commissionable / Stop / Power off                                                                        | Stop the ED                                                                                                                                                                                                                                                                                   |
| (2_4) Run `~/script/renew-comee_env.sh` on the End Device to clear its stored Wi-Fi credentials and restart wpa_supplicant ... | Run the commands in [Reset the ED's Wi-Fi](#reset-the-eds-wi-fi) (step 4) on the ED, or its serial-port line, then Enter. The script named in the prompt is a hangover from Wi-Fi PAF testing, whose test kit supplies it. A BLE-only set-up does not have it; those commands do the same job |
| (2_6) Ensure the DUT (proxy) is factory-reset                                                                                  | Already done by the CP restart. Start the ED                                                                                                                                                                                                                                                  |

The result is the `TC-COMPRO-2.x [PASSED]` line at the end; steps marked
`NOT_APPLICABLE` cover features a BLE-only CP does not have. th-cli keeps each
run's log in `output_logs/` under the directory it was run from.
