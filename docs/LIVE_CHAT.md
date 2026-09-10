# Live chat in the companion

Open `laptop/Start-Duck2Companion.cmd`. Live chat runs locally with no API key.
It supports lane following, junction choices, straight speed profiles, Stop
and pause controls. The parser is a hand-written command grammar, not a trained
language model; [implementation and provenance](CHATBOT_PROVENANCE.md) explains
its development and the template investigation.

## Starting

1. Prepare the updated source with `tools/Start-Duck2-DrivingMode.cmd` while
   duck2 is stationary. This uses the existing source-mounted runtime and does
   not require a new Docker Desktop build. It changes robot controller ownership,
   so follow the preparation/cleanup instructions in [USAGE.md](USAGE.md).
2. Open the companion, connect, and select the directed starting lane and a
   destination red line. The map supplies an initial queue of junction turns.
   To start with no planned turns, select the same directed lane for both.
3. Leave **Live chat (ask when queue ends)** checked. Confirm placement and
   press **Start selected route** while observing duck2.
4. Open **Camera & status → Live chat** beside the camera. The panel reports
   the lane, future queue, speed profile, pause state and red-stop countdown.

The app refuses a live-chat Start if the controller does not advertise the
new session protocol. Reopen the app and prepare the updated robot source;
an already running window/container does not reload changed Python files.

Unchecking Live chat before Start retains map-only operation, which ends at
the destination red line. In live-chat mode, that line instead prompts for
another instruction and ends the run after 60 seconds without departure.

## Supported messages

| Message | Result |
| --- | --- |
| `The next turn should be right` | Replace all future junction turns with right |
| `Left after that one` | Append left after the current future queue |
| `Next left then right` | Replace the queue with left, right |
| `Clear turns` | Empty the future queue; wait at the next red line |
| `Status` / `Where are we?` | Show the reported lane and queue |
| `Slow speed`, `normal speed`, `fast speed` | Choose one straight-road profile |
| `Speed up` / `slow down` | Move one profile higher/lower |
| `Pause for 5 seconds` / `stop for 5 seconds` | Immediately request zero output; preserve queue; resume after five seconds if healthy |
| `Stop` / `pause` | Immediately pause; end run if not continued within 30 seconds |
| `Continue` / `resume` | Resume a pause; do not bypass a red stop |
| `Quit`, `end run`, `stop the run` | End the run immediately through Stop |

Press Enter to send; Shift+Enter adds a line. Capitalization and common punctuation are normalized, but arbitrary typos are
not corrected. Timed pauses accept numeric durations up to 3600 seconds.
Wording outside the supported
grammar asks for clarification without sending a movement command. Negated,
conditional, reverse and obstacle requests are not executed. Ambiguous stop
wording is treated as pause rather than ending the run. An invalid timed
request is clarified rather than silently converted to an indefinite pause.

## Queue and position rules

The queue lives on the laptop. Each proposed sequence is checked against the
directed map before replacing anything. Straight ahead at a T-junction without
a straight exit is rejected. A turn already accepted by the robot cannot be
edited midway through the crossing; edits affect subsequent junctions.

At a red line the robot holds its normal dwell, then accepts one validated
instruction for that exact run, lane and junction index. Empty queue prompts
**Where should I go next?** The 60-second limit starts when the robot arrives
at the red stop, including the dwell. Chatting or sending an invalid turn does
not restart this clock. No instruction causes an indefinite unmarked crossing;
all existing crossing deadlines remain in effect.

The robot reports outgoing-lane reacquisition and its turn/index. The laptop
advances its directed-lane estimate only after matching this report to the
instruction it sent. This is route-history tracking, not global localization:
physically moving the bot requires a new confirmed starting position. Lost
acknowledgments are reconciled from status or retried with the same identifier,
so a retry cannot consume a second turn. Unexpected run/position reports
suspend queue delivery. Stop remains available.

## Straight-only changes

The straight-road profiles request centre speeds of **0.09 / 0.10 / 0.11**
(slow / normal / fast). Normal is the default. These are normalized wheel
requests, not metres per second. The profile path caps either wheel at 0.12
and retains the controller's acceleration limits. They are conservative
software candidates, not a new measured guarantee of lane containment.

Profiles apply only after stable, ordered yellow/white boundaries with near
support, small image heading/lateral error and small steering demand indicate
a straight. Missing/curved/uncertain evidence immediately uses the existing
controller speed. Speed increases are rejected outside a confirmed straight.
Selecting a lower profile during a turn changes the future straight setting
only. Curve, sharp-bend, junction-entry, pivot and reacquisition profiles retain
their existing calibrated values and steering logic. Legacy global speed-scale
commands are rejected during a live-chat run.

A pause takes effect as soon as the robot accepts the command, even during a
curve, bend or intersection. The command callback publishes zero before its
acknowledgment; it does not wait for a straight, another frame, or the next
junction. The requested duration is measured by the robot's monotonic clock
from that acceptance. Network delivery and physical braking still take time.
Timed pauses resume only with a healthy connection and fresh camera.
Indefinite pauses expire after 30 seconds without Continue. Future turns and
the current maneuver survive either pause. Paused time does not consume a
turn's motion budget or the red-line instruction wait; reacquisition requires
fresh continuity evidence after resume. Camera, heartbeat and watchdog clocks
continue normally. Repeating a pause does not extend its deadline. **STOP
DUCK2**, quit, or a fault cancels automatic resume and ends the run.
The app refuses to start with a controller lacking `pause_mode: immediate`.

## Supervised straight-path pause check

1. Place duck2 centred on a clear straight in its right lane. Keep the camera
   uncovered, stay beside it and keep the charging cable slack if connected.
2. Prepare the updated driving mode, then open `laptop/Start-Duck2-PauseCheck.cmd`.
   This is the same companion with `--pause-check`: it selects A → E, connects
   and opens the camera. Place duck2 after the curve on the straight. No route
   is started automatically. This mode tells the robot to end at the next red
   line, without waiting for instructions or crossing the intersection.
3. The Live chat input is empty. Type `pause for 3 seconds` yourself; you can
   choose another supported duration before sending it.
4. Confirm placement and click Start selected route when watching duck2.
   After it begins straight travel, send the message yourself. Check that it
   stops promptly, the app shows the pause countdown, and it resumes after
   about three seconds. It then follows the straight to the next red line and
   ends the test there. STOP DUCK2 remains available throughout.
5. Record physical stop, approximate pause duration and resume separately from
   the robot's reported zero output. A fault or Stop must prevent auto-resume.

An assistant-run physical Start still requires a fresh Go. The user sends the
chat message; preparation and isolated tests never send it to the real robot.

## Live junction override check

The user reported the A → E three-second pause scenario successful. For the
next scenario open `laptop/Start-Duck2-JunctionChatCheck.cmd`, after preparing
the updated controller while stopped. The launcher opens the same companion
with the selected route A → E → B → C and live chat enabled. It connects the
camera and leaves the chat input empty. The chat displays the selected route
without describing the planned test or suggesting commands.
Place duck2 after the curve on the A → E straight, confirm placement and Start
only while watching it.

1. If desired, first type and send `stop for 7s` (or `stop for 7 seconds`) on the straight. The
   robot pauses immediately and resumes automatically; its turn queue stays
   intact. Before E, send `go straight at the next junction`. This replaces the planned
   left/right queue with one straight instruction. The expected outgoing lane
   is E → C. The original remaining right turn is discarded.
2. Duck2 stops at E for its normal dwell, crosses straight, reacquires the
   outgoing right lane and follows E → C.
3. At C the queue is empty. Duck2 stays stopped and asks for the next direction;
   the robot allows 60 seconds from red-stop arrival, with a displayed countdown.
4. Send `go left` at C. From approach E → C, this is the legal exit C → B.
   Duck2 performs the existing left maneuver and resumes lane following.
   This scenario finishes at the **C → B red line**, where the robot ends the
   run and stays stopped. It does not wait for another departure command at B.
   STOP DUCK2 remains available throughout.

This check does not end at the first red line. It requires a controller that
advertises the 60-second red wait and directed final-red-line support. The indefinite chat pause still has its
separate 30-second Continue limit; explicit timed pauses use their requested
duration. The selected movement profiles, colour thresholds, two-second red dwell
and heartbeat limits remain unchanged. Straight exits now commit road steering
and route progress together after an observed corridor stays consistent for
0.3 seconds; the old E lane-search timer cannot remain attached to E → C.
Distant fragments, wrong-order boundaries and narrow patches do not establish
that corridor. Crossing deadlines remain active while the exit is unconfirmed.

The launch option is `--junction-chat-scenario` (`--junction-chat-check` remains
an alias). Each Start creates a new session, clears prior pause/junction state,
and configures the final directed red line on the robot. Opening the launcher
only connects and displays the camera; the user still confirms placement,
clicks Start, and types every command. A fresh app window does not resume an old
run. Physical performance of this revised combined sequence still needs the
user's supervised run.

## Verification and limits

2026-09-10 immediate-pause update: 75 focused native checks passed, including
the real controller with mocked ROS and the Windows companion UI. The focused
`tests/verify_live_chat_ros.py` check passed with Docker networking disabled:
all three profiles, immediate zero without straight classification, a
three-second pause (3.057 seconds between first zero and resumed wheel request),
and ending at the next red line without accepting a late resume. No physical
movement is claimed from that synthetic evidence.

The broader ROS script stopped at its existing sharp-right-wheel assertion
before reaching its chat section. This update does not retune or claim to fix
that unrelated scenario. The focused HTTP/ROS check verifies the changed path
using the same local interpreter, gateway, controller and message types.

`tests/test_live_chat.py` exercises parsing, directed-map validation, queue
replacement/append, lost acknowledgments, turn reporting and deadlines.
`tests/test_live_navigation.py` uses the actual controller with mocked ROS to
check the wheel gate, immediate pause/resume, red-stop waiting, profile isolation
and Stop priority. Native Tk tests exercise the chat panel and responsive map.

Before this immediate-pause update, the native suite passed 407 tests with one
platform/dependency skip. Both AMD64 and ARM64 images built, and the isolated Noetic transport
script passed with networking disabled, including a managed-session
Start/profile/pause/resume/Stop sequence. See [TESTING.md](TESTING.md) for the
dated results and the distinction between local ROS evidence and physical tests.

| Requirement | Verification |
| --- | --- |
| Replace/append future turns and reject unavailable exits | Parser and directed-map tests cover every approach; invalid edits leave the queue intact |
| Track reported completion without duplicate turns | Lost-acknowledgment, repeated-ID and mismatched-run/index tests; laptop/controller two-junction sequence |
| Wait at an empty red stop and end after 60 seconds from arrival | Controller tests cover dwell, late first tick, a valid turn after 55 seconds, expiry and disabled automatic departure |
| Three straight profiles, normal default, unchanged turns | Synthetic straight/curve images, wheel cap and profile tests; all three junction choices checked across every speed selection |
| Immediate timed/indefinite pauses preserve turns | Immediate-zero callback, curve/crossing pause, frozen motion budgets, resume and 30-second expiry tests |
| Stop/quit priority and connection failures | Parser, stale-epoch command rejection, UI Stop during command-lock contention, timed-resume health checks and isolated ROS heartbeat/shutdown checks |
| Usable app with no model API | Native Tk interaction, map fitting at multiple window sizes/scalings, chat input and clean-close tests |

These offline checks do not prove the new straight classifier or speed profiles
under tomorrow's lighting. Initial live verification must remain supervised.
No live-chat movement was performed as part of this implementation.

## Latest combined scenario record

The latest 2026-09-10 attempt appeared promising to the user but ended when the
battery depleted. It remains incomplete, not a successful full-scenario result.
The source snapshot was preserved outside the repository; see [TESTING](TESTING.md).

## Confirmed-curve yellow gaps

The current controller can retain a confirmed left-curve request for at most
0.8 seconds while yellow briefly disappears, only with a matching curved white
edge and fresh data. This does not change junction instructions or permit blind
turn recognition. See [the review](LIVE_CHAT_LANE_REVIEW.md) for verification.
The preview does not share the controller's turn memory; read controller status
when interpreting a short gap. Physical validation of the patch is pending.
