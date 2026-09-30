# Wardrobe System

An outfit-recommendation appliance built on the Arduino UNO Q. Every
morning it picks an outfit for you based on your schedule, the weather,
and how recently each item was worn — and tracks what you actually wore
versus what it recommended, so cost-per-wear and rotation stats stay
accurate.

This repo is a sanitized template with no personal data, seeded with
generic starting tags/schedule/clothing types. Everything here is meant
to be edited to fit your own wardrobe, schedule, and network.

## What's in here

```
admin-backend/     FastAPI + SQLite backend and web admin UI.
                    Runs on the UNO Q's Linux side (or any Linux/Mac/
                    Windows machine for development).
kiosk-app/          The physical-display app: buttons, motion sensor,
                    camera-preview scanning UI. Runs as an Arduino App
                    Lab app on the UNO Q.
```

## Architecture, in one paragraph

The **admin backend** is an ordinary FastAPI web app with a SQLite
database — it has no dependency on any Arduino-specific library and runs
identically on your laptop or on the board. It owns the database, the
recommendation engine, the weather lookup, QR generation/scanning, and
the web admin pages you use from any browser on your network. The
**kiosk app** is a separate Arduino App Lab app that only exists on the
UNO Q: it talks to the MCU side (buttons, motion sensor, an optional
scan-indicator LED) over the Router Bridge, and talks to the admin
backend purely over plain HTTP, exactly like any other client. Keeping
these two processes separate — and keeping "what the kiosk recommended"
strictly separate from "what you actually confirmed you wore" — is the
central design decision in this project; see `WearHistory` vs.
`Outfit`/`OutfitItem` in `admin-backend/db/models.py`.

## Hardware you'll need

- Arduino UNO Q
- A display (HDMI works fine — no touchscreen required; everything is
  driven by physical buttons)
- A USB camera, for scanning QR codes glued/tagged to each clothing item
- 2–3 momentary push buttons ("Wear This", "Change", "Done")
- A PIR motion sensor, to wake the display
- (Optional) an LED + resistor, lit while the camera is actively
  scanning, so you can tell the scanner is watching
- A QR code label printed and attached to every clothing item you want
  tracked (see "Generating QR labels" below)

Wiring assumed by `kiosk-app/sketch/sketch.ino` (change the pin constants
at the top of that file if you wire it differently):

| Component | Pin |
|---|---|
| "Wear This" button (other leg to GND, internal pull-up) | D2 |
| "Change" button (other leg to GND, internal pull-up) | D3 |
| PIR motion sensor output | D4 |
| "Done" button (other leg to GND, internal pull-up) | D5 |
| Scan-indicator LED (+ leg, through a resistor; other leg to GND) | D6 |

## Part 1 — Admin backend

### 1a. Local development (no hardware needed)

You can build out and test almost everything — clothing management,
tags, schedule rules, weather, statistics, even QR *scanning* if you have
a USB webcam — on a regular Windows/Mac/Linux machine before ever
touching the board.

```bash
cd admin-backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Create your own env file from the template and fill in your details:

```bash
cp deploy/wardrobe.env.example .env
```

Edit `.env`:

```
WARDROBE_DB_PATH=./data/wardrobe.db
WARDROBE_LATITUDE=<your latitude>      # e.g. from latlong.net
WARDROBE_LONGITUDE=<your longitude>
WARDROBE_CAMERA_INDEX=0                # 0 is usually the first/only USB camera
WARDROBE_WEATHER_CACHE_MINUTES=30
WARDROBE_SCAN_TIMEOUT_SECONDS=15
```

Load the env file into your shell (or use `python-dotenv`/your IDE's env
support — the app just reads `os.environ`), then run migrations and seed
data:

```bash
mkdir -p data
alembic upgrade head
python seed.py
```

Start the server:

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8080
```

Open `http://localhost:8080` — it redirects to the clothing list. From
there:

1. Go to **Clothing → + Add Item**, fill it in, save, then upload a
   photo and download/print its QR code label.
2. Go to **Tags** if you want to add tags beyond the seeded defaults.
3. The seeded weekly schedule (editable via the database today — a
   dedicated schedule editor UI is a natural next phase to build, see
   "Extending this project" below) starts everyone at Work/Formal on
   weekdays, Casual on Saturday, Church on Sunday.
4. Once you have a few items tagged for *today*, hit
   `http://localhost:8080/history` and click **"Get New Recommendation
   for Today"** to generate one and see it appear.

### 1b. Deploying to the UNO Q

The board's Linux side runs Debian, so this is the same app, just run as
a systemd service instead of `--reload`.

1. Copy the `admin-backend/` folder onto the board, e.g. to
   `/home/arduino/wardrobe/`.
2. On the board:
   ```bash
   cd /home/arduino/wardrobe
   python3 -m venv venv
   venv/bin/pip install -r requirements.txt
   mkdir -p data
   cp deploy/wardrobe.env.example wardrobe.env
   nano wardrobe.env   # fill in your real coordinates, etc.
   venv/bin/alembic upgrade head
   venv/bin/python seed.py
   ```
3. Give the board a static LAN IP (set this in your router's DHCP
   reservations, or configure it directly on the board — either way,
   pick and note down an IP like `192.168.1.50` for the steps below).
4. Install the systemd service:
   ```bash
   sudo cp deploy/wardrobe-admin.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable --now wardrobe-admin
   sudo systemctl status wardrobe-admin   # should show "active (running)"
   ```
   The unit file assumes the app lives at `/home/arduino/wardrobe` and
   runs as user `arduino` (the board's default account) — edit
   `deploy/wardrobe-admin.service`'s `User=`/`WorkingDirectory=`/
   `ExecStart=` lines first if yours differs.
5. Confirm it's reachable from another machine on your network:
   `http://<board's LAN IP>:8080`.

**Camera note:** if the USB camera is going to be plugged into the UNO Q
itself (rather than staying on a dev machine), set
`WARDROBE_CAMERA_INDEX` to whatever index it shows up as on the board
(usually still `0` if it's the only camera), and make sure the service
user is in the `video` group — the provided unit file already adds
`SupplementaryGroups=video`.

### Generating QR labels

Once an item is saved in the admin UI, its edit page has a **Download QR
(PNG)** link. The clothing list page also has a **Print QR Labels (all
active)** button that generates a multi-item label sheet PDF (3 columns,
sized for standard label sheets — see `app/qr_generator.py` if you want
to tune the layout) you can print and cut apart.

## Part 2 — Kiosk app (the physical display)

This is a separate **Arduino App Lab** app — install App Lab on the UNO
Q if you haven't, then create a new Python app there and copy
`kiosk-app/`'s three folders (`sketch/`, `python/`, `assets/`) into it,
matching that same structure.

### 2a. The sketch (MCU side)

Open `kiosk-app/sketch/sketch.ino` in the Arduino IDE (or however App Lab
wants the sketch loaded), adjust the pin constants at the top if your
wiring differs from the table above, and upload it to the board's MCU.

### 2b. The Python app (Linux side)

`kiosk-app/python/main.py` needs one thing set correctly before it'll
work: **how it reaches the admin backend.** There are two different
addresses involved, and this is the single most common thing to get
wrong when replicating this project, so read this carefully:

- The App Lab runtime runs each Python app inside its own Docker
  container, with its own network namespace. That container **cannot**
  reach the admin backend at `127.0.0.1` or at the board's own LAN IP —
  neither address means what you'd expect from inside a container.
  What *does* work is a hostname App Lab itself provides in that
  container's `/etc/hosts` for reaching the host machine — in our
  testing this showed up as `msgpack-rpc-router`, aliased to the Docker
  default-bridge gateway IP. **Confirm this for your own setup** by
  running, from inside the container (`docker exec -it <container> sh`,
  or however App Lab lets you get a shell in it):
  ```bash
  cat /etc/hosts
  ```
  and using whichever hostname is aliased there (or fall back to the
  gateway IP itself, e.g. `172.17.0.1`, found via
  `docker inspect <container> --format '{{json .NetworkSettings.Networks}}'`
  on the host) as `WARDROBE_ADMIN_API_URL`.
- Separately, the kiosk's **web page** (viewed from a browser on your
  phone/laptop, at `http://<board LAN IP>:7000`) loads clothing photos
  directly from the admin backend — and a browser on a *different*
  machine has no route to that container-internal hostname at all. This
  needs the board's real, LAN-routable address, set via
  `WARDROBE_ADMIN_PUBLIC_URL`.

Set both as environment variables on the kiosk app (however App Lab lets
you configure an app's environment — check its settings/config UI), or
just edit the two defaults directly in `main.py`:

```python
ADMIN_API_BASE_URL    = os.environ.get("WARDROBE_ADMIN_API_URL", "http://msgpack-rpc-router:8080")
ADMIN_PUBLIC_BASE_URL = os.environ.get("WARDROBE_ADMIN_PUBLIC_URL", "http://192.168.1.50:8080")
```

Also make sure `SCAN_API_TIMEOUT_SECONDS` in `main.py` stays a good 5–10
seconds *longer* than `WARDROBE_SCAN_TIMEOUT_SECONDS` in the backend's
env file — the kiosk holds a POST request open for the whole camera scan
window, and if its own timeout is shorter, a real, working scan attempt
looks identical to a dead connection.

### 2c. Running it

Once both env values are correct and the sketch is uploaded, run the app
through App Lab. Point a browser at `http://<board LAN IP>:7000` — you
should see "Loading…" briefly, then either today's recommended outfit
or, if nothing's eligible yet (e.g. a fresh install with no clothing
tagged for today), an error screen explaining exactly why (see
"Troubleshooting" below).

To have the kiosk app launch automatically on boot, use App Lab's own
mechanism for setting a default app (as of this writing:
`arduino-app-cli properties set default <your-app-id>` — check current
Arduino documentation, since App Lab is actively evolving).

## How the recommendation engine works

`admin-backend/app/recommendation.py` is deliberately not a black box.
For each required slot (top, bottom — extend `REQUIRED_SLOTS` for
shoes/jackets/etc. later), it:

1. Filters clothing down to items eligible for today: right status
   (`active`), matching at least one of today's required tags (from the
   day's schedule rule, or a special event's tags if one's active), and
   within the item's stated temperature range / rain tolerance for
   today's weather.
2. Scores every eligible candidate on a transparent, additive scale:
   rotation (days since last worn), a same-day-of-week/Sunday-specific
   rotation bonus (so church polos rotate independently of everyday
   wear), a manual preference boost you can set per item, and a small
   randomized tie-break margin so it doesn't always pick the exact same
   "best" item when several are equally good.
3. Picks the highest-scoring item per slot, saves it as today's `Outfit`,
   and — critically — does **not** record it as worn. A wear is only
   ever written to `WearHistory` when you either press "Wear This" on
   the kiosk, or scan an item through the "Change" workflow. See Section
   20 of the original project brief (embedded in the code comments) for
   why this distinction matters for accurate cost-per-wear stats.

Run `python recommend_test.py` from `admin-backend/` for a CLI view of
the full scoring breakdown for today, useful for tuning without touching
hardware. `GET /api/recommendation/today?debug=true` returns the same
breakdown over HTTP; add `&force=true` to throw away today's cached
outfit and generate a fresh one (also available as a button on the
History admin page).

## Design decisions worth knowing before you extend this

- **Recommendation ≠ wear.** Don't be tempted to write to `WearHistory`
  anywhere except the two places that already do (`recommendation.py`'s
  confirm endpoint, and `scan.py`'s scan-confirm endpoint).
- **Cost-per-wear is never stored** — it's computed from
  `purchase_price / total_wears` at read time (see
  `app/serializers.py`/`app/stats.py`), so it's always consistent with
  the underlying wear count.
- **SQLite foreign keys are not enforced** by this project's default
  connection setup. `delete_clothing()` in `app/routers/clothing.py`
  explicitly checks for (and blocks on) real wear history, and cleans up
  any dangling *recommendation-only* references before deleting — if you
  add new tables that reference `clothing_items`, remember to handle
  deletion the same way, or you'll get orphaned rows instead of a clean
  `IntegrityError`.
- **The kiosk never opens a second camera handle.** The live scan
  preview (`GET /api/scan/preview.mjpg`) reads a frame cache that the
  actual scan loop (`qr_scanner.py`) fills while it's running — it does
  not, and must not, call `cv2.VideoCapture` itself.

## Troubleshooting

- **"No valid outfit could be generated for today"** — this is a
  legitimate result, not a bug: it means no clothing item currently has
  both the right tag(s) for today and a weather range that fits. Check
  `GET /api/recommendation/today` directly (it returns a `debug` object
  breaking down exactly which slot(s) came up empty) and either tag more
  items appropriately or adjust today's schedule rule.
- **Kiosk shows "Can't connect to ... (ConnectionError)"** — almost
  always the container-networking issue described in "2b" above. Verify
  `WARDROBE_ADMIN_API_URL` against your board's actual `/etc/hosts`
  inside the kiosk container.
- **Photos won't "update"** — check the browser's Network tab first: if
  the upload actually returned `200 OK`, the file did save; you're
  looking at a stale browser image cache, not a failed upload (every
  photo `<img>` in this project is already cache-busted via
  `?v=<updated_at>`, so this shouldn't recur, but it's the first thing
  to check if you add a new photo `<img>` somewhere).
- **Camera scan always says "camera not available"** — confirm the
  camera shows up as a device at all (`WARDROBE_CAMERA_INDEX`), and that
  nothing else (a second app, a leftover Python process) is already
  holding it open — this project always releases the camera in a
  `finally` block, but a crashed prior process can leave it locked until
  reboot.

## Extending this project

The schema and API were built with these in mind — none of them should
require a rewrite:

- Additional outfit slots (shoes, jackets, accessories) — extend
  `REQUIRED_SLOTS` in `recommendation.py` and the `clothing_type`'s
  `outfit_slot` values.
- A dedicated schedule-rule editor in the web UI (currently seeded via
  `seed.py`/edited directly in the database) — the `ScheduleRule`/
  `SpecialEvent` tables already support it.
- Tunable recommendation weights from the web UI — `app/settings.py`'s
  `Setting` key/value table already backs every weight the scorer uses;
  it just doesn't have a settings-editor page yet.
- Multiple users/wardrobes, laundry/donated status, NFC instead of QR,
  a mobile view of the kiosk page — all additive changes to the existing
  schema, not structural ones.
