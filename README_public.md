# MapQA

A visual question-answering dataset over **maps and charts**: 11,176 question/answer
pairs grounded in 525 map images, each labeled with the reasoning skills the question
requires and the visual failure modes it is likely to trigger in a multimodal model.

The maps are mostly real documents — FAA sectional and terminal area charts, nautical
charts, topographic sheets, city zoning and cadastral maps, insurance maps, soil surveys,
military strategic and tactical maps — plus six fantasy maps. Most images come from the
Library of Congress; the rest from the National Park Service, university digital
collections, municipal government sites, and Internet Archive captures of FAA and NOAA
chart PDFs.

25% of the maps in the full corpus have been held back as a private evaluation set and
are not part of this release. The split was made by map, so no map in this release appears in the
held-back portion.

**Images are not included in this repository.** Every record carries the source URL for
its image; see [Getting the images](#getting-the-images).

---

## Contents

| File | What it is |
| --- | --- |
| `open_release.jsonl` | The dataset. One JSON object per line, 11,176 lines. |
| `question_skills_17.txt` | The question-skill taxonomy: 17 labels with descriptions and examples, plus the Multi-Part / Chained structural flag. |
| `failure_modes_9.txt` | The MLLM visual-failure-mode taxonomy (9 labels) and its source. |
| `README_public.md` / `README_public.html` | This document. |

---

## At a glance

| | |
| --- | --- |
| Question/answer pairs | 11,176 |
| Distinct image files | 525 (503 `.jpg`, 22 `.png`) |
| Distinct map sets | 525 (495 single images, 30 chart + legend pairs) |
| Topics | 4 |
| Map categories | 23 |
| Question-skill labels | 17 (multi-label) |
| Visual failure-mode labels | 9 (multi-label; 8 occur in the data) |
| Questions needing two images | 550 (`multi_map: true`) |
| Multi-part / chained questions | 1,097 |

A *map set* is a distinct value of the `Map` field — the image or image pair a question is
asked about. Image files and map sets both number 525, but they are not the same 525: of
the image files, 465 appear only in single-image questions, 30 only inside a chart+legend
pair, and 30 in both roles.

Questions per topic:

| Topic | Questions | Maps |
| --- | ---: | ---: |
| Aviation | 4,187 | 165 |
| Natural World | 2,805 | 143 |
| Urban | 2,412 | 118 |
| Military | 1,772 | 99 |

---

## Record schema

Every line has all 15 fields, always present and always of the same type.

| Field | Type | Description |
| --- | --- | --- |
| `uid` | string | UUID. Unique per record. |
| `Question` | string | The question text. |
| `Label` | string | The gold answer. Usually a short phrase. Multi-part answers are not consistently delimited — see [Evaluating](#evaluating). |
| `Question Skill` | string | One or more skill labels from the 17-label taxonomy, joined with `"; "`. |
| `Quasi Logical Expression` | string | A semi-formal restatement of the question as quantified clauses — `for some x, x is "…", return …`. Makes the retrieval and reasoning steps explicit. |
| `Expression Complexity` | string | An integer, stored as a string, scoring the structural complexity of the quasi-logical expression. Observed range: 3–10. |
| `Map URL` | string | Source URL for the image. For `multi_map` records, two URLs joined with `", "`, in the same order as `Map`. |
| `Map` | string | Image filename. For `multi_map` records, two filenames joined with `", "`. |
| `Map Category` | string | Document type, e.g. `sectional chart`, `soil map`, `zoning map`. One of 23 values. |
| `Map Description` | string | A short prose description of what the map shows. Constant for a given map. |
| `Map Location` | string | The place the map covers, e.g. `Newton County Georgia`, `Des Moines, Iowa`. |
| `Topic` | string | One of `Aviation`, `Natural World`, `Urban`, `Military`. |
| `potential_failure_modes` | string | One or more visual failure-mode labels from the 9-label taxonomy, joined with `"; "`. |
| `multi_map` | bool | `true` if the question requires both images (typically a chart plus its legend). |
| `Multi-Part / Chained` | bool | `true` if the question contains two or more linked asks. A structural flag, independent of the skill labels. |

`Map`, `Map Category`, `Map Description`, `Map Location`, and `Topic` are properties of the
map, not the question — they repeat identically across every question about that map.
`Map URL` is per-record: for a small number of Library of Congress maps the URL carries a
`r=x,y,w,h,rot` region parameter (written `?r=…` or `&r=…`) that deep-links the part of the sheet the question is
about, so the same image file can appear with several URLs that differ only in that
parameter.

### Parsing note

Both `Map` and `Map URL` use `", "` (comma **and space**) as their list separator, and
the separator only appears when `multi_map` is `true`. Do not split on a bare `,` — LoC
region parameters contain commas.

```python
def split_maps(field: str) -> list[str]:
    return field.split(", ") if ", " in field else [field]
```

### Example — single map

```json
{
  "uid": "184f3c62-a8e3-4157-aef1-ccbda01e7648",
  "Question": "Which soil type number is most common around Porterdale, and what slope category does it represent in the legend?",
  "Label": "7; deep, well-drained soils that have a loamy surface layer and a clayey subsoil",
  "Question Skill": "Attribute / Value Lookup; Legend & Symbology Interpretation; Spatial Relation; Extremum / Comparison Selection",
  "Quasi Logical Expression": "For all x and some y, x is soil type numbers around Porterdale in descending order of frequency, y is the slope category that x[0] represents in the legend, return x[0] and y.",
  "Expression Complexity": "4",
  "Map URL": "https://www.loc.gov/item/82693767/",
  "Map": "NewtonCountySoil.jpg",
  "Map Category": "soil map",
  "Map Description": "The image is a general soil map of Newton County, Georgia, created by the U.S. Department of Agriculture's Soil Conservation Service. It displays various colored regions representing different soil types and includes a legend explaining each type. The map is detailed with roads and boundaries within the county.",
  "Map Location": "Newton County Georgia",
  "Topic": "Natural World",
  "potential_failure_modes": "Positional and Relational Context; Structural and Physical Characteristics; Text",
  "multi_map": false,
  "Multi-Part / Chained": true
}
```

### Example — two maps (chart + legend)

```json
{
  "uid": "a18bba0b-a7d5-429b-b71f-59baea6c196c",
  "Question": "What is the height of the obstruction listed just outside of Guthrie Center on this map?",
  "Label": "1,641 feet",
  "Question Skill": "Attribute / Value Lookup; Spatial Relation",
  "Quasi Logical Expression": "for some x and some y, x is the symbol at Legend[obstruction], y is the number marking the area just outside of Guthrie Center[x], return y.",
  "Expression Complexity": "4",
  "Map URL": "https://tile.loc.gov/image-services/iiif/service:gmd:...:ca001124r/full/full/0/default.jpg, https://tile.loc.gov/image-services/iiif/service:gmd:...:ca001124v/full/full/0/default.jpg",
  "Map": "DesMoines1970Sec.jpg, DesMoines1970Legend.jpg",
  "Map Category": "sectional chart",
  "Map Description": "The images depict a sectional aeronautical chart of Des Moines. The first image shows a detailed map marked with various aviation-related symbols, airspaces, and elevations. The second image provides explanatory text and legends, including aeronautical symbols, topographical symbols, flight procedures, and emergency signals.",
  "Map Location": "Des Moines, Iowa",
  "Topic": "Aviation",
  "potential_failure_modes": "Positional and Relational Context; Text",
  "multi_map": true,
  "Multi-Part / Chained": false
}
```

Here the legend is the *back* of the same printed sheet (`…1124r` recto, `…1124v` verso).
The question cannot be answered from the chart alone — the solver has to look up the
obstruction symbol in the legend image first.

---

## Question skills (17 labels)

These describe the skills a human or model must use to answer a question. A question may
have multiple labels. Labels are not mutually exclusive and are assigned whenever the
corresponding operation is materially required to solve the question.

| # | Label | Description | Count |
| --- | --- | --- | ---: |
| 1 | **Feature Identification** | Identify or name a mapped entity or feature. *What city is shown?* | 5,333 |
| 2 | **Text / Label Reading** | Read or transcribe printed map text, labels, annotations, stamps, or other literal wording. | 1,037 |
| 3 | **Attribute / Value Lookup** | Retrieve a property, classification, designation, or value associated with a specified feature or location. Numeric and categorical attributes both count. | 4,828 |
| 4 | **Map Metadata Lookup** | Retrieve information about the map *as a document* — scale, projection, datum, title, date, publisher, sheet number, contour interval. | 463 |
| 5 | **Legend & Symbology Interpretation** | Interpret how symbols, colors, line styles, fills, or patterns map to meanings or categories. | 1,176 |
| 6 | **Spatial Relation** | Determine the spatial relationship between entities: cardinal direction, relative position, adjacency, containment, betweenness, bordering. | 5,965 |
| 7 | **Map-Frame Position** | Use position relative to the map sheet itself — corner, edge, margin, center, quadrant, top, bottom. | 1,528 |
| 8 | **Bearing / Direction Reasoning** | Use, compute, or reason with a bearing, radial, azimuth, or compass direction. | 1,026 |
| 9 | **Nearest / Farthest Selection** | Search candidate features and select the spatially closest or farthest from a reference. | 1,121 |
| 10 | **Extremum / Comparison Selection** | Compare candidates and select an extreme — highest, lowest, largest, smallest, most, least. | 1,332 |
| 11 | **Route / Connectivity Tracing** | Follow a road, trail, waterway, airway, or network to determine connections, crossings, sequence, or destination. | 762 |
| 12 | **Grid / Coordinate Referencing** | Use grids, graticules, coordinates, lat/long, UTM, or similar reference systems. | 349 |
| 13 | **Counting / Enumeration** | Count instances of a feature class or enumerate all members of a defined set. | 441 |
| 14 | **Measurement / Quantitative Reasoning** | Measure or calculate distance, length, area, proportion, percentage, density, average, or ratio. | 239 |
| 15 | **Presence / Absence / Anomaly Detection** | Determine whether something is present, absent, missing, possible, or unusual. | 156 |
| 16 | **Description / Characterization** | Characterize or describe a place, feature, region, or condition rather than retrieve one explicit value. | 94 |
| 17 | **Interpretation / Inference** | Draw a conclusion from map evidence that is not stated as a direct lookup. | 72 |

Counts are the number of questions carrying each label; they sum to more than 11,176
because the labels are multi-assigned.

### Secondary structural flag

**Multi-Part / Chained** — the question contains two or more linked asks or sequential
operations. Independent of the skill labels; a multi-part question receives every
substantive skill needed to answer it.

> *Which VOR station is closest to Kapaau, and what is its frequency?*
> → Nearest / Farthest Selection; Feature Identification; Attribute / Value Lookup

---

## Visual failure modes (9 labels)

Source: Tong, Liu, Zhai, Ma, LeCun, Xie. *"Eyes Wide Shut? Exploring the Visual
Shortcomings of Multimodal LLMs."* [arXiv:2401.06209](https://arxiv.org/abs/2401.06209) — the MMVP benchmark.

These labels describe the kind of visual detail a model must resolve in order to answer
correctly. They are **failure modes rather than skills**: each names a category of visual
information that CLIP-based multimodal LLMs systematically fail to encode, and so tend to
get wrong.

| # | Label | What must be resolved | Count |
| --- | --- | --- | ---: |
| 1 | **Orientation and Direction** | Which way something faces, points, turns, or moves — left/right handedness, facing direction, heading of motion. | 1,346 |
| 2 | **Presence of Specific Features** | Whether a particular element, part, or detail exists in the image at all. Failure is hallucinating an absent feature or missing a present one. | 183 |
| 3 | **State and Condition** | The transient state or configuration of an object rather than its identity — open/closed, on/off, wet/dry, full/empty, moving/at rest. | 104 |
| 4 | **Quantity and Count** | The number of objects, parts, or repeated features; exact counts and coarse numeric comparisons. | 296 |
| 5 | **Positional and Relational Context** | Spatial position and relationships — on/under, in front/behind, inside/outside, adjacency, contact vs. separation, relative arrangement. | 8,889 |
| 6 | **Color and Appearance** | Color, shade, tone, pattern, or surface appearance of a specified object or region. | 520 |
| 7 | **Structural and Physical Characteristics** | Physical attributes and structural form — shape, material, texture, thickness, proportion, how parts are joined. | 439 |
| 8 | **Text** | Read, transcribe, or resolve text, letters, numbers, logos, or symbols rendered inside the image. | 10,618 |
| 9 | **Viewpoint and Perspective** | The vantage point of the image — camera angle, elevation, viewing side. | 0 |

**Text** and **Positional and Relational Context** dominate, which is what you would
expect of maps: nearly every question requires reading something printed on the sheet,
and most require locating it relative to something else. **Viewpoint and Perspective** is
part of the taxonomy but does not occur in this dataset — maps are all plan-view
documents.

---

## Map categories

| Category | Questions | Maps |
| --- | ---: | ---: |
| sectional chart | 3,252 | 122 |
| strategic | 1,308 | 65 |
| nautical chart | 1,002 | 39 |
| topographic | 968 | 52 |
| transportation | 834 | 42 |
| city map | 630 | 23 |
| soil map | 549 | 23 |
| terminal area chart | 483 | 26 |
| tactical | 464 | 34 |
| helicopter | 420 | 14 |
| zoning map | 204 | 7 |
| cadastral | 201 | 9 |
| insurance map | 191 | 11 |
| linguistic map | 126 | 5 |
| postal roads | 100 | 8 |
| administrative | 77 | 8 |
| survey map | 74 | 9 |
| hydrologic map | 61 | 6 |
| distribution map | 60 | 6 |
| park map | 57 | 6 |
| fantasy map | 53 | 6 |
| aviation charts | 32 | 3 |
| historic | 30 | 1 |

The category vocabulary is free text and was not normalized: `aviation charts`
coexists with the more specific `sectional chart`, `terminal area
chart`, and `helicopter`. `park map` and `distribution map` each appear under two
different `Topic` values. Group these yourself if you need a clean partition.

---

## Getting the images

The images are not distributed here. Each record's `Map URL` is the source it was taken
from; `Map` is the filename it was saved under. Build your image directory keyed by the
`Map` filename so records resolve by name.

```python
import json

pairs = set()
with open("open_release.jsonl") as f:
    for line in f:
        r = json.loads(line)
        names = r["Map"].split(", ") if ", " in r["Map"] else [r["Map"]]
        urls  = r["Map URL"].split(", ") if ", " in r["Map URL"] else [r["Map URL"]]
        pairs.update(zip(names, urls))

print(len(pairs), "image/URL pairs to fetch")
```

That prints 532 — a few more than the 525 image files, because of the region-parameter
URLs described above.

Source hosts, by number of distinct URLs: `www.loc.gov` (278), `web.archive.org` (107),
`tile.loc.gov` (94), `www.nps.gov` (13), and a long tail of university, municipal, and
community sites.

Four kinds of URL appear:

- **`tile.loc.gov/image-services/iiif/…/full/full/0/default.jpg`** — a direct IIIF image
  request. Fetching it gives you the JPEG. These are large (tens of MB for a full
  sectional chart).
- **`www.loc.gov/item/…`** — a Library of Congress item page, not an image, and the most
  common single form (203 URLs). Open it and download the highest-resolution JPEG offered.
- **`www.loc.gov/resource/…`** — a Library of Congress viewer page, not an image (75 URLs).
  Open it and download the highest-resolution JPEG offered. An `r=x,y,w,h,rot` query
  parameter, if present, positions the viewer on the region the question concerns. This is
  the only URL form that carries the region parameter, and the dataset image is still the
  full sheet unless the filename says otherwise.
- **`.pdf` URLs, usually via `web.archive.org`** — FAA and NOAA chart PDFs. A `#page=N`
  fragment names the page the image came from; render that page to an image.

### Two things that will trip you up

**Truncated Wayback captures of FAA chart PDFs.** Wayback caps a stored response at
1,048,576 bytes (1 MiB), and these charts run from about 2 MB to 174 MB, so a capped
capture holds only the first megabyte of the file. Every truncation found in this dataset
came from a single crawl on **2024-12-12**, between roughly 01:00 and 02:50; captures of
the same URLs from other dates are whole. No URL in this release points at a 2024-12-12
capture, but Wayback can redirect you to a different capture, so check any chart PDF you
download.

The failure is easy to miss. A truncated PDF often still shows its first page and then
fails on the rest, or reports a damage error only on scrolling — so "it opened" is not
evidence the file is intact. **What is evidence: the file size.** If a chart PDF downloads
to exactly 1,048,576 bytes, it is a truncated capture, not a damaged download, and
refetching that same capture returns the same fragment.

To fix one, pick a different capture of the same URL:

```
https://web.archive.org/web/*/https://aeronav.faa.gov/visual/10-31-2024/PDFs/*
```

That lists every capture Wayback holds for this chart edition. Choose one from a date
other than 2024-12-12 — the 2025-02-08 crawl was whole for every chart checked — and
prefer the largest available capture.

> **Stay on the same URL path.** The path `/visual/10-31-2024/PDFs/` pins the chart
> edition, so any capture of that exact path is the same published chart and only the
> capture date differs. Do **not** substitute the live file at `aeronav.faa.gov`, and do
> not take a capture from a different edition directory. The FAA reissues charts in place
> under the same filename every 56 days, so a live or newer-edition copy can be a later
> printing than the image in this dataset — the airspace on it may genuinely differ, and
> an answer keyed to the dataset image can be wrong against it.

**The four Hawaiian Islands images.** Four dataset images come from one two-page document,
`Hawaiian_Islands.pdf`:

| Image | Where it comes from |
| --- | --- |
| `Hawaiian_Islands.png` | page 1 |
| `Honolulu_Inset_SEC.jpg` | page 2, center of the page |
| `Samoan_Islands_Inset_SEC.jpg` | page 2, bottom left of the page |
| `MarianasIslandsInsetSEC.jpg` | page 2, bottom right of the page |

Each of the three insets was cropped out as its own image. The legend on page 2 was not
included.

The URLs for all three insets end at `#page=2` and carry no region. That is deliberate
rather than an omission: the insets are not rectangular, and the only region syntax a PDF
URL offers (`viewrect`) takes a rectangle, so any region we could write would be a
bounding box that pulled in chart content belonging to the neighboring insets. `#page=2`
plus the position words above says exactly as much as is true. (`viewrect` is also ignored
by the built-in PDF viewers in Chrome and Firefox, while `#page=` works in all of them.)

---

## Using the dataset

### Load it

```python
import json
import pandas as pd

records = [json.loads(line) for line in open("open_release.jsonl")]
df = pd.DataFrame(records)

# multi-label fields -> lists
df["skills"] = df["Question Skill"].str.split("; ")
df["failure_modes"] = df["potential_failure_modes"].str.split("; ")
df["complexity"] = df["Expression Complexity"].astype(int)
```

### Slice it

```python
# One skill
bearing = df[df["skills"].apply(lambda s: "Bearing / Direction Reasoning" in s)]

# Single-image questions only (skip the chart+legend pairs)
single = df[~df["multi_map"]]

# Hardest chained questions
hard = df[df["Multi-Part / Chained"] & (df["complexity"] >= 7)]

# Held-out split by map, so no map appears in both sides
maps = sorted(df["Map"].unique())
test_maps = set(maps[::5])
train, test = df[~df["Map"].isin(test_maps)], df[df["Map"].isin(test_maps)]
```

### Split by map, not by row

The median map has 26 questions, and questions about the same sheet share context
heavily. Splitting randomly by row leaks the map — and its `Map Description` —
across train and test. **Group by `Map` (or by `Map Location`) when you build splits.**
The same rule governs the 25% of maps held back for testing: the holdout was taken by
map, and `multi_map` pairs were kept together so no image can straddle the boundary.

### Evaluating

Answers are short free text, not multiple choice: `"20 feet"`, `"NNW"`, `"1,641 feet"`,
`"rodgers canyon"`. Exact string match will understate performance — capitalization,
units, and phrasing vary. Normalize case and whitespace at minimum, and prefer a
judged or fuzzy match for anything beyond a smoke test.

**Multi-part answers are not consistently delimited.** Of the 1,097 `Multi-Part / Chained`
records, 114 separate their parts with `;`, 867 use a comma (`Lovelock, 116.5`), and 116
use no delimiter at all (`Philip 108.4`). There is no single split character, so scoring
parts independently means either matching each expected part against the whole answer
string or judging the answer as a unit.

`Question Skill` and `potential_failure_modes` are the diagnostic axes: report accuracy
per label rather than one aggregate number, and the rare labels
(Interpretation / Inference at 72 questions, State and Condition at 104) will need their
error bars taken seriously.

### What the Quasi Logical Expression is for

It is not an executable program and there is no interpreter for it. It is a semi-formal
restatement that makes explicit which entities must be found and what operation combines
them:

```
Q    : What is the bearing from the center of the map to Scott Township?
QLE  : for some x and some y, x is the center of the map, y is Scott Township on the
       map, bearing from x to y?
```

Use it to inspect what a question actually demands, to build chain-of-thought supervision,
or to check whether a model's stated reasoning found the right intermediate entities.
`Expression Complexity` scores the structural complexity of this expression and is the
dataset's built-in difficulty proxy. It is very nearly the number of comma-separated
phrases in the expression, but 322 records in this release score inconsistently with
their own phrase count — including the example above, which ends in a question rather
than a `return` clause. Treat the score as approximate when bucketing by difficulty.
