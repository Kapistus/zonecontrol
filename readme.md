Zone Control

A live, GPS-based area-control game for airsoft. Two teams (Red and Yellow) compete to physically stand inside whichever numbered zone is currently "active" on the field; each second spent inside it while it's active adds to that team's control-time score for that zone. The active zone rotates automatically on a timer, admin runs the round from a phone like everyone else, and every device sees a live map of the game state — but only its own team's positions, never the enemy's.

Runs as a single static page (playable straight from GitHub Pages) backed by Supabase for shared state, auth, and real-time sync.

Features
PIN-based login — one PIN each for Red, Yellow, and Admin. PINs are checked server-side and never appear anywhere in the shipped code.
Real fog of war — enforced by Postgres Row Level Security, not just hidden in the UI. A player's device can only ever read its own team's live positions; a modified client can't work around it.
9 numbered zones, each a fixed 10m-diameter circle, with custom display names, shown on a live satellite map (Google Maps).
Automatic zone rotation — runs entirely server-side via a scheduled Postgres job (pg_cron), not dependent on any single phone staying open or connected.
Live per-team control-time scoreboard, per zone, synced to every connected device in real time.
Whole-round elapsed timer, visible to everyone, pauses/resumes/resets correctly with the game state.
Admin controls in a slide-out drawer: Start / Pause / Resume / End, zone-duration adjustment (1-minute steps), and the option for admin to join a team and play too.
Player trail + heading indicator — a faint line from a player's last position to their current one, so teammates can see which direction someone's moving, not just a static dot.
GPS-only positioning for every role, with no way to manually set or fake a position from within the app.
Automatic cleanup of stale positions (a device that disconnects, crashes, or loses signal drops off the map within about a minute) and an instant wipe of live positions the moment admin ends a round.
How it's built
Frontend: a single index.html file — vanilla JS, no build step, no framework. Deployed as a static site via GitHub Pages.
Map: Google Maps JavaScript API, using a free Maps Demo Key (no billing account required).
Backend: Supabase — Postgres, Row Level Security, Realtime, anonymous Auth, and pg_cron for the server-side zone rotation and cleanup jobs.

Nothing runs on a server you have to maintain. GitHub Pages and Supabase are both always-on hosted services.

Setup
1. Supabase project

Create a project at supabase.com, then:

Run schema.sql — paste the whole file into SQL Editor and run it. It creates every table, function, and policy, and schedules the two pg_cron jobs (zone rotation and stale-position cleanup). Safe to re-run at any time if needed.
Enable Anonymous Sign-Ins — Authentication → Sign In / Providers → toggle it on. This can't be done via SQL. Without it, the PIN screen won't be able to create a session at all.
Confirm pg_cron is active — the schema tries to enable it automatically; if that line errors with a permissions message, enable it manually under Database → Extensions instead, then re-run just the bottom section of schema.sql.
Set your real PINs — the schema ships with placeholder PINs (110011 / 220022 / 999999 for Red / Yellow / Admin). Change them before a real game:
sql
   update team_pins set pin = 'your-new-pin' where team = 'red';
2. Google Maps key

Go to developers.google.com/maps/demo-key, sign in with a Google account (no card needed), and grab a Demo Key. Paste it into index.html, replacing YOUR_DEMO_KEY_HERE near the top of the <head>.

The Demo Key is meant for prototyping rather than guaranteed indefinite production use, and is subject to its own daily quota — if you outgrow it, the same code works with a standard billed key, just add billing to the same key later.

3. Configure your field

Near the top of index.html's <script> block:

CONFIG.SUPABASE_URL / CONFIG.SUPABASE_ANON_KEY — from your Supabase project's API settings. The anon/publishable key is safe to expose publicly; it's meant to ship in client code, and Row Level Security is what actually gates access.
CONFIG.FIELD_CENTER — where the map opens by default. Set this to your actual field, not a zone's coordinates.
ZONES array — each zone's id, display name, and lat/lng. Right-click a point on Google Maps to copy its coordinates. Zone count is currently hardcoded to 9 in both the client (ZONES.length) and the server-side rotation function (v_zone_count in rotate_zone_if_due()) — if you add or remove a zone, update both.
4. Deploy

Rename the HTML file to index.html if it isn't already, push it to this repo, then enable GitHub Pages: Settings → Pages → Source: Deploy from a branch → main / (root). Your game will be live at https://<username>.github.io/<repo>/.

Playing
Open the URL on a phone, enter a team PIN (or the admin PIN).
Admin: tap the ☰ ADMIN button (top-left of the map) to open the control drawer — Start the round, adjust zone duration, pause/resume/end, and optionally join Red or Yellow to play as well as run the game.
Players: just keep the app open and move around the field. The active zone is highlighted on the map; standing inside it while it's active earns your team control time.
The ☰ ZONES button (also top-left) shows every zone's name and live control-time totals for both teams, collapsed by default.
Known limitations
Testing requires real GPS hardware. There's no simulated/manual position option anywhere in the app (removed intentionally — it was a cheating vector). Desktop browsers fall back to inaccurate IP-based location, so meaningful testing needs an actual phone outdoors.
Zone count is duplicated between the client (ZONES array) and the database (rotate_zone_if_due()'s hardcoded count) rather than living in one shared source of truth. A future improvement would move zone definitions into their own Supabase table.
Google Maps Demo Key is officially for prototyping, not guaranteed production use, and carries an unpublished daily quota.
