+++
title = 'Launch from Vertex AI Workbench'
weight = 10
date = 2026-09-03
+++

# Copy FASTQs from BaseSpace using notebook 07

This walkthrough runs a BaseSpace-to-APGAP transfer end-to-end from [notebook 07](../../../notebook-templates/#07-launch-basespace-copy) on your Vertex AI Workbench. You'll create a batch-upload endpoint in the Portal, authenticate to BaseSpace once, pick files from a project, and transfer them. Downstream, APGAP's compliance cascade routes the files into your lab's analytical-dataset bucket automatically.

## Prerequisites

Before starting, confirm the items on the shared [Before you begin](../../../prerequisites/) checklist. Beyond those, this tutorial specifically needs:

- **A Workbench that opens cleanly and can list your project's buckets.** If notebook 01 runs without errors, you are in good shape.
- **An Illumina BaseSpace account with access to the project you want to pull from.** BaseSpace project invites are handled outside APGAP; ask the person who ran the sequencing to invite you.
- **Permission to create a batch-upload endpoint from the Portal.** Lab members typically have this by default; if the Batch Endpoints tab is not visible on your lab's page, contact your Platform Admin.
- **Enough Workbench disk to buffer the largest single file.** The Workbench default (100 GB) is fine for typical FASTQs (100 MB to 2 GB each); larger files may need a resize.

## Create a batch-upload endpoint in the Portal

The endpoint gives you two things: an ingest bucket URI in the `batch-upload-lab*` pattern, and a scoped service account key JSON that has write access to just that bucket. Endpoints are **lab-scoped**, so you create them from your lab's page, not from an individual project.

1. Open the APGAP Portal, navigate to your **lab's page**, and switch to the **Batch Endpoints** tab. Existing endpoints (if any) are listed with their status (Active, Expired) and file count.

![APGAP Portal lab page Batch Endpoints tab showing existing endpoints listed with ID, ingest bucket, kind, user, lab, status, and file count columns](/images/basespace-copy/portal-01-batch-endpoints-tab.png)

2. Click **Upload Sequences** (top-right of the sequences area of your lab page), then pick **Batch - BaseSpace** from the dropdown. The other options (`GUI Upload`, `Command Line (Batch)`) are for different upload flows, not this pipeline.

![Upload Sequences dropdown menu open with three options visible: GUI Upload, Command Line (Batch), and Batch - BaseSpace highlighted](/images/basespace-copy/portal-02-upload-sequences-menu.png)

3. In the **Batch Upload - BaseSpace** dialog, set:
   - **Lifetime Duration**: how long the endpoint stays active. Weeks or months; pick something matched to how long your upload will take. Endpoints expire after this window and uploads through them start failing with permission errors. You can always create a fresh one.
   - **Description**: something that ties the endpoint to what you are uploading (for example, `WGS run 2026-09-02, samples 1-24`).

Click **Confirm**.

![Batch Upload BaseSpace create dialog with Lifetime Duration set to 1 Weeks and Description filled in with Docs capture 2026-09-09](/images/basespace-copy/portal-03-create-endpoint-dialog.png)

4. The Portal creates the endpoint and presents the credentials.

![Batch Upload BaseSpace endpoint-created dialog showing Upload to lab, Service Key filename with copy button, URI with copy button, and a link to the APGAP Nextflow Basespace workflow](/images/basespace-copy/portal-04-endpoint-created-key-copy.png)

Copy both values now; the dialog closes after you dismiss it and the Service Key contents are only exposed here.

- **Service Key**: click the copy-to-clipboard button beside the filename. This copies the **JSON contents** to your clipboard (not the file itself). Save it locally as a backup if you want, but you do not upload it to the Workbench: you'll paste the JSON contents directly into a widget in the notebook.
- **URI**: click the copy-to-clipboard button beside the URI. What lands on your clipboard is just the bucket name in the shape `batch-upload-lab<lab-id>-<timestamp>-<uuid>`, without the `gs://` scheme. Paste it somewhere you can retrieve it (a scratch text file, your terminal); you'll add the `gs://` prefix when you paste it into the notebook widget in a moment. (The widget also auto-prepends `gs://` if you paste without it, so this is forgiving.)

Do not share the key JSON in Slack or paste it into a public gist. It has write access to your lab's ingest bucket for the endpoint's lifetime.

## Open notebook 07

1. From the Portal, open your project's page and click **NOTEBOOK LINK** in the Vertex AI Workbench card. JupyterLab opens in a new browser tab.

![APGAP Portal project page with the Vertex AI Workbench card and the NOTEBOOK LINK button visible](/images/shared/portal-project-notebook-link-button.png)

2. In the JupyterLab file browser (left sidebar), navigate to `apgap-notebooks/notebooks/` and click `07-launch-basespace-copy.ipynb`.

![JupyterLab file browser open at /home/jupyter/apgap-notebooks/notebooks/ with 07-launch-basespace-copy.ipynb selected](/images/shared/jupyterlab-filebrowser-nb07.png)

3. When JupyterLab prompts for a kernel, pick **`Python 3 (Local)`** under "Start python Kernel". Do not pick `Python 3 (ipykernel)`, `PyTorch`, or `TensorFlow`; those environments are missing libraries this notebook needs.

![JupyterLab Select Kernel dialog with Python 3 (Local) highlighted](/images/shared/jupyterlab-kernel-selector.png)

## BaseSpace source parameters

The first code cell holds parameters for what to transfer from BaseSpace: which project, whether to filter FASTQs only, whether to force re-upload previously-transferred files, and where to buffer downloads locally. For a first run, the defaults are fine:

- `BASESPACE_PROJECT_ID` blank: the notebook auto-selects if you have exactly one BaseSpace project, or prompts you to pick from a list.
- `FASTQ_ONLY = True`: skip auxiliary files (workflow metadata, log files, etc.).
- `SELECTED_FILE_IDS = []`: transfer everything that matches `FASTQ_ONLY`.
- `FORCE_REUPLOAD = False`: skip files already recorded in `~/.apgap/transferred.json`.
- `LOCAL_STAGING_DIR = "/tmp/basespace-transfer"`: where BaseSpace downloads land temporarily before upload.

Run the parameter cell.

## Paste the endpoint URL and SA key

The next code cell opens a widget with two paste-in fields. Fill both from the endpoint you created in the Portal:

- **Bucket URL**: the ingest bucket URI, prefixed with `gs://`. Example: `gs://batch-upload-lab6-20260909180601-fa38565e`.
- **SA key JSON**: the entire SA key JSON contents (starts with `{ "type": "service_account", ...`). Do not paste the bucket URI here.

Click **Save endpoint config**. The widget writes both values to files outside the notebook directory (`~/.apgap/sa-key.json` mode 600, and `~/.apgap/endpoint.json`) and prints a confirmation.

![Endpoint config widget after saving showing Bucket URL, SA key JSON textarea contents redacted, Save endpoint config button, and success messages confirming both files were written to /home/jupyter/.apgap/](/images/basespace-copy/vertex-02-endpoint-widget-saved.png)

The private key never lands inside the `.ipynb` file, so it cannot leak into a git commit even if you save and push the notebook. If both files already exist from a previous run, this cell prints "Endpoint config already saved" and skips the widget. To re-paste for a new endpoint, delete both files and re-run the cell.

## Verify the endpoint config

The next code cell reads the saved files back, validates the SA key JSON, and prints the service account email, GCP project, ingest bucket URL, and local staging directory. If either file is missing or the SA key does not look like a valid GCP service account JSON, this cell errors with a clear pointer at the fix.

![Verification cell output showing Service account, Project, Ingest bucket, and Local staging paths, with the service account email and GCP project redacted](/images/basespace-copy/vertex-03-endpoint-verification.png)

If you see the service account and project printed cleanly, you are ready to install the BaseSpace CLI.

## Install the BaseSpace CLI

The Workbench image doesn't ship the BaseSpace CLI or the `google-cloud-storage` Python package. The next cell installs both.

- **`bs` CLI**: downloaded from Illumina as a standalone binary and dropped into `~/.local/bin/`. This is the exact path the auth cell in the next section calls.
- **`google-cloud-storage`**: installed via pip into the user site-packages.

Run the install cell. Wall clock: about 1-2 minutes cold. If either install fails (network glitch, Illumina's release site 5xx-ing), just re-run the cell.

## Authenticate to BaseSpace (one-time setup)

This step is the most likely stuck point on the first run. It needs a browser-based OAuth flow that runs in a JupyterLab terminal, not inside a cell.

On the first run, the notebook's auth cell detects that no BaseSpace credentials exist yet and prints step-by-step instructions before raising to stop execution.

![Notebook output showing No BaseSpace credentials found, First-time setup needed, followed by five numbered steps for running bs auth in a JupyterLab terminal, ending in a RuntimeError with the same instructions](/images/basespace-copy/vertex-04-bs-auth-prompt.png)

Follow those steps:

1. Open a JupyterLab terminal (**File → New → Terminal**).
2. Run `~/.local/bin/bs auth`. The terminal prints a URL and waits.
3. Copy the URL into a browser tab on your local machine where you are signed in to Illumina BaseSpace, and click Accept.
4. The terminal picks up the completed handshake and prints something like `You are now authorized as <your Illumina email>` and confirms it wrote the token to `~/.basespace/default.cfg`.
5. Return to the notebook and re-run the auth cell. It confirms credentials are found and continues.

![JupyterLab terminal showing the bs auth command, the Illumina OAuth URL, and a Welcome success message with the user name redacted](/images/basespace-copy/vertex-05-bs-auth-terminal.png)

This is a one-time step per Workbench VM. On subsequent notebook runs the cached credentials at `~/.basespace/default.cfg` get picked up automatically and the auth cell just prints "BaseSpace credentials found."

## Find files on BaseSpace

The next cell enumerates BaseSpace projects visible to your account. If you have access to exactly one project, it auto-selects that project and saves the choice to `~/.apgap/endpoint.json` for subsequent runs. If you have access to more than one, it renders a dropdown widget with a **Use this project** button; select a project, click the button, and re-run the cell to continue.

Then the file-discovery step lists everything in the selected project, applies the `FASTQ_ONLY` filter from the parameter cell, and prints a table of file IDs, sizes, and filenames. If `SELECTED_FILE_IDS` was populated in the parameter cell, only those IDs are kept; otherwise all files matching the filter are queued.

![Notebook output showing auto-selected the only project, listing files in the BaseSpace project, project contains 308 files total 194 FASTQ plus 114 auxiliary, FASTQ_ONLY True so transferring 194 FASTQ file(s) only, then a table of file IDs, sizes, and filenames (redacted), with a total size of 2.78 GB](/images/basespace-copy/vertex-06-project-file-listing.png)

**To transfer only specific files**: run cells through the file discovery once with `SELECTED_FILE_IDS = []` to see the listing, note the IDs of the files you want (leftmost column), then interrupt the kernel, edit `SELECTED_FILE_IDS` in the parameter cell to that list, and re-run from the parameter cell forward.

## Pre-transfer summary

Before starting the transfer, the notebook prints a concise summary of what is about to happen: the BaseSpace source project, the ingest bucket destination, the file count, the total size, and an estimated wall clock. It also restates the cascade behavior so you know what to expect after upload.

![Pre-transfer summary showing About to transfer with Source BaseSpace project 478730705, Destination ingest bucket URI, Files 194, Size 2.78 GB, Estimated time, followed by explanatory text that after files land in the ingest bucket the APGAP backend cascade takes over automatically](/images/basespace-copy/vertex-07-pre-transfer-summary.png)

Review it. If the file count or size looks off, stop and adjust `SELECTED_FILE_IDS`, `BASESPACE_PROJECT_ID`, or `FASTQ_ONLY` in the parameter cell before continuing.

## Run the transfer

The transfer cell iterates through the selected files. For each file it checks two skip conditions: is the filename already recorded in `~/.apgap/transferred.json` (from a prior successful run), and is the file already present in the ingest bucket (from an interrupted run whose files haven't cleared the cascade yet). If either matches, the cell skips that file. Otherwise it downloads from BaseSpace to `LOCAL_STAGING_DIR`, uploads to the ingest bucket with the endpoint SA credentials, deletes the local temp file to reclaim disk, and records success in the transfer log (keyed by filename).

Progress prints as each file completes with a per-file `ok` status, elapsed time, and throughput.

![Transfer cell in progress showing per-file lines each with a position out of total counter, filename (redacted), size in MB, and an ok status line with elapsed seconds and throughput in MB per second](/images/basespace-copy/vertex-08-transfer-in-progress.png)

The cell handles interruption gracefully. If you cancel partway, resuming picks up from the first not-yet-completed file. Wall clock varies with total transfer size, dominated by BaseSpace's outbound throughput.

When the transfer finishes, the cell prints a summary of what moved.

![Transfer summary showing Uploaded 194 files, Skipped transfer log 0, Skipped still in bucket 0, Failed 0, Wall clock 427 seconds, Bytes moved 2.78 GB, Average throughput 6.7 MB per second](/images/basespace-copy/vertex-09-transfer-summary.png)

## What success looks like

Once the transfer summary shows the expected count with zero failures, the post-transfer inventory cell lists what is currently in the ingest bucket. Files appear briefly and vanish over the next few minutes as the compliance cascade (Pub/Sub then DLP scan then SRA scrubber then move to lab bucket) processes each one.

![Post-transfer inventory cell output showing Current ingest bucket contents with sample filenames (redacted) and sizes for files remaining in the bucket](/images/basespace-copy/vertex-10-post-transfer-inventory.png)

Files still visible in the ingest bucket are ones the cascade has not caught up to yet. The tail of the inventory prints a count of remaining files and reminds you that once the cascade completes, they show up in the lab bucket and become readable by notebook 02.

![Bottom of the inventory listing followed by a summary line, N files still in ingest bucket, once the cascade processes them they show up in your lab bucket and are visible in notebook 02](/images/basespace-copy/vertex-11-cascade-explainer.png)

## Verification checklist

<details>
<summary>Click through these before considering the transfer complete</summary>

- [ ] Transfer cell printed the expected uploaded/skipped/failed counts with no red errors.
- [ ] Post-transfer inventory ran without permission errors on the ingest bucket.
- [ ] Wait 5-15 minutes for the cascade to catch up, then check your lab's analytical-dataset bucket, or the **Sequences tab on your lab's page in the Portal** (endpoints are lab-scoped, so ingested files land on the lab-level Sequences view, not on a specific project's page). Navigate: Portal → your lab's page → **Sequences** tab.
- [ ] If any file is still stuck in the ingest bucket after 30 minutes, the cascade may have failed on it; check the lab's Sequences tab for a `FAILED` status and contact your Platform Admin.
- [ ] If you plan to launch a pipeline against the transferred data next, run [notebook 02](../../../notebook-templates/#02-read-your-data) first to confirm the files are readable from your project's analytical-dataset bucket.

</details>

## Where to go next

- Back to the [basespace-copy overview](../) for the compliance-cascade explanation, parameter reference, and troubleshooting.
- For the equivalent flow launched from the Seqera Platform Launchpad, see [Launch from Seqera](../tutorial-seqera/).
- To read the FASTQs you just transferred: [notebook 02 (`02-read-your-data`)](../../../notebook-templates/#02-read-your-data).
- To launch a pipeline against them: [notebook 03 (`03-launch-a-pipeline`)](../../../notebook-templates/#03-launch-a-pipeline).
