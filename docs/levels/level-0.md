# Level 0 — Walk to the Goal

Train a Fall Guy to walk toward a goal and avoid walls — the first real training run of the project.

## Overview

Level 0 is where things start getting fun: you train an agent that actually adapts to a realistic environment, rather than the sanity-check Test Environment. The Fall Guy can walk around using a movement equivalent to WASD (or a combination of those keys). It sees the world through its direction to the goal and its own facing. The goal is to reach a large green ball and avoid touching the walls, which carry a penalty. The goal spawns at a random angle and distance from the agent each episode, so the agent can't just memorize one fixed path.

This is the first level where you're expected to actually reason about hyperparameters rather than just run a working config — only a curated subset of `Level 0.yaml`'s values are exposed for editing (marked `# EXPERIMENT HERE`), each with a short comment describing what it does.

## Environment mechanics

- **Movement**: WASD-equivalent walking, no jump or dive yet.
- **Goal**: a large green ball the agent must reach. Turn on Gizmos in the Game view (with the Fall Guy selected) to visualise its sensors.
- **Walls**: touching a wall gives a penalty but does not end the episode.
- **Randomized goal placement**: the goal spawns at a random angle and distance from the agent every episode.

## Observation space

| Observation | Description |
|---|---|
| Direction to goal | The agent's sensed direction toward the green ball |
| Own facing | The direction the agent is currently facing |

The Fall Guy doesn't see the scene directly — only what its sensors report, visualised via Gizmos in the Unity Game view.

## Action space

- **Movement** (WASD-equivalent): forward / back / left / right, or combinations.

## Reward structure

| Event | Reward |
|---|---|
| Reaching the goal | +10 |
| Touching a wall | Penalty |
| Every step | Tiny penalty, to encourage speed |

## Config notes (`Level 0.yaml`)

Unlike Levels 1-3, **only a curated subset of hyperparameters is exposed here**, each marked `# EXPERIMENT HERE` with an inline comment explaining what it controls:

- `batch_size: 512` — how many steps to process at once.
- `buffer_size: 2048` — total steps collected before updating weights (must be a multiple of `batch_size`).
- `hidden_units: 64` / `num_layers: 2` — size and depth of the neural network.
- `time_horizon: 32` — how many steps of "memory" are used for rewards.
- `max_steps: 200000` — total simulation steps before training ends. By far the smallest budget of any level — this is meant to be a fast, iterative first training run, not a long one.

Your job is to play with these values and get the agent working. Reading [ML-Agents' training configuration docs](https://unity-technologies.github.io/ml-agents/Training-Configuration-File/) or asking a chatbot to explain a hyperparameter is fine — but pick the values yourself. The point isn't to finish fastest, it's to actually learn how RL training responds to these knobs.

## Training target

**Target Mean Reward: >9.5**

## How to train

1. Open `Levels/Level 0/Level 0 Unity` in Unity.
2. Open the scene: `Assets/Environment.unity`.
3. Learn how the hyperparameters in `Level 0.yaml` work.
4. Edit `Level 0.yaml`, experimenting with the values marked `# EXPERIMENT HERE`.
5. Run the backend:
   ```bash
   mlagents-learn "Levels/Level 0/Level 0.yaml" --run-id=run-1
   ```
6. Click Play in Unity.
7. Observe the mean reward and the agent's behavior to spot potential weaknesses.
8. Repeat steps 3-7 until you get a well-functioning agent and a high mean reward. Change `--run-id` for a fresh run, or use `--resume` to continue one.
9. Copy the trained model from `results` into the Fall Guy folder in Unity Assets.
10. Assign the model in Behaviour Parameters and run in inference without the backend.

## Troubleshooting

- **Agent wanders randomly with no improvement**: double check the reward is actually firing on goal contact and wall contact — a silent reward-wiring issue looks identical to "the agent hasn't learned yet."
- **Reward plateaus below target**: try increasing `buffer_size` or `max_steps` before assuming the environment itself is broken — 200k steps is a small budget, and a slightly undertrained run can look like a stuck one.
- **Agent hugs walls or moves erratically**: revisit `time_horizon` — too short a horizon can make it hard for the agent to connect a wall bump to the resulting penalty a few steps later.
- For general "my agent isn't learning" debugging, see the [TensorBoard guide](../installation/tensorboard.md).

## Screenshots & video

!!! info "Media coming soon"
    This section is a placeholder for gameplay screenshots and/or a short training video for Level 0. Contributions welcome — drop media in `docs/assets/` and link it here.

## See also

- [README — Level 0](https://github.com/AISoc-UNSW/FallGuys#level-0) for the quick-start version of this page.
- [TensorBoard guide](../installation/tensorboard.md)
- [ML Agent Theory](../introduction/ml-agent-theory.md)
