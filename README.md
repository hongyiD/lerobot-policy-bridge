# lerobot-policy-bridge

A localhost HTTP service that hosts one [LeRobot](https://github.com/huggingface/lerobot) policy rollout session and exposes it through task-level commands: load a policy, connect a robot, start / change task / reset / stop, and take camera snapshots. An LLM agent can direct the robot through this API without ever sending joint actions; the real-time control loop runs entirely inside the bridge process.

It ships with a simulated SO-101 follower arm, so the whole chain runs end to end with a real pretrained policy and no hardware.

This is the robot-side half of the LeRobot integration for [AgenticX](https://github.com/DemonDamon/AgenticX), submitted upstream in [AgenticX PR #62](https://github.com/DemonDamon/AgenticX/pull/62). The agent-side half (robot tools, non-waivable `risk=robot` confirmations, Meta-Agent prompt) lives in that PR. This repository holds the bridge on its own, with its commit history. The Python package is named `agx-robot-bridge` and its default paths live under `~/.agenticx/`, matching the upstream integration; the bridge itself does not import or depend on AgenticX.

```
LLM agent ──task-level HTTP──▶ agx-robot-bridge ──▶ LeRobot RolloutController ──▶ robot
           (localhost only,     (session state machine,  (real-time control loop,     (SO-101 or the
            bearer token)        safety checks)            in-process)                  simulated arm)
```

## Design

The agent should direct a robot at the task level while the real-time control loop stays in a separate process. The bridge is that process boundary:

- **Local only, authenticated.** Every route, including `/health`, requires a bearer token read from a 0600 token file. The server binds to loopback only, and the unauthenticated API docs are off.
- **One session, bounded motion.** Only one hardware-owning session exists at a time, and creating a session requires a per-step motion limit (`robot.max_relative_target`). `robot.extra` cannot override the robot type, port, id, cameras or motion limit, and keys that could inject extra rollout arguments are rejected.
- **Fail closed on calibration.** Before loading, the arm is connected once without calibration; an uncalibrated arm fails as `calibration_required`, with the exact `lerobot-calibrate` command as the hint. Any interactive input request inside the process fails fast instead of blocking or swallowing the agent's commands.
- **Stop always releases the robot.** Stop works from every state, is idempotent and waits for an in-flight load. It halts the control loop, returns the arm home, reads the joint pose back and reports whether it is within tolerance, then disconnects the hardware. SIGTERM stops the active session before the process exits.
- **Works offline.** The vision backbone's ImageNet init download is skipped by default; trained weights still come from the checkpoint.

These constraints come from a no-hardware proof of concept that drove `lerobot-rollout --interactive` directly through an agent's generic shell tools; see [docs/poc.md](docs/poc.md).

## Install

```bash
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -e '.[test]'            # fake backend only
uv pip install -e '.[lerobot,test]'    # real robot
```

- `--backend fake`: a pure-Python simulated backend that needs neither torch nor hardware, for tests and integration work.
- `--backend lerobot`: the real backend, wrapping LeRobot's `RolloutController` (requires the `lerobot` extra).

The `lerobot` extra is pinned to an upstream git revision (see `pyproject.toml`) because `RolloutController` is not in any published release yet. `serve --backend lerobot` checks for that API at startup; if it is missing, it prints the reason and exits with code 2.

### Simulated robot (no hardware)

`sim/lerobot_robot_agx_sim` is a simulated SO-101 follower plugin (robot type `agx_sim_so101`). Its joint and camera features (six `<joint>.pos`, `front` / `handeye` at 480×640) match SO-101 ACT policies, so it can run a real policy directly:

```bash
uv pip install -e sim/lerobot_robot_agx_sim   # into the same environment that has the lerobot extra
```

The plugin does not declare a lerobot dependency, so it never replaces the pinned revision above. Pass `{"calibrated": false}` in `robot.extra` to simulate an uncalibrated robot, or `{"camera_names": [...]}` to change the camera set.

## Run

```bash
agx-robot-bridge serve --backend fake          # default 127.0.0.1:8766
agx-robot-bridge serve --backend lerobot --port 8766
```

| Option | Default | Notes |
|---|---|---|
| `--host` | `127.0.0.1` | Only `127.0.0.1` / `localhost` / `::1` are accepted; anything else exits with code 2 |
| `--port` | `8766` | |
| `--backend` | `lerobot` | `lerobot` or `fake` |
| `--token-file` | `~/.agenticx/robot_bridge.token` | Generated with mode 0600 if missing or empty |
| `--snapshot-dir` | `~/.agenticx/robot/snapshots` | Where snapshot PNGs are written |

Logs go to stderr and to `~/.agenticx/logs/robot_bridge/bridge.log` (including access logs, never the token).

At startup the process replaces `input()` with a function that raises immediately and points stdin at `/dev/null`. Any library code that asks for interactive input (typically the calibration prompt of an uncalibrated robot) makes the session fail with `stdin_input_blocked` instead of hanging or consuming input. Calibrate beforehand in a terminal with `lerobot-calibrate`.

On SIGTERM or Ctrl+C the bridge first stops the current session (stop inference, return home, verify the pose, disconnect the hardware) and then exits. A supervisor should allow a kill grace period of at least 15 seconds.

## HTTP API

Every endpoint, including `/health`, requires `Authorization: Bearer <token>`. Errors always have the shape `{"ok": false, "error_code", "error", "hint"}`. Only one hardware-owning session is allowed at a time.

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Version, backend name, Python and lerobot versions |
| POST | `/session` | Create a session and load it in the background; returns 202 with `session_id`. `robot.max_relative_target` is required, see below |
| GET | `/session/{sid}?since=<seq>` | State, events with `seq > since`, `pose_check`, error details |
| POST | `/session/{sid}/start` | Start a run (only from `idle`) |
| POST | `/session/{sid}/task` | `{"task": "..."}`: change the task; takes effect at the next inference |
| POST | `/session/{sid}/reset` | Stop the current motion, return to the initial pose and restore the initial task; the return-to-home check follows after about 0.5 s |
| POST | `/session/{sid}/stop` | Callable from any state, idempotent; returns 202 `stopping` while loading |
| GET | `/session/{sid}/snapshot?camera=<name>` | Base64 PNG (width ≤ 640) and the path it was written to |

Session states: `loading` → `idle` ⇄ `running` / `resetting`; failures end in `failed`; stopping goes `stopping` → `stopped`.

### Creating a session (`POST /session`)

```json
{
  "robot": {"type": "so101_follower", "port": "/dev/tty.usbmodem1", "id": "arm1",
            "max_relative_target": 10.0, "cameras": {}, "extra": {}},
  "policy_path": "/path/to/pretrained_model",
  "task": "pick up the cube",
  "fps": 30, "duration_s": 0, "device": "cpu", "home_tolerance": 5.0, "offline_backbone": true
}
```

- `robot.extra` passes through other robot config options (for example `{"use_degrees": true}`); object and array values are passed as JSON. It cannot override `type` / `port` / `id` / `cameras` / `max_relative_target`.
- `offline_backbone` (default `true`) skips the ImageNet init download of the vision backbone. Trained weights still load from the checkpoint, so this works offline.
- Before loading, the robot is connected once without calibration. An uncalibrated robot fails immediately as `calibration_required`, and `hint` gives the matching `lerobot-calibrate` command. If any load step fails, the robot is disconnected if it was already connected.
- After `stop` and `reset`, the return to home reads the joint pose back and compares it with the initial pose (`pose_check`). An error above `home_tolerance` (same units as the joint readings) gives `verified: false`.
- With CPU inference the control loop often runs below the target fps; the "Control loop is running slower" warning in the log is expected.

## Tests

```bash
python -m pytest tests -q
```

The package carries its own pytest config. Without lerobot installed, the real-backend tests are skipped and the rest need neither torch nor hardware: 45 passed, 5 skipped.

Real-backend integration tests (a real policy on the simulated robot; needs the `lerobot` extra and the simulated arm plugin):

```bash
AGX_ROBOT_BRIDGE_POLICY_PATH=/path/to/act_so101_checkpoint AGX_ROBOT_BRIDGE_DEVICE=cpu \
  python -m pytest tests/test_lerobot_backend_sim.py -v
```

With the ACT checkpoint [`kai-yamada/act_so101_pick_cube`](https://huggingface.co/kai-yamada/act_so101_pick_cube) on CPU, the full suite gives 50 passed. ACT does not read the language instruction, so these runs exercise the `/task` command path, not a change in policy behavior; that needs a language-conditioned VLA policy.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). LeRobot is an optional runtime dependency and is not vendored.
