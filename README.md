# About: MCU STOP Mode & MHU Multicore Demo
For Ensembe E1, E3, E4, E5, E6, E7 or E8
  - RTSS-HE logs are on LP-UART (RX P9_1 and TX P9_2)
  - RTSS-HP logs are on UART4 (RX P12_1 and TX P12_2)

For Ensemble E1C or Balletto B1
  - RTSS-HE logs are on LP-UART (RX P2_0 and TX P7_1)

Changing the UART instance and pinmux is done via the retarget_config.h file.
```
    .
    ├── ...
    ├── device/
    │   ├── ensemble/
    │   |   ├── RTE/
    │   │   │   ├── Services/
    │   │   |   │   ├── AE1C1F4051920PH_M55_HE/retarget_config.h
    │   │   |   │   ├── AE722F80F55D5LS_M55_HE/retarget_config.h
    │   │   |   │   ├── AE722F80F55D5LS_M55_HP/retarget_config.h
    │   │   |   |   ├── AE822FA0E5597LS0_M55_HE/retarget_config.h
    │   │   |   |   └── AE822FA0E5597LS0_M55_HP/retarget_config.h
    │   │   │   └── ...
    │   │   └── ...
    │   └── ...
    └── ...
```

Refer to the sample ATOC data structure at the end of this README. The "deferred" flag instructs the Secure Enclave to skip booting a core during power-on.

At first power-on, only one core, the HE-M55, is booted by the Secure Enclave. The HE core detects that there are no pending wake events. It assumes that the MCU is being powered on for the first time and so it performs the first-time system setup. Those steps are to configure the aiPM Off Profile, initialize UART for application logging, configure the LPTIMER as its wakeup source, and finally it puts the MCU into global STOP Mode. The MCU stays in STOP Mode until the next LPTIMER expiration (configured for 1000 ms by default). Each LPTIMER expiration is a wake event that brings the MCU out of STOP Mode.

In response to the wake event, the HE-M55 core spends some time awake in a while(1) loop for 100 ms. This is just to demonstrate the HE core is awake and busy with some task. When the task is complete then the HE core returns the MCU to STOP Mode. Every tenth LPTIMER wake event the HE-M55 core uses the Message Handling Unit (MHU) to wake up the HP-M55 core. While waiting for a response from the HP core, the HE will enter deep sleep (subsystem stays on).

When the HP core receives a message via the MHU, it will run for some time in a while(1) loop before returning a response to the HE core. Again, this is just to demonstrate the HP core is awake and busy with some task. If the HP does not respond in time the HE core will timeout and reset it like a watchdog.

For Ensemble E1, E1C or Balletto B1, we will only need to build a binary for the RTSS-HE. MCU STOP Mode is demonstrated with only one core.


# Building the binaries
Open the directory using VSCode and switch to the CMSIS View (CTRL+SHIFT+ALT+S).
Click the gear icon labeled "Manage Solution Settings". Here you will choose the
Active Target. If you are planning to run this demo on the Devkit-e7, then you
will want to build binaries for the E7-HE and E7-HP. Otherwise, for the Devkit-e8,
then build binaries for the E8-HE and E8-HP. Use this Manage Solution tab to
switch between Target Types and use the hammer icon in the CMSIS View to build
the application.

After the binaries are built, switch to the Explorer View (CTRL+SHIFT+E).
The binaries will be located in the out directory, for example:
```
.
├── ...
├── out/
│   ├── app_he/
│   |   ├── E1C-HE/
│   |   |   ├── debug/app_he.bin
│   |   |   └── release/app_he.bin
│   |   ├── E7-HE/
│   |   |   ├── debug/app_he.bin
│   |   |   └── release/app_he.bin
│   |   └── E8-HE/
│   |       ├── debug/app_he.bin
│   |       └── release/app_he.bin
│   └── app_hp/
│       ├── E7-HP/
│       |   ├── debug/app_hp.bin
│       |   └── release/app_hp.bin
│       └── E8-HP/
│           ├── debug/app_hp.bin
│           └── release/app_hp.bin
└── ...
```


# Debugging the binaries
- Switch to the Debug and Run view (CTRL+SHIFT+D) and press F5
- **TIP:** You may want to use SE tools integration to install the CPU stubs


# Programming the binaries
Use the below json to configure your ATOC. Copy the binaries to the app-release-exec/build/images folder and proceed with the usual steps of generating the ATOC and writing to MRAM.

```
{
  "DEVICE": {
    "disabled" : false,
    "binary": "app-device-config-no-hfxo.json",
    "version" : "1.0.0",
    "signed": false
  },
  "HE_MRAM": {
    "disabled" : false,
    "binary": "M55_HE_img.bin",
    "mramAddress": "0x80000000",
    "version": "1.0.0",
    "cpu_id": "M55_HE",
    "flags": ["boot"],
    "signed": false
  },
  "HP_MRAM": {
    "disabled" : false,
    "binary": "M55_HP_img.bin",
    "mramAddress": "0x80200000",
    "version": "1.0.0",
    "cpu_id": "M55_HP",
    "flags": ["boot","deferred"],
    "signed": false
  }
}
```


# Power Measurement on Alif DevKit
Refer to the power measurement points described in the [aiPM Examples User Guide](https://github.com/alifsemi/alif_ensemble-vscode-aiPMExamples/blob/main/Documentation/aiPM_Examples.md)


# GitHub Workflow (Beginner Friendly)
This section gives a safe, step-by-step workflow so each behavior change is isolated and easy to review.

## Branch plan for this StopMode fork
- `docs/readme-workflow-and-observations`
- `feature/e8-led-bringup-baseline`
- `feature/e8-led-toggle-on-mhu-wake`
- optional later: `feature/e8-rtt-logging`

## Pull request order
1. PR 1: README updates only
2. PR 2: HP LED ON at cold boot proof only
3. PR 3: HP LED toggle once per MHU wake only
4. PR 4: Optional RTT/logging/power helper updates

Keep each PR focused on one behavior change.

## One-time setup (GitHub Desktop)
1. Install GitHub Desktop: https://desktop.github.com/
2. Sign in with your GitHub account.
3. Clone your StopMode fork:
   - File -> Clone repository
   - Choose your fork
   - Select local path
   - Click Clone
4. Open the cloned folder in VS Code.

## PR 1 walkthrough — GitHub Desktop (README only)
1. Make sure you are on your default branch (usually `main`).
2. Pull latest changes:
   - In GitHub Desktop, click **Fetch origin**
   - If offered, click **Pull origin**
3. Create a new branch:
   - **Current Branch** -> **New Branch**
   - Name: `docs/readme-workflow-and-observations`
   - Base: `main`
   - Click **Create Branch**
4. Edit README.md in VS Code and save.
5. Commit in GitHub Desktop:
   - Review changed file list — check only README.md
   - Summary: `docs: update README with workflow and test guidance`
   - Click **Commit to docs/readme-workflow-and-observations**
6. Push branch:
   - Click **Publish branch** (first push) or **Push origin**
7. Open pull request:
   - Click **Create Pull Request** in GitHub Desktop
   - Verify base is `main`
   - Title: `docs: README workflow and validation guidance`
   - In PR description include:
     - Target board and device
     - Build type used
     - Programming method used
     - Expected vs observed behavior (for docs PR: no functional change)
     - Short validation notes
8. Submit PR.

## PR 1 walkthrough — GitHub website only
1. Open your repository page on github.com.
2. Create branch:
   - Click the branch selector (top left, above file list, says `main`)
   - Type `docs/readme-workflow-and-observations`
   - Click **Create branch: docs/readme-workflow-and-observations from main**
3. Open README.md and click the pencil icon (Edit this file).
4. Make your README changes.
5. Scroll to **Commit changes**:
   - Commit message: `docs: update README with workflow and test guidance`
   - Choose **Commit directly to docs/readme-workflow-and-observations**
   - Click **Commit changes**
6. Click **Compare & pull request**.
7. Fill PR title and description, then click **Create pull request**.

## Repeat for PR 2 and PR 3
For each next PR:
1. Checkout `main` and pull latest.
2. Create the next feature branch from updated `main`.
3. Make only the scoped change for that PR.
4. Commit with a single-purpose message.
5. Push and open PR.
6. Merge and re-test before starting the next branch.

## Suggested commit message style
```
docs: update README with workflow and test guidance
feat(app_hp): set cold-boot LED proof behavior
feat(app_hp): toggle HP LED on MHU wake and persist state
```

## Definition of done for LED behavior PRs
- HP LED turns ON at cold boot during bring-up phase.
- HP LED toggles exactly once per valid HP MHU wake event.
- No unintended toggles during non-MHU wakes.
- Behavior repeats across at least 3 full program-reset runs.
- Release build validates behavior without debugger attached.
