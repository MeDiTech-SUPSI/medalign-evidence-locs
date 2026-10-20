# Locating Clinician-Cited Evidence for Retrieval-Augmented Generation over Longitudinal Electronic Health Records

Code accompanying the [paper]() (under review).

## Overview

This repository extends [MedAlign](https://doi.org/10.1609/aaai.v38i20.30205) with evidence locations for evaluating retrieval over longitudinal electronic health records (EHRs). MedAlign provides realistic clinical instructions and clinician-written answers based on records containing both structured events and clinical notes. Its answers include cited evidence strings, but these citations do not directly identify the corresponding events in the record. Mapping them to candidate events adds retrieval references alongside the existing generation annotations, supporting evaluation of both stages of single-patient retrieval-augmented generation (RAG).

The method identifies evidence locations for 239 of 302 distinct instruction–patient pairs. Structured events and note chunks are evaluated separately using three metrics: **hit@k** indicates whether at least one located event or chunk is retrieved; **event recall (R@k)** measures the fraction of located events or chunks retrieved; and **evidence recall (evidence_R@k)** measures the fraction of evidence entities recovered. An evidence entity may correspond to several alternative locations, and retrieving any one of them counts as recovery of that entity. Among BM25 and four dense baselines, **bge-base-en-v1.5** achieves the highest top-50 evidence recall for notes (**58.3%**), while **all-MiniLM-L6-v2** performs best for structured events (**78.6%**). Metrics are macro-averaged across instruction–patient pairs within each modality. These metrics measure recovery of clinician-cited evidence rather than comprehensive retrieval quality.

The repository includes a MedAlign-specific MEDS-to-OMOP CDM 5.4 conversion pipeline to facilitate the transfer of future solutions to observational clinical research settings.

## Data Access and Requirements

Authorized access to MedAlign must be obtained through the Redivis portal in accordance with the dataset’s data-use agreement. Reproducing the workflow requires MedAlign records in MEDS format and the annotation files. Localization outputs retain source annotation text and must be handled under the same access restrictions as the input data.

Create and activate a Conda environment, then install the required packages:

```bash
conda create -y -n medalign-evidence-locs python pip
conda activate medalign-evidence-locs
python -m pip install "psycopg[binary]" pandas pyarrow numpy rank_bm25 sentence-transformers
```

## Converting MedAlign from MEDS to OMOP

First, build the vocabulary bundle using [Athena](https://athena.ohdsi.org/) vocabulary release v5.0 (27 February 2026), including the following vocabularies:

> ABMS, CMS Place of Service, CPT4, CVX, Cancer Modifier, HCPCS, HES Specialty, ICD10CM, ICD10PCS, ICD9CM, ICD9Proc, ICDO3, LOINC, Medicare Specialty, NAACCR, NUCC, PPI, RxNorm, RxNorm Extension, and SNOMED. Athena also bundles Domain, Ethnicity, Gender, Provider, Race, Type Concept, UCUM, Visit, and Vocabulary.

Then generate [OMOP CDM 5.4](https://ohdsi.github.io/CommonDataModel/) tables as CSV files. [OMOP Lite](https://github.com/Health-Informatics-UoN/omop-lite) can be used to load the tables into PostgreSQL.

```bash
python scripts/build_vocab_names.py \
  --meds-dir /path/to/meds_omop_medalign \
  --athena-dir /path/to/athena_dir \
  --out out/vocab_bundle

python scripts/meds_to_omop.py \
  --meds-dir /path/to/meds_omop_medalign \
  --vocab-bundle out/vocab_bundle \
  --out out/medalign_omop_tables
```

## Evidence Localization

Map clinician-cited evidence strings and structured snippets from clinician-written answers to candidate structured OMOP events, note events, or demographic information. The output comprises distinct evidence instances, each with a list of alternative candidates.

```bash
python scripts/localize.py \
  --host <host> \
  --port <port> \
  --dbname <dbname> \
  --user <user> \
  --password <password> \
  --schema <schema> \
  --instructions-path /path/to/all-instructions-rouge.tsv \
  --responses-path /path/to/clinician-instruction-responses.tsv \
  --out out
```

## Retrieval Evaluation

Evaluate retrieval within each patient record.

```bash
python scripts/evaluate.py \
  --host <host> \
  --port <port> \
  --dbname <dbname> \
  --user <user> \
  --password <password> \
  --schema <schema> \
  --gold-path out/localization.jsonl \
  --retrievers dense \
  --model BAAI/bge-base-en-v1.5 \
  --out out
```
