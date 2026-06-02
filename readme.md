# Scorebook

**A browser-based baseball game scoring and pitch tracking application built for the Guelph Royals (Canadian Baseball League).**

Scorebook is a digital replacement for the physical scorebook — combining live game scoring (à la GameChanger) with full pitch-by-pitch tracking including pitch type, pitch location, and batted ball placement. At the end of every game, data exports directly to a Google Sheet matching the Royals' existing pitch log format exactly.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Features](#features)
4. [Data Schema](#data-schema)
5. [Google Sheets Export Format](#google-sheets-export-format)
6. [App Flow](#app-flow)
7. [UI Components](#ui-components)
8. [Supabase Database Schema](#supabase-database-schema)
9. [Google Sheets Integration](#google-sheets-integration)
10. [Pitch Location System](#pitch-location-system)
11. [Spray Chart System](#spray-chart-system)
12. [Outcome Values](#outcome-values)
13. [Pitch Types](#pitch-types)
14. [Future Considerations](#future-considerations)

---

## Project Overview

**Name:** Scorebook  
**Platform:** Web app (browser)  
**Primary users:** Guelph Royals coaching staff and analysts  
**Purpose:** Live game scoring + pitch tracking with direct export to the Royals' Google Sheets pitch log

### The Problem

GameChanger handles live scoring well but lacks pitch-level detail (type, location). The Royals maintain a separate Google Sheet with full pitch data that has to be entered manually after games. Scorebook combines both workflows into one — score the game live, track every pitch in real time, and export the complete dataset automatically when the game ends.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth (for app login) |
| Export Auth | Google OAuth 2.0 (user logs in with their Google account) |
| Export Target | Google Sheets API v4 |
| Hosting | TBD (Vercel or Netlify recommended) |

### Key decisions
- **Supabase (fresh project):** All game and pitch data persists to a new dedicated Supabase project. This enables multi-device access (e.g. scorer on iPad, analyst on laptop simultaneously).
- **Google OAuth:** The user authenticates with their own Google account to authorize Sheets writes. No hardcoded service account credentials.
- **React:** Component-based architecture maps cleanly to the app's modular UI (strike zone, field diagram, scorecard, pitch log).

---

## Features

### Game Setup
- Enter home team and away team names
- Enter game date
- Enter starting lineups (batter order + pitcher for each team)
- Record batter side (R/L) and pitcher side (R/L) per player

### Live Scoring (Pitch-by-Pitch)
- Display current game state: inning, half inning (Top/Bottom), outs, balls, strikes, count, score
- Display current batter and pitcher with their handedness
- Display current baserunner state (runners on 1B, 2B, 3B)
- For each pitch, scorer inputs:
  1. **Pitch type** — FB or Offspeed (button select)
  2. **Pitch location** — click exact spot on interactive strike zone image (X/Y coordinates recorded)
  3. **Outcome** — select from full outcome list (button/dropdown)
  4. If outcome is a **ball in play**: drag dot on overhead field diagram to record batted ball landing spot (Spray Chart X/Y), then select **Quality of Contact**

### Auto-Derived Fields
- **Count** — derived from cumulative Balls/Strikes for the at-bat (e.g. "2-1"), not manually entered
- **Half Inning** — derived from current inning state (Top/Bottom)

### Scorecard View
- Live traditional box score updating in real time
- Running pitch log table showing all pitches in the current game

### Post-Game Export
- On game completion, user clicks Export
- OAuth flow prompts user to log in with Google (if not already authenticated)
- Full pitch log exports to a Google Sheet in the exact column format of the Royals' existing spreadsheet
- Each pitch becomes one row; column order is preserved exactly

---

## Data Schema

Every row in the exported sheet (and in Supabase) represents **one pitch**. The 23 columns, in order:

| # | Column | Type | Source |
|---|---|---|---|
| 1 | Half Inning | String | Derived (Top/Bottom from inning state) |
| 2 | Inning | Integer | Live game state |
| 3 | Outs | Integer | Live game state (0–3) |
| 4 | Balls | Integer | Live count state (0–3) |
| 5 | Strikes | Integer | Live count state (0–2) |
| 6 | Count | String | Derived from Balls-Strikes (e.g. "2-1") |
| 7 | Batter_Team | String | Game setup |
| 8 | Pitcher_Team | String | Game setup |
| 9 | Date | Date | Game setup |
| 10 | Home Team | String | Game setup |
| 11 | Away Team | String | Game setup |
| 12 | Time To Plate (sec) with Man on First | Float | Manual entry (optional) |
| 13 | Batter | String | Lineup |
| 14 | Pitcher | String | Lineup |
| 15 | Batter_Side | String | R or L |
| 16 | Pitcher_Side | String | R or L |
| 17 | Pitch_Type | String | FB or Offspeed |
| 18 | Outcome | String | Selected from outcome list |
| 19 | Quality of Contact | String | Ground Ball / Line Drive / Pop Up / Fly Ball (only on ball in play) |
| 20 | Spray Chart | String/Float | X/Y from drag on field diagram (ball in play only) |
| 21 | Runners | String | Baserunner state at time of pitch (e.g. "1B", "1B,3B", "Empty") |
| 22 | Pitch_Location_X | Float | X coordinate from click on strike zone image |
| 23 | Pitch_Location_Y | Float | Y coordinate from click on strike zone image |

---

## Google Sheets Export Format

- Export appends rows to an existing Google Sheet (the Royals' pitch log)
- Row 1 is the header row (already exists in the sheet — export does NOT re-write headers)
- Each pitch appends as one new row in column order above
- Columns map 1:1 to the schema — no transformation, no renaming
- Export is triggered manually by the scorer at end of game
- Auth: Google OAuth 2.0, user signs in with their Google account

---

## App Flow

```
1. GAME SETUP
   └── Enter teams, date, lineups, handedness
   
2. LIVE SCORING SCREEN
   ├── Game state panel (inning, outs, count, score, runners)
   ├── Current matchup (batter vs pitcher)
   └── Pitch entry flow:
       ├── Step 1: Select pitch type (FB / Offspeed)
       ├── Step 2: Click location on strike zone → X/Y recorded
       ├── Step 3: Select outcome
       └── If Ball in Play:
           ├── Step 4: Drag dot on field diagram → Spray Chart X/Y recorded
           └── Step 5: Select Quality of Contact

3. PITCH LOG TABLE
   └── Live scrolling table of all pitches this game

4. BOX SCORE
   └── Traditional inning-by-inning score grid, updating live

5. END GAME
   └── Click "Export to Google Sheets"
       ├── OAuth login (if not already authed)
       └── Appends all pitch rows to Royals sheet
```

---

## UI Components

### Strike Zone
- Interactive SVG/Canvas component
- Visually matches the Royals' zone image:
  - Outer yellow area = chase/waste zone
  - Pink/beige area = shadow/borderline zone
  - Dashed green border = strike zone boundary
  - Purple 4x4 grid = heart of the zone (16 cells)
- User clicks exact spot → dot plotted visually → X/Y coordinates stored
- Coordinates normalized to a consistent scale (0.0–1.0 range) regardless of screen size

### Field Diagram (Spray Chart)
- Overhead baseball diamond SVG
- Triggered only when outcome = ball in play
- User drags a dot to the landing spot of the batted ball
- Final position stored as X/Y coordinates (Spray_Chart column)
- Matches GameChanger-style drag interaction

### Pitch Entry Panel
- Pitch type: two large buttons (FB / Offspeed)
- Outcome: grid of buttons covering all 29 outcomes, grouped logically:
  - **Balls:** Ball, Walk, Intentional Walk, Hit By Pitch
  - **Strikes:** Called Strike, Swinging Strike, Foul
  - **Strikeouts:** Strikeout Looking, Strikeout Swinging, Dropped Third Strike Looking, Dropped Third Strike Swinging
  - **Balls in Play:** Single, Double, Triple, Home Run, Groundout, Double Play, Triple Play, Popout, Flyout, Lineout, Sacrifice Bunt, Sacrifice Fly
  - **Baserunning/Other:** Pickoff, Caught Stealing, Error, Truncated Out, Batter Interference, Catcher Interference, Additional Out

### Game State Panel
- Current inning + half (Top/Bottom)
- Outs (0/1/2)
- Count (Balls-Strikes)
- Score
- Baserunner indicator (diamond graphic with lit bases)

---

## Supabase Database Schema

### `games`
```sql
id uuid PRIMARY KEY
date date
home_team text
away_team text
created_at timestamptz
```

### `players`
```sql
id uuid PRIMARY KEY
game_id uuid REFERENCES games(id)
name text
team text
batting_order integer
batter_side text  -- R or L
pitcher_side text -- R or L
role text         -- batter | pitcher
```

### `pitches`
```sql
id uuid PRIMARY KEY
game_id uuid REFERENCES games(id)
half_inning text
inning integer
outs integer
balls integer
strikes integer
batter_team text
pitcher_team text
date date
home_team text
away_team text
time_to_plate float
batter text
pitcher text
batter_side text
pitcher_side text
pitch_type text
outcome text
quality_of_contact text
spray_chart_x float
spray_chart_y float
runners text
pitch_location_x float
pitch_location_y float
created_at timestamptz
```

---

## Google Sheets Integration

- **Auth method:** Google OAuth 2.0
- **Scopes required:** `https://www.googleapis.com/auth/spreadsheets`
- **Flow:**
  1. User clicks "Export to Google Sheets"
  2. If not authed: Google OAuth popup → user grants access
  3. User pastes or selects target Sheet ID
  4. App appends all pitch rows from current game to the sheet
  5. Success confirmation shown
- **Library:** `gapi` (Google API JS client) or `googleapis` Node package
- **Append method:** `spreadsheets.values.append` with `valueInputOption: RAW`

---

## Pitch Location System

- Strike zone rendered as interactive SVG
- Zone layers (outside to inside):
  - Chase zone (outermost)
  - Shadow/borderline zone
  - Strike zone (dashed border)
  - Heart of zone (4x4 purple grid = 16 cells)
- On click: dot rendered at click position
- Coordinates stored as normalized floats (0.0–1.0) relative to the full zone image dimensions
- Coordinate origin: top-left (0,0), bottom-right (1,1)

---

## Spray Chart System

- Triggered only when outcome is a ball in play
- Overhead field SVG (standard baseball diamond view from above)
- A draggable dot appears when ball-in-play outcome is selected
- User drags dot to landing spot
- Coordinates stored as normalized floats (0.0–1.0)
- After placing dot, Quality of Contact buttons appear:
  - Ground Ball
  - Line Drive
  - Pop Up
  - Fly Ball

---

## Outcome Values

Full list of 30 valid outcome values:

```
Ball
Walk
Foul
Called Strike
Swinging Strike
Strikeout Looking
Strikeout Swinging
Dropped Third Strike Looking
Dropped Third Strike Swinging
Single
Double
Triple
Home Run
Groundout
Double Play
Triple Play
Popout
Flyout
Lineout
Sacrifice Bunt
Sacrifice Fly
Intentional Walk
Hit By Pitch
Error
Pickoff
Caught Stealing
Truncated Out
Batter Interference
Catcher Interference
Additional Out
```

### Ball-in-Play outcomes (trigger Spray Chart + Quality of Contact):
```
Single, Double, Triple, Home Run, Groundout, Double Play, Triple Play,
Popout, Flyout, Lineout, Sacrifice Bunt, Sacrifice Fly, Error
```

---

## Pitch Types

```
FB        (Fastball)
Offspeed  (All breaking/off-speed pitches)
```

---

## Future Considerations

These are out of scope for v1 but natural extensions:

- **Pitch velocity** field
- **More granular pitch types** (4FB, 2FB, SL, CB, CH, CT, Splitter)
- **Umpire zone tracking** (called strike vs actual strike zone)
- **Pitcher heat maps** from Pitch_Location_X/Y data
- **Per-batter spray chart overlays**
- **Multi-game season dashboard**
- **CBL-wide analytics** (if other teams adopt Scorebook)
- **Mobile PWA** for iPad use on the field
- **Real-time collaboration** (two scorers on same game)
- **Integration with Data Diamond** (data-diamond.ca) — auto-push game data to the RE24 engine and xContact model pipeline