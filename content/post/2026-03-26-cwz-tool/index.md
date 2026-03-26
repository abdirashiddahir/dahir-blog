---
title: "Building a V2X Work Zone Tool From Scratch"
subtitle: "How a Transportation Engineer Used AI to Bridge R, Python, and ASN.1"
author: "Dahir"
date: "2026-03-26"
slug: cwz-tool
---

*A first-person account of developing a Connected Work Zone platform for a state DOT — from R Shiny dashboards to federally-validated UPER binary encoding — with Claude AI as a coding partner.*

---

## The Starting Point: An R User With a V2X Problem

I'm a transportation engineer. My daily tools are R, GIS, and traffic models — not Python, not ASN.1, and definitely not binary encoding protocols. When a state DOT asked me to build a tool that could generate J2735-compliant Traveler Information Messages (TIMs) for connected vehicles, I didn't know what UPER encoding was. I didn't know that `choiceID: 6` means absolute lat/lon coordinates. I didn't know that a 620KB Python file generated from an ASN.1 schema would become the backbone of our encoding pipeline.

What I did know was R Shiny. And I had Claude.

This article is about what happened next — 19,777 lines of R, a Python encoder validated by the federal government, deployment on shinyapps.io, and the lessons learned when a non-conventional programmer builds a multilingual platform with AI assistance.

---

## What We Built

The **Connected Work Zone (CWZ) Tool** is a Shiny web application that:

1. **Ingests** the state's 398+ active work zones from a DOT API and Google Sheets
2. **Displays** them on an interactive Leaflet map with 82,320 road segments
3. **Generates** J2735-compliant TravelerInformation Messages (TIMs) — the standard V2X format that roadside units broadcast to connected vehicles
4. **Validates** the JSON structure against 38 rules derived from SAE J2735
5. **Encodes** JSON to UPER HEX binary using the official federal schema
6. **Decodes** UPER HEX back to JSON for round-trip verification
7. **Deploys** on shinyapps.io with Python encoding working through reticulate

---

## Why Connected Work Zone — and Why It Matters

### From WZDx to CWZ

Most state DOTs share work zone data using the [Work Zone Data Exchange (WZDx)](https://datahub.transportation.gov/Roadways-and-Bridges/Work-Zone-Data-Feed-Registry/69qe-yiui/data_preview) specification — a GeoJSON-based format designed for sharing work zone information between agencies, navigation apps, and traffic management centers. The [USDOT Work Zone Data Feed Registry](https://datahub.transportation.gov/Roadways-and-Bridges/Work-Zone-Data-Feed-Registry/69qe-yiui/data_preview) tracks over 20 states publishing feeds:

| State | Feed | Version |
|-------|------|---------|
| Ohio | publicapi.ohgo.com | WZDx 4.2 |
| Texas | api.drivetexas.org | WZDx 4.2 |
| Massachusetts | feed.massdot-swzm.com | WZDx 4.1 |
| Washington | wzdx.wsdot.wa.gov | WZDx 4.2 |
| Kentucky | storage.googleapis.com | WZDx 4.1 |
| Indiana | in.carsprogram.org | WZDx 4.1 |
| Delaware | wzdx.e-dot.com | WZDx 4.1 |
| Michigan | mdotjcpd.state.mi.us | WZDx 4.0 |
| **Colorado** | **data.cotrip.org** | **CWZ 1.0** |

In December 2024, the [Connected Work Zone (CWZ) Implementation Guide and Standard v01.00](https://www.ite.org) was published by AASHTO, ITE, NEMA, and SAE International, with USDOT support. The CWZ standard is **not a replacement for WZDx — it's its evolution**. The cover page itself reads: *"Previously Work Zone Data Exchange (WZDx)."*

### What CWZ adds to WZDx

CWZ maintains full backward compatibility with WZDx's GeoJSON data exchange format. Both use the same WorkZoneFeed and GeoJSON structure for sharing work zone data between systems. What CWZ adds is guidance and requirements for the **Connected Vehicle Environment (CVE)** — the layer where work zone data reaches vehicles directly through roadside units:

- **SAE J2945/4 compatibility** — mapping CWZ WorkZoneFeed fields to Roadside Safety Message (RSM) data concepts (Table 8 in the standard)
- **NTCIP 1218 compatibility** — mapping CWZ DeviceFeed data to RSU configuration (Table 9 in the standard)
- **SAE J2735 TIM guidance** — how work zone information maps to Traveler Information Messages that RSUs broadcast
- **Field device data** — new DeviceFeed schemas for arrow boards, cameras, DMS signs, traffic sensors, traffic signals, and roadside units

### Where UPER hex encoding fits in

UPER (Unaligned Packed Encoding Rules) hex encoding is **not exclusive to CWZ** — it's part of the SAE J2735 standard, which defines the binary message formats for all V2X communication (BSMs, MAPs, SPaTs, TIMs, and more). Any V2X deployment uses UPER encoding, whether or not it follows CWZ.

What CWZ provides is the **mapping** — how to translate work zone data fields (speed limits, lane closures, event types) from the GeoJSON feed into J2735 TIM fields for broadcast. The standard acknowledges this is not always straightforward: *"There are challenges to providing a one-to-one mapping between the CWZ elements and those of SAE."* Different agencies currently have different approaches to expressing work zone information using SAE ITIS codes.

### What our tool does

Colorado is the only state that has adopted CWZ 1.0 so far (other states are still on WZDx 4.x). Our tool works with both — it ingests work zone data from any state DOT's API feed, then converts it into J2735-compliant TIMs for V2X deployment. The GeoJSON data goes in; the UPER binary that roadside units broadcast comes out.

---

## The Tech Stack: Five Languages, One App

Here's what surprised me most: building a modern V2X tool requires *five different technologies*, each doing something the others can't.

| Language | Role | Lines | Why it's irreplaceable |
|----------|------|-------|----------------------|
| **R** | Web app, maps, data processing | 19,777 | Shiny framework — the only language where `runApp()` gives you a full web dashboard |
| **Python** | UPER binary encoding/decoding | ~400 | pycrate library — no R equivalent exists for ASN.1 UPER encoding |
| **JavaScript** | Browser interactivity | ~200 (embedded) | Map clicks, clipboard copy, toast notifications — runs in the user's browser |
| **HTML/CSS** | UI styling | ~500 (embedded) | The dark professional theme, responsive layout, custom components |
| **ASN.1** | Data schema definition | 620KB (compiled) | J2735 standard — defines every V2X message type and encoding rule |

The revelation: **no single language could build this tool**. R can't encode UPER binary. Python can't serve Shiny dashboards. JavaScript can't read Google Sheets. Each language does exactly one job, and the magic is in how they connect.

---

## The Bridge: reticulate and the Pain of R + Python

The hardest technical challenge wasn't the encoding algorithm — it was getting R to call Python reliably across three environments (local Windows, VS Code, shinyapps.io).

### system2() — works locally, fails on server

```r
result <- system2("python", args = c("encode_tim_pycrate.py", json_file))
```

This calls Python as a subprocess. Works on my laptop where Python is installed. Completely fails on shinyapps.io because there's no system Python binary available.

### reticulate + RETICULATE_PYTHON — Windows Store nightmare

```r
library(reticulate)
Sys.setenv(RETICULATE_PYTHON = "C:/Users/.../python3.exe")
```

On Windows, `python3.exe` is a Microsoft Store *stub* that redirects to the Store app — not an actual Python interpreter. reticulate downloads its own Python 3.12 via `uv`, creating a separate environment where pycrate isn't installed. We had to manually install packages into reticulate's managed Python:

```bash
uv pip install pycrate --python "C:/Users/.../reticulate/uv/cache/.../python.exe"
```

### py_require() — the solution

```r
library(reticulate)
py_require(packages = "pycrate", python_version = ">=3.9")
```

One line. Works locally, works on shinyapps.io, works on AWS. The `py_require()` function (introduced in reticulate 1.41) tells both local and server environments exactly what Python packages are needed. shinyapps.io reads this and auto-provisions the correct Python environment during deployment.

**Lesson learned:** When bridging R and Python, use the *highest-level abstraction available*. Don't manage Python paths, virtual environments, or package installation manually. Let `py_require()` handle it.

---

## The Encoding Journey: From JSON to Federal Validation

### "Your JSON is proprietary"

Our first attempt at sending TIM data to the RSU vendor was met with feedback that our JSON followed a proprietary format and was not portable. The recommendation was to use the standard UPER HEX encoded format instead.

This feedback changed the trajectory of the project. We needed to convert our JSON to UPER binary — the packed bit-level encoding that V2X radios actually broadcast.

### Finding the schema

UPER encoding requires an ASN.1 schema — a formal definition of every field, its bit width, and encoding rules. The SAE J2735 schema is copyrighted and costs thousands of dollars. But the USDOT had already compiled it into a Python file for their [j2735decoder](https://github.com/usdot-fhwa-stol/j2735decoder) project:

```
j2735decoder_J2735_201603_combined_mobility.py  — 620KB
```

This single file contains every J2735 message type (BSM, MAP, SPaT, TIM, and 10+ others) compiled for the pycrate library. The j2735decoder authors only wrote decode functions for BSM, MAP, and SPaT. We wrote the TIM encoder and decoder on top of their schema.

### Bare vs MessageFrame — the breakthrough

Our first encoded hex failed on the federal validator. The issue: we were wrapping the TIM in a `MessageFrame` envelope (messageId=31), but the validator expected *bare* TravelerInformation when "TravelerInformation" is selected in the dropdown.

```
MessageFrame wrapped:  001F80AD6001AA67...  → FAILED (garbage decoded)
Bare TravelerInformation:  6001AA67...     → GREEN ✓
```

The `001F` prefix (messageId=31 in UPER) confused the validator's bit alignment. Removing it and selecting "TravelerInformation" manually produced a perfect decode.

**Lesson learned:** When validating V2X messages, match the encoding format to the validator's expectations. The RSU adds the MessageFrame during broadcast — you don't need to pre-wrap it.

### RSU vendor format compatibility

The RSU vendor's sample TIM JSON uses a proprietary format for certain fields:

```json
"viewAngle": {"bytes": "AAA=", "unusedBits": 0, "from000_0to022_5degrees": false}

"viewAngle": [0, 16]
```

We wrote `convert_bitstring()` to auto-detect and translate Base64-encoded BIT STRING fields, `long_` field names (with trailing underscore), and 16 named boolean direction flags.

---

## The Four Bugs That Taught Me Shiny Architecture

**New work zones invisible on startup.** The startup observer loaded the `entries` sheet but never loaded the `newworkzone` sheet. One missing `read_sheet()` call.

**Colorado map grey tiles.** Leaflet maps render grey when their container is hidden during initialization. A global `MutationObserver` watching for any DOM change and calling `invalidateSize()` on all Leaflet containers fixed every map in the app.

**Dropdown crashing silently on missing columns.** Accessing `nwz$rsm_schema` when the column doesn't exist in the Google Sheet kills the reactive observer silently. Wrapping every column access in `"col" %in% names(df)` prevents this.

**File upload overriding paste.** The encoder's reactive checked file upload before paste text. If a file was uploaded earlier in the session, it always used that file — ignoring new paste content. Reversing the priority (paste first when non-empty) fixed it.

All four bugs were about *assumptions* — assuming a sheet is loaded, assuming a column exists, assuming a container is visible, assuming the user's last input was a file. Defensive programming in Shiny means checking every assumption.

---

## Working With Claude AI: What It's Actually Like

I've been asked: "Did AI write your app?" The honest answer: **Claude was a coding partner, not an autopilot.**

**What Claude excelled at:**

- **Bridging knowledge gaps**: I'd say "I need to encode JSON to UPER HEX" and Claude would explain ASN.1, CHOICE types, bit packing, and then write the encoder — connecting concepts I'd never encountered
- **Reading source code at scale**: Claude analyzed the 620KB federal schema file, the 88 Java files from the fedgov encoder, and the 19,777-line app.R to find specific functions and patterns
- **Debugging across languages**: When the reticulate bridge failed, Claude traced the issue from R error messages through Python's import system to the Windows Store Python stub — three languages in one debug session
- **Writing reference documentation**: Every HTML reference document (architecture, encoding, deployment, HeadingSlice, vendor compatibility) was generated with accurate code snippets and interactive visualizations

**What Claude couldn't do:**

- **Make architectural decisions**: "Should we use MessageFrame or bare encoding?" required testing on the actual federal validator and understanding the agency's workflow. Claude provided options; I chose.
- **Understand the domain**: V2X deployment is a niche field. Claude knew the J2735 standard but not the agency's operational constraints, vendor-specific quirks, or the politics of sending hex files to a hardware vendor.
- **Guarantee correctness**: Every encoder output was verified against the federal validator. Claude's code passed — but I verified every GREEN independently.

**The real productivity multiplier:**

Without Claude, I would have needed a Python developer for the encoder (~2 weeks), a V2X engineer for the ASN.1 schema work (~1 week), a DevOps person for the shinyapps.io + reticulate deployment (~3 days), and a frontend developer for the dark-themed validator UI (~1 week).

With Claude, I did all of this myself — iteratively, with mistakes, with debugging — in about two weeks of intensive work. The key was that Claude could context-switch between R, Python, JavaScript, ASN.1, and deployment in a single conversation.

---

## Deployment: One Codebase, Multiple Platforms

The final app deploys from a single folder:

```
R_deploy/
├── app.R                          (19,777 lines — the entire Shiny app)
├── encode_tim_pycrate.py          (Python encoder — called via reticulate)
├── requirements.txt               (Python dependencies)
├── asn1/
│   └── j2735decoder_...py         (620KB federal J2735 schema)
├── geo_data/                      (pre-computed state boundaries)
├── Road_Inventory_Filtered/       (82,320 road segments)
└── www/logo.png
```

This same folder works on local (RStudio or VS Code) with `shiny::runApp()`, on shinyapps.io with `rsconnect::deployApp()` using `py_require()` to handle Python, and on AWS EC2 with Shiny Server and system Python.

The `py_require()` approach means no `.python-version` files, no `.Rprofile` hacks, no manual virtual environment management. One line declares the dependency; the platform figures out the rest.

---

## What I'd Do Differently

**Start with pycrate from day one.** We initially wrote a custom ASN.1 schema (J2735-TIM-Complete.asn) for the `asn1tools` library, then discovered it produced different hex than the federal validator expects. The USDOT's pre-compiled pycrate schema was the answer all along.

**Test on the federal validator early and often.** We spent time perfecting JSON structure before discovering that the *encoding* was the real challenge. The federal validator should be your first validation target, not your last.

**Don't fight reticulate's Python management.** Our attempts to force reticulate to use a specific Python binary (`RETICULATE_PYTHON`, `use_python()`, `use_virtualenv()`) all caused problems. `py_require()` is the correct abstraction layer — trust it.

**Document everything in HTML, not markdown.** The interactive HTML reference documents (with live Base64 converters, clickable compass diagrams, syntax-highlighted code) turned out to be more useful than any README. They're self-contained, run in any browser, and serve as both documentation and teaching tools.

---

## The Numbers

| Metric | Value |
|--------|-------|
| Total R code | 19,777 lines |
| R packages used | 18 |
| observeEvent handlers | 110 |
| Render functions | 133 |
| JavaScript calls | 86 |
| Python encoder | 400 lines |
| Federal schema | 620KB (594 ASN.1 types) |
| JSON validation rules | 38 |
| Work zones processed | 398+ active |
| Road segments displayed | 82,320 |
| Federal validator result | GREEN |
| Deployment platforms | 3 (local, shinyapps.io, AWS) |
| HTML reference documents | 8 |
| Bugs found and fixed | 4 critical |

---

## For Other Non-Conventional Programmers

If you're a domain expert (transportation, civil engineering, public health, environmental science) who uses R and is being asked to build something beyond a basic Shiny dashboard:

**Your domain knowledge is the hard part.** The coding can be assisted by AI. Understanding that `duratonTime` is a typo in the J2735 standard that you have to match exactly — that's domain knowledge no AI has.

**Multilingual is normal now.** A modern data application will likely use 2-4 languages. R for the framework, Python for specialized libraries, JavaScript for browser interactions. This isn't a sign of poor architecture — it's a sign of using the right tool for each job.

**AI is a multiplier, not a replacement.** Claude wrote most of the encoder. I verified every byte against the federal validator. Claude designed the UI. I tested every button on the deployed server. The human in the loop isn't optional — especially when the output drives roadside infrastructure.

**Deploy early, debug on the server.** Local development hides platform-specific issues. Our reticulate + Windows + RStudio + shinyapps.io debugging chain was painful but necessary. The earlier you deploy, the earlier you find these issues.

---

*This tool was developed for a state DOT's Connected Work Zone program. The federal J2735 schema is from [usdot-fhwa-stol/j2735decoder](https://github.com/usdot-fhwa-stol/j2735decoder) (Apache 2.0 license).*

*This article was written with Claude AI based on the context, experience, and guidance of Abdirashid Dahir. Built with R Shiny, Python pycrate, and Claude AI.*
