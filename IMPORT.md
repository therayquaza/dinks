# The dinks data file

One JSON file carries everything dinks stores. `GET /api/export` produces one and
`POST /api/import` consumes one, so an export is always a valid import and the
app's own history is never trapped inside it.

## Shape

```json
{
  "format": "dinks-import",
  "version": 1,
  "source": "flo",
  "exported_at": "2026-09-29T12:00:00Z",
  "periods": [
    { "started_on": "2021-05-04", "ended_on": "2021-05-08", "flow": "heavy", "notes": "",
      "days": [
        { "date": "2021-05-04", "flow": "medium" },
        { "date": "2021-05-08", "flow": "light" }
      ] }
  ],
  "symptoms": [
    { "recorded_on": "2021-05-16", "kind": "Acne", "severity": 3, "notes": "" },
    { "recorded_on": "2021-09-18", "kind": "note", "severity": 3, "notes": "Mood: Calm, Sad." }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `format` | no | Must be `"dinks-import"` when present. A file naming any other format is rejected. |
| `version` | no | Must be `1` or lower. A newer version is rejected rather than half-read. |
| `source` | no | Free-form provenance, e.g. `"flo"`. Informational. |
| `exported_at` | no | RFC 3339 timestamp of when the file was produced. Ignored by the server. |
| `periods` | yes | May be empty. Must be an array. |
| `symptoms` | yes | May be empty. Must be an array. |

`format` and `version` are optional so a hand-written file works, but anything
that *generates* files should write them: they are what lets a future revision be
recognised instead of silently misread.

### Periods

| Field | Required | Notes |
|---|---|---|
| `started_on` | yes | `YYYY-MM-DD`. |
| `ended_on` | no | `YYYY-MM-DD`, on or after `started_on`. Omit or leave empty for a period that has not ended. |
| `flow` | no | `light`, `medium`, `heavy`, or any string. Defaults to `unknown`. This is the period's **summary** flow. |
| `days` | no | Per-day flow. See below. |
| `notes` | no | Free text. |

A period is the only unit of cycle data: cycle length is derived from the gaps
between consecutive `started_on` values, so one entry per period is the whole
model.

#### Per-day flow

Flow usually is not constant across a period — it is typically lighter on the
first and last days. `days` records that, as `{ "date": "YYYY-MM-DD", "flow": "…" }`
entries.

`flow` stays authoritative: **any day not listed in `days` uses the period's
summary `flow`.** A `days` entry that matches the summary is redundant and is
normally left out, which also means a file can express "the same flow
throughout" simply by omitting `days`.

Each `date` must fall inside the period — between `started_on` and `ended_on`, or
at or after `started_on` for a period that has not ended. A day outside it, a
malformed date, or the same date twice is rejected before anything is written.

`days` is optional and additive: a file without it imports exactly as it did
before, and a period with no per-day detail behaves as it always did.

### Check-ins

| Field | Required | Notes |
|---|---|---|
| `recorded_on` | yes | `YYYY-MM-DD` — the day the entry applies to, not the day it was typed. |
| `kind` | yes | Non-empty. Any string; dinks does not constrain the vocabulary. |
| `severity` | yes | Integer 1–5. |
| `notes` | no | Free text. |

`kind: "note"` is the convention for a day's free text, and it is what the UI
writes when you save a note. Moods live inside that text as `Mood: Sad, Tired.`
so a day holds one note rather than one record per mood.

#### Sex, libido and custom indicators

`kind` is free-form, so three groups of entries ride on the same records rather
than on new tables:

| Prefix | What it is |
|---|---|
| `Sex — protected`, `Sex — unprotected`, `Sex — oral`, `Sex — toys`, `Orgasm` | Intercourse and orgasm, as the Flo export records them. |
| `Libido — high`, `Libido — moderate`, `Libido — low`, `Libido — none` | Sexual desire for that day. At most one level per day. |
| `track:<key>` | A value for a custom indicator the member defined in Preferences, such as `track:body_weight`. The key is the slugified label; the number goes in `notes`, and `severity` is unused (1). |

Because these are ordinary check-ins they need no special handling in an import
file: they round-trip through export, appear in the calendar and stats, and are
covered by the same validation as any other entry.

**These kinds are never shared with a partner by default.** A partner link
starts with the status fields only, and sex, libido and notes are opt-in per
person — see the share scope on a partner link.

`ids` may appear on records — an export includes them — and are ignored; the
server assigns them.

## Importing

```
POST /api/import?mode=merge      # default
POST /api/import?mode=replace
```

| Mode | Effect |
|---|---|
| `merge` | Adds every record the account does not already have. Re-running the same file changes nothing, so a retry after a failed request is safe. |
| `replace` | Deletes every existing period and check-in, then writes the file. The swap is transactional. |

The response reports what actually happened, with skips counted rather than
silently dropped:

```json
{
  "mode": "merge",
  "periods_imported": 65,
  "periods_skipped": 0,
  "symptoms_imported": 170,
  "symptoms_skipped": 0
}
```

A record counts as already present when a period starts on the same day, or a
check-in has the same `recorded_on` and `kind` pair. A file that duplicates
itself is rejected rather than collapsed.

## Guarantees and limits

- **All or nothing.** Every record is validated before anything is written, so
  one bad date leaves the account untouched instead of half-migrated. Errors
  name the offending index (`periods[7]: ended_on must be on or after started_on`).
- **4 MiB** per request, and **50 000** records per file.
- `kind` is free-form. dinks will store and display a kind it has never seen;
  it simply has no emoji for it.

## Coming from another app

dinks has no converter for every tracker. `backend/cmd/flo2dinks` converts a Flo
account export:

```sh
cd apps/dinks/backend
go run ./cmd/flo2dinks -in flo-export.json -out dinks-import.json
```

It writes a file the import endpoint accepts directly, and prints what it could
not carry across — a conversion is never silently lossy. Writing a converter for
another tracker means mapping its data onto the two arrays above; Flo's mapping
is in `cmd/flo2dinks/main.go` as a worked example.
