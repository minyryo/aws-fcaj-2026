# FCJ Workshop Template

A Hugo-based workshop site template using the `hugo-theme-learn` theme, supporting English and Vietnamese content.

## Local Development

**Prerequisites:** Hugo v0.126+ (extended build recommended)

```bash
# Start local dev server with draft content
hugo server -D
```

Open `http://localhost:1313` in your browser. The server live-reloads on file changes.

```bash
# Build static site to public/
hugo
```

## Changelog

### 2026-06-20

**Content: Week 1 Worklog (`content/1-Worklog/1.1-Week1/`)**
- Added polished team kick-off meeting minutes (06/15/2026): architecture decision table, task distribution across FE / BE / AWS Admin tracks, and Week 2 next steps.
- Updated Day 2 task row to reference the kick-off meeting.

**Content: Proposal (`content/2-Proposal/`)**
- Added Section 3 — Core Application Features: explicit table mapping each feature (Auth, Booking Management, Payment) to its deployment model (Monolith vs. Serverless) with rationale.
- Renumbered all subsequent sections accordingly.
- Added second architecture diagram (detailed flow, `court_booking_hybrid_v1.png`) to the Solution Architecture section.

**Assets**
- Added `static/images/2-Proposal/court_booking_hybrid_v1.png` — detailed numbered-flow architecture diagram.
- Added `static/images/2-Proposal/court_booking_hybrid_v2.png` — overview architecture diagram.

---

### 2026-06-15

**Content: Proposal (`content/2-Proposal/`)**
- Rewrote both `_index.md` and `_index.vi.md` to reflect the court booking hybrid AWS architecture (replaced IoT weather platform sample content).

**Content: Week 1 Worklog (`content/1-Worklog/1.1-Week1/`)**
- Updated objectives, task table, and achievements to align with the court booking project context.
- Updated all task dates to the week of 06/15–06/19/2026.

**Hugo fixes (`layouts/shortcodes/`, `config.toml`)**
- Replaced deprecated `getJSON` with `resources.GetRemote` + `transform.Unmarshal` in `layouts/shortcodes/ghcontributors.html` — `getJSON` was removed in Hugo v0.126.
- Replaced deprecated `languageName` with `label` in `config.toml` for both `en` and `vi` language blocks — `languageName` was deprecated in Hugo v0.158.
