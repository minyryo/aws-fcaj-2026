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

## Deployment

Live at: **https://minyryo.github.io/aws-fcaj-2026/**

Deployed automatically via GitHub Actions (`.github/workflows/deploy.yml`) on every push to `main`.

## Changelog

### 2026-06-27

**Content: Week 2 Worklog (`content/1-Worklog/1.2-Week2/`)**
- Added AWS Skill Builder learning achievements: Cloud Quest Cloud Practitioner (4 solutions — Networking Concepts, Cloud Computing Essentials, Computing Solutions, Cloud First Steps) with reputation points table; AWS Escape Room CLF-C02 exam prep.
- Added polished team meeting minutes for 06/27/2026: Hieu presented payment logic and DB design; Danh's UI draft reviewed asynchronously; 6-track task distribution table (FE design/tech stack, BE tech stack, Hieu on Amplify + architecture, Thanh on Booking API, Nguyen on Auth API).
- Rewritten VI file (`_index.vi.md`) to match EN file exactly: correct dates (22/06–27/06/2026), translated AWS Skill Builder section, translated meeting minutes, all deliverable tables.
- Updated task table dates from 08/11–08/15/2025 (old placeholder) to 06/22–06/27/2026.

---

### 2026-06-22

**GitHub Pages deployment**
- Fixed CSS/JS and image loading broken by subpath deployment: replaced `relativeURLs = true` with `canonifyURLs = true` in `config.toml` — canonifyURLs prepends the full `baseURL` to all absolute paths in the HTML output, which is compatible with `hugo-theme-learn` and works correctly at a subpath.
- Removed `--baseURL` override and `configure-pages` step from workflow — `config.toml` baseURL is now the single source of truth, eliminating the double-path bug (`/aws-fcaj-2026/aws-fcaj-2026/`).
- Deleted duplicate `hugo.yml` workflow (was failing due to `publish_branch: main`).
- Added `.gitignore`: `public/`, `resources/_gen/`, `.hugo_build.lock`, OS and editor files.

**GitHub Actions workflow (`.github/workflows/deploy.yml`)**
- Final working setup: Hugo extended v0.163.1, `actions/checkout@v4` with `submodules: recursive`, `actions/deploy-pages@v4`.
- `baseURL` set to `https://minyryo.github.io/aws-fcaj-2026/` in `config.toml`.

---

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
