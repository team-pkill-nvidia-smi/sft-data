# sft-data

SFT datasets for training a vision-language model to emit **SVG geometric-diagram completions** (and, in the mixed set, answer short geometry questions).

Two datasets are included, differing in format and task mix:

| dir | format | samples | task(s) | size |
|---|---|---:|---|---:|
| `mix_v1_tar/` | ShareGPT-style list content | 30,000 | 24k SVG-edit + 6k short-answer geometry | ~655 MB |
| `sft_llava_prod_foundational_jsonl/` | LLaVA conversations | 10,099 | SVG-edit only | ~258 MB |

Both use the same SVG output convention (see below).

---

## 1. `mix_v1_tar/`

Despite the name, the packed tar (`train_00000.tar`, 678 MB) is **not tracked** — it exceeds GitHub's 100 MB per-file limit. The unpacked files are equivalent and can be re-packed trivially (see *Re-packing* below).

### Files

```
mix_v1_tar/
├── samples.jsonl          # 30,000 lines, 59 MB
└── images/                # 26,101 PNGs: 000000.png .. 026100.png
```

Image count (26,101) is less than sample count (30,000) because geometry3k items re-use the same diagram across multiple questions (one image is referenced by up to 3 samples).

### Schema (one JSON object per line)

```json
{
  "messages": [
    {"role": "user", "content": [
      {"type": "image", "image": "images/000000.png"},
      {"type": "text",  "text":  "<prompt>"}
    ]},
    {"role": "assistant", "content": [
      {"type": "text", "text": "<answer>"}
    ]}
  ]
}
```

Image paths are **relative to `mix_v1_tar/`**.

### Task mix

- **24,000 SVG-edit samples** (MathCanvas-derived). User text is:
  ```
  You are given a partially-drawn geometric diagram and an edit instruction. Extend the diagram by emitting an SVG. Draw existing geometry with black solid strokes and newly-added elements with red dashed strokes. Draw existing points as black dots and newly-added points as red dots.

  Base: <figure description>
  Edit: <construction instruction>
  ```
  Assistant emits a complete `<svg>` document (see *SVG output convention* below).
  Assistant length: mean 1,675 chars (range 766–3,084).

- **6,000 short-answer geometry samples** (geometry3k-style). User text is a short question like `Find x.`, `Use parallelogram to find $z$`, `If CDFG is a kite, find $m\angle D$`.
  Assistant emits a numeric or LaTeX-formatted answer only — mean 3 chars (range 1–34). Some diagrams are shared across 2–3 questions.

### Re-packing into a tarball

If you need the WebDataset-style tar used by some loaders:

```bash
cd mix_v1_tar
tar -cf train_00000.tar samples.jsonl images/
```

---

## 2. `sft_llava_prod_foundational_jsonl/`

A 10,099-sample LLaVA-format slice drawn from the MathCanvas *foundational* shards. SVG-edit task only (same prompt template as above).

### Files

```
sft_llava_prod_foundational_jsonl/
├── train_00000.jsonl         # 5,044 samples, 12 MB
├── train_00500.jsonl         # 5,055 samples, 12 MB
└── images/
    ├── bin_00000/bin_00000/  # 5,044 PNGs, referenced by train_00000.jsonl
    └── bin_00500/bin_00500/  # 5,055 PNGs, referenced by train_00500.jsonl
```

Image paths in the JSONL are relative to `sft_llava_prod_foundational_jsonl/` and have the form `images/bin_00000/{source_id}_edit_{N}.png`.

### Schema (LLaVA-style)

```json
{
  "id": "364117/edit_2",
  "image": "images/bin_00000/364117_edit_2.png",
  "conversations": [
    {"from": "human", "value": "<image>\nYou are given a partially-drawn geometric diagram...\n\nBase: ...\n\nEdit: ..."},
    {"from": "gpt",   "value": "<svg>...</svg>"}
  ]
}
```

The `<image>` placeholder marks where the image token is inserted by the SFT framework.

---

## SVG output convention (both datasets)

All SVG-edit targets follow the same structure: a 368.64×368.64 viewBox, `<symbol>` definitions for point markers, then four layers.

```svg
<svg viewBox="0 0 368.64 368.64" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <symbol id="pt"     overflow="visible"><circle r="1.94" fill="black"/></symbol>
    <symbol id="new-pt" overflow="visible"><circle r="1.94" fill="red"/></symbol>
  </defs>

  <!-- existing geometry: black solid -->
  <g id="base" stroke="black" stroke-width="1.2" fill="none">
    <line x1="..." y1="..." x2="..." y2="..."/>  <!-- labeled segment e.g. BA -->
    ...
  </g>

  <!-- newly-added geometry: red dashed -->
  <g id="edit" stroke="red" stroke-width="1.5" stroke-dasharray="5.55,2.4" fill="none">
    <line .../>
  </g>

  <!-- existing points (black dots) -->
  <use href="#pt"     x="..." y="..."/>  <!-- A -->
  ...
  <!-- newly-added points (red dots) -->
  <use href="#new-pt" x="..." y="..."/>  <!-- H -->

  <!-- italic Times New Roman labels -->
  <g id="labels" font-family="Times New Roman" font-style="italic" font-size="12">
    <text x="..." y="...">A</text>
    ...
  </g>
</svg>
```

Reward/evaluation code in downstream RL work relies on this exact structure (`id="base"` / `id="edit"` groupings, `#pt` / `#new-pt` markers).

---

## Summary stats

```
mix_v1_tar
  samples      : 30,000  (24,000 SVG-edit  +  6,000 short-answer)
  images       : 26,101  unique PNGs
  jsonl size   :  59 MB
  images size  : 596 MB

sft_llava_prod_foundational_jsonl
  samples      : 10,099  (all SVG-edit)
  images       : 10,099  unique PNGs
  jsonl size   :  25 MB
  images size  : 235 MB
```
