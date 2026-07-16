# Level 2 — Full Obstacle Course

Run a complete start-to-finish obstacle course, the full "Fall Guys" experience.

## Overview

Level 2 combines everything from the earlier levels — walking, jumping — and adds **diving** (hold Ctrl), then puts it all inside a long course rather than a single static arena. The course is packed with small and large rotating spinners, jump gaps, and death zones. This is the hardest and longest environment of the three: it's a long-horizon credit-assignment problem where the agent has to learn a sequence of different skills (timing a jump here, diving under an obstacle there) and chain them together over the whole course.

## Environment mechanics

- **Course layout**: a single continuous path from a start point to a finish line, with obstacles distributed along the way — small spinners, large spinners, jump gaps, and death zones that instantly end the episode on contact.
- **Diving**: new action this level (Ctrl), used to pass under or through certain obstacles that jumping alone can't clear.
- **Checkpoints**: the course is conceptually divided into segments (past the small spinners, over the jump, past each big spinner, onto the final platform) — each cleared once, in order.
- **`randomizeSpawn`**: a configurable option on the agent that lets it spawn partway through the course instead of always at the start. Enabling this during training helps the agent get exposure to later obstacles without first having to solve everything before them from scratch every episode — useful for exploration once the early segments are already learned.

## Observation space

| Observation | Description |
|---|---|
| Course progress | How far along the start→finish axis the agent has traveled |
| Lateral offset | Distance from the ideal line down the course |
| Velocity | Current agent velocity |
| Height | Current agent height (relevant for jump/dive state) |
| Jump availability | Whether a jump is currently usable |
| Dive availability | Whether a dive is currently usable |
| Next spinner position | Position of the next spinner obstacle ahead |
| Next spinner rotation | Rotation state of the next spinner obstacle ahead |

Only the *next* upcoming spinner is observed, not the whole course — this keeps the observation space a fixed size regardless of course length and forces the agent to react to what's immediately relevant rather than needing global course knowledge.

## Action space

Movement + jump (from Level 0/1), plus:

- **Dive** (binary) — new this level, holding Ctrl.

## Reward structure

| Event | Reward |
|---|---|
| Forward progress along the course | Positive, proportional to distance gained |
| Backward movement | Negative — discourages retreating |
| Clearing a checkpoint (past small spinners, over the jump, past each big spinner, final platform) | One-time bonus per checkpoint |
| Reaching the finish line | Bonus |
| Entering a death zone | Penalty, episode ends |
| Jumping | Small cost |
| Diving | Small cost |

The progress-based shaping reward is the main signal, with checkpoint bonuses layered on top to make sure the agent gets credit for real milestones even if raw forward-progress reward is noisy near an obstacle. The small jump/dive costs exist purely to discourage the agent from spamming those actions when they aren't needed.

## Config notes (`Level 2.yaml`)

The **full hyperparameter set is exposed** — treat this as the level where everything learned tuning Level 0/1 gets applied at once.

- `time_horizon: 512` — much longer than Level 1's 64. Needed because the reward signal for a decision made early in the course (e.g. positioning before a jump) may not pay off until several actions later.
- `max_steps: 20000000` — by far the largest step budget of the three levels; expect training runs measured in millions of steps and correspondingly longer wall-clock time.
- `gamma: 0.995` — same discount factor as Level 1, appropriate given the long-horizon dependencies in this course.
- `batch_size: 1024` / `buffer_size: 20480`, `hidden_units: 256` — bigger network and batch than Level 1, reflecting the larger observation space and harder policy this level requires.

Watch out for **reward hacking on the progress signal**: because forward progress alone is rewarded continuously, an under-trained agent can get stuck oscillating near the first obstacle it can't clear, "farming" small forward/backward reward without actually learning to pass it. If you see reward plateau early with high episode counts, check whether the agent is actually reaching later checkpoints or just oscillating near the start.

## Training target

As high as possible — ideally the agent consistently clears the later checkpoints and reaches the finish line. Watch the `Environment/MaxProgress` stat in TensorBoard specifically: it's a much more informative signal than raw mean reward for tracking whether the agent is actually getting further down the course over time.

## How to train

1. Open `Levels/Level 2/Level 2 Unity` in Unity.
2. Open the scene: `Assets/Environment.unity`.
3. Review and tune the hyperparameters in `Level 2.yaml`.
4. Run the backend:
   ```bash
   mlagents-learn "Levels/Level 2/Level 2.yaml" --run-id=run-1
   ```
5. Click Play in Unity.
6. Watch mean reward and `Environment/MaxProgress` in TensorBoard to see how far the agent is getting.
7. Repeat steps 3–6, iterating on the config. Change `--run-id` for a fresh run, or use `--resume` to continue one.
8. Copy the trained model from `results` into the Assets folder in Unity.
9. Assign the model in Behaviour Parameters and run in inference without the backend.

## Troubleshooting

- **`MaxProgress` plateaus early**: the agent is likely stuck at a specific obstacle. Watch it in Unity directly to see if it's oscillating rather than attempting to pass.
- **Agent jumps/dives constantly regardless of need**: the small per-action costs may not be large enough relative to how easy it is to trigger them accidentally — double check the action costs are actually being applied on every jump/dive, not just successful ones.
- **Training is extremely slow to show any progress**: expected to some degree given `max_steps: 20000000`, but if mean reward is completely flat after a large fraction of training, revisit `time_horizon` and `buffer_size` before assuming it just needs more steps.
- For general reward-curve reading and "agent isn't learning" diagnostics, see the [TensorBoard guide](../installation/tensorboard.md).

## See also

- [README — Level 2](https://github.com/AISoc-UNSW/FallGuys#level-2) for the quick-start version of this page.
- [TensorBoard guide](../installation/tensorboard.md)
- [ML Agent Theory](../introduction/ml-agent-theory.md)
