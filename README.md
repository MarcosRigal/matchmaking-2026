# Snake Game Workshop: Participant Edition

Build a distributed snake player with Python and RTI Connext DDS. You will connect
your player to a game controller, add keyboard controls, and make a food-seeking
bot.

**Start here:** complete the setup below, then follow the
[step-by-step workbook](WORKSHOP.md). This is the **starter repository**: the
snake applications intentionally contain unfinished TODOs and will not run
successfully until you complete the required exercises.

The [solutions repository](https://github.com/MarcosRigal/solutions-matchmaking-2026)
contains the reference implementation and a separate answer walkthrough.

## Learning goals and route

You should be comfortable with Python classes, tuples, loops, and command-line
arguments. No prior DDS knowledge is required.

| Stage | Suggested time | Outcome |
| --- | --- | --- |
| Setup and DDS introduction | 15 minutes | Environment ready; understand the two topics |
| Exercise 1: DDS initialization | 20 minutes | Publish snake positions and subscribe to food |
| Exercise 2: keyboard controls | 15 minutes | Turn without reversing; quit with `q` |
| Exercise 3: bot movement | 20 minutes | Choose a direction toward food |
| Multiplayer checks and discussion | 15 minutes | Share a domain and diagnose communication |

These are suggested timings, not measured completion times.

## What the applications actually display

- [SnakeGameApplication.py](SnakeGameApplication.py) publishes food, receives
  snake positions, detects food collection, and prints a **text leaderboard**.
- [SnakeApplication.py](SnakeApplication.py) uses `curses` for keyboard input on
  Linux/macOS. It does **not** draw a board in the terminal.
- [SnakeApplicationWindows.py](SnakeApplicationWindows.py) uses `keyboard`
  instead of `curses`. Choose this file on Windows.
- [utils.py](utils.py) defines the shared DDS data types.

For a graphical view, ask your facilitator for RTI Shapes Demo. Use the same DDS
domain and create **subscribers** for `Square` (snake heads) and `Circle` (food),
with compatible types and QoS. Start the Circle subscriber before the controller
publishes food. Do not add Shapes publishers during the basic exercises: they
would inject additional game data.

The Python scripts do not launch Shapes Demo. Only the snake head is published,
not a rendered snake body. No snake growth, wall death, or snake-to-snake collision
handling is implemented.

## Setup

### 1. Check prerequisites

- Git and a real interactive terminal.
- A supported CPython installation; **Python 3.11 or 3.12 is recommended**.
  The pinned `rti.connext==7.7.0` requires Python **3.10 or newer**.
  Its published wheels cover CPython 3.10-3.14 on Linux x86-64/ARM64,
  Windows x64, and macOS Apple Silicon. Do not assume an arbitrary newer
  interpreter or Intel Mac has a matching wheel.
- An RTI license provided by your facilitator or your own RTI installation.
  Ask for it before the workshop; do not commit or share license contents.
- A facilitator-assigned DDS domain ID. Every communicating process must use
  the same ID; different groups should use different domains.

See the [7.7.0 package release](https://pypi.org/project/rti.connext/7.7.0/)
and [RTI Python getting-started documentation](https://community.rti.com/static/documentation/connext-dds/7.7.0/doc/api/connext_dds/api_python/getting_started.html)
for platform and installation details.

### 2. Clone and create a virtual environment

```bash
git clone https://github.com/MarcosRigal/matchmaking-2026.git
cd matchmaking-2026
```

Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Windows PowerShell (example using Python 3.12):

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, use `.\.venv\Scripts\python.exe` in place of
`python` for subsequent commands; changing machine-wide execution policy is not
necessary.

### 3. Install the runtime dependencies

With the virtual environment active, on any supported platform:

```bash
python -m pip install "rti.connext==7.7.0" "keyboard==0.13.5"
```

These are the third-party packages directly imported by the workshop scripts.
`curses` is part of supported Unix Python installations; Windows uses the
separate keyboard-based script.

[requirements.txt](requirements.txt) also includes `evdev`, `python-xlib`,
and `pynput`, which the current scripts do not directly use. The list includes
Linux-specific dependencies, so it is **not a portable Windows/macOS install
command**. Linux users who need the full pinned environment can instead run
`python -m pip install -r requirements.txt`; `evdev` may require native build
prerequisites. The minimal command above is sufficient for these exercises.

### 4. Configure the license in each terminal

Point to the license you were given, substituting its actual absolute path.

Linux/macOS:

```bash
export RTI_LICENSE_FILE="/absolute/path/to/rti_license.dat"
```

Windows PowerShell:

```powershell
$env:RTI_LICENSE_FILE = "C:\absolute\path\to\rti_license.dat"
```

Activate the virtual environment and set the license in **every new terminal**
used to run a process.

### 5. Verify the environment before editing

```bash
python --version
python -c "import rti.connextdds as dds; import keyboard; from utils import ShapeTypeExtended; print('Imports OK')"
python -c "import rti.connextdds as dds; participant = dds.DomainParticipant(0); print('DDS participant OK')"
python SnakeGameApplication.py --help
```

The participant check uses domain `0` just for setup; substitute your assigned
domain if required. Import success alone does not prove that your license permits
creating DDS entities.

Then check the entry point for your platform:

```bash
# Linux/macOS
python SnakeApplication.py --help
```

```powershell
# Windows
python SnakeApplicationWindows.py --help
```

Expected: successful imports, `DDS participant OK`, and argument help. Do not
expect the unfinished snake application to run yet.

## Run commands to use during the exercises

Use separate terminals in this repository with the environment activated.
Replace example domain `0` with the ID assigned to your group.

Controller (one per domain):

```bash
python SnakeGameApplication.py --domain-id 0
```

Human-controlled snake, after exercises 1 and 2:

```bash
python SnakeApplication.py --snake-id BLUE --domain-id 0
```

Bot, after exercises 1 and 3:

```bash
python SnakeApplication.py --snake-id YELLOW --domain-id 0 --is-bot
```

On Windows, replace the snake filename in both commands with
`SnakeApplicationWindows.py`. Use a different `--snake-id` for each player;
the ID is the keyed DDS `color` field. Uppercase standard colors also work well
with Shapes Demo.

After exercise 2, `q` stops a human-controlled snake. Use `Ctrl+C` to stop the
controller or a bot.

**Startup caveat:** the controller publishes food at startup and when food is
collected, not on every loop. With the default QoS, do not rely on a late-joining
snake receiving the earlier food sample. For the bot checkpoint, start the snake
first, allow discovery, then start (or restart) the controller. See the
[workbook](WORKSHOP.md) for the full verification sequence.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| No matching `rti.connext` distribution | Check Python version, CPU architecture, and the published 7.7.0 wheels; use a supported interpreter. |
| Installation fails on `evdev` or `python-xlib` | Use the minimal dependency command above, especially on Windows/macOS. |
| `ModuleNotFoundError` | Activate the environment in this terminal; install using that interpreter's `python -m pip`. |
| DDS participant/license error | Check `RTI_LICENSE_FILE`, file access, and license validity with the facilitator. |
| Missing `snake_writer` or `food_reader` attribute | Exercise 1 is incomplete, or the entities were not stored on `self`. |
| Bot fails while unpacking `self.direction` | Exercise 3 still returns `None`; every path must return a two-number direction tuple. |
| Blank snake terminal | Expected: the script handles input but does not render a board. Use Shapes Demo for graphics. |
| Empty leaderboard | Players appear only after collecting food, not merely after joining. |
| Bot continues straight | Check that a valid Circle sample arrived; restart the controller after reader discovery. |
| Shapes or peers cannot see a snake | Match domain, topic name, type and compatible QoS. Check network/firewall/VPN restrictions with the facilitator; do not disable security controls globally. |
| No keyboard response | Focus the Unix terminal; check that you chose the appropriate platform script. Windows global keyboard hooks may be restricted by local policy. |
| Terminal behaves oddly after interruption | On Linux/macOS, run `stty sane` in the affected terminal. |

For the first multiplayer test, use one machine and several terminals. Move to
multiple machines on the facilitator-approved network after the local check
works.

## Continue

Open the [participant workbook](WORKSHOP.md) and start with Exercise 1.
