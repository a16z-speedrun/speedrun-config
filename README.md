# speedrun-config

One small, public source of truth for the dates and versions that Speedrun's
**founder-facing surfaces** read at runtime — so we can re-date a cohort or ship
a tool update by editing one file here, with no app rebuild or reinstall.

It is consumed by:

- **[`sr-chrome-extension`](https://github.com/a16z-speedrun/sr-chrome-extension)**: the Demo Day new-tab countdown
- **[`speedrun-statusline`](https://github.com/a16z-speedrun/speedrun-statusline)**: the Claude Code status line
- **[`sr-raise`](https://github.com/a16z-speedrun/sr-raise)**: the fundraising coach, which reads
  a cohort's `kickoff`, `progress_day`, `demo_day` and `raise_ends` to work out which
  phase of the raise a founder is in and scope its advice to that phase

Each fetches [`cohort.json`](./cohort.json) (raw, public, no auth), caches it
locally, and **falls back to its own baked-in defaults** if the fetch fails —
so a network hiccup never breaks a countdown.

## `cohort.json`

```jsonc
{
  "cohorts": {
    "speedrun": {                                  // the CURRENT Speedrun cohort (founder-facing
      "cohort": "SR008",                           // alias; repointed to the next cohort once
      "milestone": "Demo Day",                     // the previous Demo Day has passed)
      "progress_day": "2027-03-17T09:00:00-07:00", // optional; splits the pre-Demo-Day
                                                   // stretch into "build" and "ramp"
      "demo_day": "2027-04-13T09:00:00-07:00",     // ISO-8601 with explicit offset
      "date_label": "April 13, 2027"
    },
    "alpha": {                                     // Speedrun Alpha Showcase
      "cohort": "Alpha",
      "milestone": "Showcase",
      "demo_day": "2027-03-01T09:00:00-08:00",
      "date_label": "March 1, 2027"
    },
    "SR007": {                                     // one entry per Speedrun cohort, keyed by
      "cohort": "SR007",                           // its cohort code; sr-raise reads these
      "milestone": "Demo Day",
      "progress_day": "2026-09-08T09:00:00-07:00",
      "demo_day": "2026-10-06T09:00:00-07:00",
      "raise_ends": "2026-12-06T23:59:00-08:00",   // optional; last day this cohort's own
                                                   // post-Demo-Day timeline applies
      "date_label": "October 6, 2026"
    },
    "SR008": {
      "cohort": "SR008",
      "milestone": "Demo Day",
      "kickoff": "2027-02-02T09:00:00-08:00",      // optional; the day the cohort starts
      "progress_day": "2027-03-17T09:00:00-07:00",
      "demo_day": "2027-04-13T09:00:00-07:00",
      "date_label": "April 13, 2027"
    },
    "SR009": {                                     // a cohort the program has only dated
      "cohort": "SR009",                           // Demo Day for; kickoff + progress_day
      "milestone": "Demo Day",                     // follow when they are set
      "demo_day": "2027-10-05T09:00:00-07:00",
      "date_label": "October 5, 2027"
    }
  },
  "statusline": {
    "latest_version": "1.1.0"                      // drives the status line's "⬆ update" nudge
  }
}
```

Each consumer reads only what it needs: the extension reads `cohorts.speedrun` and
`cohorts.alpha` by key (it has a toggle); the status line reads only `cohorts.speedrun`
plus `statusline.latest_version`; sr-raise resolves a company's calendar by the
company's **own cohort code** (`cohorts.SR007` for an SR007 company, `cohorts.SR008`
for an SR008 company) and reads `cohorts.speedrun` only for an asker with no cohort
on file. No consumer iterates every key under `cohorts`, so adding a per-cohort entry
is invisible to the extension and the status line.

**Always use an explicit UTC offset** on every date (`-07:00` for PDT, `-08:00` for PST -- Pacific is on PST from early November to mid-March) so the
countdown is identical on every machine in every timezone.

`progress_day` is **optional**. Where it is set, it marks the point a cohort turns
from building toward the raise, so the two halves of the run-up can be told apart —
before it, "stay heads-down"; after it, "start the sequencing arc." A cohort without
it simply has no such split, and the field is absent for cohorts whose program has no
equivalent milestone (Alpha, whose milestone is a Showcase).

## Per-cohort entries

`cohorts.speedrun` and `cohorts.alpha` are **aliases**: `speedrun` always points at the
current (or next) Speedrun cohort and is repointed once a Demo Day has passed. Their
shape never changes, because the extension and the status line read them by key.

Repointing `speedrun` must not drag the previous cohort onto the next cohort's
calendar. An SR007 company is still in its post-Demo-Day raise for about two months
after Demo Day, while an SR008 company is counting down to kickoff; the two timelines
never mix and neither applies globally. So every Speedrun cohort also gets its **own
entry keyed by cohort code** (`cohorts.SR007`, `cohorts.SR008`, ...), and sr-raise
reads the entry that matches each company's cohort.

| Field | Required | Meaning |
|---|---|---|
| `cohort` | yes | The cohort code (`SR007`, `SR008`, `Alpha`). For a per-cohort entry it must equal the key. |
| `milestone` | yes | What `demo_day` counts down to (`Demo Day`, `Showcase`). |
| `kickoff` | no | ISO-8601 with offset. The day the cohort starts. Before it, sr-raise counts down to kickoff instead of to Progress Day. Omit when unknown or long past. |
| `progress_day` | no | ISO-8601 with offset. See above. |
| `demo_day` | yes | ISO-8601 with offset. The cohort's Demo Day (or Showcase). |
| `raise_ends` | no | ISO-8601 with offset. The last day the cohort's own post-Demo-Day timeline applies; after it a company of that cohort is coached as an alumnus of it. Absent means sr-raise uses its baked-in window plan for that code. |
| `date_label` | yes | Human-readable date shown under the countdown. |
| `alumni_demo_day`, `raise_windows` | no | Already-supported optional overrides that sr-raise honors; leave them out unless you are tuning the coach. |

How sr-raise resolves the calendar for a company whose cohort code is `C`:

1. `cohorts[C]` in the fetched config; else
2. any fetched entry whose `cohort` field equals `C`; else
3. sr-raise's own baked-in fallback keyed by `C`.

A malformed entry falls back the same way a malformed date does today. sr-raise's
baked-in fallbacks must mirror the entries here (and its `speedrun` fallback must
mirror `cohorts.speedrun` on `main`), so when you add or re-date a cohort here, open
the matching change in sr-raise.

## How to make a change

| To… | Edit | Effect |
|---|---|---|
| Re-date a cohort | the cohort's `demo_day` / `date_label`, in its `cohorts.<CODE>` entry and, if it is the current cohort, in `cohorts.speedrun` too | Consumers pick it up automatically (extension on next load, status line within ~6h) |
| Move Progress Day | the cohort's `progress_day` (same two places) | sr-raise re-scopes which phase of advice founders get |
| End a cohort's own raise timeline | the cohort's `raise_ends` | After that day sr-raise coaches that cohort's companies as alumni of it |
| Add the next cohort | add `cohorts.<CODE>` with `kickoff` (once known), `progress_day`, `demo_day` and `date_label`; set `raise_ends` on the previous cohort's entry and leave it in place; repoint `cohorts.speedrun` to the new cohort once the previous Demo Day has passed; mirror the new entry in sr-raise's baked-in fallback | sr-raise coaches each cohort on its own timeline; the extension and status line follow `speedrun` |
| Push a status line update | `statusline.latest_version` | Installs surface the `⬆ update` nudge |

Commit to `main` — that's the published version. There is no build step.

## Note on the long-term home

The architecturally-correct home for canonical program data (cohorts, dates) is
**mission-control's `gold` schema** — the contract every app reads. This repo is a
pragmatic public projection for unauthenticated founder surfaces until a public
read-only `gold` endpoint exists; at that point this file becomes a thin mirror of it.
