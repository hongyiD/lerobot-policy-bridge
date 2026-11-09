# No-hardware proof of concept behind the bridge design

- Date: 2026-09-27
- Goal: without changing any AgenticX code, use only AgenticX's existing generic tools (`bash_bg_start` / `bash_bg_poll` / `bash_bg_input` / `bash_bg_stop` / `bash_exec` / `view_image`) to drive a `lerobot-rollout --interactive=true` session that loads a **real pretrained policy**.
- Result: **PASS**. The full command chain (launch → `/start` → look at a snapshot → `/subtask` → `/reset` → `/start` → `/stop`, plus a forced stop while running) worked. The runs also surfaced four safety and engineering constraints that a generic shell-tool path cannot satisfy (see [Findings](#findings)). The bridge in this repository exists to meet them.

The PoC scripts lived in a separate research workspace and are not part of this repository. The simulated arm written for the PoC later became `sim/lerobot_robot_agx_sim`.

## Environment

| Item | Value |
|---|---|
| lerobot | 0.6.2, editable install at upstream SHA `e595b79` |
| Python / torch | 3.12.13 / 2.11.0, device `mps` (Apple Silicon) |
| Policy | [`kai-yamada/act_so101_pick_cube`](https://huggingface.co/kai-yamada/act_so101_pick_cube) (ACT, ~52M parameters, `model.safetensors` 206.7 MB, Apache-2.0) |
| Policy I/O | Inputs `observation.state`[6] + `observation.images.front`[3,480,640] + `observation.images.handeye`[3,480,640]; output `action`[6] |

## Simulated robot (PoC scaffolding)

Registered as `--robot.type=agx_sim_so101` following LeRobot's third-party plugin convention (package prefix `lerobot_robot_`, `XxxConfig` plus `Xxx` in the same module):

- Six SO-101 joints `<joint>.pos` and two 480×640 cameras `front` / `handeye`, matching the policy features one to one.
- Joints follow their targets with first-order dynamics in a separate 100 Hz servo thread, imitating real servos moving on their own. `send_action` applies joint limits and the per-step `max_relative_target` clamp, with the same semantics as the real SO follower.
- The cameras reproduce the concurrency structure of `OpenCVCamera`: a background frame-reader thread, `frame_lock` and `read_latest(max_age_ms)`. Frames show the joint readings and a simple arm sketch.
- A separate snapshot thread calls `read_latest` every second and writes `snapshot_front.png` (atomic replace) and `status.json`, standing in for a concurrent reader like the bridge's `/snapshot`.
- With `--robot.calibrated=false`, `calibrate()` blocks in `input()`, just like the real `SOFollower.calibrate()`.

## What was run

### 1. Command-line pipe

`printf` wrote `/help → /start → /subtask → /reset → /start → /stop` to stdin in sequence; exit code 0.

- About 3 s from launch to a ready session (1 s policy load, plus connection and inference engine).
- First `/start` segment: 18.5 Hz, dragged down by a 4.8 s MPS warm-up on the first inference. Second segment: 28.5 Hz (target 30 Hz), mean inference 8.3 ms.
- `/subtask` echoed `Task: 'pick up the cube' → 'place the cube in the bowl'`; `/reset` echoed `Robot reset — holding at initial position`.
- Note: ACT does not read the language instruction, so `/subtask` only verified the command path here, not a change in policy behavior. That needs a VLA such as SmolVLA or π0.

### 2. Driven through AgenticX tools (no AgenticX code changes)

The driver script only called AgenticX's tool dispatch entry point (`dispatch_tool_async`), with the same tool names and arguments the Meta-Agent LLM would emit. The confirmation gate recorded every confirmation request and approved only `bash_bg_start` (simulating the user clicking "Allow" once); everything else was refused.

Scenario A (full command chain, stdin kept open):

| Check | Result |
|---|---|
| `bash_bg_start` launches the rollout; `bash_bg_poll` sees the ready prompt | ✅ |
| `bash_bg_input "/start"` → `Rollout running` | ✅ |
| `view_image(snapshot_front.png)` attaches the image to the next turn | ✅ (pending=1) |
| `bash_exec cat status.json` reads joints and counters | ✅ |
| `/subtask` takes effect | ✅ |
| `/reset` returns to the initial pose | ✅ |
| `/start` again | ✅ |
| Session ends on `/stop` alone, with stdin still open | ✅ (exited in 4.0 s, exit 0, including the return to home) |
| Max deviation of the final pose from the initial pose | 0.023 |
| Concurrent `read_latest` from the snapshot thread | 18 succeeded / 0 errors |

Scenario B (forced stop with `bash_bg_stop`, i.e. SIGTERM, after 131 control cycles):

| Check | Result |
|---|---|
| SIGTERM triggers a graceful shutdown (stop inference → return home → disconnect) | ✅ |
| `bash_bg_stop` duration | 3.86 s (its SIGKILL fallback threshold is 5 s) |
| Max deviation of the final pose from the initial pose | 0.008 |
| Concurrent snapshots | 13 succeeded / 0 errors |

Confirmation log: two requests, both from `bash_bg_start`, with `risk=non_whitelisted` and the message "lerobot-rollout is not in the known read-only command set; run it anyway?". `bash_bg_input` (including the `/start` that sets the robot moving), `bash_bg_poll`, `bash_bg_stop` and `view_image` never triggered a confirmation.

### 3. Questions checked

1. **Does a calibrated robot trigger `input()`?** Calibrated: no (all runs above). Uncalibrated (`--robot.calibrated=false`): `robot.connect()` (`rollout/context.py:394`, `calibrate=True` by default) enters `calibrate()`, and **the first `/start` written to the pipe is swallowed by `input()`** (log: `calibrate() consumed stdin line: '/start'`). The session never reaches the running state, and the connection step blocks until a line arrives on stdin.
2. **Is `read_latest` safe alongside the control loop?** Statically, `OpenCVCamera.read_latest` (`cameras/opencv/camera_opencv.py:585-615`) takes the frame under `frame_lock` without blocking. At runtime, 44 concurrent snapshots over three runs on the simulated camera with the same structure gave 0 errors. Real USB cameras were not tested.

LeRobot file and line references are at upstream SHA `e595b79`.

## Findings

1. **F-1: calibration swallows stdin (high).** An uncalibrated real robot consumes the agent's first command as the calibration confirmation. The bridge must run a calibration preflight before launch (or `connect(calibrate=False)` plus an `is_calibrated` check), and fail closed when it sees a prompt like `Press ENTER to use provided calibration file`. The zero-code `bash_bg_input` path cannot do this.
   - In the bridge: the arm is connected once without calibration before loading and fails as `calibration_required`; `input()` raises and stdin is `/dev/null`, so any prompt fails as `stdin_input_blocked`.
2. **F-2: "allowed to launch" is not "allowed to move" (high).** AgenticX's confirmation gate stops only once, at process launch, with `non_whitelisted`; afterwards `/start`, `/subtask` and `/reset` run with no confirmation. (Corrected on re-check the same day: in full-auto mode AgenticX uses `RiskAwareAutoConfirmGate`, and `non_whitelisted` is a protected risk, so the launch **still** prompts.) The launch confirmation itself has two bypasses: the confirmation card lets the user choose "allow all" for `non_whitelisted`, and the confirmation check consults `permissions.allowed_tools` first (matched by tool name with `fnmatch`). Robot motion commands therefore need their own `risk=robot` that bypasses both shortcuts, rather than borrowing bash's risk classification.
   - Addressed on the AgenticX side in [PR #62](https://github.com/DemonDamon/AgenticX/pull/62): robot tools use `risk=robot`, and those confirmations are non-waivable.
3. **F-3: little margin on forced stop (medium).** LeRobot returns home gracefully on SIGTERM, but the return is a hard-coded 3 s, 50 Hz open-loop interpolation (`rollout/strategies/core.py:228`). Together with stopping inference, it measured 3.75–3.86 s, only about 1 s short of `bash_bg_stop`'s 5 s SIGKILL. On a real robot with slower inference or a longer return, the process could be killed mid-return and leave the robot in an arbitrary pose. The fix: send `/stop` first and wait for the return, with SIGTERM/SIGKILL only as a configurable fallback. The interpolation also **does not read back** whether the robot arrived (it returns `True` once the commands are sent), so on real hardware the bridge should read the pose back and verify it.
   - In the bridge: stop halts the control loop, returns home, reads the joint pose back and reports `pose_check` against `home_tolerance`; SIGTERM stops the session before exit, and the documented kill grace period is at least 15 s.
4. **F-4: offline deployment needs the backbone download off (medium).** When ACT is constructed, it first downloads ResNet18 ImageNet weights from `download.pytorch.org` (later overwritten by the checkpoint); with restricted network access it fails at startup with `URLError`. `--policy.pretrained_backbone_weights=null` makes it fully offline. A bridge for enterprise or intranet use should apply such overrides by default.
   - In the bridge: `offline_backbone` defaults to `true`.

Other observations:

- The `/stop` handler prints no echo, so in pipe mode it is hard to tell apart from EOF. Scenario A verified separately that `/stop` alone ends the session while stdin stays open.
- The first inference has a ~4.8 s MPS warm-up, so "motion has started" cannot be judged from the `Rollout running` text alone; use the control-loop counter or pose changes.
- `bash_bg_poll` returns plain-text logs that the agent must parse with regexes, so a structured robot status is still needed.

## Out of scope

- Real hardware (SO-101 etc.), real USB cameras and the real calibration flow.
- Language-conditioned VLA policies (the semantic effect of `/subtask`), `/vqa` and `/autosteer` (these need policies with a text head, such as EO1 or Wall-X).
- End-to-end driving through the desktop app or an LLM conversation (the PoC called the tool dispatch entry point directly, which is equivalent to "the LLM has decided to call these tools").
- Any change to AgenticX production code.
