# CCS2026 Artifacts - From Fault to Oracle: Breaking Google Pixel TPU Hardware Memory Isolation via Trusted Deserialization Faults

[Artifact Evaluation appendix (PDF)](ccs2026-ae-template.pdf) · [LaTeX source](ccs2026-ae-template.tex)

This package supports three evaluation stages: acquire two EdgeTPU memory
regions from a prepared Pixel 9a, import the resulting dump logs into IDA and
recover candidate functions, and scan reconstructed firmware for candidate
fault-oracle memory-access sites. Reviewers can start with the device, the
bundled logs, or the prepared IDA database. The PDF gives the formal AE
appendix; this README gives the commands and checks for each stage.

## 1. Description and requirements

### 1.1 Access to the prepared device

Live acquisition requires access to the authors' prepared, rooted Pixel 9a
through the reviewer Tailscale network. Obtain the network invitation and SSH
connection details through the artifact-review channel. Root access is already
enabled: after connecting, run `su` to enter a root shell. No additional
rooting steps are required. Contact the authors through that channel if the
invitation expires or the device is offline. Stages 2 and 3 work offline.

Only run acquisition on the prepared device or another authorized test device.
It stops the EdgeTPU logging service and may reboot the device or interrupt
SSH. After reconnecting, enter root and stop logging again before resuming.
Restore logging when acquisition finishes. Import and recovery change the IDA
database, so work on a copy.

### 1.2 Dependencies

| Task | Requirements |
| --- | --- |
| Live acquisition | Prepared Pixel 9a with `/dev/edgetpu-soc`, Tailscale, SSH, and root access |
| Offline import and recovery | IDA Pro 9.4 with IDAPython |
| Fault-oracle scanning | IDA Pro 9.4 with a licensed ARM Hex-Rays decompiler |
| Optional rebuild and deployment | Android NDK and ADB access to the device |

IDA and Hex-Rays are proprietary and are not bundled. The build script uses
the NDK's `darwin-x86_64` host directory and
`aarch64-linux-android34-clang`; adjust its toolchain path on other host
platforms. SSH access does not imply ADB access. Minimum workstation RAM and
disk requirements have not been measured.

### 1.3 Package contents

| Path | Purpose |
| --- | --- |
| [`POC/DumpAuto_stable/`](POC/DumpAuto_stable/) | `runner` and `cracktpu` acquisition binaries, source, and `compile.sh` |
| [`Public_artifacts/edgetpu-rio.fw`](Public_artifacts/edgetpu-rio.fw) | Raw firmware for a fresh IDA import |
| [`Public_artifacts/0x0ff00000_0x0ff0E4FF_dump.log`](Public_artifacts/0x0ff00000_0x0ff0E4FF_dump.log) | First reference dump |
| [`Public_artifacts/0x10000000_0x1000ea80_dump.log`](Public_artifacts/0x10000000_0x1000ea80_dump.log) | Second reference dump |
| [`Public_artifacts/import_dump_log_to_ida.py`](Public_artifacts/import_dump_log_to_ida.py) | Patch dump words into an IDA database |
| [`Public_artifacts/recover_tpu_funcs.py`](Public_artifacts/recover_tpu_funcs.py) | Heuristic function recovery |
| [`Public_artifacts/faultoracle_scanner_mc.py`](Public_artifacts/faultoracle_scanner_mc.py) | Hex-Rays microcode scanner |
| [`edgetpu-rio_eval_re.fw.i64`](edgetpu-rio_eval_re.fw.i64) | Prepared IDA Pro 9.4 database |

Each dump record contains an address and a 32-bit value, optionally followed
by a tag such as `cache_unstable`:

```text
0x0ff00000:0xbf182800
0x0ff00018:0x43db6a23:cache_unstable
```

The bundled input sizes are:

| Reference log | Words | Tagged words | Last word address |
| --- | ---: | ---: | --- |
| `0x0ff00000_0x0ff0E4FF_dump.log` | 14,656 | 2,175 | `0x0ff0e4fc` |
| `0x10000000_0x1000ea80_dump.log` | 15,009 | 2,150 | `0x1000ea80` |

These counts describe the supplied inputs. Unstable locations in a fresh
acquisition need not match the bundled values byte for byte.

## 2. Set up and basic test

For the shortest offline check, copy and open
`edgetpu-rio_eval_re.fw.i64` in IDA Pro 9.4, wait for analysis, then use
**File > Script file** to run
`Public_artifacts/faultoracle_scanner_mc.py`. The IDA output window should
show reports for configured handlers and finish with:

```text
TOTAL sites (after per-def-site dedup): ...
```

Check the output for `no function at` and `gen_microcode failed`. A final
total alone does not show that every handler was analyzed. No verified
reference site count is supplied with this package.

For a fresh import instead, load `Public_artifacts/edgetpu-rio.fw` into IDA
as **ARM little-endian raw binary at base address `0x0ffff000`**, then save a
working database. Do not rebase or reimport the prepared `.i64` database.

## 3. Evaluation workflow

The stages correspond to the AE appendix's claims and experiments:

| Claim | Experiment | Evidence to inspect |
| --- | --- | --- |
| C1: acquire two memory regions and resume after interruption | E1 / Stage 1 | Dump logs, final addresses, and instability tags |
| C2: import words and recover candidate function boundaries | E2 / Stage 2 | Import counts and recovery output |
| C3: find candidate fault-oracle memory-access sites | E3 / Stage 3 | Per-handler scanner reports and inspected sites |

### Stage 1 / E1: memory acquisition

Connect to the prepared device using the details from Section 1.1. The
prebuilt `runner` and `cracktpu` binaries can be used if they are already
deployed in `/data/local/tmp/`. In the device shell, run:

```sh
su
stop edgetpu_logging
cd /data/local/tmp
chmod +x runner cracktpu
mkdir -p review-dumps
./runner 0x0ff00000 0x0ff0e500 ./cracktpu review-dumps/0x0ff00000_0x0ff0E4FF_dump.log
./runner 0x10000000 0x1000ea84 ./cracktpu review-dumps/0x10000000_0x1000ea80_dump.log
start edgetpu_logging
```

`runner` takes an inclusive start address, an **exclusive** end address, the
helper path, and the output log path. It advances in four-byte steps. If the
device reboots or SSH disconnects, reconnect, run `su` and
`stop edgetpu_logging` again, and rerun the same command with the same log
path. It resumes after the last valid logged address. Restore logging when
finished.

Retrieve the logs and check that their last word addresses are
`0x0ff0e4fc` and `0x1000ea80`, respectively. Review instability tags and
incomplete runs instead of requiring identical values at every location.

To rebuild the helpers on a host with an Android NDK and an ADB-connected
device, run this from the repository root (adjust the NDK path):

```sh
cd POC/DumpAuto_stable
NDK=/absolute/path/to/android-ndk sh compile.sh
```

The script compiles both binaries and pushes them to `/data/local/tmp/`.

### Stage 2 / E2: import and function recovery

Use the fresh database described in Section 2 and keep a working copy. In
IDA, run `Public_artifacts/import_dump_log_to_ida.py` through **File >
Script file**. Select the first supplied log when prompted, wait for
reanalysis, then run the script again and select the second log. If IDA has no
segment at the first dump address, the importer can offer to create
`.dump_0ff`; check the address range before accepting.

The importer reports `patched words`, `malformed lines skipped`, and
`unmapped addresses skipped`. Expect **14,656** and **15,009** patched
words, respectively, if all log addresses are mapped. Investigate malformed
or unmapped entries rather than treating those counts as unconditional.

Next, run `Public_artifacts/recover_tpu_funcs.py` through **File > Script
file**. Its `SEG_NAME` setting defaults to `ROM`; verify that it names the
segment you intend to process. Run it again for other relevant segments,
including `.dump_0ff`, after changing that setting if needed. Save the
output ending in `DONE — added ... funcs total` and inspect candidate
functions. Zero additions can mean the database was already processed or
that the wrong segment was selected.

### Stage 3 / E3: fault-oracle scanning

Open a copy of the prepared database and run
`Public_artifacts/faultoracle_scanner_mc.py` through **File > Script file**
with the ARM Hex-Rays decompiler available. The script analyzes its
configured `HANDLERS` at `MMAT_CALLS` microcode maturity.

Record the IDA and Hex-Rays versions, database identity, each handler's
report, site addresses, and the final deduplicated total. Inspect reported
instructions and pseudocode manually. Reported sites are candidates, not
automatically confirmed vulnerabilities or proof of exploitability. A
verified reference count and quantitative acceptance threshold have not been
provided.

## 4. Reuse and release notes

The bundled logs and prepared database support offline evaluation without
the Pixel. Other devices and firmware builds are not assumed compatible.
For new firmware versions, review the segment names, handler addresses, and
payload registers. Function recovery is heuristic, and scanner failures do
not establish that no candidates exist.

The archive DOI, redistribution license, measured person time and compute
time, exact paper section/table references, and paper-specific reference
results still need to be supplied for a final release. This appendix follows
the Artifact Evaluation LaTeX template version 1.0.
