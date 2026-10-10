# SO-101 Pick-and-Place in Isaac Lab (experimental)

Individual development branch from an ETH Zürich team project (reinforcement
learning for robotic pick-and-place with the SO-101 arm). It adds a
cube-to-bowl pick-and-place task to the SO-101 Isaac Lab framework and was
used to experiment with RL training in simulation.

## Status
Experimental development code. Training runs, and a state-based PPO policy
learned to pick up the cube in simulation. A policy for the complete
pick-and-place task was **not** trained to completion or evaluated (no success
rates), and the code was not re-tested after later experiments, so parts may
need fixes. The vision variant (see below) is untested.

## What I added
On top of the upstream framework (see Attribution):

- **Task `pick_place`** (Isaac Lab manager-based RL environment): the SO-101
  picks a 2 cm cube and places it in a bowl.
  - **Rewards:** reaching, gripper closing near the cube, bilateral
    contact-force grasp reward, lifting, cube-to-bowl distance, release in bowl,
    action-rate and joint-velocity penalties.
  - **Terminations:** timeout, cube out of bounds, cube placed in bowl.
  - **Observations:** joint state, last action, bowl position, cube position.
    The actor sees only the *initial* cube position (as in a real deployment),
    while the critic gets the live cube position (asymmetric actor-critic).
  - **Domain randomization:** cube/bowl placement (annular ring sampling with
    an occlusion-cone constraint), cube yaw and color, actuator gains, gripper
    and table friction, cube mass, lighting.
- **Bowl asset** (`rl_bowl.stl` + URDF + Isaac Lab config).
- **Training configs** for RSL-RL PPO (state-based) and an experimental
  **vision variant**: ResNet18 encoder on a wrist-camera image fused with
  joint state.
- `check_ee.py`: small debug script to check end-effector frame alignment
  against the wrist camera view.

## Environments
| Gym ID | Description |
|---|---|
| `Isaac-SO-ARM101-Pick-Place-State-v0` | State-based observations (the variant used for the cube pick-up experiments) |
| `Isaac-SO-ARM101-Pick-Place-v0` | Vision-based policy (wrist camera + ResNet18), untested |
| `Isaac-SO-ARM101-Pick-Place-Play-v0` | Small-scale environment for visualization |

## Setup and training
Installation follows the upstream project (`install.sh`, Isaac Sim / Isaac Lab
required). [Add your Isaac Lab version and the exact train command you used.]

## Attribution
- Based on [isaac_so_arm101](https://github.com/MuammerBay/isaac_so_arm101) by
  Muammer Bay (LycheeAI) and Louis Le Lay, BSD-3-Clause. The original license
  is kept in `isaac_so_arm101-main/LICENSE`.
- Built on [Isaac Lab](https://github.com/isaac-sim/IsaacLab) and RSL-RL.
- This repository contains my individual work from a team project; ideas and
  work of my teammates are not included.
