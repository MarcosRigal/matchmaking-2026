# Participant Workbook

[Setup and troubleshooting](README.md) |
[Reference solutions](https://github.com/MarcosRigal/solutions-matchmaking-2026/blob/main/SOLUTIONS.md)

Work in **one** snake file: [SnakeApplication.py](SnakeApplication.py) for
Linux/macOS, or [SnakeApplicationWindows.py](SnakeApplicationWindows.py) for
Windows. Each has the same three TODOs. You do not need to implement both
platforms to complete the workshop.

Leave [SnakeGameApplication.py](SnakeGameApplication.py) and [utils.py](utils.py)
unchanged for the core exercises. The controller is useful reference code for
the DDS setup.

## Before you start: the data flow

DDS is data-centric publish/subscribe communication. Applications join a
**domain**, agree on a **topic name and data type**, and exchange typed samples
through writers and readers. There is no application-level central message
broker here; the controller is a game participant, not a DDS broker.

```text
Snake player                                  Game controller
  Square DataWriter ---- ShapeTypeExtended ---> Square DataReader
  Circle DataReader <--- ShapeTypeExtended ---- Circle DataWriter
```

| Concept | In this workshop |
| --- | --- |
| DomainParticipant | Joins the assigned DDS domain |
| Topic | `Square` for snake heads, `Circle` for food; names are case-sensitive |
| Data type | Both topics use `ShapeTypeExtended` |
| DataWriter | Publishes a sample on its topic |
| DataReader | Receives samples on its topic |
| Key | `color` identifies an instance: use unique snake IDs |
| QoS | Communication policies; the starter uses defaults |

Read the type in [utils.py](utils.py): `x` and `y` are coordinates, `shapesize`
is the displayed size, and `color` is the keyed identifier. `fillKind` and
`angle` extend the base shape.

The snake loop runs approximately every 0.1 seconds:

1. Select direction from keyboard input or bot logic.
2. Move and publish the new head position.
3. Read available food samples.
4. Sleep.

Food is read **after** movement, so a newly received food position influences
the next iteration.

### Coordinate convention: read this before coding

Positions are stored as **`(x, y)`**, but directions are **`(dy, dx)`**.
The existing movement method unpacks `dy, dx = self.direction`.

| Movement | Direction `(dy, dx)` |
| --- | --- |
| Left | `(0, -5)` |
| Right | `(0, 5)` |
| Up | `(-5, 0)` |
| Down | `(5, 0)` |

Increasing `y` moves down. The board wraps using modulo: `x % 230` and
`y % 265`. Keep the existing five-unit movement step.

## Exercise 1: initialize DDS

**Find:** `TODO: Implement DDS initialization` in your snake constructor.

**Goal:** make the existing publishing and food-reading methods work without
changing them.

### Steps

1. Inspect the controller constructor. Identify its participant, topics, food
   writer, and snake reader. Which endpoints does a snake need instead?
2. Create a `dds.DomainParticipant` using the constructor's `domain_id`
   argument. Store it as `self.participant`; do not hard-code a domain.
3. Create `self.snake_topic` named `Square` and `self.food_topic` named `Circle`.
   Both must use `ShapeTypeExtended` and the same participant.
4. Create `self.snake_writer` on the snake topic using the participant's
   `implicit_publisher`.
5. Create `self.food_reader` on the food topic using its `implicit_subscriber`.
6. For parity with the reference solution, create `self.snake_reader` on the
   snake topic as well. The current game loop does not use this reader;
   distinguish a reference endpoint from one required for current behavior.
7. Follow the calls in `update_snake()` and `check_food_updates()` to check your
   attribute names. Leave their existing sample-validity check in place.

<details>
<summary>Hint 1: constructing an entity</summary>

The construction pattern is `dds.Topic(participant, topic_name, data_type)`.
A writer takes a publisher and a topic; a reader takes a subscriber and a
topic. Keep long-lived entities on `self`.

</details>

<details>
<summary>Hint 2: check the direction of communication</summary>

The controller writes Circle and reads Square. Your snake must write Square
and read Circle. A reader on Square alone cannot provide food updates.

</details>

### Checkpoint

1. Run your platform's snake command **without `--is-bot`**:
   `python SnakeApplication.py --snake-id BLUE --domain-id 0`
   (use the Windows filename on Windows).
2. It should run without missing DDS-attribute errors and move right using the
   existing `(0, 5)` direction. Keyboard input still does nothing; stop it with
   `Ctrl+C`.
3. To observe publication, subscribe to Square in Shapes Demo on the same
   domain. Expect the BLUE head to move horizontally and wrap at the edge.
   A blank snake terminal is not a failure.
4. If no viewer is available, use a debugger breakpoint immediately after
   `new_head` is calculated to inspect positions, and in the controller's
   `handle_snake_updates()` to verify receipt of valid BLUE samples. A breakpoint
   only before `write()` proves movement, not DDS delivery.
5. Start the snake reader before the controller and inspect `self.food_position`
   after `check_food_updates()` to confirm Circle reception.

**Do not run bot mode yet:** its TODO currently returns `None`, which cannot
be unpacked into `dy, dx`.

## Exercise 2: implement keyboard controls

**Find:** `TODO: Implement user input handling for controlling the snake`.

**Goal:** update direction without blocking DDS updates or allowing an immediate
180-degree turn.

### Steps

1. Use your platform's input API:
   - Linux/macOS: call `self.screen.getch()`. Initialization already enables
     non-blocking input; `-1` means no key was pressed.
   - Windows: query `keyboard.is_pressed()` with `left`, `right`, `up`, `down`,
     and `q`.
2. Map each arrow to the direction table above. Change `self.direction`;
   do not move the head in this method.
3. Reject the opposite of the current direction. For example, left must be
   ignored while moving right.
4. Preserve the current direction when there is no input or an unrelated key.
5. Set `self.running = False` for `q`. Do not call `exit()` inside the handler.
6. Keep input non-blocking. The loop must still publish positions while the
   player is not pressing keys.

<details>
<summary>Hint 1: opposite directions</summary>

If the desired direction is `(a, b)`, its opposite is `(-a, -b)`.
You can express the four guards directly using the direction table.

</details>

<details>
<summary>Hint 2: platform constants and priority</summary>

On Unix, compare with `curses.KEY_LEFT`, `curses.KEY_RIGHT`,
`curses.KEY_UP`, `curses.KEY_DOWN`, and `ord("q")`.
On Windows, an `if`/`elif` chain processes at most one matching key per loop.
The reference prioritizes left, right, up, down, then `q`; release the arrows
before pressing `q`.

</details>

### Checkpoint

Run in human mode and perform this sequence:

| Action | Expected direction |
| --- | --- |
| Start; press nothing | `(0, 5)`; movement continues |
| Press left immediately | Still `(0, 5)`; reversal rejected |
| Press down | `(5, 0)` |
| Press up immediately | Still `(5, 0)`; reversal rejected |
| Press left | `(0, -5)` |
| Press an unrelated key | Direction unchanged |
| Release arrows; press `q` | Loop stops |

Observe the head in Shapes Demo or inspect `self.direction` in a debugger.
The existing loop can publish one final movement after `q`, because it checks
`self.running` at the start of the next iteration.

## Exercise 3: implement the food-seeking bot

**Find:** `TODO: Implement bot logic to calculate the next move`.

**Goal:** return a valid `(dy, dx)` on every path using a simple greedy policy.
The caller assigns your return value to `self.direction`.

### Steps

1. If `self.food_position is None`, return the current direction. This is
   normal before the first food sample arrives.
2. Unpack the head from `self.snake[0]` as `head_x, head_y`.
3. Calculate `delta_x = food.x - head_x` and `delta_y = food.y - head_y`.
4. Compare the **absolute magnitudes** of the deltas.
5. If the vertical distance is strictly greater, return up or down based on
   the sign of `delta_y`.
6. Otherwise, return left or right based on the sign of `delta_x`.
   This means ties choose the horizontal axis.
7. Keep the step size at five. Return a tuple; do not publish samples or mutate
   the snake deque here.

<details>
<summary>Hint 1: magnitude versus sign</summary>

Magnitude chooses the axis. Sign chooses the direction along that axis.
For example, food below the head has positive `delta_y` and needs `(5, 0)`.

</details>

<details>
<summary>Hint 2: preserve the reference policy at boundaries</summary>

The comparison is `abs(delta_y) > abs(delta_x)`, not `>=`.
The horizontal branch returns right only when `delta_x > 0`; otherwise it
returns left. Consequently, exact overlap also returns left. The controller,
not this method, detects food collection.

</details>

### Checkpoint A: deterministic direction cases

Use the following cases in a debugger or a small local test. Set the head to
`(30, 40)` and call `calculate_bot_direction()` without moving the snake.
Set `self.food_position` to a `ShapeTypeExtended` with the listed coordinates.

| Food `(x, y)` | Current direction | Expected return `(dy, dx)` |
| --- | --- | --- |
| No sample (`None`) | `(0, 5)` | `(0, 5)` |
| `(50, 40)` | Any | `(0, 5)` |
| `(10, 40)` | Any | `(0, -5)` |
| `(30, 60)` | Any | `(5, 0)` |
| `(30, 20)` | Any | `(-5, 0)` |
| `(50, 60)` | Any | `(0, 5)` (tie) |
| `(10, 20)` | Any | `(0, -5)` (tie) |
| `(30, 40)` | Any | `(0, -5)` (overlap) |

The reference bot can reverse direction and does not choose the shortest route
across wrapping boundaries. Avoid adding pathfinding during the core exercise.

### Checkpoint B: end-to-end collection

1. Stop any controller already running in your domain. Keep only one controller.
2. Start your completed bot:
   `python SnakeApplication.py --snake-id YELLOW --domain-id 0 --is-bot`
   (use the Windows filename on Windows).
3. Start the Shapes subscribers if available. Allow DDS discovery, then start
   the controller in another terminal with a fixed initial target:

   ```bash
   python SnakeGameApplication.py --domain-id 0 --initial-food-x 100 --initial-food-y 100
   ```

4. Expect the bot to turn toward food, the leaderboard to show YELLOW after
   collection, and the controller to publish another food position. The score
   appears only after both coordinate differences are strictly less than five.
5. If the bot keeps moving straight, inspect `self.food_position`. If it is
   `None`, allow discovery and restart the controller. Food is published only
   at startup and after collection; matching domain IDs alone do not guarantee
   receipt of an earlier sample.
6. Stop both processes with `Ctrl+C`.

Both initial-food arguments are required to select a fixed target. Supplying
only one leaves the controller using random placement.

## Multiplayer and completion checklist

- [ ] All three TODOs are implemented in my platform's snake file.
- [ ] The DDS entity names and topic/type pairings match the controller.
- [ ] Manual input stays responsive, ignores reversal, and supports `q`.
- [ ] Every bot branch returns a tuple and passes the direction table above.
- [ ] A valid Circle sample reaches the snake.
- [ ] The controller receives Square samples and awards a food-collection point.
- [ ] Two players with distinct IDs can participate on the same domain.
- [ ] I can explain why an empty leaderboard does not mean DDS discovery failed.

For the two-player check, first use separate terminals on one machine. Add the
second player with a different ID; a late-joining bot may need the controller
restarted after discovery. Restarting the controller resets its in-memory scores.
Then try another machine on the workshop network.

## Discuss or extend after completing the core

These are optional challenges, not hidden TODO requirements:

- Why does `color` need to be unique for each snake?
- Why is the extra snake reader unused in the current player?
- How would periodic food publication or suitable durability QoS help late joiners?
- How would you choose a shorter path across a wrapping boundary?
- Why might a five-unit step and an off-grid random target produce surprising
  behavior? The controller uses a tolerance, not exact coordinate equality.
- How would you add rendering, snake growth, or stale-player cleanup?

Compare your work with the
[separate solution walkthrough](https://github.com/MarcosRigal/solutions-matchmaking-2026/blob/main/SOLUTIONS.md)
after attempting the checkpoints.
