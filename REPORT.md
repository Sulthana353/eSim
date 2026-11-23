# eSim 2.5 installation on Ubuntu 25.04

**FOSSEE Semester Long Internship · Autumn 2026 · Screening Task 4**

The eSim 2.5 installer encountered five distinct blockers on Ubuntu 25.04. After changes to the Bash installers, the NGHDL installer reached its success message and reported creating the Ngspice and NGHDL symlinks. That result came from a direct run of the edited NGHDL script. A fresh top-level installation and an eSim co-simulation test remain pending.

## Test environment and method

Tests used the official eSim 2.5 Linux ZIP on an Ubuntu 25.04 (Plucky) VMware virtual machine. GHDL reported an `x86_64-linux-gnu` build machine and GCC 14.2.0. The installer successfully installed PyQt5 5.15.11 for Python 3.13 before it reached the KiCad failure. FOSSEE lists Ubuntu 22.04, 23.04, and 24.04 for this release. [FOSSEE downloads](https://esim.fossee.in/downloads)

The initial run used `./install-eSim.sh --install` from the extracted eSim directory. After each failure, the relevant installer was inspected and the affected stage rerun. NGHDL was tested directly after extraction so its Bash script could be edited and rerun independently. Installer changes belong on the fork's `installers` branch, with readable diffs for the scripts that ship inside archives. [FOSSEE installers branch](https://github.com/FOSSEE/eSim/tree/installers)

## 1. The eSim launcher rejected Ubuntu 25.04

The untouched launcher exited before any dependency checks.

```text
Detected Ubuntu Version:
Unsupported Ubuntu version: 25.04 ()
```

The launcher read `25.04` from `/etc/os-release`, but its case statement had no matching entry. The blank field came from a separate parsing issue. The release description regex required three components, so it failed to capture `25.04`.

In `install-eSim.sh`, the version parser was adjusted to accept an optional third component and fall back to `VERSION_ID`. This preserves the existing comparison against `22.04.4`.

```bash
FULL_VERSION=$(lsb_release -d | grep -oE '[0-9]+\.[0-9]+(\.[0-9]+)?')
FULL_VERSION=${FULL_VERSION:-$VERSION_ID}
```

The `"24.04")` case was extended to `"24.04"|"25.04")` so the 25.04 run could use the bundled `install-eSim-24.04.sh`. The subsequent run reached Python dependency installation and the KiCad step, confirming that the launcher no longer blocked progress.

## 2. The KiCad 6.0 PPA had no Plucky Release file

The selected eSim installer added `ppa:kicad/kicad-6.0-releases`. APT requested the `plucky` suite and received a 404. The Ubuntu archive sources in the same log returned `Hit`. [KiCad 6.0 PPA distribution index](https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu/dists/)

```text
Err:6 https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release
  404  Not Found
E: The repository 'https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release' does not have a Release file.
```

The change in `install-eSim-scripts/install-eSim-24.04.sh` clears `kicadppa` on 25.04 and guards the existing PPA setup with `[[ -n "$kicadppa" ]]`. The package installation is outside that guard. Its package names are unpinned, so APT can resolve them from the configured Ubuntu repositories. On a VM where the failed attempt already added the PPA, remove the entry before retrying.

```bash
sudo add-apt-repository --remove --yes ppa:kicad/kicad-6.0-releases
sudo apt-get update
apt-get -s install --no-install-recommends \
    kicad kicad-footprints kicad-libraries kicad-symbols kicad-templates
```

A later run reached NGHDL extraction, demonstrating progress past the earlier PPA abort. KiCad integration still needs a schematic test. The installer copies the eSim symbol table to `~/.config/kicad/6.0/`, so verify that the installed KiCad uses the table and resolves eSim symbols.

## 3. The bundled NGHDL launcher also rejected Ubuntu 25.04

The top-level installer extracted `nghdl.zip` and then printed `Unsupported Ubuntu version: 25.04 ()` again. The installer calls `./install-nghdl.sh --install` from the extracted `nghdl/` directory, where the bundled launcher has its own version check.

The parent install path was adjusted to select the bundled 24.04 NGHDL script explicitly on 25.04 after extraction. Other releases use the existing launcher.

```bash
if [[ "$(lsb_release -rs)" == "25.04" ]]; then
    bash ./install-nghdl-scripts/install-nghdl-24.04.sh --install
else
    ./install-nghdl.sh --install
fi
```

Running that child script directly reached the compiler and Canberra dependency steps. The parent dispatch still needs verification from a fresh full install. The existing uninstall path also calls the NGHDL launcher and was outside the tested installation path.

## 4. APT could not resolve the GTK 2 Canberra module

The NGHDL dependency step requested `libcanberra-gtk-module` and `libcanberra-gtk3-module` together. APT reported `Package 'libcanberra-gtk-module' has no installation candidate`. On this VM, the GTK 3 module was already installed at version `0.30-17ubuntu2` from Plucky, and an install simulation resolved it successfully.

```bash
apt-cache policy libcanberra-gtk-module libcanberra-gtk3-module
apt-get -s install libcanberra-gtk3-module
```

In `nghdl/install-nghdl-scripts/install-nghdl-24.04.sh`, the 25.04 path requests only `libcanberra-gtk3-module`. The existing package list remains in use for other releases. The subsequent run reached GHDL configuration, confirming that the package transaction no longer blocked this stage.

## 5. GHDL 4.1.0 rejected LLVM 20.1.2

GHDL extracted successfully. Its configure step failed with `Unhandled version llvm 20.1.2`. The installer used the unversioned LLVM packages and passed `/usr/bin/llvm-config` to GHDL. GHDL 4.1.0's configure script accepts LLVM 18.1.x and rejects LLVM 20.1.x. [GHDL 4.1.0 configure source](https://github.com/ghdl/ghdl/blob/v4.1.0/configure#L268-L306)

```bash
apt-get -s install llvm-18-dev
/usr/bin/llvm-config-18 --version
```

APT reported `llvm-18-dev` already installed at `1:18.1.8-17`. The 25.04 dependency branch was changed to install `llvm-18-dev` and skip removal of the unversioned LLVM packages. The GHDL configure step on 25.04 uses the versioned executable explicitly.

```bash
./configure --with-llvm-config=/usr/bin/llvm-config-18
```

The rerun reached the simulator's install stage and printed `NGHDL installed sucessfully`, followed by messages reporting both symlinks. It also printed `rm: cannot remove '/usr/bin/ngspice': No such file or directory` while clearing the old path, then continued to symlink creation. Runtime verification still needs to confirm that the new link resolves and that the installed GHDL uses the LLVM backend.

## Clean snapshot reproduction and verification

1. Save the patched files and earlier logs outside the VM snapshot. From a clean snapshot, run the untouched eSim 2.5 ZIP and retain its launcher rejection. Use the same Ubuntu APT source configuration for both runs. If the default Ubuntu mirrors fail after Plucky's end of life, record the archive configuration as VM setup. [Ubuntu end-of-life announcement](https://lists.ubuntu.com/archives/ubuntu-announce/2026-January/000320.html)
2. Restore the clean snapshot. Apply the patched eSim launcher and child script. Include the patched NGHDL child script in the archive because `unzip -o nghdl.zip` overwrites the extracted copy each time. From the eSim directory, run `zip nghdl.zip nghdl/install-nghdl-scripts/install-nghdl-24.04.sh` and `unzip -tq nghdl.zip`. Keep a readable script diff alongside the installer changes.
3. From the extracted eSim directory, capture each run with the following commands. Use `baseline` for the untouched run and `patched` for the patched run. Save the baseline log before restoring.

```bash
run=patched
set -o pipefail
./install-eSim.sh --install 2>&1 | tee "esim-2.5-ubuntu-25.04-$run.log"
installer_status=${PIPESTATUS[0]}
printf 'Installer exit status = %s\n' "$installer_status" |
    tee -a "esim-2.5-ubuntu-25.04-$run.log"
```

4. After the full patched run, check `readlink -e /usr/bin/ngspice`, `ngspice --version`, and `ghdl --version`. Launch the eSim GUI, open a schematic with an eSim custom symbol in KiCad, and run an NGHDL mixed-signal example. Record the example name, tool versions, and observed output in the submission.
