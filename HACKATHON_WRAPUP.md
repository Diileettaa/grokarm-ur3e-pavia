# Hackathon wrap-up — where GrokArm UR3e stopped

*Grok Hackathon at the Museum, ctrlalt.museum, Pavia, 4–5 November 2026 — Track 04 (Voice Control).*
*Status written on 6 November 2026 for commit `09c1e15` (last commit of the event, 5 Nov 19:08 CET).*

> **Draft for review.** This page summarises the state of the code at the end of the event. Statements are taken from
> the repository (README, `PROJECT_STATUS.md`, `docs/`, commit messages). Nothing here was re-run for this write-up.

## What the project is

Spoken or typed commands control a Universal Robots UR3e arm. Seven languages are supported, Italian first. Grok Voice
understands the request and **proposes** one of a fixed set of skills. A deterministic backend gate validates the
proposal. The arm moves only after the operator says an explicit "yes". STOP works from a button, the space bar or a
spoken keyword, and none of these paths waits for the model.

In the second half of the event the team added a top-down stereo camera, coloured paper-cup detection and a passive
3D-printed half-ring tool. The goal was to slide the ring around a cup and lift it.

## Demo video

[GrokArm_Track04_Pavia.mp4](docs/GrokArm_Track04_Pavia.mp4) (2:33, English narration). It was made for the jury on
5 November 2026 and shows the voice loop, live commands on the real UR3e with the original audio, the safety design,
the measurements and the stereo-depth view.

Team: Wu Xinran · Zakarya Mesrar · Diletta · Niccolò Gherlinzoni.

## Status at the end of the event

| Part | State | Evidence |
| --- | --- | --- |
| Voice / text control, safety gate, five named-pose skills | Tested end to end on URSim (synthesised speech, browser, Grok Voice) | `PROJECT_STATUS.md` Phase 6 |
| Rules vs Grok baseline (81 Italian cases, 4 Nov) | Rules 76/81, Grok text mode 78/81; the safety metrics are 100% for both because the gate enforces them | README §17 |
| STOP latency (URSim in Docker, emulated) | UI button → arm halted: median 228 ms; spoken "Fermati!" end of speech → STOP: median 857 ms | README §19 |
| STOP on the real UR3e | One logged run: STOP acknowledged by the robot in 23.9 ms, arm halted in 416 ms (n = 1) | session log shown in the demo video |
| Spoken commands on the real UR3e | «Girati di 45 gradi verso destra» (turn the base 45°) and «destra sopra» (Grok offers the poses, the operator picks one) executed after spoken confirmation | demo video |
| Connection to the real arm | Robot model and PolyScope version read from the dashboard; the beam direction of the gripper was measured on the arm | commits `161b06e`, `b595454` |
| Cup detection from the top camera | Works for coloured cups. Colouring the white inside of each cup made detection much more reliable. Several known errors remain, listed below. | `backend/app/vision/` |
| Stereo depth (ELP USB stereo camera) | Calibrated on site with a checkerboard shown on a laptop screen; gives the table plane (RANSAC) and cup height | `backend/stereo_depth/`, `docs/STEREO_DEPTH.md` |
| Pointing at a cup on the real arm | Reliable in our tests without the gripper. With the printed gripper it often works, but it fails when the planned motion brings the gripper into the arm itself. | team observation |
| Collision checking for the printed gripper | The guard uses UR's official UR3e collision meshes + FCL for the gripper vs the arm, the table and (from `09c1e15`) the arm vs itself. In an offline comparison on 3,000 random poses, the earlier capsule model had marked 198 of 979 gripper–arm collisions as safe. | `backend/app/skills/mesh_model.py`, commit `3437a74` |
| Cup grasp (`grab_cup`) | Planned and guarded, but it **never succeeded on the real arm** during the event | `backend/app/skills/grasp.py` |
| Cup release (`place_cup`) | Implemented, **not tested** | — |
| Side camera to confirm a grasp | Designed as a separate patch, **not merged** | — |

## Known limitations

- **Cup position.** Seen from above, the inside of a paper cup is white, so detection picks up only a thin band of
  colour on the wall. The position is taken from that band, which can shift it by 1–2 cm. The ring has only a few
  millimetres of clearance around the cup base.
- **Parallax.** The camera maps image points to the table plane. Points above the table, such as the ring at about
  11 cm, appear shifted. This was observed roughly and has not been measured.
- **Calibration.** The four tape corners in `data/plane.json` were recorded by touching them. They were not checked
  against an independent fifth point.
- **No grasp sensing.** The tool is passive and has no force feedback. Nothing in the software confirms that a cup is
  actually held.
- **Tests.** Three older tests failed on the team machine at the end of the event:
  `test_real_mode_requires_explicit_config`, `test_real_mode_refuses_unverified_poses` and
  `test_execution_timeout_is_fault`.
- **Measurements.** Apart from the single logged real-arm STOP (n = 1), all latency figures come from URSim under
  emulation. They are not real-robot statistics.
- **Safety.** The software STOP is a convenience layer. It does not replace the robot's physical E-stop, which lab
  staff kept under their control during every real-arm session.
- **Self-collision with the gripper.** With the printed gripper fitted, some planned motions still bring it into the arm. The mesh-based self-collision check added at the end of the event has not been verified on the real arm.

## Possible next steps

1. Estimate the cup centre from the rim ellipse or from stereo depth, instead of from the bottom of the colour band.
2. Measure the camera error and parallax against 3–5 points touched with the tool tip, at table height and at
   10 cm above it.
3. Verify the grasp on the real arm without a cup, then with a cup, at reduced speed.
4. Fix or update the three failing tests.

## Credits

This repository merges the work of three team versions, as listed in the README:
[Voice-Control-Robot (VOCE)](https://github.com/MulatYazew/Voice-Control-Robot),
[voice-arm](https://github.com/strinconegianluca/voice-arm) and
[grokarm-ur3e](https://github.com/13032585789-spec/grokarm-ur3e). Much of the code was written with AI coding
assistants (Cursor, Claude), as the commit authors show. The UR3e collision meshes come from
Universal Robots' `Universal_Robots_ROS2_Description` (BSD-3-Clause, licence in `config/meshes/ur3e/`). The project code
is under the Apache-2.0 licence in `LICENSE`.

Thanks to comPVter and ctrlalt.museum for hosting the event and giving us hands-on access to the arm. Thanks also to the
sponsor Scotti, whose rice boxes, stacked three high, served as our overhead camera mount.
