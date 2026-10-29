<!-- cpanse/skills:fgcz-infrastructure/skills/autonomous-render/SKILL.md | https://github.com/cpanse/skills/blob/HEAD/fgcz-infrastructure/skills/autonomous-render/SKILL.md -->

---
name: autonomous-render
description: "Autonomous R Markdown and Python report rendering with self-correcting validation. Use when asked to render an Rmd or Python report, debug a rendering failure, or submit a report rendering job. Triggers on: render, knit, rmarkdown::render, SBATCH render, render report, fix rendering error, re-render. Encodes a retry-and-validate loop that fixes common errors (S4 objects, missing packages, barcode mismatches, plot failures) without asking the user for guidance."
---

# Autonomous Report Rendering Skill

## Overview

Render R Markdown or Python reports autonomously with a validate-and-retry loop.
The goal is to collapse multi-round debugging sessions into a single invocation
by self-correcting common failures.

**Key principle:** Do NOT ask the user for guidance between iterations. Fix issues
using your judgment and only report back when done or after exhausting retries.

---

## Step 1 — Read and understand the report

Before rendering, read the full Rmd/py file and identify:

1. **Data inputs**: What objects/files does it load? (`.qs2`, `.h5ad`, `.rds`, paths)
2. **Expected outputs**: Plots, tables, saved files, HTML sections
3. **Dependencies**: Required R/Python packages (check library/import statements)
4. **Metadata assumptions**: Which columns does it expect? (`Sample`, `CellType_Harmonized_Detailed`, etc.)
5. **Cache files**: Does it use intermediate `.qs2` caching? Which cache files exist already?

Create a mental checklist of what "success" looks like for this report.

---

## Step 2 — Pre-flight validation

Before rendering, verify prerequisites exist:

```bash
# Check input files exist
ls -la /path/to/input_data.qs2

# Check required packages are available
Rscript -e 'stopifnot(requireNamespace("Seurat")); stopifnot(requireNamespace("qs2"))'

# For Python reports
python -c "import scanpy; import anndata"

# Check working directory has enough disk space
df -h /srv/GT/analysis/pXXXXX/
```

If any prerequisite is missing, fix it before rendering (install package, correct path, etc.).

---

## Step 3 — Render the report

### Local rendering (small reports, <10 min)

```bash
module load Dev/R/4.5.0
cd /srv/GT/analysis/pXXXXX/
Rscript -e "rmarkdown::render('report.Rmd')"
```

### SBATCH rendering (large reports, >10 min or >16GB RAM)

Use or adapt the existing SBATCH script if one exists (`SBATCH_render_*.sh`).
Otherwise create one following the CLAUDE.md template:

```bash
#!/bin/bash
#SBATCH --job-name=pXXXXX_render
#SBATCH --output=/srv/GT/analysis/pXXXXX/render_%j.log
#SBATCH --error=/srv/GT/analysis/pXXXXX/render_%j.err
#SBATCH --time=04:00:00
#SBATCH --mem-per-cpu=8G
#SBATCH --cpus-per-task=8
#SBATCH --partition=employee

module load Dev/R/4.5.0
cd /srv/GT/analysis/pXXXXX/
Rscript -e "rmarkdown::render('report.Rmd')"
echo "Completed at $(date)"
```

After submitting, monitor with:
```bash
squeue -u $USER  # Check if running
sacct -j JOBID --format=JobID,State,ExitCode,MaxRSS,Elapsed  # After completion
```

---

## Step 4 — Check for errors

After rendering completes (or fails), check in this order:

### 4a. Exit code and error log

```bash
# For SBATCH jobs
cat /srv/GT/analysis/pXXXXX/render_JOBID.err | tail -50
cat /srv/GT/analysis/pXXXXX/render_JOBID.log | tail -50

# For local rendering — error is inline
```

### 4b. Common error patterns and fixes

| Error pattern | Likely cause | Fix |
|--------------|-------------|-----|
| `cannot open the connection` / `No such file` | Wrong input path | Verify path exists, check for typos |
| `object 'X' not found` | Variable not defined in scope | Check chunk execution order, missing `include=FALSE` chunk |
| `no slot of name "X"` / `is not a valid slot` | S4 object API change (Seurat v5) | Use `class()` and `str()` to inspect, use modern accessor |
| `cannot coerce class 'Assay5'` | Seurat v5 layer issue | Add `JoinLayers()` before the operation |
| `Error in loadNamespace` / `there is no package` | Missing R package | `pak::pkg_install("pkg")` |
| `cannot allocate vector of size` | Out of memory | Increase `--mem-per-cpu` or add `gc()` calls |
| `Column 'X' not found` / `Unknown column` | Metadata column missing or renamed | Check `colnames(obj@meta.data)`, fix column name |
| `Column 'X' already exists` | Duplicate metadata column | Remove duplicate before adding |
| `DimReduc umap not found` | UMAP not computed | Add `RunUMAP()` step |
| `non-unique values when setting 'row.names'` | Duplicate chunk labels in Rmd | Find and rename duplicate chunk labels |
| `viewport may be too small` | `coord_fixed()` with patchwork | Replace with `theme(aspect.ratio = 1)` |
| `package 'X' was installed under R version Y` | R version mismatch | Reinstall package with current R |

### 4c. Apply fix and re-render

1. Read the error message carefully
2. Locate the failing line in the Rmd/py file
3. Apply the fix using the Edit tool
4. Re-render
5. **Repeat up to 5 times total** — if still failing after 5 attempts, report all errors to user

---

## Step 5 — Validate the output

After successful rendering, validate the HTML output:

### 5a. Basic checks

```bash
# HTML exists and is non-trivial
ls -la report.html
wc -c report.html  # Should be >100KB for most reports

# Check for R warnings/errors embedded in HTML
grep -c "Error in\|error:\|Warning:" render_JOBID.log
```

### 5b. Content validation

```bash
# Count plot images in HTML (base64 encoded PNGs)
grep -co 'data:image/png;base64' report.html

# Check for NA/NULL text that suggests data issues
grep -c '>NA<\|>NULL<\|>NaN<' report.html
```

### 5c. Report-specific validation

If the Rmd loads a Seurat object, verify key metrics match expectations:
- Cell count matches what's documented
- Number of samples/patients is correct
- No unexpected NA values in grouping columns

If validation fails, trace back to the data loading/filtering logic, fix, and re-render.

---

## Step 6 — Post-render actions

After successful rendering and validation:

1. **Report summary** to user:
   - Rendering succeeded/failed
   - Number of fix iterations needed
   - What was fixed (if anything)
   - Output file location and size
   - Any warnings worth noting

2. **If user wants to deliver to gStore**, use:
   ```bash
   g-req copynow /srv/GT/analysis/pXXXXX/report.html /srv/gstore/projects/pXXXXX/Analyses_Paul/
   ```
   Then provide the web link:
   `https://fgcz-sushi.uzh.ch/projects/pXXXXX/Analyses_Paul/report.html`

---

## Retry budget

- **Max 5 render attempts** per invocation
- After each failure, the fix should be different — do not retry the same thing
- If stuck on the same error after 2 attempts, try an alternative approach
- After 5 failures, stop and present all error messages to the user with your diagnosis

---

## What NOT to do

- Do NOT ask the user for help between iterations — use your judgment
- Do NOT skip the pre-flight check (Step 2) — missing inputs cause the most wasted time
- Do NOT use `--no-verify` or skip any safety checks
- Do NOT modify the report's analytical conclusions — only fix technical rendering issues
- Do NOT delete cache files unless you're sure they're corrupted
- Do NOT render on the login node if the report needs >16GB RAM — use SBATCH
