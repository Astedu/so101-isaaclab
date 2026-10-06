# SO-ARM101 Pick & Place with Reinforcement Learning (Isaac Lab)

[![Isaac Sim](https://img.shields.io/badge/IsaacSim-5.1.0-76B900.svg)](https://docs.isaacsim.omniverse.nvidia.com/latest/index.html)
[![Isaac Lab](https://img.shields.io/badge/IsaacLab-2.3.0-8A2BE2.svg)](https://isaac-sim.github.io/IsaacLab/main/index.html)
[![Python](https://img.shields.io/badge/python-3.11-3776AB.svg)](https://www.python.org/)
[![uv](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/uv/main/assets/badge/v0.json)](https://github.com/astral-sh/uv)

Simulation environment and training code for teaching a low-cost **SO-ARM101** robot arm to **pick up a cube and place it in a bowl**, trained with PPO (RSL-RL) in NVIDIA Isaac Lab. Built as part of a university robot learning group project (*Group Project 3: Singulation, Reinforcement Learning*).

> **About this repository:** this is my development branch. It was used for environment design, reward shaping, physics and domain-randomization experiments, so it also contains experiment logs and debugging scripts. It builds on the open-source [`isaac_so_arm101`](https://github.com/MuammerBay/isaac_so_arm101) framework (see [Credits](#credits)).

---

## What I built

The upstream framework provides the robot model plus Reach and Lift tasks. The **pick-and-place task** and everything around it was developed in this project:

- **Custom pick-and-place environment** (`Isaac-SO-ARM101-Pick-Place-*`): cube, bowl, table, wrist camera and contact sensors in a manager-based Isaac Lab environment.
- **Shaped, multi-stage reward** combining reaching, gripper closing, force-aware grasping, lifting, transport to the bowl and a release bonus, plus small action/velocity regularizers (see [Reward design](#reward-design)).
- **Domain randomization** for sim-to-real transfer: object and bowl placement, cube yaw, cube color, lighting, actuator gains, friction and object mass.
- **Two policy variants**: a state-based MLP policy (the one used for the main training runs) and a vision-based policy with a ResNet18 encoder on wrist-camera images.
- **Alternative RL algorithm**: a SAC training script using `skrl`, alongside the default PPO pipeline.
- **Debug tooling**: `check_ee.py` for inspecting end-effector frames and cube alignment in a minimal scene.

## Task overview

| | |
|---|---|
| Robot | SO-ARM101 (6-DoF arm with parallel gripper) |
| Task | Grasp a cube from a randomized position and release it in a bowl |
| Algorithm | PPO via [RSL-RL](https://github.com/leggedrobotics/rsl_rl) (SAC via skrl also included) |
| Parallel envs | 4096 by default (2048 in the example commands below) |
| Control | Joint-position actions (arm + gripper), 10 s episodes, sim dt 0.01 s, decimation 2 |
| Success | Cube resting in the bowl for 7 consecutive steps (episode terminates) |

### Registered environments

| Task ID | Description |
|---|---|
| `Isaac-SO-ARM101-Pick-Place-State-v0` | State-based observations, MLP policy. Used for the main training runs. |
| `Isaac-SO-ARM101-Pick-Place-v0` | Vision-based observations (wrist camera), ResNet18 actor-critic. |
| `Isaac-SO-ARM101-Pick-Place-Play-v0` | Evaluation / playback variant. |

### Observations

- **State policy (actor):** initial cube position in the robot root frame, bowl center position, joint positions, joint velocities, last action.
- **State policy (critic):** same, but with the *live* cube position (asymmetric actor-critic).
- **Vision policy:** wrist camera image, bowl position, joint positions/velocities, last action.

### Reward design

The reward is a weighted sum of terms (defined in `tasks/pick_place/mdp/rewards.py`, weights in `pick_place_env_cfg.py`):

| Stage | Term | Idea |
|---|---|---|
| Reach | `object_ee_distance` | `1 - tanh(d/std)` between gripper and cube |
| Grasp | `gripper_close_when_near` | Reward closing the gripper only when close to the cube |
| Grasp | `object_grasped_contact_continuous` | Bilateral finger contact force, balanced and opposing (via cosine similarity of finger forces) |
| Lift | `lifting_object_grasped` | Lift height ramp gated by a valid grasp |
| Transport | `object_bowl_distance` | Distance to a point above the bowl, gated by minimum height |
| Place | `object_released_in_zone` | Large bonus for releasing the cube over the bowl |
| Regularization | `action_rate`, `joint_vel`, `alive` | Small penalties for smooth, time-efficient motion |

Several additional terms are implemented but currently disabled (commented out in the config) after ablation experiments: gripper aperture and force-tracking rewards, a cube-moved-before-grasp penalty, a table-contact penalty and a time penalty. They are kept for reference.

### Domain randomization

| What | Range / method |
|---|---|
| Cube and bowl placement | Sampled on annular regions in front of the robot with keep-out and camera-occlusion constraints, plus a safe fallback set |
| Cube yaw | Full 360° |
| Cube color | Random from a six-color palette |
| Lighting | Dome and sphere light intensity/color randomization |
| Actuator gains | Stiffness and damping scaled by 0.9 to 1.1 |
| Gripper friction | Static 0.6 to 1.0, dynamic 0.4 to 0.7 |
| Table friction | Static 0.5 to 0.7, dynamic 0.4 to 0.6 |
| Object mass | Scaled by 0.7 to 1.3 |

## Installation

Requirements: Linux x86_64 (or Windows), an NVIDIA GPU with a recent driver, Python 3.11 and [`uv`](https://docs.astral.sh/uv/). Isaac Sim 5.1 / Isaac Lab 2.3.0 are installed through `uv` from the project's `pyproject.toml`.

```bash
# 1. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Clone and install
git clone https://github.com/Astedu/so101-isaaclab.git
cd so101-isaaclab
chmod +x install.sh
./install.sh
```

`install.sh` installs a system dependency (`libglu1-mesa`), pins `setuptools` so that `flatdict` builds, and runs `uv sync` inside `isaac_so_arm101-main/`. It uses `sudo apt-get`, so review it before running on a shared machine.

## Usage

All commands are run from `isaac_so_arm101-main/`.

```bash
cd isaac_so_arm101-main

# List registered environments
uv run list_envs

# Sanity check the environment with dummy agents
uv run zero_agent   --task Isaac-SO-ARM101-Pick-Place-State-v0
uv run random_agent --task Isaac-SO-ARM101-Pick-Place-State-v0

# Train (PPO, headless)
uv run train --task Isaac-SO-ARM101-Pick-Place-State-v0 --num_envs 2048 --headless

# Play back a trained policy and record a video
uv run play --task Isaac-SO-ARM101-Pick-Place-State-v0 --num_envs 16 --video --headless

# Play back a specific checkpoint
uv run play --task Isaac-SO-ARM101-Pick-Place-State-v0 --num_envs 16 --video --headless \
  --checkpoint <path/to/model_XXX.pt>

# Alternative: SAC with skrl
uv run train_sac --task Isaac-SO-ARM101-Pick-Place-State-v0 --num_envs 64
```

Training logs and checkpoints are written to `logs/rsl_rl/pick_place_state/<timestamp>/` (TensorBoard event files, `model_*.pt`, exported `policy.pt` / `policy.onnx`, and the config/git diff used for the run).

```bash
tensorboard --logdir logs/rsl_rl/pick_place_state
```

### Key hyperparameters (state-based PPO)

| Parameter | Value |
|---|---|
| Network | MLP, actor/critic `[256, 128, 64]`, ELU |
| Rollout length | 64 steps per env |
| Iterations | 3000 |
| Learning rate | 1e-4 (adaptive, desired KL 0.01) |
| Discount / GAE | γ = 0.995, λ = 0.95 |
| Clip / entropy | 0.2 / 0.005 |
| Observation normalization | Empirical normalization on |

## Project structure

```
.
├── install.sh                       # environment setup script
├── isaac_so_arm101-main/            # Python package (uv project)
│   ├── pyproject.toml
│   └── src/isaac_so_arm101/
│       ├── robots/                  # SO-ARM100 / SO-ARM101 models and configs
│       ├── bowl/                    # bowl asset and config
│       ├── scripts/                 # train, play, list_envs, dummy agents, SAC (skrl)
│       └── tasks/
│           ├── reach/               # upstream task
│           ├── lift/                # upstream task
│           └── pick_place/          # this project's main contribution
│               ├── pick_place_env_cfg.py   # scene, observations, rewards, events, terminations
│               ├── joint_pos_env_cfg.py    # robot, camera and action setup; State/Vision/Play variants
│               ├── mdp/                    # rewards, observations, events (randomization), terminations
│               ├── networks/               # ResNet18 actor-critic
│               ├── agents/                 # RSL-RL PPO configs
│               └── check_ee.py             # end-effector / alignment debug scene
├── logs/rsl_rl/pick_place_state/    # training runs, checkpoints, exported policies
└── outputs/                         # Hydra run configs
```

## Status and roadmap

- [x] Pick-and-place environment with randomized scene
- [x] Shaped reward with grasp-force and release terms
- [x] State-based PPO training pipeline with exportable policy (`.pt` / `.onnx`)
- [x] Domain randomization (placement, color, lighting, dynamics)
- [ ] Vision-based policy (ResNet18) training and tuning
- [ ] Behavior cloning pre-training followed by RL fine-tuning
- [ ] Color-conditioned cube selection (pick the cube of a requested color)
- [ ] Sim-to-real deployment on the physical SO-ARM101

<!-- TODO: add a results section once you have numbers, e.g.
     success rate over N eval episodes, a training reward curve screenshot,
     and a short GIF of a successful pick and place (from `play --video`). -->

## Team

Group project with Leon Auspurg and Masiar Etemadi. My contributions are the pick-and-place environment, reward shaping, physics and domain-randomization experiments, and the RL training setup in this repository.

<!-- TODO: edit the line above so it matches exactly what you did and what teammates did. -->

## Credits

- [`MuammerBay/isaac_so_arm101`](https://github.com/MuammerBay/isaac_so_arm101) (Muammer Bay, Louis Le Lay): the base framework, robot assets, and Reach/Lift tasks this repository builds on. BSD-3-Clause.
- [NVIDIA Isaac Lab](https://isaac-sim.github.io/IsaacLab/) and [RSL-RL](https://github.com/leggedrobotics/rsl_rl).
- [skrl](https://skrl.readthedocs.io/) for the SAC script.
- Reference material used during planning: [LeRobot](https://github.com/huggingface/lerobot), [NVIDIA Sim-to-Real SO-101 Workshop](https://github.com/isaac-sim/Sim-to-Real-SO-101-Workshop).

## License

BSD-3-Clause, inherited from the upstream project. See `isaac_so_arm101-main/LICENSE`.
