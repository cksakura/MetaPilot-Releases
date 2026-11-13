# MetaPilot — Releases

**MetaPilot** is a GUI and command-line workflow for metaproteomics. On a MAG catalogue it selects the genomes that explain a sample, builds a sample-specific database, and searches that database.

This repository hosts the **downloadable release**. The source, and the description of how identification works, are in the [code repository](https://github.com/cksakura/MetaPilot).

**Version 1.2.0.** Please report issues with the version, the parameter file, and the steps to reproduce.

The release contains:

- A Windows desktop application for interactive analysis. The portable package includes a Java 21 runtime and JavaFX, so the GUI does not need a separate Java install.
- `metapilot-cli-1.2.0-shaded.jar` for Linux servers and HPC.
- `config/meta_resources.params` for the paths of external tools.
- A resources archive for the external tools and databases used by the Windows application.

Building from source is not part of this install.

## Download

Grab the latest build from the **[Releases](../../releases)** page.

- **Portable ZIP (no install):** `MetaPilot-<version>-win64.zip`. Unzip and run `MetaPilot.exe`.
- **Resources archive:** `Resources.7z`. Extract it with the application. The GUI reads DIA-NN, FragPipe, and the other tools from this tree.
- **Linux / HPC:** `metapilot-cli-1.2.0-shaded.jar` together with `config/`.

## System requirements

**Windows GUI**

- **OS:** Windows 10 or 11 (x64)
- **RAM:** 8 GB minimum (16–128 GB recommended for large datasets)
- **Disk:** about 6–20 GB for the application and resources, plus space for the study data

**Linux and HPC**

MetaPilot runs on Java 17 or newer. The HPC runs for this version use Java 21.

| Tool | Required for |
| --- | --- |
| FragPipe 24, with MSFragger, Crystal-C, MSBooster, and the DIA-NN 1.8.x binary under its `tools/` directory | DDA MAG FragPipe and DDA open search. MetaPilot uses that bundled 1.8 binary only to predict spectra. A DIA search does not use it |
| DIA-NN 2.x, installed separately and set as `DIANN` | DIA Marker, DIA Seed, DIA Target, and DIA Real-Time |
| FlashLFQ | Peptide quantification. On Linux this is the .NET build, so `dotnet` must be on `PATH` |
| Python 3 | Report pages |
| Functional annotation database | `functional_annotation/` |
| ProteoWizard `msconvert` | Thermo `.raw` input on a DDA workflow. mzML input does not need it. DIA Marker reads the DIA files directly |

Open search reads MSFragger, Crystal-C, and MSBooster from `FRAGPIPE_TOOLS`. It does not launch the FragPipe GUI.

## Windows quick start

1. Download `MetaPilot-<version>-win64.zip` and `Resources.7z` from **[Releases](../../releases)**.
2. Extract both to a short path (for example `C:\Apps\MetaPilot\`).
3. Double-click `MetaPilot.exe`. If SmartScreen blocks the unsigned program, run `Start_MetaPilot.bat` from the same folder. It clears the download mark and starts the application.
4. In the resource settings, point each tool at the copy inside the extracted resources folder.
5. Register the MAG catalogue (human-gut v2.0.2 for the studies in this version), then choose a workflow and save the parameter file. The same file is what the command line runs.

## Highlights

- A GUI for end-to-end metaproteomics, with the same workflows available on HPC.
- MAG catalogue download and registration, so a study can be repeated from one parameter file.
- Structured outputs and HTML reports.
- DDA open search, and DIA Marker Deep, are the two workflows written out below.

## 1. Install the command line

Copy the jar and the `config` directory to a path that every compute node can read. The two must stay together:

```text
/opt/metapilot/metapilot-cli-1.2.0-shaded.jar
/opt/metapilot/config/meta_resources.params
```

Run from that directory, or pass `--resources` explicitly. Check Java and the tool paths:

```bash
cd /opt/metapilot
java -version
java -jar metapilot-cli-1.2.0-shaded.jar resources-check
```

Set a path that `resources-check` reports as missing:

```bash
java -jar metapilot-cli-1.2.0-shaded.jar resources-set \
  --set FRAGPIPE=/opt/fragpipe-24.0/bin/fragpipe \
  --set FRAGPIPE_TOOLS=/opt/fragpipe-24.0/tools \
  --set DIANN=/opt/diann/diann \
  --set FLASHLFQ=/opt/metapilot/FlashLFQ/CMD \
  --set PYTHON=/opt/conda/envs/metapilot/bin/python \
  --set FUNCDB=/data/metapilot/function/func_def.db \
  --print
```

`DIANN` is the program a DIA search runs. Point it at the separate DIA-NN 2 installation. Do not point it at `tools/diann/1.8.2_beta_8/.../diann-1.8.1.8`. Open search finds that 1.8 binary itself from `FRAGPIPE_TOOLS` and uses it for spectrum prediction.

`dotnet --version` must succeed on the compute node before a run that quantifies peptides.

## 2. Prepare the MAG catalogue

Formal studies in this version use the human-gut catalogue v2.0.2. Download it once, then register it. Both examples below use this registration.

```bash
java -jar metapilot-cli-1.2.0-shaded.jar mag-download \
  --catalogue human-gut \
  --version v2.0.2 \
  --out /data/metapilot/MAG \
  --resume

java -jar metapilot-cli-1.2.0-shaded.jar mag-register \
  --catalogue human-gut \
  --version v2.0.2 \
  --identifier MGYG \
  --path /data/metapilot/MAG/human-gut/v2.0.2

java -jar metapilot-cli-1.2.0-shaded.jar mag-list
```

The registered folder must contain `original_db/`, `genomes-taxa.tsv`, and `Marker4.fasta`. Open search builds `Marker4_td.fasta` beside it on the first run and reuses that file afterwards. DIA Marker builds the Marker4 spectral library on the first run and reuses it afterwards. The pangenome check, used by open search, looks for `protein_catalogue/uhgp-100.faa.gz` under the same version. Leave `uhgp100_path` empty to use that default, or set it to the catalogue directory.

## 3. Open search

Use a parameter file saved from the Windows GUI, or start from this example. These fields decide the run:

```json
{
  "workflow": "DDA_MAG_OPENSEARCH",
  "input_output": {
    "mag_db_version": "v2.0.2",
    "mag_database": "human-gut",
    "append_host_db": false,
    "host_database": "/data/metapilot/host/human_reviewed.fasta",
    "output_directory": "/data/studies/demo",
    "experiments": [
      {
        "file_path": "/data/studies/demo/mzML/sample_a.mzML",
        "experiment_name": "sample_a",
        "fraction": 1
      }
    ]
  },
  "additional_parameters": {
    "identification_version": "v2",
    "genome_selection_strategy": "Comprehensive",
    "threads": 24,
    "pangenome_variant_check": "on"
  }
}
```

Paths in this file are read on the compute node. A drive letter from the Windows machine will not resolve there. `threads` should equal the CPUs you request. `append_host_db` adds the host FASTA to every sample database after genome selection; set it only when host proteins are part of the question. `Focus` selects fewer genomes than `Comprehensive`.

Check the file before submitting:

```bash
java -jar metapilot-cli-1.2.0-shaded.jar params-validate \
  --params /data/studies/demo/mag_opensearch_parameters.json
```

Request memory beyond the Java heap. A 100 GB heap needs a job of about 160 GB, because MSFragger, DIA-NN, and the XGBoost library sit outside it. A study of a few dozen mzML files is a one-node job: 24 CPUs and 72 hours is a reasonable start; large cohorts need more of both.

```bash
#!/bin/bash
#SBATCH --job-name=metapilot-open
#SBATCH --cpus-per-task=24
#SBATCH --mem=160G
#SBATCH --time=72:00:00
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

export JAVA_HOME=/opt/java/jdk-21
export DOTNET_ROOT=/opt/dotnet
export PATH="$DOTNET_ROOT:$JAVA_HOME/bin:$PATH"

cd /opt/metapilot
java -Xms16g -Xmx100g -XX:+UseParallelGC \
  -jar metapilot-cli-1.2.0-shaded.jar mag-opensearch \
  --params /data/studies/demo/mag_opensearch_parameters.json
```

Submit it with `sbatch run_open.slurm.sh` from the directory where you want the Slurm logs.

The run writes `/data/studies/demo/opensearch_MAG_search/`. The first file to open is `final_summary.tsv`: one row per raw file with the MS/MS count, the number identified, the identification rate, the peptide count, and the PSM count. `combined/validated_psms.tsv` holds the accepted PSMs. `final_peptides.tsv` holds the peptide table, including the shift class. `taxonomy_analysis/Genome.tsv` and `functional_annotation/` are the genome and function reports.

### Running the same open search again

Submit the same command. A step whose output is already present and still matches the current parameters is skipped: the marker search, MSBooster, genome selection, and the stage-2 searches.

Genome selection is skipped whenever `search_results/<sample>/marker/<sample>.fasta` exists. After a change to stage-1 rescoring or to the genome-selection settings, delete that FASTA. The stage-2 searches are then older than the new database and run again. A sample that previously produced no stage-1 PSMs has no FASTA and is selected again on the next run without this step.

Do not run two jobs on the same mzML files. MSFragger writes `<sample>.mzBIN_calibrated` next to each mzML, and the second job will overwrite it. Queue the second job with `--dependency=afterany:<jobid>`.

### Open-search entrapment

To measure the error of an open search, run the entrapment command on the same parameter file. It reuses the marker search and genome selection, adds shuffled proteins to the stage-2 database, and writes `opensearch_entrapment_MAG_search/`. It stops before quantification and taxonomy. Read `combined/entrapment_summary.tsv`. This evaluation does not replace the normal open-search result.

```bash
java -Xms16g -Xmx100g -XX:+UseParallelGC \
  -jar metapilot-cli-1.2.0-shaded.jar mag-opensearch-entrapment \
  --params /data/studies/demo/mag_opensearch_parameters.json
```

## 4. DIA Marker Deep

This is the standard Deep workflow: one Marker4 search across every DIA file, then iterative rounds on all files together, five genomes per round. The result folder is `marker_deep/`.

DIA files may be `.raw` or `.dia`. Paths are read on the compute node.

```json
{
  "workflow": "DIA_MARKER",
  "input_output": {
    "output_directory": "/data/studies/demo",
    "micro_db": "human-gut",
    "append_host_db": false,
    "host_database": "/data/metapilot/host/human_reviewed.fasta",
    "experiments": [
      {
        "file_path": "/data/studies/demo/dia/sample_a.raw",
        "experiment_name": "sample_a"
      }
    ]
  },
  "mag_database": {
    "catalogue_id": "human-gut",
    "identifier": "MGYG",
    "catalogue_version": "v2.0.2"
  },
  "dia_settings": {
    "library_search": true,
    "marker_mode": "DEEP",
    "genomes_per_round": 5
  },
  "basic_parameters": {
    "threads": 24,
    "ms2_scan_mode": "DIA"
  }
}
```

`marker_mode` is `DEEP` for this together workflow. `FOCUS` and `COMPREHENSIVE` are the single-pass marker modes and write `marker_focus/` and `marker_comprehensive/`. `DEEP_PER_FILE` repeats the iterative search once per file; use it for a large cohort, and leave `DEEP` as the standard run.

```bash
java -jar metapilot-cli-1.2.0-shaded.jar params-validate \
  --params /data/studies/demo/dia_marker_parameters.json
```

```bash
#!/bin/bash
#SBATCH --job-name=metapilot-dia-deep
#SBATCH --cpus-per-task=24
#SBATCH --mem=160G
#SBATCH --time=72:00:00
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

export JAVA_HOME=/opt/java/jdk-21
export DOTNET_ROOT=/opt/dotnet
export PATH="$DOTNET_ROOT:$JAVA_HOME/bin:$PATH"

cd /opt/metapilot
java -Xms16g -Xmx100g -XX:+UseParallelGC \
  -jar metapilot-cli-1.2.0-shaded.jar dia-marker \
  --params /data/studies/demo/dia_marker_parameters.json
```

The run writes `/data/studies/demo/marker_deep/`. The Marker4 search is `marker/marker.parquet`. Genome ranking is `marker/genome_ranked.tsv` and `marker/genome_selected.tsv`. The quantified peptide and protein tables, the taxonomy report, and the functional annotation are under `marker_deep/` after the final combined library search.

A second launch skips the Marker4 search when `marker/marker.parquet` is already present, and skips genome ranking when both genome tables are present. To repeat genome selection, delete those two tables. To repeat the Marker4 search, delete `marker.parquet`.

For an error estimate of Deep together, set `entrapment_enabled` to `true` in `dia_settings`, or run `dia-marker-deep-entrapment` on the same parameter file. That run writes `marker_deep_entrapment/` and does not replace `marker_deep/`.

## Other workflows

| Goal | Command or GUI workflow |
| --- | --- |
| Closed DDA search of the selected MAG genomes | `mag-fragpipe` |
| DIA from libraries you already have | `dia-seed` |
| Standard DDA against one database, without genome selection | DDA IGC in the Windows GUI |
| pFind MAG search | Windows GUI or `pfind-mag` |

`mag-fragpipe` and `dia-seed` take the same `--params` and `--resources` form as the two examples above.

## When a job fails

The Slurm `.err` file is usually empty, because the workflow log is in the result folder: `opensearch_MAG_search/run.log` or `logging.log` for an open search, and `logging.log` under the study folder for DIA Marker. The last lines of that log name the sample and the step.

| Message | What to do |
| --- | --- |
| `resources-check` reports a missing tool | Fix the path in `config/meta_resources.params`. The compute node must see the same path |
| `dotnet` is not found | Export `DOTNET_ROOT` in the job. FlashLFQ needs it |
| Marker search did not start | The spectrum path in the parameter file is not visible on the node |
| Stage-1 rescoring accepted 0 targets | The file has no usable marker evidence. Later samples continue. Delete nothing; the next run selects that sample again |
| Genome selection was skipped and the genomes look stale | Open search: delete `marker/<sample>.fasta`. DIA Marker Deep: delete `marker/genome_ranked.tsv` and `marker/genome_selected.tsv`. Then submit again |
| Java was killed near the memory limit | Raise `--mem`. Keep `-Xmx` about 50 GB under the job memory |
| Two open-search jobs searched the same mzML | Rerun them one after another so `.mzBIN_calibrated` is written by one job |

With a report, include the MetaPilot version, the parameter file, `resources-check` output, and the last 80 lines of the workflow log.
