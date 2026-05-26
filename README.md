# Single-cell pipeline: how to run

This project is a notebook-based single-cell RNA-seq workflow (Cell Ranger outputs → Scanpy analysis).  
It is designed to be reproducible by keeping **data paths** and **sample metadata** in `configs/` rather than hardcoding them in the notebooks.

## Dataset

The data used by this workflow correspond to the ArrayExpress/BioStudies dataset:

**Single-cell RNAseq of WT and OCRL-KO kidney organoids either untreated or treated with IB-MECA**

ArrayExpress accession: [`E-MTAB-16673`](https://www.ebi.ac.uk/biostudies/ArrayExpress/studies/E-MTAB-16673)

Access to the BioStudies record may require an access key. The key is not included in this repository.  
Please contact the workflow maintainer and analysis lead, Alessandro Giordano (`a.giordano@tigem.it`), to request access.

Once access is granted, open:

```text
https://www.ebi.ac.uk/biostudies/ArrayExpress/studies/E-MTAB-16673?key=<ACCESS_KEY>
```

---

## What you need

### Software
- Python (tested with **Python 3.12**; Python 3.11 should also work)
- Required Python packages (see `requirements.txt` and `requirements-lock.txt`)

> If you already have an environment, you can skip installation, but package versions should be compatible with the lock file.

### Input data
The first notebook expects **Cell Ranger count output**, not raw FASTQ files.

Each sample must contain:

```text
outs/filtered_feature_bc_matrix.h5
```

The public FASTQ files from ArrayExpress/EBI must first be processed with Cell Ranger. If you already have the processed Cell Ranger outputs, the repository can read them directly.

---

## Generate count matrices from ArrayExpress FASTQ files

The ArrayExpress record provides raw FASTQ files. To run this workflow from FASTQs, first generate gene-barcode UMI count matrices with **Cell Ranger v9.0.1** using the 10x Genomics human reference:

```text
refdata-gex-GRCh38-2024-A
```

Cell Ranger documentation: [cellranger count command-line arguments](https://www.10xgenomics.com/support/software/cell-ranger/latest/getting-started/cr-command-line-arguments)

Example setup:

```bash
FASTQS=/path/to/E-MTAB-16673_fastqs
REF=/path/to/refdata-gex-GRCh38-2024-A
OUT=/path/to/Data

mkdir -p "$OUT"
cd "$OUT"
```

Run one `cellranger count` job per sample:

```bash
while read -r OUT_ID FASTQ_PREFIX; do
  cellranger count \
    --id="$OUT_ID" \
    --sample="$FASTQ_PREFIX" \
    --fastqs="$FASTQS" \
    --transcriptome="$REF" \
    --create-bam=false
done <<'EOF'
LES-25-0001 LES_25_0001_S1
LES-25-0002 LES_25_0002_S2
LES-25-0003 LES_25_0003_S3
LES-25-0004 LES_25_0004_S4
LES-25-0005 LES_25_0005_S5
LES-25-0006 LES_25_0006_S6
LES-25-0007 LES_25_0007_S7
LES-25-0008 LES_25_0008_S8
EOF
```

Optional resource flags can be added depending on the machine or cluster:

```bash
--localcores=8 --localmem=64
```

After completion, each sample should contain:

```text
<OUT>/<sample_id>/outs/filtered_feature_bc_matrix.h5
```

For example:

```text
Data/LES-25-0005/outs/filtered_feature_bc_matrix.h5
```

These folders are the inputs used by `notebooks/01_singlecell_qc_de.ipynb`.

---

## Project layout (must be respected)

Run notebooks from the **project root** (the folder containing `configs/` and `notebooks/`).

```text
project_root/
├── configs/
│   ├── config.yaml
│   ├── samples.template.yaml
│   ├── metadata.csv
│   └── sample_map.yaml        # optional (see below)
├── notebooks/
│   ├── 01_singlecell_qc_de.ipynb
│   └── 02_functional_gprofiler.ipynb
└── Data/                         # or an external data folder configured in config.yaml
    ├── <sample_folder_1>/
    │   └── outs/filtered_feature_bc_matrix.h5
    ├── <sample_folder_2>/
    │   └── outs/filtered_feature_bc_matrix.h5
    └── ...


```


**Important:** the pipeline expects Cell Ranger output files at:

```text
<data_root>/<sample_folder>/outs/filtered_feature_bc_matrix.h5
```

where `<data_root>` is configured in `configs/config.yaml`.

## Configuration files

### 1) `configs/config.yaml`
Defines where data are located and where outputs will be written.

*Example:*
```yaml
paths:
  data_root: "Data"
  outdir_anndata: "results/anndata"
```

For this dataset, if the processed data folder is in Downloads:

```yaml
paths:
  data_root: "/Users/alessandro/Downloads/Data"
  outdir_anndata: "results/anndata"
  outdir_figures: "results/figures"
  outdir_deg: "results/deg"
```

* `data_root`: folder that contains sample folders. It can be relative to the project root or an absolute path.
* `outdir_anndata`: where `.h5ad` outputs will be written. The directory is created automatically.

### 2) `configs/samples.template.yaml`
Defines the order of samples using aliases (public-safe IDs).

*Example:*
```yaml
ordered_aliases:
  - Sample_01
  - Sample_02
  - Sample_03
  - Sample_04
  - Sample_05
  - Sample_06
  - Sample_07
  - Sample_08
```

### 3) `configs/metadata.csv`
Defines per-sample metadata keyed by alias.

**Required columns:**
* `alias`
* `barcode_id`
* `sample_pool_name`
* `species`
* `private_genetic_data`
* `condition`

*Example:*
```csv
alias,barcode_id,sample_pool_name,species,private_genetic_data,condition
Sample_01,LES-25-0005,KONT1A,Homo_sapiens,No,KOuntreat
Sample_02,LES-25-0008,KOIB1B,Homo_sapiens,No,KOtreat
...
```

> **Note:** The notebook attaches these fields to `adata.obs` for downstream analysis.

### 4) (Optional) `configs/sample_map.yaml`
Only needed when the folder names in `Data/` are NOT the same as the aliases.

*Example (aliases → real folder names):*
```yaml
sample_map:
  Sample_01: LES-25-0005
  Sample_02: LES-25-0008
  Sample_03: LES-25-0002
  Sample_04: LES-25-0001
  Sample_05: LES-25-0006
  Sample_06: LES-25-0007
  Sample_07: LES-25-0003
  Sample_08: LES-25-0004
```

---

## How it works

* If `sample_map.yaml` exists, the pipeline uses the mapped folder names.
* If `sample_map.yaml` does not exist, the pipeline assumes folder names equal to aliases (e.g., `Data/Sample_01/`).
* The notebook reads samples in the order specified by `samples.template.yaml`.
* Metadata from `metadata.csv` are attached to `adata.obs`.
* `Genotype`, `Treatment`, and `Full_Condition` are derived from the `condition` column.

Expected condition mapping:

```text
KOuntreat  -> KO, Non-treated, KO Non-treated
KOtreat    -> KO, IB,          KO IB
CTRL       -> WT, Non-treated, WT Non-treated
CTRLTreat  -> WT, IB,          WT IB
```

---

## Install Python environment

From the project root:

```bash
cd /path/to/SingleCellOrganoids_public
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-lock.txt
```

If `scanpy==1.11.5` cannot be installed, check your Python version:

```bash
python --version
```

Use Python 3.11 or newer.

---

## Test the data configuration before running the notebook

Run this command from the project root:

```bash
python - <<'PY'
from pathlib import Path
import pandas as pd
import yaml
import scanpy as sc

cfg = yaml.safe_load(open("configs/config.yaml"))
data_root = Path(cfg["paths"]["data_root"])
ordered_aliases = yaml.safe_load(open("configs/samples.template.yaml"))["ordered_aliases"]

map_path = Path("configs/sample_map.yaml")
sample_map = yaml.safe_load(open(map_path))["sample_map"] if map_path.exists() else {}
meta = pd.read_csv("configs/metadata.csv").set_index("alias")

adatas = []
for alias in ordered_aliases:
    folder = sample_map.get(alias, alias)
    f = data_root / folder / "outs" / "filtered_feature_bc_matrix.h5"
    if not f.exists():
        raise FileNotFoundError(f"Missing file for {alias}: {f}")

    cond = meta.loc[alias, "condition"]
    genotype = "WT" if cond.startswith("CTRL") else "KO"
    treatment = "IB" if cond in ["KOtreat", "CTRLTreat"] else "Non-treated"
    full_condition = f"{genotype} {treatment}"

    adata = sc.read_10x_h5(str(f))
    adata.var_names_make_unique()
    adata.obs["Sample"] = alias
    adata.obs["Condition"] = cond
    adata.obs["Genotype"] = genotype
    adata.obs["Treatment"] = treatment
    adata.obs["Full_Condition"] = full_condition
    adatas.append(adata)

    print(f"{alias} -> {folder}: shape={adata.shape}, Full_Condition={full_condition}")

adata = sc.concat(adatas, axis=0)
adata.obs_names_make_unique()

print()
print("Final adata shape:", adata.shape)
print("obs names unique:", adata.obs_names.is_unique)
print("var names unique:", adata.var_names.is_unique)
print("PASS: adata configuration works")
PY
```

Expected result for the current dataset:

```text
Final adata shape: (22431, 38606)
obs names unique: True
var names unique: True
PASS: adata configuration works
```

---

## Run the notebooks

**1) Start Jupyter from the project root**
Open a terminal and run:

```bash
cd <project_root>
jupyter notebook
```

Then run notebooks in order:

1. `notebooks/01_singlecell_qc_de.ipynb`
2. `notebooks/02_functional_gprofiler.ipynb`

---

## Functional enrichment inputs

The second notebook, `notebooks/02_functional_gprofiler.ipynb`, runs g:Profiler enrichment on differential expression tables.

It reads input Excel files from the folder configured in `configs/config.yaml`:

```yaml
paths:
  input_gene_expression: "results/gene_expression"
  outdir_gprofiler: "results/gprofiler"
```

Each input file must be an `.xlsx` table with at least these columns:

```text
Gene_Name
logfoldchanges
pvals_adj
```

Example:

| Gene_Name | logfoldchanges | pvals_adj |
|---|---:|---:|
| LAMP1 | 1.20 | 0.001 |
| COL1A1 | -1.10 | 0.002 |
| GAPDH | 0.10 | 0.500 |

The notebook uses these thresholds:

```text
Upregulated:   logfoldchanges >  0.5 and pvals_adj < 0.05
Downregulated: logfoldchanges < -0.5 and pvals_adj < 0.05
```

For each input `.xlsx`, the notebook writes g:Profiler results to `outdir_gprofiler`, creating separate files for upregulated and downregulated genes:

```text
<input_file>_Upregulated.xlsx
<input_file>_Downregulated.xlsx
```

Each output workbook contains separate sheets for:

```text
GO_BP
GO_CC
KEGG
```

The g:Profiler step requires internet access.

### Bubble plot inputs

The second part of the notebook generates GO:CC bubble plots from g:Profiler Excel outputs. It reads files from:

```yaml
paths:
  gprofiler_renal_xlsx_dir: "results/gprofiler_all/Renal"
  gprofiler_renal_fig_dir: "results/figures/gprofiler_all/Renal/Figures1"
```

Input files for the bubble plot must follow this naming pattern:

```text
<contrast>__<cell_type>_Upregulated.xlsx
<contrast>__<cell_type>_Downregulated.xlsx
```

Example:

```text
KO_IB_vs_KO_Non-treated__Proximal_Tubule_Epithelial_Cell_Upregulated.xlsx
KO_IB_vs_KO_Non-treated__Proximal_Tubule_Epithelial_Cell_Downregulated.xlsx
```

The notebook reads the second sheet (`GO_CC`) from each workbook and writes PDF figures to `gprofiler_renal_fig_dir`.

---

## Quick sanity checks

### Check working directory
In a notebook cell:

```python
import os
print(os.getcwd())
```

It must be the `project_root` (where `configs/` exists).

### Check that input files are found
The notebook prints warnings for missing files:

```text
[WARN] Missing file for Sample_XX: .../outs/filtered_feature_bc_matrix.h5
```

If you see warnings:
* Verify sample folder names under `Data/`
* Verify `sample_map.yaml` (if you are using real folder names)
* Verify that each folder contains `outs/filtered_feature_bc_matrix.h5`
* Verify `data_root` in `configs/config.yaml`

---

## Outputs

The pipeline writes outputs (e.g., `.h5ad`) under the configured output folders (e.g., `results/anndata/`). 
