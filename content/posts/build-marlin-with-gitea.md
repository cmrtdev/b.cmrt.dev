+++
date = '2026-01-27T20:15:49+01:00'
title = 'Build Marlin With Gitea Actions'
+++

Or: how I got tired of having to use the excellent [Auto Build Marlin](https://marlinfw.org/docs/basics/auto_build_marlin.html) and decided to automate the build process even more.

## Background

I run a local [Gitea](https://github.com/go-gitea/gitea) for all my `git` needs; recently(-ish)[^ish] they introduced Gitea Actions, for CI/CD, instead of relying on external tools like Woodpecker.

I also have a Creality Ender-3 Pro 3D printer, whose stock firmware is... Not great..

From what I remember, it does not (or at least did not, when I got it) have [Thermal Runaway](https://en.wikipedia.org/wiki/Thermal_runaway) protection enabled in the firmware[^trp].

This can be a problem if your printer's thermistor is broken and telling the printer that the hot-end is cold, even though the heater block is pumping watt after watt of heat into it. This can cause a fire, which is Very Not Good.

Thermal Runaway protection does a relatively simple (AND NOT TO BE RELIED ON) check: if the thermistor has been saying "hot-end is cold" for too long after the printer started heating, it will emergency stop the printer to prevent a fire (because it assumes that there is an issue with the thermistor).

On top of this, at the time the Ender-3 Pro shipped with Marlin `1.x`, and since I like to live on the edge (and wanted to customize my printer's name), I wanted my printer on the newer Marlin `2.x`.

The [official instructions](https://marlinfw.org/docs/basics/install.html) for building the firmware are relatively simple, and it has getting easier and easier, especially after the release of Auto Build Marlin (ABM), a VSCode extension that - after setting a couple parameters - does the whole job for you: it builds it, you upload it to the printer.

Of course, this still requires you to figure out exactly which options to set for your printer. For most common printers the awesome developers of Marlin (and its contributors) provide pre-built [configuration files](https://github.com/MarlinFirmware/Configurations) that you can use (e.g. they set the correct build plate size, and which basic features the printer has), but I wanted more: I wanted BLTouch (because I paid for it) and automatic bed levelling; I wanted host action commands (for [OctoPrint](https://octoprint.org/)); I wanted to be able to pause my prints to add magnets into them; and, as I mentioned, I wanted to give my printer its own name.

## How I started

To build your own Marlin you first download both "generic Marlin" and the Configurations, then take your printer's `Configuration.h` and `Configuration_adv.h` (and all the other `.h` files they give you) and copy them over to the Marlin source folder. You comment/uncomment/edit the C header files to enable/disable/edit the things you want, then use ABM to build it.

I started with a `README.md`: I wrote down all the configurations I wanted to change, and changed them, then Auto Build Marlin-ed the firmware and flashed it.

Dear reader: can you feel the annoyance? The boredom? How is there no way to make this easier?

I quickly realized that Marlin developers probably had a similar issue, what with them having to build a million different version of the firmware, and they did the work I really, really didn't want to do: in the main repository is a `buildroot/bin` directory with scripts that can:

* Enable and disable features (a.k.a. comment/uncomment C header lines)
* Add, remove, and change settings (a.k.a. edit specific C header lines)
* Fetch the configuration of a specific printer (I don't really need it, but they do)
* Do a lot more things that I don't really care about, but that are essential for building a solid product

So I wrote a shell script that did that the commenting/uncommenting/editing of the files for me, and for a while it was good... But I wanted more.

I figured out I could invoke [PlatformIO](https://platformio.org/), the actual build system that ABM uses, directly and save myself the ABM step. Marlin actually [tells you](https://marlinfw.org/docs/basics/install_platformio_cli.html) how to do that. But that still requires... Typing...

And so we come to a few days ago.

## Gitea Actions

The plan is simple:

1. I can `git clone` the firmware and configurations
2. I have a shell script that makes all the changes I need, unattended
3. Write a Gitea Actions workflow that does #1 and #2, then runs `platformio` and builds the firmware
4. And while I'm at it, it "releases" it (in my internal Gitea) so all I have to do is download it and add it to the printer

While I could also auto-push the firmware, my printer spends most of its life powered off, so that's kinda pointless (I'd have to turn it on manually anyway).

### Configuration updater script

The script that updates the settings is fairly simple. I won't go through all the settings because, TBH, I don't remember what most of them are for: they've been collected from articles, guides, and the likes, and I didn't keep track of them.

The script lives in a `scripts` subfolder in a dedicated Gitea repository.

First it does a `cd` to the subfolder where the actual Marlin code will be downloaded, and then checks that that folder has a `Marlin` subfolder. That's where the code, including the `Configuration.h` and `Configuration_adv.h` files have to be.

```bash
#!/bin/bash
cd $1

if [ ! -d "./Marlin" ]; then
    echo "Can't find a Marlin subfolder"
    exit 1
fi

for exe in opt_disable opt_enable opt_set; do
    if [ ! -x "./buildroot/bin/${exe}" ] ; then
        echo "Can't find ./buildroot/bin/${exe}. Aborting."
        exit 127
    fi
done

echo "Starting to patch"
opt_set="./buildroot/bin/opt_set"
opt_enable="./buildroot/bin/opt_enable"
opt_disable="./buildroot/bin/opt_disable"

# Configuration.h
$opt_set CUSTOM_MACHINE_NAME '"Your Machine Name"'    # Set to the machine name
$opt_set X_BED_SIZE 230
$opt_enable S_CURVE_ACCELERATION
$opt_enable INDIVIDUAL_AXIS_HOMING_MENU
$opt_enable LCD_BED_TRAMMING
$opt_enable BLTOUCH
$opt_disable Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN
$opt_enable USE_PROBE_FOR_Z_HOMING
$opt_set NOZZLE_TO_PROBE_OFFSET "{ -45.1, -7.5, 0 }"
$opt_set PROBING_MARGIN 20
$opt_enable LCD_BED_LEVELING
$opt_enable AUTO_BED_LEVELING_UBL
$opt_enable RESTORE_LEVELING_AFTER_G28
$opt_enable Z_SAFE_HOMING
$opt_enable NOZZLE_PARK_FEATURE                       # Filament Change

# Configuration_adv.h
$opt_set BLTOUCH_DELAY 200
$opt_enable HOST_ACTION_COMMANDS
$opt_enable HOST_PROMPT_SUPPORT         # For multiple things, incl. Filament Change
$opt_enable LONG_FILENAME_HOST_SUPPORT
$opt_enable EMERGENCY_PARSER
$opt_enable AUTO_REPORT_TEMPERATURES
$opt_enable AUTO_REPORT_POSITION
$opt_enable M114_DETAIL
$opt_enable M114_REALTIME
$opt_enable GCODE_CASE_INSENSITIVE
$opt_enable ADVANCED_PAUSE_FEATURE      # Filament Change
$opt_enable PARK_HEAD_ON_PAUSE          # Filament Change (to add the G-code M125 Pause and Park)
```

The `opt_*` commands figure out by themselves which file to modify, which is pretty neat if you ask me.

> [!IMPORTANT]
> This script is (obviously) tailored to **my** printer. For example, it forces BLTouch support, and to do that is DISABLES THE Z-AXIS ENDSTOP. If you don't have a BLTouch installed, using this will break your Z-axis. \
> Edit the script to only do what your printer can, or you will suffer the consequences (and I take no responsibility for it). \
> **You have been warned.**

### Workflow

> [!WARNING]
> As-is this workflow and script will work to build firmware for a Creality Ender-3 Pro **with a v4.2.2 motherboard** and my own setup. Make sure you know what you're doing before using it, and you change all the correct things. \
> **I am NOT responsible for you bricking or breaking your own printer, setting fire to your house, or any outcome (positive or negative) from using this without knowing what you're doing.**

I have mirrors of [Marlin](https://github.com/MarlinFirmware/Marlin/) and [MarlinConfigurations](https://github.com/MarlinFirmware/Configurations) on Gitea, which is easier than pulling from GitHub every time (`actions/checkout` doesn't really do that by default, and I didn't want to figure out how to do that, so I set up Gitea mirrors).

`mirrors/Marlin` is downloaded in the `Code` subfolder, and `mirrors/Marlin-Configurations` in the `Configurations` subfolder. After all the checkouts, the working directory looks like this (not showing files that don't matter):

```text
├── .gitea
│   └── workflows
│       └── marlin.yaml
├── Code
│   ├── buildroot
│   │   ├── bin
|   |   |   [...]
│   │   │   ├── opt_add
│   │   │   ├── opt_disable
│   │   │   ├── opt_enable
│   │   │   ├── opt_find
│   │   │   ├── opt_set
|   |   |   [...]
|   |   [...]
│   ├── config
│   ├── docker
│   ├── docs
│   ├── ini
│   ├── LICENSE
│   ├── Makefile
│   ├── Marlin
│   │   ├── config.ini
│   │   ├── Configuration_adv.h
│   │   ├── Configuration.h
|   |   |   [...]
│   │   └── Version.h
│   ├── platformio.ini
│   ├── README.md
│   └── test
├── Configurations
│   ├── bin
│   ├── config
|   │   └── examples
│   │       ├── [...]
|   │       ├── Creality
│   │       │   ├── [...]
│   │       │   ├── Ender-3 Pro
│   │       │   │   ├── [...]
│   │       │   │   ├── CrealityV422
│   │       │   │   │   ├── _Bootscreen.h
│   │       │   │   │   ├── Configuration_adv.h
│   │       │   │   │   ├── Configuration.h
│   │       │   │   │   ├── README.md
│   │       │   │   │   └── _Statusscreen.h
│   │       │   │   ├── [...]
│   ├── Configurations.sublime-project
│   ├── LICENSE
│   └── README.md
└── scripts
    └── update_configurations.sh
```

This is `marlin.yaml` in `.gitea/workflows`:

```yaml
name: Build Marlin
run-name: Build Marlin
on: [push]

env:
  MARLIN_TAG: 2.1.2.7
  MARLIN_BOARD: STM32F103RE_creality

jobs:
  Prepare-Marlin:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository code
        uses: actions/checkout@v6
      - name: Check out Marlin
        uses: actions/checkout@v6
        env:
          NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt
        with:
          repository: mirrors/Marlin
          ref: ${{ env.MARLIN_TAG }}
          path: ./Code
          token: ${{ secrets.REPOSITORY_ACCESS }}
      - name: Check out Marlin Configurations
        uses: actions/checkout@v6
        env:
          NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt
        with:
          repository: mirrors/Marlin-Configurations
          ref: ${{ env.MARLIN_TAG }}
          path: ./Configurations
          token: ${{ secrets.REPOSITORY_ACCESS }}
      - name: Copy Creality header files to Marlin code directory
        run: cp Configurations/config/examples/Creality/Ender-3\ Pro/CrealityV422/*.h Code/Marlin/
      - name: Update configuration using script
        # update_configurations.sh <marlin_code_directory>
        run: bash ./scripts/update_configurations.sh Code
      - name: "DEBUG: Contents of Configuration.h"
        run: cat Code/Marlin/Configuration.h
      - name: "DEBUG: Contents of Configuration_adv.h"
        run: cat Code/Marlin/Configuration_adv.h
      - name: Upload code to be ready for the next step
        # https://github.com/go-gitea/gitea/issues/36024
        uses: https://github.com/christopherHX/gitea-upload-artifact@v4
        env:
          NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt
        with:
          name: marlin-code
          path: Code
          retention-days: 1 # Don't need this for more than a day.
  Build-Marlin:
    runs-on: ubuntu-latest
    needs: Prepare-Marlin
    steps:
      - name: Get artifacts
        # https://github.com/go-gitea/gitea/issues/36024
        uses: https://github.com/christopherHX/gitea-download-artifact@v4
        env:
          NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt
        with:
          name: marlin-code
          path: .
      # https://docs.platformio.org/en/stable/integration/ci/github-actions.html
      # Without actions/cache@v4 because it just times out, and this still works without it.
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install PlatformIO Core
        run: pip install --upgrade platformio
      - name: Build Marlin
        run: platformio run -e $MARLIN_BOARD
      - name: Upload Firmware Release
        # https://gitea.com/actions/gitea-release-action
        uses: akkuman/gitea-release-action@v1
        env:
          NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt
        with:
          name: v${{ env.MARLIN_TAG }}
          tag_name: v${{ env.MARLIN_TAG }}
          files: |-
            .pio/build/${{ env.MARLIN_TAG }}/firmware-*.bin
```

It's fairly self-explanatory, but here's a few key points:

#### Static version/tag

This workflow will build a specific version of the firmware, defined by the environment variable `MARLIN_TAG`.

That's because I don't want to deal with trying to figure out whatever `latest` is and then set up automation to check and build-when-needed.

If I was running a print farm I may look into it, but since I need to be present to update the firmware, changing the variable and pushing a commit, then waiting the 5 minutes[^actually] it takes to build it before sending it to the printer is very much not a problem.

#### `MARLIN_BOARD`

`MARLIN_BOARD` is also a variable, set to `STM32F103RE_creality`. It's used only twice, but it's easy to find (and I had to change it before, when they [renamed it](https://github.com/MarlinFirmware/Marlin/issues/23902#issuecomment-1067487080) between Marlin 2.0.x and 2.1.x)

#### `secrets.REPOSITORY_ACCESS`

I created a [Gitea Secret](https://docs.gitea.com/usage/actions/secrets) called `REPOSITORY_ACCESS`, which contains a Personal Access Token for a dedicated user that can clone the internal `Marlin` and `Marlin-Configurations` repositories, as they're owned by a dedicated user, different from the one that runs the workflow.

Depending on how your environment is set up, this can be removed and the normal `${{ secrets.GITHUB_TOKEN }}` (yes: `GITHUB`, not `GITEA`) used, or you could force `ssh` for fetching.

This works for me, and if it's not broken...

#### `NODE_EXTRA_CA_CERTS`

You may have noticed most actions have `NODE_EXTRA_CA_CERTS: /etc/ssl/certs/ca-certificates.crt`. That's because my internal SSL certificate is self-signed, and the action runners don't like that.

If you don't have your own private CA, or self-signed certs, and you're using e.g. Let's Encrypt certificates, comment those entries out and you'll be fine.

> [!IMPORTANT]
> If you are in the same boat as me, remember to mount `/etc/ssl/certs/ca-certificates.crt` into the Docker container for the Actions Runner. \
> Many thanks to [this forum post](https://forum.gitea.com/t/cannot-checkout-a-repository-hosted-on-a-gitea-instance-using-self-signed-certificate-server-certificate-verification-failed/7903/5) for explaining it; I made a couple changes to those instructions because e.g. I don't have `/etc/ca-certificates`, but they're fairly simple changes. Don't forget `valid_volumes` (you have one guess as to why I'm pointing it out).

#### Using two jobs

Could I have done this in a single job? Yes. Why did I choose to do this in multiple jobs? Because I could, and this is my first time playing with Gitea Actions.

No, really: you can just get rid of the second one, I just like having it.

This also means I have artifacts that I don't really need (the code from the `Prepare-Marlin` job), but the resulting zipfile is pretty small and expires in 1 day, so I don't really care.

#### Actions cache

What about `actions/cache@v4`? It's in the official PlatformIO guidance, but in my case it was timing out, and I'm pretty sure (albeit not certain) that it's because those instructions are for GitHub. The step was timing out after being unable to contact the actions server, so I removed it.

Could it be beneficial? Maybe, but I'm building something that takes very little resources and time, so I don't care.

#### Debugging a workflow from within the container

_(Credit goes to `mariusrugan` from [this forum post](https://forum.gitea.com/t/unable-to-run-workflow-on-private-internal-repository/9160))_

I found it extremely useful to be able to `docker exec` my way into the workflow containers to see what was happening (e.g. when I didn't realize that multiple steps meant a clean repo every time, and couldn't figure out why the workflow was failing).

To do that, add this step wherever you want the workflow to stop:

```yaml
- name: Debug
  shell: bash
  run: |
    sleep 3600
```

Wait until that step is running and then, on the machine running the workflow run `docker ps` to show all running containers.

Find the container you want, for example `GITEA-ACTIONS-TASK-89_WORKFLOW-Build-Marlin_JOB-Prepare-Marlin`, and `docker exec -it GITEA-ACTIONS-TASK-89_WORKFLOW-Build-Marlin_JOB-Prepare-Marlin bash`. Explore, figure out, then exit the container and cancel the workflow to stop it.

Obviously this only works if you're using an image that does have `bash` (or a shell), but for this workflow, which uses `ubuntu-latest`, it's perfect.

And that's all. Happy Auto-Auto Building Marlin!

[^ish]: `Starting with Gitea 1.19, Gitea Actions are available as a built-in CI/CD solution.` <https://docs.gitea.com/usage/actions/overview>
[^trp]: <https://www.reddit.com/r/ender3/comments/a9q9jk/for_new_ender_3_owners_a_quick_note_on_thermal/>
[^actually]: The last build took 3m19s. Sure, this is not my first run and I didn't have do redownload most of the Docker images, but still...
