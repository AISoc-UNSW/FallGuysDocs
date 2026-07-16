# Tensorboard

TensorBoard is a dashboard that shows training graphs. Instead of only watching the agent move in Unity, you can use it to understand whether the agent is actually learning.

This page has two views: **Student View** covers how to launch TensorBoard and read the graphs while training. **Developer View** covers how TensorBoard support is wired up per level, for contributors adding a new level to the project.

=== "Student View"

    ### Why TensorBoard matters

    In reinforcement learning, the agent improves through trial and error. Sometimes it looks like the agent is improving, but the training graphs tell a different story. TensorBoard helps you answer questions like:

    - Is the agent getting better over time?
    - Is the reward increasing?
    - Is the agent stuck?
    - Is the agent acting too randomly?
    - Is training unstable?
    - Is the agent exploiting the reward function in a strange way?

    For this course, TensorBoard is mainly a debugging and learning tool.

    ### Before you start

    Open a terminal in the repository root and activate the ML-Agents environment:

    ```bash
    conda activate mlagents
    ```

    You do not need to manually install TensorBoard if the course environment is already set up correctly. If the script says TensorBoard is missing, run:

    ```bash
    pip install tensorboard
    ```

    ### How to launch TensorBoard

    Use two terminals: one runs ML-Agents training, the other runs TensorBoard.

    **Terminal 1** (example for Level 0):

    ```bash
    conda activate mlagents
    mlagents-learn "Levels/Level 0/Level 0.yaml" --run-id=level0
    ```

    When ML-Agents asks you to start training, press Play in Unity.

    **Terminal 2:**

    ```bash
    conda activate mlagents
    ./scripts/tensorboard.sh level0
    ```

    On Windows:

    ```bat
    conda activate mlagents
    scripts\tensorboard.bat level0
    ```

    The same pattern works for any level, e.g. `./scripts/tensorboard.sh level1`.

    ### Opening the dashboard

    After running the TensorBoard script, the terminal will print a URL similar to:

    ```text
    http://localhost:6006
    ```

    Open that URL in your browser. If port `6006` is already in use, the macOS/Linux script may choose another port such as `http://localhost:6007` — use the exact URL printed in your terminal.

    !!! tip "Graphs not appearing immediately?"
        This is normal. TensorBoard can be opened before training starts — it will begin showing graphs once ML-Agents creates training logs. In this project, ML-Agents writes TensorBoard summaries every `summary_freq` steps (10,000 by default), so wait until training has taken that many steps. You can either manually refresh, or click the gear icon (top right, not the settings one) and check "Reload Data".

    ### The graphs to focus on

    TensorBoard shows many graphs, but you don't need to understand all of them. Focus on these first:

    1. `Environment/Cumulative Reward`
    2. `Environment/Episode Length`
    3. `Losses/Policy Loss`
    4. `Losses/Value Loss`
    5. `Policy/Entropy`
    6. `Policy/Learning Rate`

    **`Environment/Cumulative Reward`** — the most important graph. Shows how much reward the agent receives per episode; usually you want this to increase over time.

    - **Goes up** → the agent is probably learning, the reward function is giving useful feedback, observations/actions are likely connected correctly.
    - **Stays flat** → the agent may not understand what to do, rewards may be too rare, observations may be missing/incorrect, or the goal may be too hard to reach.
    - **Jumps up and down wildly** → training may be unstable, learning rate may be too high, the reward function may be easy to exploit, or the environment may be inconsistent.

    **`Environment/Episode Length`** — how long an episode lasts before it ends. Longer is not always better: if the agent gets a longer episode by standing still and avoiding punishment, that's not good learning. Always compare episode length with cumulative reward and what you see in Unity.

    - **Length and reward both increase** → the agent may be surviving longer and performing better.
    - **Length increases but reward stays flat** → the agent may be wasting time; the reward function may not encourage progress strongly enough.
    - **Length is very short** → the agent may be failing quickly; the level may be too punishing at the start.

    **`Losses/Policy Loss`** — how much the agent's decision-making policy is changing during training. Use it as a rough stability signal, not something to calculate yourself. Small changes and mild fluctuations are normal; large repeated spikes can mean training is unstable, the learning rate is too high, updates are too aggressive, or the reward signal is noisy. Policy loss doesn't need to smoothly decrease forever — RL graphs are often noisy.

    **`Losses/Value Loss`** — how well the agent is predicting future reward. If it gradually becomes more stable, the agent is getting better at estimating which situations are useful. If it stays very high or spikes constantly, the agent may not understand which actions lead to reward, rewards may be too delayed/inconsistent, or the environment may be too difficult for the current setup.

    **`Policy/Entropy`** — how random the agent's actions are. High = exploring many actions; low = becoming consistent and confident.

    - **High at the start** → normal, the agent is exploring.
    - **Slowly decreases** → usually good, moving from exploration toward a strategy.
    - **Stays high too long** → the agent may still be acting randomly; rewards may be too rare.
    - **Drops very low too early** → the agent may stop exploring before finding a good strategy, and get stuck repeating a weak behavior.

    **`Policy/Learning Rate`** — controls how large the training updates are. In this project the schedule is linear, so it gradually decreases during training. Higher = faster but less stable changes; lower = slower but more stable. Useful for explaining why learning may slow down later in training.

    ### A simple way to read TensorBoard

    1. Look at Cumulative Reward.
    2. Check Episode Length.
    3. Compare the graphs with what the agent is doing in Unity.
    4. Check Entropy to see whether the agent is still exploring.
    5. Check Policy Loss and Value Loss for instability.

    Don't judge training from one graph alone — e.g. if reward is increasing and the agent looks better in Unity, training is probably going well even if some loss graphs are noisy.

    ### Common training patterns

    **Good training**: cumulative reward generally increases, episode length makes sense for the level's goal, entropy gradually decreases, policy/value loss don't spike constantly, the agent looks better in Unity.

    **Agent is not learning**: cumulative reward stays flat, episode length doesn't improve, entropy stays high, the agent moves randomly in Unity. Check: is the reward clear? Can the agent reach the goal? Are observations/actions connected correctly? Is the level too difficult at the start?

    **Training is unstable**: reward rises then suddenly drops, policy loss has large repeated spikes, value loss is very noisy, the agent changes behavior suddenly. Try: lower the learning rate, avoid changing too many hyperparameters at once, check whether the reward function can be exploited, make sure the agent isn't receiving conflicting rewards.

    **Agent finds a strange shortcut**: sometimes the agent learns to get reward in a way that doesn't match the intended level goal (e.g. standing still if that avoids punishment). Watch the agent in Unity, compare with the reward graph, and change the reward function so it better matches the real goal.

    !!! note "Extra graphs"
        Depending on the ML-Agents version and training settings, TensorBoard may show extra graphs beyond the ones above. You don't need to focus on them at first — start with reward, episode length, entropy, policy loss, value loss, and learning rate. Once those make sense, extra graphs can be used for deeper debugging.

    ### Quick troubleshooting

    **TensorBoard says command not found** — activate the environment and install it:

    ```bash
    conda activate mlagents
    pip install tensorboard
    ```

    **TensorBoard opens but no graphs show** — did you start training? Did you press Play in Unity? Are you using the correct level command? Has training reached at least `summary_freq` steps (10,000 by default)?

    **The wrong run is showing** — make sure the run ID matches the level (`level0 -> results/level0`, `level1 -> results/level1`, etc.).

    **Port already in use** — on macOS/Linux the script automatically tries the next available port; on Windows pass a different port manually: `scripts\tensorboard.bat level0 6007`.

    ### Main takeaway

    TensorBoard helps you connect what the agent is doing in Unity with what's happening during training. If the agent looks better, the reward graph should usually support that. If the graph looks strange, check the reward function, observations, actions, level design, and hyperparameters.

=== "Developer View"

    This view is for contributors adding TensorBoard support to a new Fall Guys RL level. TensorBoard support is already set up for Level 0 (`level0`) and Level 1 (`level1`) — students should not need to manually choose log folders; they run one small script from the repository root.

    ### Current setup

    The TensorBoard launchers live in:

    ```text
    scripts/
    |-- tensorboard.sh
    |-- tensorboard.bat
    ```

    The scripts map a short level name to the correct ML-Agents results folder, e.g. `level0 -> results/level0`. This means contributors can tell students to run `./scripts/tensorboard.sh level0` (or `scripts\tensorboard.bat level0` on Windows).

    ### How ML-Agents creates TensorBoard logs

    When training starts, ML-Agents writes training summaries into the run's results folder:

    ```text
    results/
    |-- level0/
    |-- level1/
    ```

    Each run folder contains the trained model, checkpoints, configuration, run logs, and TensorBoard event files. TensorBoard reads those event files and turns them into graphs.

    ### YAML requirements

    Each level's training config should include `summary_freq`:

    ```yaml
    summary_freq: 10000
    ```

    This tells ML-Agents how often to write TensorBoard summaries. If too high, students wait a long time before graphs appear; if too low, ML-Agents writes logs more often than necessary. `10000` is a good default for this course.

    ### Run IDs

    The script expects training to use a run ID that matches the level name:

    ```bash
    mlagents-learn "Levels/Level 0/Level 0.yaml" --run-id=level0
    ```

    `--run-id` controls where ML-Agents writes results (`--run-id=level0 -> results/level0`). If the run ID doesn't match the script's mapping, TensorBoard may open but show no graphs.

    ### Adding TensorBoard support for a new level

    Suppose a contributor adds Level 2:

    1. **Choose a simple level key**, e.g. `level2`.
    2. **Train using that run ID**: `mlagents-learn "Levels/Level 2/Level 2.yaml" --run-id=level2` — this creates `results/level2`.
    3. **Check the YAML config** includes `summary_freq: 10000`.
    4. **Update the macOS/Linux script** (`scripts/tensorboard.sh`) with a new mapping:
       ```bash
       elif [ "$LEVEL" = "level2" ]; then
           LOGDIR="results/level2"
       ```
       Also update the usage text so contributors know the new level exists.
    5. **Update the Windows script** (`scripts/tensorboard.bat`):
       ```bat
       ) else if "%LEVEL%"=="level2" (
           set LOGDIR=results\level2
       ```
       Also update the usage text.
    6. **Test the script**: `./scripts/tensorboard.sh level2` (or `scripts\tensorboard.bat level2` on Windows) should print a local URL such as `http://localhost:6006`.

    ### What to verify before merging a new level

    - The level YAML has `summary_freq: 10000`.
    - The training command uses the same run ID as the TensorBoard script.
    - The script maps the level key to the correct `results` folder.
    - TensorBoard opens without extra setup after `conda activate mlagents`.
    - The dashboard shows graphs after training starts.
    - The student docs mention the new level key if students are expected to use it.

    ### Common problems

    **TensorBoard opens but there are no graphs** — training hasn't started yet, the run ID doesn't match the script's log directory, ML-Agents hasn't reached the next `summary_freq` step yet, or the wrong level key was passed to the script.

    **TensorBoard command not found** — the environment probably doesn't have TensorBoard installed or activated:

    ```bash
    conda activate mlagents
    pip install tensorboard
    ```

    **Port 6006 already in use** — the macOS/Linux script automatically checks for the next available port; on Windows pass a different port manually: `scripts\tensorboard.bat level0 6007`.

    ### Contributor rule of thumb

    For every new level, keep the naming consistent: `level key = run ID = results folder` (e.g. `level2 -> --run-id=level2 -> results/level2`). This keeps TensorBoard easy for students to launch and easy for contributors to maintain.
