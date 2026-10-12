# Zona Calcio Lab clip format

A small JSON format to exchange tagged moments of a football match between
Zona Calcio Lab tools: **PlayTrace** (live tagging), **MatchRazor** (automated
video analysis) and **ClipTrimmer** (clip export).

Each tool keeps its own internal files and reads/writes this format at the
edges, so a match tagged in PlayTrace can be cut in ClipTrimmer or reviewed in
MatchRazor without a dedicated importer for every pair of tools.

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

All fields are optional.

## `sources`

```json
{"id": "main", "uri": "https://www.youtube.com/watch?v=XXXXXXXXXXX", "kickoffs": {"1": 312.0, "2": 3611.5}}
```

| Field | Required | Description |
|---|---|---|
| `id` | yes | Unique within the file, referenced by clips |
| `uri` | no | Absolute local path or YouTube link (`https://www.youtube.com/watch?v=…`). Missing when there is no video yet (live tagging) |
| `kickoffs` | no | Kick-off position of each period in this video, in seconds: keys `"1"`, `"2"` and, if needed, `"3"`, `"4"` (extra time) |
| `title` | no | Human-readable label |

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
| `extra` | no | Object with tool-specific fields, see below |

A clip has either `t` or both `start` and `end`, never both kinds.

### `extra`

Fields that only one tool understands go in `extra`, under the tool's name, so
other tools can ignore them and still write them back unchanged:

```json
"extra": {
  "matchrazor": {"possesso": "osservata", "descrizione": "Uscita bassa con il terzino"},
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

## Mapping from the current formats

| Concept | PlayTrace export (schema 2) | MatchRazor clip JSON | ClipTrimmer playlist 1 |
|---|---|---|---|
| Match | `match_info` | `partita` | — |
| Video | — | master video + `offset` | `clips[].source` |
| Instant | event, `time_period` "23:45 1T" → `period` 1, `t` 1425 | `t` | — |
| Section | — | `start`, `end` | `start`, `end` |
| Label | `tag_name` | `fase` | `name` |
| Team | `team` | (observed team) | — |

## License

The specification, the JSON Schema and the examples are licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE): any
tool may implement the format, crediting Zona Calcio Lab.

© 2026 Roberto Valtancoli / Zona Calcio Lab
