# ML Agent Theory

The [RL Theory](rl-theory.md) page explained the mathematical ideas. This page explains how those ideas map onto the actual files and components you'll interact with in the project.

!!! abstract "The key insight"
    The agent-environment loop from the RL Theory page is exactly what's running here — it's just split physically across **two separate programs** (Unity and Python) that communicate over a network socket. Everything else on this page is explaining where each part of that loop lives.

## Two Programs, One Training Run

=== "Just the Basics"

    When you run `mlagents-learn` and press Play in Unity, you're starting two separate processes that talk to each other continuously:

    ```
    ┌──────────────────────────┐         ┌──────────────────────────┐
    │         UNITY            │ ──────> │       PYTHON             │
    │   (the environment)      │  obs +  │   (mlagents-learn)       │
    │                          │  reward │                          │
    │ • Renders the scene      │         │ • Holds the neural net   │
    │ • Runs physics           │         │ • Runs PPO algorithm     │
    │ • Fires sensors          │ <────── │ • Computes gradients     │
    │ • Applies actions        │ action  │ • Updates weights        │
    │ • Computes rewards       │         │ • Logs to TensorBoard    │
    │ • Reports episode end    │         │                          │
    └──────────────────────────┘         └──────────────────────────┘
               gRPC / Communicator API (local socket)
    ```

    **Unity knows nothing about learning.** It just simulates the scene, fires sensors, applies whatever action it's handed, and reports back numbers. It never sees the hyperparameters.

    **Python knows nothing about the scene.** It receives a vector of floats (observations) and a reward scalar, runs the maths, and sends back a vector of floats (actions). It never touches the Unity physics.

    This split is why you need *both* running: `mlagents-learn` waits for Unity to connect, Unity needs `mlagents-learn` to tell it what to do.

=== "Good Understanding"

    No additional depth needed here — see [The Training Config File & Hyperparameters](#the-training-config-file-hyperparameters) below for how you configure the Python side of this split.

=== "Deep Knowledge"

    See [Under the Hood](#under-the-hood) below for exactly how the gRPC connection between these two programs works.

## The Agent Action Loop

=== "Just the Basics"

    On the Unity side, the loop is implemented across three C# methods on the `Agent` script, plus one component that controls timing:

    ```
    CollectObservations()
            │  fills the observation vector
            │  (sensor readings → Python)
            ▼
    Decision Requester  ──── fires "make a decision" every N steps
            │                 (not every physics frame)
            │
            ▼  Python returns an action
    OnActionReceived()
            │  applies the action to move the Fall Guy
            │
            ▼
    AddReward() / EndEpisode()
            │  scores the transition
            └─ ends episode if goal reached, agent fails, or timer expires
    ```

    **`CollectObservations(sensor)`** — called by ML-Agents to fill the observation vector. For a ray-perception setup this is populated automatically by the sensor component; for other observations (positions, velocities, angles) it's populated manually in code.

    **`OnActionReceived(actions)`** — called with the action chosen by the network (or heuristic, or human). Translates the action vector into actual movement.

    **`AddReward(float)` / `SetReward(float)`** — adds to (or replaces) the cumulative reward for this step. Called whenever the agent does something scoreable: e.g. `AddReward(1f)` for progress, `AddReward(-0.1f)` for touching a wall.

    **`EndEpisode()`** — signals that the current episode is over and the agent should be reset. The environment resets, a new episode begins, and the cycle continues.

    **The Decision Requester component** controls *how often* a decision is requested from Python:

    ```
    Decision Period = 5
    → Unity runs 5 physics steps
    → Requests a decision from Python
    → Python sends back an action
    → Unity applies that action for the next 5 steps
    → Requests another decision
    → ...
    ```

    **Why not every frame?** A physics simulation runs at 50+ frames per second. Requesting a neural network decision every single frame would be wasteful — the agent's position barely changes between frames, so getting a new action every frame adds no useful information. A decision period of 5–10 lets the agent "commit" to an action for a short burst, which is more natural for physical movement and much more efficient. Setting `Decision Period` higher makes the agent less reactive but faster to train; lower makes it more reactive but more expensive per unit of real time.

=== "Good Understanding"

    No additional depth needed here — the mechanics above are the whole picture at the Unity/C# level.

=== "Deep Knowledge"

    See [Under the Hood](#under-the-hood) below for how the Academy singleton orchestrates this loop across potentially many agents at once.

## Behaviour Parameters & Heuristic Mode

=== "Just the Basics"

    The **Behaviour Parameters** component is the contract between Unity and the Python trainer. It tells both sides the shape of what they're exchanging:

    | Setting | What it controls |
    |---|---|
    | **Behavior Name** | Must exactly match the key in your training `.yaml` |
    | **Vector Observation size** | Number of floats the agent sends to Python each step |
    | **Action Space** | How many action branches and what values they can take |
    | **Model** | Slot to drop in a trained `.onnx` for inference without Python |
    | **Behavior Type** | Controls training vs. inference vs. heuristic mode |

    If the Behavior Name doesn't match the yaml, `mlagents-learn` will either ignore the agent or crash. This is one of the most common setup mistakes.

    **Behavior Type: Heuristic Only** routes control through a `Heuristic()` method instead of the neural network:

    ```csharp
    public override void Heuristic(in ActionBuffers actionsOut)
    {
        var discreteActions = actionsOut.DiscreteActions;
        discreteActions[0] = (int)(Input.GetKey(KeyCode.W) ? 1 : 0); // forward
        discreteActions[1] = (int)(Input.GetKey(KeyCode.S) ? 1 : 0); // back
        // etc.
    }
    ```

    You can manually drive the Fall Guy using the keyboard, with **no Python backend running at all**. This lets you:

    - **Verify the environment works** before spending time on training
    - **Confirm rewards fire correctly** (watch the reward readout in Unity)
    - **Check episode resets** happen when and how you expect
    - **Debug observations** by seeing if sensors visually make sense (toggle Gizmos in the Game view)

    !!! tip "Always test in heuristic mode before starting a training run"
        It saves debugging time — a broken reward or observation wiring looks identical to "the agent just hasn't learned yet" if you only ever look at a training curve.

=== "Good Understanding"

    No additional depth needed here.

=== "Deep Knowledge"

    No additional depth needed here.

## Ray Perception Sensors

=== "Just the Basics"

    A Fall Guy's "eyes" are often **ray-perception sensors** — a set of rays cast outward from the agent in multiple directions. Each ray can detect objects (walls, goals, hazards) and returns:

    - Whether it hit anything
    - What it hit (by object tag)
    - How far away the hit is (normalised distance)

    This produces a flat vector of floats that becomes part of the observation vector. You can visualise the raycasts in the Unity editor by selecting the agent and enabling Gizmos in the Game view — you'll see coloured lines shooting out from it. The agent never sees the Unity scene directly; it only sees these numbers (plus any manually-added observations, like positions or velocities).

=== "Good Understanding"

    No additional depth needed here.

=== "Deep Knowledge"

    Sensor readings are just floats appended to the same observation vector that travels over the Communicator API — see [Under the Hood](#under-the-hood) below for how that transport actually works.

## The Training Config File & Hyperparameters

=== "Just the Basics"

    This section is one layer deeper than what's required to get a first training run going — the level-specific `.yaml` files already ship with sensible starting values (see the [Level docs](../levels/level-1.md)). Read this tab's Good Understanding content once you're ready to tune those values yourself.

=== "Good Understanding"

    The yaml file configures every aspect of the training run. The structure maps directly onto the training concepts from [RL Theory](rl-theory.md):

    ```yaml
    behaviors:
      BehaviorName:                  # must match Behavior Name in Unity
        trainer_type: ppo

        hyperparameters:
          batch_size: 512             # experiences per gradient step
          buffer_size: 10240          # experiences to collect before any update
          learning_rate: 3.0e-4       # optimiser step size
          beta: 5.0e-3                # entropy bonus (exploration)
          epsilon: 0.2                # PPO clip range
          lambd: 0.95                 # GAE lambda (advantage estimation)
          num_epoch: 3                # gradient passes per buffer

        network_settings:
          hidden_units: 128           # neurons per layer
          num_layers: 2               # depth of the policy network

        reward_signals:
          extrinsic:
            gamma: 0.99               # discount factor
            strength: 1.0

        max_steps: 500000             # total environment steps
        time_horizon: 64              # max steps per rollout segment
    ```

    **Data collection:**

    | Param | What it does | If too low | If too high |
    |---|---|---|---|
    | `buffer_size` | Experiences collected before *any* update | Noisy, unstable updates | Slow to start learning |
    | `batch_size` | Experiences per gradient step | Noisy gradients | Slow updates; must be < buffer_size |
    | `time_horizon` | Steps per rollout segment before cutting | Agent can't learn long sequences | Slow to update; more stale data |

    **Update quality:**

    | Param | What it does | If too low | If too high |
    |---|---|---|---|
    | `learning_rate` | Optimiser step size | Very slow learning | Unstable, diverges |
    | `epsilon` | PPO clip range | Policy changes too slowly | Defeats the purpose of PPO |
    | `num_epoch` | Gradient passes per buffer | Underuses data | Overfits to current buffer |
    | `lambd` | GAE trade-off: value estimate vs. actual rewards | High bias (over-trusts value net) | High variance (noisy) |

    **Exploration and long-term thinking:**

    | Param | What it does | Notes |
    |---|---|---|
    | `beta` | Entropy bonus, rewards random actions | High early → more exploration, drops naturally as policy improves |
    | `gamma` | Discount factor | Higher values suit levels where the goal/payoff is far away |

    **Network size:**

    | Param | What it does | Notes |
    |---|---|---|
    | `hidden_units` | Neurons per layer | Larger = more capacity but slower |
    | `num_layers` | Network depth | More layers = more complex representations |

    For how to launch and read TensorBoard once training is running, see the [TensorBoard installation guide](../installation/tensorboard.md).

=== "Deep Knowledge"

    See [Under the Hood](#under-the-hood) below for the Communicator API, the `.onnx` export format, and how multiple agents share a single policy during training — all of which are relevant once you're tuning this config seriously.

## Under the Hood

=== "Just the Basics"

    This section is deeper plumbing context, not required to get training working.

=== "Good Understanding"

    No additional depth needed here — see the Deep Knowledge tab for the full picture.

=== "Deep Knowledge"

    **How the Communicator API works.** ML-Agents uses a **gRPC** (Google Remote Procedure Call) protocol to pass data between Unity and Python. At startup:

    1. `mlagents-learn` starts listening on a local port (default 5004).
    2. Unity starts, the Academy initialises, and connects to that port.
    3. Each Academy step: Unity packages observations + rewards into a protobuf message and sends it to Python.
    4. Python runs inference (or training), packages actions into a protobuf response, sends it back.
    5. Unity unpacks the actions and applies them.

    This all happens on `localhost`, no internet connection required. The `--run-id` flag you pass to `mlagents-learn` determines where logs and model checkpoints are saved: `results/<run-id>/`.

    **The `.onnx` file and inference mode.** Once training is complete, `mlagents-learn` saves the policy as an **ONNX** (Open Neural Network Exchange) file — a standard format for exporting neural networks so they can run outside the framework they were trained in. When you drop the `.onnx` into the **Model** slot in Behaviour Parameters and set Behavior Type to **Inference Only**, Unity runs the neural network entirely inside its own runtime, no Python needed. This is how you'd ship a trained agent in a real game.

    **Multiple agents training in parallel.** You can have many copies of an agent in the same scene, all training simultaneously. Each agent runs its own instance of the observation-action-reward loop, but they all share the same policy network and contribute experiences to the same buffer. This is one of the key ways to speed up training: more agents means `buffer_size` fills faster, which means more updates per wall-clock minute. The yaml stays identical; Unity just needs more agent GameObjects in the scene.

    **The Academy.** The **Academy** is a hidden singleton in ML-Agents that orchestrates the entire Unity side of training. You never interact with it directly in recent versions of ML-Agents (it auto-initialises), but it manages the connection to the Python communicator, steps all agents in sync, handles episode resets, and tracks global step count. In older ML-Agents documentation you may see references to subclassing Academy — this is no longer required; the component-based approach (Behaviour Parameters, Decision Requester) replaced it.

## See also

- [RL Theory](rl-theory.md) — the mathematical foundations behind this page
- [Level docs](../levels/level-1.md) — see these components applied to a specific level
- [TensorBoard installation guide](../installation/tensorboard.md) — launching and reading training graphs
