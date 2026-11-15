# Zona Calcio Lab clip format

A small JSON format to exchange tagged moments of a football match between
Zona Calcio Lab tools, such as **PlayTrace** (live tagging) and the tools for
video analysis and clip export.

Each tool keeps its own internal files and reads/writes this format at the
edges, so a match tagged in PlayTrace can be cut into clips or reviewed in a
video analysis tool without a dedicated importer for every pair of tools.

- Format id: `zcl-clips`
- Current version: **1**
- File extension: `.zclclips.json` (any `.json` is accepted)
- Encoding: UTF-8
- JSON Schema: [`schema/zcl-clips-1.schema.json`](schema/zcl-clips-1.schema.json)
- Examples: [`examples/`](examples)

## Overview

```json
{
  "format": "zcl-clips",
  "version": 1,
  "generator": {"name": "PlayTrace", "version": "1.4.0"},
  "match": {...},
  "sources": [...],
  "clips": [...]
}
```

| Field | Required | Description |
|---|---|---|
| `format` | yes | Always `"zcl-clips"` |
| `version` | yes | Integer, the format version (see [Versioning](#versioning)) |
| `generator` | no | Tool that wrote the file: `name`, `version` |
| `match` | no | Match information |
| `sources` | yes | Videos (or the live recording) the clips refer to, may be empty |
| `clips` | yes | Tagged moments, in the order they should be shown |

## Times

All times are **numbers of seconds** (decimals allowed), never formatted
strings, so tools do not need to parse `MM:SS`, `45+2` or similar.

A clip's times are measured in one of two ways:

- **Video time** (no `period` on the clip): seconds from the start of the
  source video.
- **Period time** (`period` set): seconds from the kick-off of that period.
  Added time simply continues: 45+2 in the first half is `2820`.

Period time is what a live tagging app knows (PlayTrace has no video). To turn
it into video time, a tool needs the kick-off position of that period in the
video, from `sources[].kickoffs`; if it is missing, the tool asks the user.

```
video_time = sources[source].kickoffs[period] + clip_time
```

## `match`

| Field | Description |
|---|---|
| `competition` | Competition name |
| `date` | `YYYY-MM-DD` |
| `venue` | Venue |
| `home`, `away` | Team names |
| `period_duration` | Nominal minutes per period (e.g. `45`) |
| `observed` | `"home"` or `"away"`: the team the analysis is about, when there is one. Tools that think in terms of observed/opponent map it to `team` |

All fields are optional.

## `sources`

```json
{"id": "main", "uri": "https://www.youtube.com/watch?v=XXXXXXXXXXX", "kickoffs": {"1": 312.0, "2": 3611.5}, "ends": {"1": 3105.0, "2": 6498.0}}
```

| Field | Required | Description |
|---|---|---|
| `id` | yes | Unique within the file, referenced by clips |
| `uri` | no | Absolute local path or YouTube link (`https://www.youtube.com/watch?v=…`). Missing when there is no video yet (live tagging) |
| `kickoffs` | no | Kick-off position of each period in this video, in seconds: keys `"1"`, `"2"` and, if needed, `"3"`, `"4"` (extra time) |
| `ends` | no | End position of each period in this video (final whistle, added time included), in seconds, same keys as `kickoffs`. `ends[p] > kickoffs[p]` when both are present |
| `title` | no | Human-readable label |

`ends` is for tools that work on whole periods, for example cutting the
halves out of a full match recording or skipping the half-time break.

A file with clips from several videos lists several sources.

## `clips`

Each clip is either an **instant** or a **section**:

```json
{"source": "main", "period": 1, "t": 1425.0, "pre": 5, "post": 3, "label": "Shot", "team": "home"}
{"source": "main", "start": 763.5, "end": 781.0, "label": "Build-up", "name": "12.43 build-up"}
```

| Field | Required | Description |
|---|---|---|
| `source` | yes | `id` of a source |
| `period` | no | `1`, `2`, `3`, `4`. When set, times are period time; otherwise video time |
| `t` | instant | Moment of the event |
| `pre`, `post` | no | Instant only: seconds to keep before and after `t` when it becomes a clip. Tools use their own default when missing (PlayTrace XML uses 5 and 3) |
| `start`, `end` | section | Section bounds, `end > start` |
| `label` | no | Category of the moment (tag, phase), e.g. `"Shot"` |
| `team` | no | `"home"`, `"away"` or missing (not attributed) |
| `name` | no | Free name, used for example as the output file name |
| `note` | no | Free text |
| `favorite` | no | `true` when the clip is marked as a favourite. Missing means not a favourite |
| `restart` | no | Restart the clip starts from: `"kick_off"`, `"goal_kick"`, `"throw_in"`, `"free_kick"`, `"corner"`, `"penalty"`. Tools show their own translated labels |
| `origin` | no | Pitch cell where the action starts, see [Pitch cells](#pitch-cells) |
| `extra` | no | Object with tool-specific fields, see below |

A clip has either `t` or both `start` and `end`, never both kinds.

### Pitch cells

`origin` places the start of the action on a 4 × 4 grid, seen by the **reference
team** attacking towards the opponent's goal:

```json
"origin": {"column": 3, "row": "B"}
```

- **Reference team**: `match.observed` when present, otherwise the home team.
- `column` `1`-`4` along the pitch: `1` the 26 m in front of the reference team's
  own goal (about 10 m beyond its penalty area), `2` and `3` the rest of its own
  half and of the opponent's half, split by the halfway line, `4` the 26 m in front
  of the opponent's goal.
- `row` `"A"`-`"D"` across the pitch, left to right for the reference team facing
  the opponent's goal: `A` and `D` outside the width of the penalty area, `B` and
  `C` inside it, split by the middle of the pitch.

The cell does not depend on the camera: a tool that reads cells on the screen
converts them with the attacking direction of the reference team in that period.
Coarser areas follow from the cell: thirds `1 | 2+3 | 4`, halves `1+2 | 3+4`,
lanes `A | B+C | D`, sides `A+B | C+D`.

### Period of video-time clips

A clip in video time has no `period`. When a tool needs it (to show `2T 05:30` or
to filter by half), it takes the period whose `kickoffs`-`ends` range of the
clip's source contains the clip start; without `ends`, the last kick-off before
the start. A clip outside every range (half-time break) has no period.

### `extra`

Fields that only one tool understands go in `extra`, under the tool's name, so
other tools can ignore them and still write them back unchanged:

```json
"extra": {
  "videoanalysis": {"possession": "observed", "description": "Build-up through the full-back"},
  "playtrace": {"tag_id": 3, "timestamp_absolute": "12/10/26 15:23:45"}
}
```

A tool must keep `extra` entries it does not know when it rewrites a file.

## Versioning

- Adding optional fields does **not** change the version: readers must ignore
  fields they do not know.
- Removing or changing the meaning of a field increases `version`. A reader
  refuses files with a version higher than the one it supports, with a clear
  message.

## Mapping from PlayTrace

| Concept | PlayTrace export (schema 2) | zcl-clips |
|---|---|---|
| Match | `match_info` | `match` |
| Video | — | `sources[]` without `uri` |
| Instant | event, `time_period` "23:45 1T" | `period` 1, `t` 1425 |
| Label | `tag_name` | `label` |
| Team | `team` | `team` |
| Other fields | `tag_id`, `timestamp_absolute` | `extra.playtrace` |

## License

The specification, the JSON Schema and the examples are licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE): any
tool may implement the format, crediting Zona Calcio Lab.

© 2026 Roberto Valtancoli / Zona Calcio Lab
