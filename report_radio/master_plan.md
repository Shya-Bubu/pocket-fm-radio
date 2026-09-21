# Master Plan — FM Pocket Radio Report Generation

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    MASTER (Kiro/Opus)                     │
│  Orchestrates, reviews, ensures quality & consistency    │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌───────────────┐        ┌───────────────┐
│   FREEMAN     │        │   ENGINEER    │
│ (DeepSeek V4  │        │ (Kiro/Opus    │
│  Flash)       │        │  sub-agent)   │
│               │        │               │
│ Reads 42K+    │        │ Takes Freeman │
│ lines of chat │        │ output +      │
│ + all docs    │        │ template →    │
│               │        │ produces      │
│ Outputs:      │        │ final LaTeX   │
│ freeman_      │        │ report        │
│ output.md     │        │               │
└───────────────┘        └───────────────┘
```

## Workflow Steps

### Step 1: Freeman reads everything (DeepSeek V4 Flash)
- Feed DeepSeek the `freeman.md` instructions
- Feed it `master.md` first (300-line overview)
- Then feed it `llmchats.txt` in chunks (it has 1M+ context)
- Then feed it all feedback files
- Output: `freeman_output.md` (comprehensive structured data)

### Step 2: Master reviews Freeman output
- Kiro reads `freeman_output.md`
- Checks for completeness against `master.md`
- Identifies gaps → sends Freeman back for specific details if needed

### Step 3: Engineer compiles the report
- Kiro sub-agent reads `engineer.md` instructions
- Reads `freeman_output.md` for content
- Reads `../report/main.tex` for template styling
- Produces all LaTeX files in `report_radio/`

### Step 4: Master reviews & compiles
- Kiro reviews all .tex files for consistency
- Runs xelatex to compile
- Fixes any compilation errors
- Final PDF ready

## Why DeepSeek V4 Flash for Freeman?
- **Context window:** 1M+ tokens — can ingest the entire 42K-line chat log in one shot
- **Speed:** Flash variant is fast for extraction tasks
- **Cost:** Much cheaper than Opus for bulk reading
- **Task fit:** Freeman's job is extraction/summarization, not creative writing — Flash excels here

## Alternative: Gemma 4 31B
- Also viable for Freeman role
- Smaller context window than DeepSeek V4 Flash
- Would need chunked processing
- Good at structured extraction
- **Verdict:** DeepSeek V4 Flash is better for this specific task due to context length

## File Locations
- Instructions: `report_radio/freeman.md`, `report_radio/engineer.md`
- Freeman output: `report_radio/freeman_output.md`
- Final report: `report_radio/main.tex` + `report_radio/part*.tex`
- Template reference: `report/main.tex` (line-follower)
- Source data: `llmchats.txt`, `master.md`, `feedback_*.md`, `FM_*.md`

## Current Status
- [x] Master plan created
- [x] Freeman instructions written
- [x] Engineer instructions written
- [ ] Freeman processes source material → freeman_output.md
- [ ] Master reviews freeman_output.md
- [ ] Engineer produces LaTeX files
- [ ] Master compiles and reviews PDF
- [ ] Images added by user
