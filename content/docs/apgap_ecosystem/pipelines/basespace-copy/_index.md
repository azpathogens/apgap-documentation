+++
title = 'basespace-copy'
weight = 10
date = 2026-09-03
bookCollapseSection = true
+++

# basespace-copy

**basespace-copy** moves FASTQ files out of Illumina BaseSpace and into your lab's APGAP-analyzable storage. It handles the transfer through a compliance cascade so incoming data goes through the required PII and human-read checks before it lands in a bucket your other notebooks can read.

## When to use it

Reach for basespace-copy when new sequencing runs have finished on Illumina BaseSpace and you need those FASTQs available to the downstream APGAP analysis tools (notebook 02, notebook 03, notebook 06). Typical use cases:

- New sequencing run just completed and you want the FASTQs on-cloud, ready to launch a pipeline against.
- Backfilling older BaseSpace runs into APGAP so you can rerun them under the same analytical stack.
- Automating ingest for a lab that runs sequencing frequently and doesn't want a manual download-then-upload step for every batch.

If your FASTQs are already in a GCS bucket your project can read, or if you're pulling from NCBI SRA rather than BaseSpace, you don't need this pipeline. See [notebook 04 (`04-download-from-public-repos`)](../../notebook-templates/#04-download-from-public-repos) for the SRA / ENA / GenBank path.

## How it works (receiving-dock analogy)

Think of basespace-copy as the loading dock at a secure facility. A shipment arrives from an outside sender (BaseSpace), it's placed on the intake pad (the batch-upload endpoint bucket), a security team scans it for anything sensitive (DLP scan for PII), removes what shouldn't leave the facility (SRA scrubber removes human reads), and only then forwards the cleaned shipment to the recipient's mailbox (your lab's analytical-dataset bucket).

Under the hood, each transfer moves files through this cascade:

```
Illumina BaseSpace
      │
      │  bs download + gsutil cp  (this pipeline's job ends here)
      ▼
Portal batch-upload endpoint bucket (gs://batch-upload-lab*)
      │
      │  Pub/Sub OBJECT_FINALIZE  (APGAP backend, automatic)
      ▼
DLP scan  (checks for PII, quarantines anything sensitive)
      │
      ▼
SRA scrubber  (removes reads matching the human genome)
      │
      ▼
Lab's analytical-dataset bucket  (visible to notebooks 02, 03, 06)
```

The pipeline's only job is authenticating to BaseSpace, authenticating to the endpoint bucket as its scoped service account, and transferring files. Everything downstream of the ingest bucket is APGAP backend territory and runs automatically.

## Inputs

| Input | Format | Notes |
| --- | --- | --- |
| BaseSpace project | project ID or name | The project you want to pull FASTQs from. Nb 07 lists projects interactively if you don't set one. |
| BaseSpace access token | string | Obtained via `bs auth` on any machine, or from the BaseSpace developer portal. See troubleshooting for which one to use. |
| Batch-upload endpoint | Portal-created endpoint | Created from your project's page in the APGAP Portal. Gives you two things: an ingest bucket URI (`gs://batch-upload-lab*`) and a scoped service account key JSON. |
| Service account key | JSON file | Downloaded from the Portal at endpoint creation time. Scoped to the ingest bucket only and short-lived; do not reuse across endpoints. |

The service account key is the sensitive piece. Notebook 07 takes the JSON contents pasted into a widget textarea (not a file upload) and writes them to `~/.apgap/sa-key.json` at chmod 600, outside the notebook directory. Widget values are runtime-only, so the key never lands in `.ipynb` output. The Seqera pipeline reads the key from a Pipeline Secret (a workspace-scoped credential that Seqera injects as an environment variable at launch time), base64-encoded so the multi-line JSON fits in an env var.

## Outputs

basespace-copy doesn't produce analytical outputs of its own. Its output is a set of FASTQ files written to the batch-upload endpoint bucket, which then triggers the compliance cascade.

Where files land depends on which stage you're watching:

- **Immediately after transfer**: files appear in the batch-upload endpoint bucket (`gs://batch-upload-lab<...>`). They vanish over the next few minutes as the cascade picks them up. That is expected behavior.
- **After DLP scan + SRA scrubber** (typically 5-15 minutes for a standard batch; individual very large files can take longer): cleaned files land in your lab's analytical-dataset bucket. Once they're there, notebook 02 and downstream pipelines can read them.
- **On the APGAP Portal**: newly-ingested files appear on the **Sequences tab of your lab's page** (not the project page) once the cascade completes.

## Two ways to launch it

**From Vertex AI Workbench (notebook 07).** Open `07-launch-basespace-copy.ipynb` in your Workbench. Paste the endpoint URL and SA key JSON into the endpoint-config widget, authenticate to BaseSpace once via `bs auth` in a terminal, and run the transfer. Best fit when you want a visible transfer with interactive project + file selection.

**From Seqera Platform Launchpad.** Register (or launch a pre-registered) `basespace-copy-scrubber` pipeline. Store your BaseSpace token and the base64-encoded SA key as Seqera Pipeline Secrets, then launch with a small params file naming the BaseSpace file IDs and the target endpoint bucket. Best fit when you want a persistent launch definition and a centralized run history across the workspace. On APGAP deployments the pipeline is usually pre-registered; the [Launch from Seqera](tutorial-seqera/) tutorial covers both the everyday launch flow and the admin flow to register it from scratch.

Step-by-step walkthroughs for both paths land in the tutorial pages under this section.

## Parameters

The Seqera pipeline exposes two required parameters plus two Pipeline Secrets. The notebook path uses code-cell variables instead of CLI parameters; see the notebook's parameter cell for the equivalents.

| Parameter | Where set | What it controls |
| --- | --- | --- |
| `--input_files` | Params file | List of BaseSpace file IDs to download. Get these by running `bs contents project -i <project-id>` on any authenticated machine, or from the BaseSpace web UI. |
| `--outdir` | Params file | The batch-upload endpoint bucket URI, in the shape `gs://batch-upload-lab<lab-id>-<timestamp>-<uuid>` (the Portal fills in the concrete values when you create the endpoint). Must match the `batch-upload-lab` prefix pattern; the scrubber cascade only fires on buckets that do. |
| `BASESPACE_ACCESS_TOKEN` | Seqera Pipeline Secret | Your BaseSpace API access token. Use the token from `~/.basespace/default.cfg` after `bs auth`, not the developer portal token. See troubleshooting. |
| `BATCH_UPLOAD_SA_KEY_B64` | Seqera Pipeline Secret | Base64-encoded content of the SA key JSON downloaded from the Portal endpoint. Rotate per endpoint (endpoint keys are short-lived and scoped to one bucket). |

The pipeline uses an explicit `gsutil cp` inside the upload process rather than Nextflow's `publishDir` (Nextflow's built-in mechanism for copying outputs to a destination bucket). This is intentional. The compute environment's default service account cannot write to the batch-upload endpoint bucket, and `publishDir` runs from the head process using that ambient identity. Running the upload from inside the process script lets it authenticate as the scoped batch-upload service account instead.

## Troubleshooting

<details>
<summary><code>bs auth</code> browser flow doesn't complete or times out</summary>

`bs auth` opens an OAuth flow that requires a browser session logged in to Illumina. On a Workbench VM you don't have a browser, so `bs auth` prints a URL and waits.

**Fix**: copy the printed URL, open it in a browser tab on your local machine where you're already signed in to Illumina BaseSpace, complete the login, and return to the Workbench terminal. The `bs` CLI polls Illumina and picks up the completed handshake automatically. If the flow times out, re-run `bs auth`.

Once authenticated, credentials are written to `~/.basespace/default.cfg` on the Workbench and persist across sessions. You only do this once per Workbench VM.
</details>

<details>
<summary>Uploads fail with 401 or 403 from BaseSpace</summary>

You're using the wrong BaseSpace token. Two tokens exist:

- **`bs auth` token**: written to `~/.basespace/default.cfg` after the OAuth flow. This is what the `bs` CLI reads, and what the Seqera pipeline expects in `BASESPACE_ACCESS_TOKEN`.
- **Developer portal token**: issued through the BaseSpace developer console for API app registration. Different scope, different audience, does not work for `bs download`.

**Fix**: use the token from `~/.basespace/default.cfg` (the `accessToken` field). If you don't have one yet, run `bs auth` once to generate it.
</details>

<details>
<summary>Notebook cell errors with <code>FileNotFoundError</code> on <code>~/.apgap/sa-key.json</code></summary>

The endpoint-config widget cell (near the top of the notebook) hasn't been saved yet. The widget writes `~/.apgap/sa-key.json` and `~/.apgap/endpoint.json` when you click **Save endpoint config**; the verification cell right after it reads those files back. If you skip the widget's Save button, the verification cell errors with a `FileNotFoundError` pointing at whichever file is missing.

**Fix**: scroll up to the widget cell, confirm both fields are filled in (Bucket URL and SA key JSON), click **Save endpoint config**, and re-run the verification cell. To start over with a new endpoint, delete both files first (`rm ~/.apgap/sa-key.json ~/.apgap/endpoint.json`) and re-run the widget cell.
</details>

<details>
<summary>Upload succeeds but files vanish from the endpoint bucket</summary>

Expected behavior. The batch-upload endpoint bucket is a transient staging area. Once a file lands there, the APGAP compliance cascade (DLP scan → SRA scrubber → move to lab bucket) picks it up within seconds to minutes and moves it out.

**Where the files are now**: check your lab's analytical-dataset bucket, or the Sequences tab on your lab's page in the Portal (endpoints are lab-scoped, so ingested files land on the lab's Sequences view rather than a specific project's). If files never appear there and never come back to the endpoint bucket, either the cascade failed or the endpoint bucket wasn't the expected `gs://batch-upload-lab<...>` pattern (see next entry).
</details>

<details>
<summary>Files land in the endpoint bucket but never move to the lab bucket</summary>

The scrubber cascade only fires on buckets matching the `batch-upload-lab` prefix. If you set `--outdir` (Seqera) or the endpoint URL in the notebook widget to a different bucket, the transfer works but nothing downstream picks the files up.

**Fix**: create a batch-upload endpoint from the Portal (your lab's page → **Upload Sequences** menu → **Batch - BaseSpace**). The Portal generates the bucket name with the `batch-upload-lab<...>` shape; use exactly the URI it gives you. Do not point the pipeline at an arbitrary bucket in your project even if you can write to it.
</details>

<details>
<summary>Batch-upload endpoint expired or SA key rejected</summary>

Endpoint keys are scoped to one ingest bucket and have a Lifetime Duration you pick when you create the endpoint (Portal offers a Weeks or Months range; you set the exact number). Once the endpoint expires, uploads fail with a permissions error even if the key file itself is untouched.

**Fix**: create a fresh endpoint from the Portal, download the new SA key, and update your notebook parameter (or your Seqera Pipeline Secret) with the new value. The old endpoint bucket is left behind but empty. For Seqera Secrets, base64-re-encode the new key before storing it.
</details>

<details>
<summary>Seqera launch errors with <code>invalid credentials</code> after storing the SA key Secret</summary>

Seqera Pipeline Secrets are environment-variable-only. A JSON SA key spans multiple lines and contains special characters, which can be truncated or misinterpreted when pasted directly. This pipeline expects the key **base64-encoded** so the whole JSON fits cleanly into an env var, and the process script decodes it back to a file at runtime.

**Fix**: encode the SA key JSON first, then paste the base64 string into the Secret. Quote the path if it contains spaces.

```
# Linux / macOS
base64 -w 0 < "path/to/sa-key.json"

# PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("path/to/sa-key.json"))
```

Store the resulting single-line string as the `BATCH_UPLOAD_SA_KEY_B64` Secret. The pipeline handles the decode automatically.
</details>

<details>
<summary><code>bs download</code> hangs partway through a large file</summary>

BaseSpace's outbound throughput is the bottleneck for large transfers. A 10 GB FASTQ can take 30+ minutes. Nothing you can do about the wall clock, but the `bs` CLI does support resuming; if a download fails partway, re-running `bs download file -i <id>` picks up from where it stopped as long as the local partial file is still there.

**Fix**: if a specific file has been stuck for more than an hour, cancel and re-run just that file's download. If BaseSpace itself is returning errors, wait and retry; Illumina's API occasionally rate-limits.
</details>

## Source

- Pipeline (Seqera path): registered as `basespace-copy-scrubber` on the workspace; source repository is [`azpathogens/APGAP-nextflow-basespace`](https://github.com/azpathogens/APGAP-nextflow-basespace) on the `feature/scrubber-routing` revision. The Portal batch-upload endpoint dialog also links to this repository under "Nextflow Workflow."
- Notebook (Vertex path): [notebook 07 (`07-launch-basespace-copy`)](../../notebook-templates/#07-launch-basespace-copy) in [`apgap-notebooks`](https://github.com/azpathogens/apgap-notebooks).
- Downstream analysis: [notebook 02 (`02-read-your-data`)](../../notebook-templates/#02-read-your-data) reads the scrubbed FASTQs once they land in the lab bucket; [notebook 03 (`03-launch-a-pipeline`)](../../notebook-templates/#03-launch-a-pipeline) launches viralrecon (or another nf-core pipeline) against them.
