# loom-watch

Loom on the Apple Watch, for keeping an eye on agents from the wrist. It is
built for looking and nudging, not for working: there is no terminal, plan or
diff on it. It ships inside the Loom iPhone app, which lives in
[loom-desktop](https://github.com/FutureMLS-Lab/loom-desktop).

- **Home** puts what needs you first — finishes you have not looked at, and
  agents stopped on a question — then what is working, then agents waiting at
  a prompt. Each row says what the agent last said, or what it is running.
  Stopped tasks are under **All Tasks**.
- **A task** shows the agent's latest words (headings, lists and tables
  flattened to fit), the tool running now, and the steps since your last
  message. From there: answer its question, pick a numbered reply, dictate or
  scribble a message, send *Continue* / *Yes* / *No*, interrupt with Esc, or
  start and stop the agent. Opening a finish acknowledges it everywhere, as
  on the other clients.
- **When an agent stops**, the watch taps if Loom is on screen, and otherwise
  posts a notification with the agent's last words. **Reply** on the
  notification sends straight to the agent.
- **The complication** shows how many agents are working and how many
  finishes wait for you, in the circular, corner, inline and rectangular
  families.

It gets its servers from the iPhone over WatchConnectivity — a token is no
fun to type on a watch — and follows the phone when the phone switches
server; picking another one on the watch sticks until the phone switches
again. Requests go to the server directly, over the watch's Wi-Fi or through
the phone.

What a watch allows in the background sets the limits. While Loom is on
screen it polls every 4 s, and every 20 s while it stays frontmost with the
wrist down. After that, watchOS wakes it a few times an hour at most — up to
four with the complication on the face, far fewer without — so a finish can
take a while to be announced unless the app is up. The task view never reads
`/api/tasks/<slug>`, which inlines every markdown file under the task; the
pane target comes from `claude-sessions` instead.

## Building

The watch app is not a project of its own. It compiles loom-desktop's
networking, models and server list, and the iPhone app embeds it, so
loom-desktop's `iOS/project.yml` includes [`watch.yml`](watch.yml) and the
watch targets become part of the iPhone project. Check the two out side by
side:

```bash
git clone https://github.com/FutureMLS-Lab/loom-desktop.git
git clone https://github.com/FutureMLS-Lab/loom-watch.git
cd loom-desktop/iOS && xcodegen generate   # Loom.xcodeproj: iPhone app, watch app, complication
```

`scripts/ios-install.sh` and `scripts/ios-testflight.sh` in loom-desktop
deliver the watch app with the iPhone app. From the iPhone's Watch app,
install it under *Available Apps* if automatic app install is off. The first
signed build registers `engineering.loom.ios.watchkitapp`, its widget
extension and the app group `group.engineering.loom` for the team.

To try it without a phone or a server, against loom-desktop's mock:

```bash
xcodebuild -downloadPlatform watchOS   # once: a watchOS simulator runtime
scripts/watch-sim.sh                   # → /tmp/loom-watch/shots/home.png
LOOM_WATCH_OPEN=quant scripts/watch-sim.sh
```

Debug builds read `-LoomWatchServer <url>` (and `-LoomWatchToken`) in place
of the list the phone would send, and `-LoomWatchOpen <title>` opens a task
once it is listed.

## Files

| File | What it is |
| --- | --- |
| `LoomWatch/Workspace/WatchStore.swift` | Polls activity, reads ahead the conversations worth a row, acknowledges finishes |
| `LoomWatch/Task/TaskSession.swift` | The open task: its feed's tail, sending, answering, Esc, start and stop |
| `LoomWatch/Supervision/` | Which finishes are news (`FinishLedger`), taps and notifications, background checks |
| `LoomWatch/Sync/PhoneSync.swift` | Watch side of the server list |
| `LoomWatch/Glance/Glance.swift` | What the complication shows, shared with the widget through the app group |
| `LoomWatchWidgets/` | The complication, which also fetches for itself when its timeline is due |
| `watch.yml` | The two targets, as the XcodeGen spec loom-desktop's `iOS/project.yml` includes |
| `scripts/watch-sim.sh` | Build the watch app, run it in a simulator against the mock, and save a PNG |

These are in loom-desktop, and the watch depends on them:

| File | What it is |
| --- | --- |
| `iOS/Loom/App/WatchSync.swift` | iPhone side of the server list: sends it whenever it changes, and on request. Its keys are the ones `PhoneSync` reads, so change both together |
| `Sources/LoomDesktop/LoomAPI.swift`, `Models.swift`, `LoomServers.swift` | The client, payloads and server list, shared with the desktop and iPhone apps. Keep them free of AppKit and UIKit |
| `iOS/Loom/Common/Cancellation.swift` | Tells a request that was given up on from one that failed |
| `scripts/mock-loom.py` | Offline mock of the Loom API, which `watch-sim.sh` runs |

## Licence

Noncommercial use only, under the
[PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0)
terms in [`LICENSE`](LICENSE). Commercial use needs a separate licence from
FutureMLS-Lab.
