# Research Paper Metadata Extraction Pipeline (Google Colab)

## Introduction

This notebook takes a folder of scientific PDFs and turns them into a single, structured Excel matrix: one row per paper, one column per extracted attribute (title, authors, DOI, funding, methodology, technology, research theme, and more — 19 fields in total). It's built to run entirely in Google Colab against Google Drive, so a batch of papers can be processed over more than one sitting without losing work.

Each paper is read in full, sent to Google's Gemini model in a single schema-enforced call, and the structured result is saved incrementally so the run can be safely stopped and resumed. The end product is a styled `.xlsx` workbook saved to Drive and downloaded automatically at the end of the run.

## Background

Reading and coding a stack of research papers by hand — pulling out bibliographic details plus more interpretive fields like methodology, technology, and research theme — is slow and inconsistent across papers and reviewers. This pipeline automates that first pass: an LLM reads each paper's full text and fills in the same 19-field schema every time, with an explicit instruction to answer "NA" rather than guess when something isn't stated in the text.

This notebook is the Colab-adapted version of a broader effort to build LLM-assisted bibliometric tooling for NOAA marine sanctuary research literature (work that has separately covered the Flower Garden Banks and Monterey Bay National Marine Sanctuary bibliographies). Earlier versions of this particular pipeline made one LLM call per field per paper — at 30 papers x 19 fields that's 570 calls — and built each paper's context by joining overlapping text chunks, which silently duplicated a large share of every paper's text in every prompt. This version consolidates all 19 fields into a single structured call per paper, drops the chunk-overlap duplication from the context sent to the LLM (chunking is kept only for a separate, not-yet-active embeddings/vector-search step), and adds retry-with-backoff and resumability so a 30-paper batch can survive rate limits and Colab session interruptions.

As with any LLM-assisted extraction, the interpretive fields in particular (main topic, research theme, refined research topic, key methodology, technology, branch of science) are a starting point for review, not a final verified answer — they should be spot-checked against the source PDF before being treated as ground truth.

## Usage Guide

### Prerequisites

- A Google account with Google Drive and Google Colab access
- A Gemini API key
- Your PDFs uploaded to a folder in Google Drive (e.g. `My Drive/papers`)

### Setup

1. **Open the notebook in Colab** and paste each `# Cell N` block from the script into its own notebook cell, in order.
2. **Add your API key to Colab Secrets** — click the key icon in the left sidebar, add a secret named `GEMINI_API_KEY` with your Gemini API key as the value, and toggle "Notebook access" on.
3. **Create your papers folder in Drive** — e.g. `My Drive/papers` — and upload all the PDFs you want processed into it.
4. **Update `PDF_DIR`** in the config cell if you used a different folder name or path than `/content/drive/MyDrive/papers`.

### Running the pipeline

1. **Run Cell 1** to install dependencies.
2. **Run Cell 2** — this mounts your Google Drive (you'll be asked to authorize access) and pulls in your Gemini API key from Secrets.
3. **Run Cells 3-7** in order to define the configuration, the extraction schema, the PDF-loading functions, the Gemini extraction logic, and the Excel export function.
4. **Run Cell 8** to execute the full pipeline: it discovers every PDF in `PDF_DIR`, loads each paper's text, sends one structured extraction request per paper (several run concurrently), saves results to Drive as it goes, exports the styled Excel workbook, previews the first 10 rows in the notebook, and downloads the finished file to your computer.

### If something goes wrong mid-run

- **Rate limit or transient errors**: each paper's extraction call retries automatically (up to `MAX_RETRIES`, with exponential backoff). If you see a lot of retry messages, lower `MAX_WORKERS` (default 5) to reduce concurrent calls.
- **Colab disconnects or times out**: your progress isn't lost. Results are saved to `OUTPUT_JSON` in Drive after every paper. Just re-run Cell 8 (or the whole notebook) — papers already recorded there are skipped automatically, and only the remaining papers are processed.
- **A specific paper keeps failing**: after `MAX_RETRIES` attempts it's recorded with `"N/A"` in every field rather than blocking the rest of the batch, so you can identify and re-run it individually later.

### Output

- `research_papers_extraction_matrix.xlsx` — the final formatted matrix, saved to your Drive output folder and downloaded to your computer.
- `paper_records.json` — the raw structured results per paper, also in Drive; this is what makes the resume behavior possible and can be reloaded independently of the Excel export if needed.

### Notes & Known Limitations

- Model name strings (`gemini-3.1-pro-preview` for extraction, `gemini-embedding-2-preview` for the embeddings step) should be checked against your current Gemini API access, since preview model names change over time.
- The embeddings/Chroma vector store setup referenced in the pipeline is scaffolding for a future retrieval step (e.g. cross-paper semantic search) — it currently has no effect on the metadata extraction output.
- Treat the interpretive fields as a first-pass draft; verify against the source text before using them in downstream analysis.

## Possible Next Steps

- Build a small hand-verified gold-standard subsample to validate field-level extraction accuracy.
- Activate the embedding/Chroma store for cross-paper semantic search or question answering.
- Add a lightweight review step (e.g. flagging low-confidence or "NA"-heavy records) before the matrix is treated as final.
