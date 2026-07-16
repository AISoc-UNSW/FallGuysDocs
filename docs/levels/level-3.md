# Level 3 — Self-Play Soccer

A competitive 1v1 soccer environment where two Fall Guys train against each other via self-play.

!!! note "Not yet in the FallGuys README"
    Level 3 was added most recently. See the [README's Level 3 section](https://github.com/AISoc-UNSW/FallGuys#level-3) for the quick-start version — this page goes deeper.

## Overview

Level 3 is a genuinely different kind of problem from Levels 0–2: instead of a single agent solving a fixed environment, two Fall Guys (one per team) compete on a soccer field, and both are trained simultaneously via **self-play** — each agent's opponent is itself (or a recent snapshot of itself), rather than a hand-scripted AI. The objective is to move the ball into the opponent's goal while defending your own.

This introduces concepts the earlier levels don't touch at all: competitive multi-agent training, opponent snapshotting, and ELO-based matchmaking to keep the difficulty of the opponent appropriately matched to the agent's current skill.

## Environment mechanics

- **Field, goals, ball**: a standard soccer field with a goal at each end and a single ball.
- **Movement**: WASD-style forward/back/left/right movement, inherited from earlier levels.
- **Jump and dive**: both available, as in Levels 1/2 — diving in particular is used offensively here (see kick mechanics below).
- **Action masking**: jump and dive are disabled in the action space (masked out) whenever the agent isn't in a state where they're usable — e.g. dive requires the agent be in a state where `CanDive` is true, and jump requires `CanJump`. This keeps the policy from wasting exploration on actions that can't do anything at a given moment.
- **Kicking**: there's no explicit "kick" action — the ball is moved by physical contact. When the agent's controller collides with the ball, an impulse is applied to the ball in the direction from the agent to the ball, scaled by how fast the agent was moving (a stationary bump barely moves the ball; a running or diving hit sends it much further). Diving into the ball applies a stronger horizontal and vertical impulse than a standing kick.

## Observation space

20 observations per agent, all relative to the field and expressed as normalized/clamped values:

| Observation | Description |
|---|---|
| Self position (x, z) | Own position relative to field center, normalized by field half-size |
| Self velocity (x, z) | Own horizontal velocity, normalized |
| Self facing (x, z) | Own forward direction |
| Ball relative position (x, z) | Ball position relative to self, normalized by field size |
| Ball velocity (x, z) | Ball velocity, normalized by max ball speed |
| Opponent relative position (x, z) | Opponent position relative to self |
| Opponent velocity (x, z) | Opponent velocity |
| Own goal relative position (x, z) | Own goal position relative to self |
| Opponent goal relative position (x, z) | Opponent's goal position relative to self |
| Ball progress toward opponent goal | Scalar, 0–1, how far the ball currently is along the own-goal→opponent-goal axis |
| Grounded state | Whether the agent is currently on the ground |

Everything is framed relative to the agent itself (rather than absolute field coordinates), which is what lets the same policy generalize regardless of which side of the field the agent currently defends.

## Action space

6 discrete actions, all binary:

- Forward / back / left / right (movement)
- Jump
- Dive

Jump and dive are masked out (disabled) when unusable, rather than simply having no effect — this is a meaningfully different design choice from Level 1/2, where those actions were always available.

## Reward structure

| Event | Reward |
|---|---|
| Ball progress toward opponent's goal (per step) | Small positive shaping reward, proportional to progress gained |
| Closing distance to the ball (per step) | Small positive shaping reward, clamped to avoid large spikes |
| Every step | Small negative per-step penalty |
| Attempting a dive | Small negative cost |
| Making contact with the ball (kicking it) | Flat positive bonus |

This is a shaped-reward design layered on top of what's ultimately a sparse win/loss competitive signal — the shaping (ball progress, closing distance, kick bonus) exists to give the agent *something* to learn from early in training, before it's good enough to reliably score goals. The per-step penalty and dive cost exist to discourage passive play and dive-spamming respectively.

## Self-play configuration (`Level 3.yaml`)

Level 3 is the only level with a `self_play` block, which controls how each agent's opponent is selected during training:

- `save_steps: 50000` — how often a snapshot of the current policy is saved into the opponent pool.
- `team_change: 200000` — how often the two teams effectively swap which snapshot pool they're being trained against.
- `swap_steps: 2000` — how often, within a training window, the opponent is swapped to a different snapshot.
- `window: 10` — the size of the snapshot pool the training process samples opponents from (keeps a rolling set of recent policy versions, not just the single latest one).
- `play_against_latest_model_ratio: 0.5` — the probability of facing the most recent policy snapshot vs. an older one from the pool. A 50/50 split balances "training against something current" against "not overfitting to only ever seeing the latest version of yourself."
- `initial_elo: 1200` — starting ELO rating for matchmaking; ELO is tracked and updated as agents play against each other, which is what lets the training process (and you, when monitoring) gauge whether the policy is actually improving in a relative sense, since there's no fixed opponent to measure absolute performance against.

## Training scale

- `trainer_type: ppo`, `time_horizon: 512`, `gamma: 0.995` — long horizon and high discount, similar reasoning to Level 2: individual actions (a well-timed intercept, a good kick) may not pay off in reward for several steps.
- `max_steps: 20000000` — same order of magnitude as Level 2. Self-play training is generally slower to show clear improvement than single-agent training, since the "difficulty" of the environment (the opponent) is a moving target that improves alongside the agent.

## Manual play-testing (Heuristic mode)

The agent supports a heuristic mode that lets two humans play against each other on one keyboard — useful for sanity-checking the physics and reward wiring before committing to a real training run:

- **Team 0**: WASD to move, Space to jump, Left Ctrl to dive.
- **Team 1**: Arrow keys to move, Enter to jump, Right Shift to dive.

This is worth doing at least once before a long training run — it's a fast way to confirm kicking, goal detection, and controls all behave as expected without waiting on a training curve to tell you something's wrong.

## Tuning and stability notes

- **Watch ELO trend, not a fixed reward target.** Unlike Levels 0–2, there's no single number to aim for here — "success" is a policy whose ELO keeps climbing and that reliably out-competes older snapshots of itself, not a fixed mean-reward threshold.
- **Policy collapse / cycling** is the most common self-play failure mode: an agent can get temporarily very good at beating whatever it's currently facing without making general progress, then get exploited once the opponent pool rotates. If ELO oscillates without a clear upward trend over a long window, this is the most likely cause — consider whether `window` and `play_against_latest_model_ratio` need adjusting to give the agent a more diverse, less exploitable set of opponents.
- **If goals essentially never happen**, check the shaped rewards (ball progress, distance-closing, kick bonus) are actually firing in TensorBoard before assuming the policy itself is bad — a reward-wiring bug (e.g. a goal trigger not firing) looks identical to "the agent hasn't learned to score" from the mean-reward curve alone.

## How to train

1. Open `Levels/Level 3/Level 3 Unity` in Unity.
2. Open the scene: `Assets/Environment.unity`.
3. Review the `self_play` block and PPO hyperparameters in `Level 3.yaml`.
4. Run the backend:
   ```bash
   mlagents-learn "Levels/Level 3/Level 3.yaml" --run-id=run-1
   ```
5. Click Play in Unity.
6. Watch ELO and mean reward in TensorBoard — ELO trend is the more meaningful signal here.
7. Repeat steps 3–6, iterating on the self-play and PPO config. Change `--run-id` for a fresh run, or use `--resume` to continue one.
8. Copy the trained model from `results` into the Assets folder in Unity.
9. Assign the model in Behaviour Parameters for both teams to run in inference without the backend.

## Troubleshooting

- **Agent never attempts to approach the ball**: check the ball-progress and distance-closing shaping rewards are non-zero early in training — if these aren't firing, the agent has no early signal to learn from.
- **ELO flat or oscillating with no clear trend**: likely policy cycling — see the stability notes above.
- **Kicks don't seem to move the ball as expected**: verify in heuristic/manual play mode first, since this isolates a physics/collision issue from a training issue.
- For general TensorBoard reading (not self-play specific), see the [TensorBoard guide](../installation/tensorboard.md).

## Screenshots & video

!!! info "Media coming soon"
    This section is a placeholder for gameplay screenshots and/or a short training video for Level 3. Contributions welcome — drop media in `docs/assets/` and link it here.

## See also

- [README — Level 3](https://github.com/AISoc-UNSW/FallGuys#level-3)
- [TensorBoard guide](../installation/tensorboard.md)
- [ML Agent Theory](../introduction/ml-agent-theory.md)
