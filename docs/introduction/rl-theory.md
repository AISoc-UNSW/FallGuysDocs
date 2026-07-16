# Reinforcement Learning Theory

Reinforcement learning (RL) is a machine learning paradigm where an agent learns entirely through experience — no labels, no correct answers handed to it upfront. This page builds up the theory from the core loop to the algorithm (PPO) that actually trains our Fall Guys.

??? tip "Watch this first (optional, 90 minutes)"
    This video is one of the fastest, most complete overviews of reinforcement learning available. It covers everything from the basic loop through MDPs, value functions, policy gradients, and PPO.

    <div class="video-embed" markdown>
    <iframe width="100%" style="aspect-ratio:16/9;border:none;border-radius:8px;" src="https://www.youtube.com/embed/VnpRp7ZglfA" title="The FASTEST introduction to Reinforcement Learning on the internet — Gonkee" loading="lazy" allowfullscreen></iframe>
    </div>

    **Recommended timestamps:**

    | Timestamp | Section |
    |---|---|
    | [4:27](https://youtu.be/VnpRp7ZglfA?t=267) | Markov Decision Processes |
    | [16:37](https://youtu.be/VnpRp7ZglfA?t=997) | Grid Example + Monte Carlo |
    | [36:22](https://youtu.be/VnpRp7ZglfA?t=2182) | Temporal Difference |
    | [50:54](https://youtu.be/VnpRp7ZglfA?t=3054) | Deep Q-Networks |
    | [58:45](https://youtu.be/VnpRp7ZglfA?t=3525) | Policy Gradients (REINFORCE → PPO) |
    | [1:20:24](https://youtu.be/VnpRp7ZglfA?t=4824) | Limitations & Future Directions |

    If you're short on time, watch **4:27 → ~35:00** (MDPs and the core loop) and **58:45 → end of that chapter** (policy gradients). That's the mathematical foundation of everything we're doing.

## The RL Loop & MDPs

=== "Just the Basics"

    The entire mechanism is a loop:

    ```
    Observe state → Choose action → Receive reward → Observe new state → repeat
    ```

    This loop runs thousands of times across many episodes. That's it — that's RL. Everything else is details about how to make it work efficiently.

    **Concretely, for a Fall Guy:**

    - **State / Observation** — sensor readings: ray-perception distances to walls, goals, hazards, and empty space, plus whatever other observations a level defines (positions, velocities, angles). The Fall Guy can't see the scene directly; it only knows what its sensors detect.
    - **Action** — a movement equivalent to WASD, plus jump/dive depending on the level.
    - **Reward** — a number computed by the Unity scene, defined per level (e.g. positive for progress, negative for hitting a wall or falling).
    - **Episode** — ends when the agent reaches a goal, fails (falls, dies), or a timer runs out.

    !!! note "State vs. Observation"
        In RL literature, "state" means the full description of the world. "Observation" means what the agent *actually perceives*, which is often a partial view. In ML-Agents, what the agent receives is always an observation — the true full state of the Unity scene is never handed to it directly.

    The agent-environment loop has a formal name: a **Markov Decision Process (MDP)**, defined by four things:

    | Component | Symbol | What it means |
    |---|---|---|
    | States | **S** | Every possible observation the agent could receive |
    | Actions | **A** | Every move the agent is allowed to make |
    | Reward function | **R** | The rule that scores each (state, action) pair |
    | Transition | **P** | How the world changes given a state and action |

    The critical assumption is the **Markov property**: *the future depends only on the current state, not on the history of how you got there.* For the Fall Guy, whatever its sensors show right now is sufficient to decide the next move — it doesn't need to remember its entire trajectory. This property is what makes the maths tractable; without it, the agent would need to track its entire history, which is computationally infeasible.

=== "Good Understanding"

    This section is foundational — the mechanics of *how* the policy actually improves from this loop are covered in [Learning Algorithms](#learning-algorithms-reinforce-to-ppo) below.

=== "Deep Knowledge"

    The Markov assumption is what makes the bootstrapped TD update in [Learning Algorithms](#learning-algorithms-reinforce-to-ppo) valid — see that section's Deep Knowledge tab for the mechanics of temporal difference learning, which directly exploits this property.

## Return, Policy, and the Value Function

=== "Just the Basics"

    **Return.** If the agent only optimised the *immediate* reward, it would behave short-sightedly. Instead, we define the **return** as the weighted sum of all future rewards:

    $$G_t = r_t + \gamma r_{t+1} + \gamma^2 r_{t+2} + \gamma^3 r_{t+3} + \ldots$$

    The **discount factor γ (gamma)**, between 0 and 1, controls the horizon:

    - **Low γ (e.g. 0.5)** → short-sighted, cares mainly about the next few steps
    - **High γ (e.g. 0.99)** → patient, willing to take a longer route for a bigger eventual payoff

    The agent's goal is to maximise **expected return**, not just the next reward — this is why it can learn strategies that involve short-term sacrifice.

    **Policy.** The **policy** (π) is the agent's behaviour: a mapping from observations to actions. In our setup, the policy is a **neural network**:

    ```
    [sensor observations...] → [neural network] → [probability distribution over actions]
    ```

    - At the **start of training**: weights are random, so the agent stumbles around with no useful strategy.
    - During **training**: the network is updated so that actions that led to high returns become more probable.
    - At **deployment**: the trained `.onnx` file is loaded in Unity and the agent runs inference without Python.

    "Training" literally means adjusting this network's weights. Nothing else about the policy changes — same architecture, same inputs, same output format. Only the numbers inside the network improve.

    **Value function.** The **value function** V(s) answers a different question to the policy: *"If I'm in state s right now, how much total return should I expect from here onward?"* It doesn't pick actions, it scores positions. Near a goal → high value. Stuck against a wall with no clear path → low value.

    The value function is used as a **baseline** during training: instead of asking "was this action good?", we ask "was this action *better than what I expected*?" That difference — actual return minus expected return — is called the **advantage**. Both the policy network and the value network are trained simultaneously; this is the "actor-critic" architecture that PPO uses.

=== "Good Understanding"

    See [Learning Algorithms](#learning-algorithms-reinforce-to-ppo) below for how return, policy, and value function combine into PPO's actual training update (the advantage is what PPO trains on).

=== "Deep Knowledge"

    See [Learning Algorithms](#learning-algorithms-reinforce-to-ppo)'s Deep Knowledge tab for how the value function is updated via temporal difference learning, and [Why This Design?](#why-this-design) for the neuroscience connection between the value function's TD error and dopamine signalling.

## RL vs. Supervised Learning

=== "Just the Basics"

    It's worth being explicit about why RL is different from ordinary supervised learning, because the differences explain why training can be slow and unstable.

    | | Supervised Learning | Reinforcement Learning |
    |---|---|---|
    | **Data source** | Fixed, labelled dataset | Generated live by the agent acting |
    | **Signal** | Correct label per example | Reward per transition (not "correct action") |
    | **Independence** | Each example is i.i.d. | Actions affect future states, not i.i.d. |
    | **Goal** | Generalise from training data | Maximise return through trial and error |
    | **Stability** | Generally stable | Can be noisy, slow, unstable |

    The non-i.i.d. nature of RL data is particularly important: a bad action now changes what the agent sees next, which changes what it learns from next. Errors compound in ways they don't in supervised learning.

=== "Good Understanding"

    No additional depth needed here — this comparison is foundational context, not something that gets more technical at a deeper tier.

=== "Deep Knowledge"

    No additional depth needed here.

## Learning Algorithms: REINFORCE to PPO

=== "Just the Basics"

    The network gets better by playing episodes and reinforcing whatever actions led to good outcomes: actions that led to high return get pushed to be more probable, actions that led to low return get pushed to be less probable. The simplest version of this idea is called **REINFORCE**; **PPO** (Proximal Policy Optimization) is a more robust, practical version of the same idea, and it's the algorithm ML-Agents uses by default.

=== "Good Understanding"

    **REINFORCE — where the loss comes from.** REINFORCE is the simplest policy gradient algorithm, and understanding it explains where the "loss function" in RL even comes from — because unlike supervised learning, it's not obvious.

    1. **Play an episode** — run the current policy, recording every (state, action, reward) tuple.
    2. **Compute returns** — for each action taken, calculate the discounted return that followed it.
    3. **Weight by return** — if the return was high, the gradient update should push *that action's probability up*; if low, push it down.
    4. **Gradient step** — backpropagate through the network exactly like supervised learning, but the "target signal" comes from the return, not a label.

    The loss is constructed as:

    $$\mathcal{L} = -\log \pi(a | s) \cdot G_t$$

    The network's log-probability of the action it took, multiplied by the return that followed. Minimising this loss pushes the network to assign higher probability to high-return actions — no labels, the correctness signal is just whatever return actually happened. This makes REINFORCE a **policy gradient method**: we're taking a gradient step directly on the policy, not on a value function.

    **PPO vs. REINFORCE.** REINFORCE works in principle but is too unstable for harder tasks. PPO fixes the key practical problems:

    | Problem with REINFORCE | PPO's fix |
    |---|---|
    | Must wait for a full episode before any update | Uses short **rollouts** (`time_horizon`), not full episodes |
    | Raw returns are very noisy | Uses the **value function as a baseline** (advantage instead of raw return) |
    | One lucky/unlucky episode can cause a huge, destabilising update | **Clips the update** (`epsilon`) — policy can't change too much in one step |
    | Each batch used for exactly one gradient step | **Reuses** each buffer for `num_epoch` passes — more efficient |

    "Proximal" means "nearby/close to" — the new policy stays close to the old one, and the clip is the mechanism that enforces this. This is why PPO is the default trainer in ML-Agents: it's REINFORCE, made robust enough to work on real environments.

    **On-policy vs. off-policy.** This is a fundamental axis RL algorithms sit on. **On-policy** (REINFORCE, PPO): you can only learn from data collected by the *current* policy — the moment the policy updates, old data is stale and gets discarded. **Off-policy** (DQN, SAC): you store experiences in a **replay buffer** and can reuse them across many updates, even if they were collected by an older policy — more sample-efficient, but updates can be less stable. PPO is on-policy: every time the policy updates, the previous rollout data is gone. ML-Agents also supports SAC (off-policy, `trainer_type: sac`) but PPO is the default.

    **The full training loop, end to end:**

    ```
    ┌─────────────────────────────────────────────────────────────┐
    │  1. INITIALISE — network weights are random                 │
    │                                                             │
    │  2. COLLECT ROLLOUT                                         │
    │     Unity runs the current policy until buffer_size         │
    │     experiences are gathered                                │
    │                                                             │
    │  3. COMPUTE ADVANTAGE                                       │
    │     Python: for each step, actual return minus              │
    │     value function estimate (GAE, controlled by lambda)     │
    │                                                             │
    │  4. UPDATE NETWORKS                                         │
    │     Python: num_epoch passes of gradient descent            │
    │     over buffer in batch_size chunks, clipped by epsilon    │
    │                                                             │
    │  5. DISCARD (on-policy!) → go back to 2                     │
    │                                                             │
    │  6. REPEAT until max_steps or target reward                 │
    └─────────────────────────────────────────────────────────────┘
    ```

    Steps 2–5 repeat continuously. Steps 2 and 3 happen in Unity and Python respectively; step 4 is entirely Python. The connection between them is a live gRPC socket (see [Two Programs, One Training Run](ml-agent-theory.md#two-programs-one-training-run)).

=== "Deep Knowledge"

    **Temporal difference (TD) learning** is how the value function gets updated without waiting for the end of an episode. Instead of using the true return (which requires seeing the whole future), it uses a **bootstrapped estimate**:

    $$V(s_t) \leftarrow V(s_t) + \alpha [r_{t+1} + \gamma V(s_{t+1}) - V(s_t)]$$

    The term in brackets — actual reward plus discounted next-value minus current-value estimate — is the TD error, and it's also closely related to the advantage in PPO. λ (lambda) in your yaml controls how far this bootstrapping looks ahead via Generalised Advantage Estimation (GAE).

    **Monte Carlo vs. TD.** Monte Carlo methods wait for the end of the episode and use the true return — unbiased but high-variance. TD methods use bootstrapped estimates — biased, but lower variance, and they can update online without waiting for episode completion. PPO uses a TD-based advantage estimate, not Monte Carlo.

## Why This Design?

=== "Just the Basics"

    This section is deeper background on *why* we use these specific methods — not required to get training working, but useful context if you're curious.

=== "Good Understanding"

    See the Deep Knowledge tab for the full rationale — it's more theory-heavy than the "Good Understanding" tier elsewhere on this page, so it's kept together rather than split.

=== "Deep Knowledge"

    **Why policy-based methods for this project?** There's a whole other family of RL algorithms — **value-based methods** like DQN — that never directly learn a policy. They learn a Q-function (Q(s, a) = expected return of taking action a in state s) and derive actions by taking the max. We use **policy-based methods (PPO)** for a few specific reasons:

    - **Action space**: our actions behave like continuous control (combined movement directions, not a small fixed menu). Policy methods naturally output a probability distribution; value methods need to take a discrete max over all actions, which becomes awkward with continuous or large action spaces.
    - **Direct optimisation**: policy gradient methods improve the behaviour itself, step by step. Value methods improve the Q-function and hope the right behaviour falls out indirectly.
    - **Smoother updates**: because the policy is a smooth probability distribution, small parameter changes produce small behaviour changes — important for a physical walking agent where erratic jumps in behaviour are costly.

    **Neuroscience connection.** The temporal difference error has a fascinating neuroscience connection: it closely mirrors the behaviour of **dopamine neurons** in primate brains during reward learning experiments (covered at [1:12:06](https://youtu.be/VnpRp7ZglfA?t=4326) in the video above if you're interested). It's a compelling argument that RL might actually describe how biological agents learn.

## See also

- [ML Agent Theory](ml-agent-theory.md) — how this theory maps onto the actual Unity + Python setup
- [Level docs](../levels/level-1.md) — see these concepts applied to specific training runs
