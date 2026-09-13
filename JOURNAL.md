# Build journal

This journal separates retrospective hardware documentation from firmware sessions recorded as the work happens.

**Status at the start of this journal:** CAD and the first PCB design are complete; the mechanical assembly is completed; firmware development/software integration is the next phase. PCB rework, electrical validation, and balancing tests remain open.

The entries below summarize earlier work. Original session dates and tracked durations were not recorded here, so no hours are claimed. These notes do not establish verified time or funding eligibility.

## Retrospective — Mechanical design and assembly

### Work completed

- Completed the CAD for the current reaction-wheel cube.
- Built the 3D-printed frame and assembled the mechanical system.
- The design uses three Nidec 24H reaction-wheel motors.

### Current outcome

The mechanical build is ready to move into electrical integration and firmware development. Final wire routing, strain relief, and powered validation remain part of integration.

### Evidence to add

- CAD source and STEP/STL exports.
- Assembly and motor-mount screenshots.
- Photos of the printed parts and assembled cube.

These files are not yet included in this repository. The existing [project media folder](https://drive.google.com/drive/folders/1Wyz73s2HkP9Nr1Q7drfzrUGMNGJb-3j-?usp=drive_link) is retained as a reference.

**Time:** not recorded; retrospective entry.

## Retrospective — Custom PCB design and integration issues

### Work completed

- Completed the first custom four-layer PCB design and sent it for fabrication.
- Identified issues that need to be addressed during hardware integration.

### Issues and planned actions

| Issue | Planned action | Verification still needed |
|---|---|---|
| An unintended VBAT connection may bypass the protection stage | Isolate the unintended connection and add a bodge wire as required by the actual board connectivity | Confirm isolation, intended power path, and protection behavior |
| STM32 header-row spacing does not match the board | Use wires between the STM32 and PCB | Document the pin mapping; verify continuity, shorts, and mechanical security |
| J7 IMU connector is unavailable | Solder the IMU wiring directly | Verify pin order, voltage compatibility, continuity, and strain relief |

The repairs are planned, not recorded as completed or tested. This journal is not a board-specific cutting or soldering procedure.

### Evidence to add

- KiCad schematic and PCB source files.
- PCB layout and 3D renders.
- Annotated images showing each issue.
- Before/after repair photos and measured verification results.

### Next step

Resolve and verify the hardware integration issues, then proceed through STM32, sensor, and motor bring-up before closed-loop control.

**Time:** not recorded; retrospective entry.

## Upcoming work — Firmware and bring-up

No successful firmware or balancing test is claimed by this entry.

- [ ] Verify PCB rework, power rails, and the as-built pin map.
- [ ] Create and upload the STM32CubeIDE project.
- [ ] Flash the STM32 and establish basic diagnostics.
- [ ] Implement and test motor-safe boot behavior.
- [ ] Bring up motor control and encoders.
- [ ] Read and calibrate the ICM-20948 v2 specified in the project plan.
- [ ] Implement and validate attitude estimation.
- [ ] Add calibration storage and fault handling.
- [ ] Test limited-output feedback with mechanical support.
- [ ] Tune and record the first repeatable balance.

## Template for future sessions

Copy the template below for each real session. Use the actual date and tracked duration; if time was not tracked, say so. Include failures and measurements as well as successful results. Record screen-based work and physical-work evidence as appropriate.

```markdown
## YYYY-MM-DD — Session title

**Time:** actual tracked duration, or "not tracked"
**Evidence:** recording, screenshots, photos, logs, or commit link

### Goal
What I intended to accomplish.

### Work and observations
What I changed, what happened, and what I measured.

### Problems and decisions
What failed, what I investigated, and why I chose the next approach.

### Result
What is now working, with evidence; what is still unverified.

### Next step
The next concrete task.
```
