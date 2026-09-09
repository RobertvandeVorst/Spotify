# Technical Documentation

**Spotify Music Analytics — Power BI report**
Model: `Spotify Claude 01` · 64,956 tracks · 12,083 artists · 1923–2021

This document is the model and measure reference for the report described in the
[README](README.md). It covers the data sources, the semantic model, every column
of the fact table, the 31 measures, and the rendering approach.

---

## Contents

1. [Overview](#1-overview)
2. [Where things live](#2-where-things-live)
3. [Data sources](#3-data-sources)
4. [Data model](#4-data-model)
5. [Table: Tracks](#5-table-tracks)
6. [Table: Corr](#6-table-corr)
7. [Derived fields](#7-derived-fields)
8. [Measures](#8-measures)
9. [Rendering approach](#9-rendering-approach)
10. [Notes and trade-offs](#10-notes-and-trade-offs)

---

## 1. Overview

The report profiles a catalogue of Spotify tracks by their audio characteristics —
danceability, energy, valence, acousticness and others — and analyses how those
characteristics relate to popularity, streams, genre, era and mood.

Key facts about the underlying data:

| | |
|---|---|
| Tracks | 64,956 (one row per Spotify track) |
| Unique artists | 12,083 |
| Detailed genres | 47, rolled up into 8 genre families |
| Release years | 1923 – 2021 |
| Average popularity | 46.5 (Spotify's 0–100 scale) |
| Columns in fact table | 45 |

Tempo, key/mode, album type and the YouTube engagement columns come from the
enrichment dataset and cover **7,471 of 64,956 tracks (11.5%)**. The `genre` label
is thinner still — it is populated for only **3,429 tracks (5.3%)**, so it is blank
even on many enriched rows. Findings that depend on either are directional, not
catalogue-wide.

The model is intentionally compact: two imported data tables plus a dedicated
measures table. Much of the report's visual richness is delivered through DAX
measures that generate HTML, rendered via an HTML-content custom visual —
see [Rendering approach](#9-rendering-approach).

---

## 2. Where things live

The report is published in the Power BI Project (`.pbip`) format, which stores the
model and report definition as plain text rather than as a binary `.pbix`:

```
Spotify Claude 01.pbip                  Project pointer file
Spotify Claude 01.SemanticModel/
  definition/
    model.tmdl                          Model-level settings, table refs
    tables/Tracks.tmdl                  Fact table: column definitions and types
    tables/Corr.tmdl                    Correlation matrix table
    tables/Measures_.tmdl               All 31 measures, full DAX
    cultures/en-US.tmdl                 Formatting culture
Spotify Claude 01.Report/
  definition/
    report.json                         Report-level config
    pages/<id>/page.json                One folder per page
    pages/<id>/visuals/<id>/visual.json One file per visual
    bookmarks/*.bookmark.json           Bookmark states (chapter navigation)
  StaticResources/
    RegisteredResources/                Page background SVGs and images
```

`Measures_.tmdl` is the file to read if you want the actual DAX behind the
HTML-generated visuals.

Not published in this repository: the `.pbix` binaries, the cleaned dataset
(`spotify_merged_clean.csv`), and the Power BI local cache (`.pbi/cache.abf`),
which contains an embedded copy of the imported data.

---

## 3. Data sources

Both tables are imported from local CSV files via Power Query, in Import storage
mode. Power Query promotes headers and applies explicit column types; no further
shaping happens in the model — all cleaning is done upstream in the preprocessing
notebook.

| Table | Source file | Encoding | Columns | Purpose |
|---|---|---|---|---|
| `Tracks` | `spotify_merged_clean.csv` | UTF-8 (65001) | 45 | Main fact table, one row per track |
| `Corr` | `correlation_matrix_long.csv` | Windows-1252 | 7 | Pre-computed correlation matrix, long format |

Both derive from two Kaggle datasets joined on `spotify_id` — see the
[Data section of the README](README.md#data) for the sources, the join logic and
the cleaning steps.

---

## 4. Data model

| Table | Type | Columns | Measures | Storage |
|---|---|---|---|---|
| `Tracks` | Data (fact) | 45 | — | Import |
| `Corr` | Data (lookup) | 7 | — | Import |
| `Measures_` | Measures only | 0 | 31 | Import |

**Relationships: none.**

`Tracks` and `Corr` are analysed independently. `Corr` holds a self-contained,
pre-aggregated correlation matrix that the heatmap measures read directly, so no
join is required. `Measures_` is a disconnected table used purely to organise the
measures — a common convention that centralises calculations and keeps them out of
the data tables.

The consequence is worth naming: because there is no relationship, `Corr` does not
respond to slicers applied to `Tracks`. The correlation heatmap always shows the
whole-catalogue correlations. Making it respond to filters would require either a
relationship, a bridge table, or recomputing correlations in DAX.

---

## 5. Table: Tracks

The central fact table — one row per Spotify track. Columns fall into five groups:
identifiers and metadata, track-level audio features, artist-level audio averages,
YouTube and stream engagement, and derived helper columns.

### Identifiers and metadata

| Column | Type | Description |
|---|---|---|
| `spotify_id` | Text | Unique Spotify track identifier — the grain of the table |
| `song_name` | Text | Track title |
| `album_name` | Text | Album the track belongs to |
| `artist_name` | Text | Primary performing artist |
| `Artist` | Text | Artist name from the enrichment dataset (display field) |
| `Album_type` | Text | Album, single or compilation |
| `release_date` | Date | Release date, formatted as Long Date. 6 rows blank (placeholder dates removed) |
| `year` | Whole number | Release year, derived from `release_date` |
| `duration` | Whole number | Track length in milliseconds (raw) |
| `popularity` | Whole number | Spotify popularity score, 0–100 |

### Track-level audio features

| Column | Type | Description |
|---|---|---|
| `danceability` | Decimal | 0–1: how suitable the track is for dancing |
| `energy` | Decimal | 0–1: intensity and activity |
| `loudness` | Decimal | Overall loudness in dB. Positive values capped at 0 during cleaning |
| `speechiness` | Decimal | 0–1: presence of spoken words |
| `acousticness` | Decimal | 0–1: confidence the track is acoustic |
| `instrumentalness` | Decimal | 0–1: likelihood the track has no vocals |
| `liveness` | Decimal | 0–1: presence of a live audience |
| `valence` | Decimal | 0–1: musical positivity |
| `Key` | Whole number | Musical key as pitch class, 0–11 |
| `mode` | Whole number | Modality: 1 = major, 0 = minor |
| `Tempo` | Decimal | Estimated tempo in BPM |
| `time_signature` | Whole number | Estimated beats per bar |

`Key`, `mode`, `Tempo` and `time_signature` come from the enrichment dataset and
are blank for 88.5% of rows.

### Artist-level averages

Denormalised onto every track row, so artist profiles can be compared without a
separate artist table.

| Column | Type |
|---|---|
| `acousticness_artist` | Decimal |
| `danceability_artist` | Decimal |
| `energy_artist` | Decimal |
| `instrumentalness_artist` | Decimal |
| `liveness_artist` | Decimal |
| `speechiness_artist` | Decimal |
| `valence_artist` | Decimal |

### Engagement

From the enrichment dataset; blank for 88.5% of rows (more, for `genre`).

| Column | Type | Description |
|---|---|---|
| `Channel` | Text | YouTube channel that published the video |
| `Views` | Whole number | YouTube view count |
| `Likes` | Whole number | YouTube like count |
| `Comments` | Whole number | YouTube comment count |
| `Stream` | Whole number | Spotify stream count |
| `genre` | Text | Detailed genre label, 47 distinct values. Populated for 3,429 tracks |

### Derived columns

Documented in full in the [next section](#7-derived-fields).

| Column | Type |
|---|---|
| `duration_min` | Decimal |
| `duration_outlier` | True/False |
| `decade` | Text |
| `era` | Text |
| `popularity_tier` | Text |
| `mood` | Text |
| `genre_group` | Text |
| `key_name` | Text |
| `mode_name` | Text |
| `yt_like_rate` | Decimal |

---

## 6. Table: Corr

A pre-computed correlation matrix in long (tidy) format — 100 rows representing a
10×10 grid of audio-feature pairs. Each row is one cell of the heatmap. The order
and label columns control axis sorting and display, so the visual does not have to
derive them.

| Column | Type | Description |
|---|---|---|
| `Feature_X` | Text | First audio feature in the pair (matrix column source) |
| `Feature_Y` | Text | Second audio feature in the pair (matrix row source) |
| `Correlation` | Decimal | Pearson correlation coefficient, −1 to 1 |
| `X_Order` | Whole number | Sort order for `Feature_X` on the axis |
| `Y_Order` | Whole number | Sort order for `Feature_Y` on the axis |
| `X_Label` | Text | Display label for `Feature_X` |
| `Y_Label` | Text | Display label for `Feature_Y` |

---

## 7. Derived fields

Ten analytical fields added on top of the raw columns. Each becomes a ready-made
slicer or axis in the report — the point is to turn abstract 0–1 numbers into
labels a reader can act on.

| Field | Built from | Values |
|---|---|---|
| `duration_min` | `duration` ÷ 60,000 | Continuous |
| `duration_outlier` | `duration_min` | `TRUE` for tracks under 30s or over 15min — **77 rows flagged, none deleted** |
| `decade` | `year` rounded down | `1920s` … `2020s` (11 buckets) |
| `era` | `year` | `Pre-1980 (Classic)`, `1980-1999`, `2000s`, `2010s`, `2020s` |
| `popularity_tier` | `popularity` | `Hit (75-100)`, `Popular (50-74)`, `Moderate (25-49)`, `Niche (1-24)`, `Unknown` |
| `mood` | `valence` × `energy` quadrant | `Happy / Energetic`, `Sad / Mellow`, `Calm / Content`, `Angry / Tense` |
| `genre_group` | `genre` | `Pop`, `Rock`, `Hip-Hop/R&B`, `Electronic/Dance`, `Country/Folk`, `Classical/Jazz`, `Metal`, `Other` — blank where `genre` is blank |
| `key_name` | `Key` | `C`, `C#`, `D` … `B` |
| `mode_name` | `mode` | `Major`, `Minor` |
| `yt_like_rate` | `Likes` ÷ `Views` | Continuous |

**On `mood`:** valence and energy are each hard to read alone. Crossing them
produces four labels anyone understands, which makes a far better slicer than two
abstract 0–1 numbers.

**On blanks:** the YouTube fields are blank for the 88.5% of tracks outside the
enrichment dataset, and `genre_group` is blank for 94.7% — genre is missing on
many enriched rows too. Visuals that slice by genre are therefore describing a
small subset, and any total shown alongside them will not reconcile with the
64,956 catalogue count.

---

## 8. Measures

All 31 measures live in the disconnected `Measures_` table. Full DAX is in
[`Measures_.tmdl`](#2-where-things-live). They split into two groups.

### 8.1 KPI and aggregate measures

Standard scalar measures, used in cards and summary readouts.

| Measure | DAX | Description |
|---|---|---|
| `Total Tracks` | `COUNTROWS ( Tracks )` | Track count in the current filter context |
| `Unique Artists` | `DISTINCTCOUNT ( Tracks[artist_name] )` | Distinct artists |
| `Avg Popularity` | `AVERAGE ( Tracks[popularity] )` | Mean popularity score, 0–100 |
| `Avg Loudness` | `AVERAGE ( Tracks[loudness] )` | Mean loudness in dB |
| `Avg Duration min` | `AVERAGE ( Tracks[duration_min] )` | Mean track length in minutes |
| `Avg Acousticness` | `AVERAGE ( Tracks[acousticness] )` | Mean acousticness, 0–1 |
| `Avg Valence` | `AVERAGE ( Tracks[valence] )` | Mean valence, 0–1 |
| `Avg Danceability` | `AVERAGE ( Tracks[danceability] )` | Mean danceability, 0–1 |
| `Year Span` | `MIN ( Tracks[year] ) & " – " & MAX ( Tracks[year] )` | Text label of earliest–latest release year in context |

### 8.2 HTML-generating measures

Each returns an HTML string rendered by an HTML-content custom visual. The DAX is
large — markup plus inline styling built with `CONCATENATEX` over the underlying
table — so they are documented here by purpose rather than in full.

| Measure | Category | Purpose |
|---|---|---|
| `Heatmap HTML` | Correlation | Renders the audio-feature correlation matrix from `Corr` as a colour-graded HTML table |
| `Heatmap HTML 2` | Correlation | Alternate styling and layout variant of the heatmap |
| `CorrelationHeatmapHTML` | Correlation | Full heatmap build — axis labels, diverging colour scale, cell values |
| `Equalizer HTML` | Decorative | Animated equalizer-bar graphic used as a header accent |
| `MP3 preview` | Media | Audio player for the pre-generated track matching the current selection |
| `Loudness by Decade (HTML)` | Trend | Average loudness across decades — the "loudness war" chart |
| `Sound Profile by Decade (HTML)` | Trend | Multi-feature audio profile by decade |
| `Duration by Decade (HTML)` | Trend | Average track duration across decades |
| `Hits vs Niche Radar (HTML)` | Radar | Radar comparing the audio fingerprint of popular vs niche tracks |
| `Pop by Danceability (HTML)` | Feature vs popularity | Relationship between danceability and popularity |
| `Pop by Energy (HTML)` | Feature vs popularity | Relationship between energy and popularity |
| `Pop by Valence (HTML)` | Feature vs popularity | Relationship between valence and popularity |
| `Pop by Acousticness (HTML)` | Feature vs popularity | Relationship between acousticness and popularity |
| `Pop by Instrumentalness (HTML)` | Feature vs popularity | Relationship between instrumentalness and popularity |
| `Pop by Duration (HTML)` | Feature vs popularity | Relationship between track duration and popularity |
| `Streams vs Popularity (HTML)` | Scatter | Spotify streams plotted against popularity score |
| `Avg Pop by Genre Family (HTML)` | Category | Average popularity across the genre groups |
| `Avg Pop by Era (HTML)` | Category | Average popularity across release eras |
| `Genre Filter (HTML) true` | Custom filter | Interactive HTML genre slicer, selected state. Display folder: *Filters* |
| `Genre Filter (HTML) false` | Custom filter | Genre slicer, unselected state. Display folder: *Filters* |
| `Mood Filter (HTML) true` | Custom filter | Interactive HTML mood slicer, selected state. Display folder: *Filters* |
| `Mood Filter (HTML) false` | Custom filter | Mood slicer, unselected state. Display folder: *Filters* |

The paired `true` / `false` filter measures exist because the HTML slicers render
their own selected and unselected states; a bookmark swaps between the two visuals
rather than restyling one.

---

## 9. Rendering approach

Three techniques carry most of the report's appearance.

**HTML and SVG generated in DAX.** The heatmap, radar, decade trends, scatter
plots, audio player and custom filter readouts are all strings assembled in a
measure and rendered through an HTML-content custom visual. Colour scales,
`aspect-ratio` boxes and `vmin`-based type sizing are written inline, which is why
the visuals scale cleanly with the page rather than with a fixed pixel grid.

**SVG page backgrounds.** Static page chrome — panels, dividers, chapter rails,
titles — is designed as SVG and applied as a page background image, with
transparent Power BI visuals layered on top. Hover and default states for the
chapter navigation are separate SVGs swapped by bookmark; they are stored under
`StaticResources/RegisteredResources/`.

**A custom range slider.** The dual-handle numeric slider on the Listening Room
page was built with the Power BI Visuals SDK, because the stock slicer cannot give
real two-ended numeric filtering across the audio-feature columns.

---

## 10. Notes and trade-offs

- **HTML-in-DAX couples visuals to string-building.** It buys precise control over
  styling that no stock visual offers, but any change to a chart means editing
  measure markup rather than clicking a formatting pane. Debugging is harder, and
  the measures are long.
- **No relationships means `Corr` is static.** The correlation heatmap always
  reflects the full catalogue and ignores page filters. This is intentional, but
  it is not obvious to a reader, and it is the first thing that would need to
  change if the heatmap were ever meant to be interactive.
- **Raw and display versions of the same concept coexist** — `duration` vs
  `duration_min`, `mode` vs `mode_name`, `Key` vs `key_name`. The raw column
  supports calculation, the derived one supports display.
- **Energy and loudness tell nearly the same story** (r = +0.75). Treating them as
  independent signals would double-count.
- **All data is imported and static.** A refresh re-reads the local CSVs; there is
  no live connection to the Spotify API. Popularity is a point-in-time metric and
  will drift toward newer releases in any later extract.
- **Enrichment coverage is thin.** Key, tempo and the YouTube metrics cover 11.5%
  of tracks; `genre` covers 5.3%. Any visual built on them describes that subset,
  and the genre breakdowns rest on the narrowest base of all.

---

*Companion to the [README](README.md). Report and analysis by
[Robert van de Vorst](https://www.linkedin.com/in/robertvandevorst/).*
