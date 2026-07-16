# Level 1 — Spinner Platform

Survive on a platform swept by rotating spinner arms above the slime.

## Overview

Level 1 keeps the goal-reaching mechanic from Level 0 out of the picture entirely and replaces it with a survival problem. The Fall Guy stands on a platform suspended above slime and can now **jump** (Space), in addition to walking. Two spinner arms rotate across the platform, trying to knock the agent into the slime below. There is no "reach the goal" signal here — the only objective is to stay alive.

This is a meaningfully different kind of RL problem from Level 0: instead of learning to navigate *toward* something, the agent has to learn to continuously *avoid* a moving threat, which means the policy has to react to a constantly-changing observation rather than converge on a single approach path.

## Environment mechanics

- **Platform + slime**: touching the slime ends the episode immediately (fall = death, no recovery).
- **Spinner arms**: two independent rotating arms sweep across the platform. Getting hit by one usually knocks the agent toward the edge or directly into the slime.
- **Jump**: the agent's only new action this level. Timing a jump to clear a spinner sweep is the core skill being learned.
- **Spawn randomization**: the agent's start position on the platform is slightly randomized each episode, so it can't memorize one fixed safe path.

## Observation space

| Observation | Description |
|---|---|
| Spinner 1 angle | Current rotation angle of the first spinner arm |
| Spinner 1 angular velocity | How fast spinner 1 is currently rotating |
| Spinner 2 angle | Current rotation angle of the second spinner arm |
| Spinner 2 angular velocity | How fast spinner 2 is currently rotating |
| Grounded state | Whether the agent is currently touching the platform |
| Polar position | The agent's own position on the platform, expressed in polar coordinates (relative to the platform's center) |

Polar coordinates make sense here because the spinners themselves sweep in a circular pattern — the agent's angular position relative to a spinner is a much more directly useful signal than raw x/z coordinates.

## Action space

Inherited from Level 0 (movement), plus:

- **Jump** (binary) — the only new action this level introduces.

## Reward structure

| Event | Reward |
|---|---|
| Every step survived | Small positive reward |
| Falling into the slime | Large negative penalty, episode ends |

There's no shaping reward pulling the agent toward any particular position — survival alone is the signal. This is intentionally sparse-ish: the agent has to discover on its own that dodging/jumping over spinners is what keeps the per-step reward flowing.

## Config notes (`Level 1.yaml`)

Unlike Level 0, **the full hyperparameter set is exposed** here (no `# EXPERIMENT HERE` markers) — this is the first level where you're expected to reason about the whole config, not just a curated subset.

- `time_horizon: 64` — short relative to Level 2/3, appropriate since a single "avoid the next spinner sweep" decision doesn't need a very long credit-assignment window.
- `max_steps: 6000000` — noticeably more than Level 0's 200k. Expect training to take meaningfully longer; this is a genuinely harder, longer-horizon problem than Level 0.
- `gamma: 0.995` — high discount factor, appropriate for an ongoing survival task where future steps matter almost as much as the current one.

## Training target

A consistently high, positive mean reward — meaning the agent survives most of each episode rather than falling into the slime early. There's no single hard numeric bar the way Level 0 has `>9.5`; watch the reward curve's trend and whether episode length is increasing over time.

## How to train

1. Open `Levels/Level 1/Level 1 Unity` in Unity.
2. Open the scene: `Assets/Environment.unity`.
3. Review the hyperparameters in `Level 1.yaml`.
4. Edit `Level 1.yaml` and experiment with the values.
5. Run the backend:
   ```bash
   mlagents-learn "Levels/Level 1/Level 1.yaml" --run-id=run-1
   ```
6. Click Play in Unity.
7. Watch the mean reward and episode length in TensorBoard, and observe the agent's behavior for weaknesses.
8. Repeat steps 3–7 until the agent reliably survives. Change `--run-id` for a fresh run, or use `--resume` to continue one.
9. Copy the trained model from `results` into the Fall Guy folder in Unity Assets.
10. Assign the model in Behaviour Parameters and run in inference without the backend.

## Troubleshooting

- **Agent doesn't move at all / stays frozen**: check that the Decision Requester and Behaviour Parameters components are correctly wired on the Fall Guy — a missing/misconfigured component is the most common reason for zero learning signal.
- **Reward curve is flat near zero**: the agent may not be discovering that jumping avoids spinner hits early enough. Consider whether `time_horizon` and `buffer_size` give it enough experience per update to stumble into a useful jump.
- **Agent survives briefly then plateaus**: often means it's learned to avoid one spinner but not react to the second — watch the two spinner angle observations against the agent's actual movement to sanity-check it's using both signals.
- For general "my agent isn't learning" debugging (reward not increasing, high variance, etc.), see the [TensorBoard guide](../installation/tensorboard.md).

## See also

- [README — Level 1](https://github.com/AISoc-UNSW/FallGuys#level-1) for the quick-start version of this page.
- [TensorBoard guide](../installation/tensorboard.md)
- [ML Agent Theory](../introduction/ml-agent-theory.md)
