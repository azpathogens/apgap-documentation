+++
title = 'Launch from Seqera Platform'
weight = 20
date = 2026-09-03
+++

# Launch basespace-copy from Seqera Platform

This walkthrough runs the `basespace-copy-scrubber` pipeline from your project's Seqera Launchpad. On most APGAP deployments the pipeline is already registered; if it is not, a separate admin-flow section at the end covers first-time registration. Best fit when you want a persistent launch definition, a centralized run history, or when several people on the same lab will trigger transfers.

## Prerequisites

Before starting, confirm the items on the shared [Before you begin](../../../prerequisites/) checklist. Beyond those, this tutorial specifically needs:

- **A Seqera Platform workspace on your project.** If your project was created with the "Deploy Seqera workspace" toggle enabled at project-creation time, this is already in place.
- **A bound compute environment on the workspace.** The Portal provisions this alongside the workspace at project creation. To confirm: in the workspace's left sidebar, click **Compute Environments**; you should see at least one row with an **Available** state. If nothing is listed, the workspace was created without a compute environment; contact your Platform Admin.
- **A pre-registered `basespace-copy-scrubber` pipeline on your workspace's Launchpad.** APGAP deployments usually ship this pre-registered; check the Launchpad first. If the pipeline row is missing, contact your Platform Admin and share the [Registering from scratch](#registering-from-scratch-admin-flow) section below (that flow is admin-only, not for regular users).
- **An Illumina BaseSpace account** with access to the project you want to pull from, and a valid BaseSpace access token (see "Prepare credentials" below).
- **Permission to create a batch-upload endpoint from the APGAP Portal.** Same as the notebook path; see the [Launch from Vertex AI Workbench](../tutorial-vertex/) tutorial for the Portal endpoint creation steps.

If you want a primer on how the Seqera workspace fits alongside your Vertex Workbench, see the [Seqera Platform primer](../../../seqera/).

## Opening the workspace

From the APGAP Portal, click the **WORKSPACE LINK** in your project's Seqera Workspace card. Seqera Platform opens in a new browser tab, dropped into your workspace's Launchpad.

![APGAP Portal project page showing the Seqera Workspace card with the WORKSPACE LINK button visible](/images/shared/portal-project-workspace-link-button.png)

![Seqera Launchpad view listing registered pipelines including basespace-copy-scrubber, tostadas-measles-vadr, and other pipelines with their versions and last-updated dates](/images/basespace-copy/seqera-01-launchpad.png)

If `basespace-copy-scrubber` is already listed, continue with "Prepare credentials" below. If it is not, jump to [Registering from scratch](#registering-from-scratch-admin-flow) first.

Clicking on the pipeline row (not the Launch button) opens its detail page, which surfaces the workflow repository URL (`azpathogens/APGAP-nextflow-basespace` on the `feature/scrubber-routing` revision) and a version-history strip.

![basespace-copy-scrubber pipeline detail page showing Name, Version, Description, Workflow repository URL, Workflow revision feature scrubber-routing, and Pipeline versions block with the default version and creation timestamp (user/date redacted)](/images/basespace-copy/seqera-02-pipeline-details.png)

## Prepare credentials

The pipeline reads two credentials from Seqera Pipeline Secrets. Set them once per workspace and they persist across launches.

### 1. Create a batch-upload endpoint (and download the SA key)

Follow the Portal endpoint creation steps from the [Vertex tutorial's "Create a batch-upload endpoint" section](../tutorial-vertex/#create-a-batch-upload-endpoint-in-the-portal). You'll come away with:

- An **ingest bucket URI** in the `gs://batch-upload-lab*` pattern.
- A **service account key JSON** file, downloaded once at endpoint creation.

Keep the key file somewhere local. You'll base64-encode it in a moment.

### 2. Get a BaseSpace access token

The pipeline authenticates to BaseSpace using an access token stored in a Pipeline Secret. Two ways to get one:

- **From `bs auth` (recommended)**: on any machine where you can run the `bs` CLI, run `bs auth` and complete the browser OAuth flow. The token lands in `~/.basespace/default.cfg`. Copy the `accessToken` value from that file.
- **From the BaseSpace developer portal**: log in at [developer.basespace.illumina.com](https://developer.basespace.illumina.com), create an API app, and copy the token issued to it.

**Use the `bs auth` token, not the developer portal token.** The two have different scopes; the developer portal token is meant for registered apps making OAuth-flow requests on behalf of users, and does not work for direct `bs download` calls. Uploads will silently fail with 401 or 403 if you use the wrong one.

### 3. Base64-encode the SA key JSON

Seqera Pipeline Secrets are environment-variable-only. A JSON file with newlines and special characters can be truncated or misinterpreted when pasted directly, so this pipeline expects the SA key **base64-encoded** into a single line. The process script decodes it back to a file at runtime.

On Linux or macOS:

```bash
base64 -w 0 < path/to/sa-key.json
```

On Windows PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("path/to/sa-key.json"))
```

Copy the resulting single-line string. You'll paste it into a Secret in the next step.

### 4. Create the two Seqera Secrets

On the Seqera workspace, navigate to **Secrets** (left sidebar).

1. Click **Add workspace secret** (top-right of the Secrets page). Name: `BASESPACE_ACCESS_TOKEN`. Value: the BaseSpace access token from step 2. Save.
2. Click **Add workspace secret** again. Name: `BATCH_UPLOAD_SA_KEY_B64`. Value: the base64 string from step 3. Save.

Both Secrets should now appear on the Secrets page with their names and `Last updated` timestamps. Seqera masks the values by default so nothing sensitive is visible in the list.

![Seqera Secrets page listing both BASESPACE_ACCESS_TOKEN and BATCH_UPLOAD_SA_KEY_B64 with Last updated and Last activity timestamps and the Add workspace secret button in the top-right](/images/basespace-copy/seqera-03-secrets-list.png)

Both Secrets get injected as environment variables into the pipeline's task containers at launch time. Rotate `BATCH_UPLOAD_SA_KEY_B64` whenever you create a new endpoint (endpoint keys are short-lived and scoped to one ingest bucket).

## Launching a run

1. On the Launchpad, click **Launch** on the `basespace-copy-scrubber` row. The launch form opens.

2. On the **General config** step, verify:
   - **Compute environment** shows your workspace's compute environment.
   - **Work directory** shows a `gs://` path in your workspace's seqera-output bucket.

Click **Next**.

3. On the **Run parameters** step, use the Params file view. Paste:

```yaml
input_files:
  - <basespace-file-id-1>
  - <basespace-file-id-2>
  - <basespace-file-id-3>
outdir: gs://batch-upload-lab<your-lab-id>-<timestamp>-<uuid>
```

Replace each `<basespace-file-id>` with an actual BaseSpace file ID (get these by running `bs contents project -i <project-id>` on any authenticated machine, or from the BaseSpace web UI). Replace `outdir` with the full ingest bucket URI from your Portal endpoint (the value starts with `gs://batch-upload-lab`).

**Do NOT** prefix the outdir value with the label `outdir:` inside the string itself. The Params file field already parses the leading `outdir:` as the key; a value like `outdir: gs://...` produces an invalid URI that Nextflow rejects.

![Seqera launch form on the Run parameters step with Params file view selected, showing the YAML with input_files list and outdir key (specific values redacted), and a warning banner explaining that uploading a params file overrides existing pipeline parameters](/images/basespace-copy/seqera-04-launch-form-params.png)

Click **Next**.

4. On the **Advanced settings** step, verify the Pipeline Secrets section lists both `BASESPACE_ACCESS_TOKEN` and `BATCH_UPLOAD_SA_KEY_B64` as available. If either is missing, go back to the Secrets page and confirm the names match exactly (case matters).

Click **Next**, review the Summary, and click **Launch**. Seqera redirects to the run detail page.

## What success looks like

The run detail page progresses through: `Submitted` → `Running` → `Succeeded`. Two process rows show:

- `DOWNLOAD_FROM_BS`: one task per file in `input_files`. Uses the `theiagen/basespace_cli` container, reads `BASESPACE_ACCESS_TOKEN`, calls `bs download file`.
- `UPLOAD_TO_INGEST`: one task per file downloaded. Uses the `google/cloud-sdk` container, activates the batch-upload SA from `BATCH_UPLOAD_SA_KEY_B64`, calls `gsutil cp`.

Both process rows should turn green with a task count matching the number of files in your `input_files` list.

![Seqera run detail page for a completed basespace-copy-scrubber run showing Succeeded status with a green check, 100% workflow run progress bar, task counters of 4 total 4 succeeded 0 failed, and DOWNLOAD_FROM_BS and UPLOAD_TO_INGEST process rows both green with 2 of 2 counts, plus a task table below listing each task with Succeeded status and the container images used](/images/basespace-copy/seqera-05-run-succeeded.png)

Expected wall clock: dominated by BaseSpace's outbound throughput, roughly 1 minute per GB per file. Small files (100 MB or less) complete in under a minute each; large files (2 GB or more) can take several minutes each. Tasks run in parallel where the compute environment allows.

Once the run finishes, the transferred files are briefly present in the ingest bucket and then move through the compliance cascade (DLP scan → SRA scrubber → lab bucket). Wait 5-15 minutes, then check your lab's analytical-dataset bucket or the **Sequences tab on your lab's page** (endpoints are lab-scoped, so ingested files land on the lab's Sequences view, not on a specific project's page).

## Verification checklist

<details>
<summary>Click through these before considering the transfer complete</summary>

- [ ] Run detail page shows **Succeeded** state.
- [ ] Both process rows (`DOWNLOAD_FROM_BS`, `UPLOAD_TO_INGEST`) are green with task counts matching `input_files`.
- [ ] No red Failed rows or Aborted rows.
- [ ] Wait 5-15 minutes, then confirm the transferred files appear in your lab's analytical-dataset bucket, or the **Sequences tab on your lab's page** (navigate: Portal → your lab's page → Sequences).
- [ ] If a file is stuck in the ingest bucket after 30 minutes, the cascade may have failed on it; check the lab's Sequences tab for a `FAILED` status and contact your Platform Admin.
- [ ] If you plan to launch a pipeline against the transferred data next, run [notebook 02](../../../notebook-templates/#02-read-your-data) first to confirm the files are readable.

</details>

## Registering from scratch (admin flow)

Only needed if `basespace-copy-scrubber` is not already on your workspace's Launchpad. Most APGAP deployments ship it pre-registered; this section is for the Platform Admin doing initial setup.

1. On the Launchpad, click **Add pipeline** in the top-right (visible in the [Launchpad screenshot above](#opening-the-workspace)).

2. Fill in:
   - **Name**: `basespace-copy-scrubber`
   - **Description**: e.g. `Copy FASTQs from BaseSpace to APGAP ingest bucket via scoped SA + Seqera Secret. Triggers scrubber cascade (Pub/Sub → DLP → SRA scrubber → lab bucket).`
   - **Compute environment**: your workspace's compute environment (usually pre-selected)
   - **Pipeline to launch**: `https://github.com/azpathogens/APGAP-nextflow-basespace`
   - **Revision**: `feature/scrubber-routing` (matches the Portal endpoint dialog's Nextflow Workflow link on APGAP deployments)
   - **Pull latest**: ON

3. In **Advanced options**, set:
   - **Work directory**: `gs://<your-workspace-seqera-output-bucket>`
   - **Pipeline secrets**: enable `BASESPACE_ACCESS_TOKEN` and `BATCH_UPLOAD_SA_KEY_B64` (these are workspace-level Secrets you already created above).

4. Click **Add**. The pipeline now appears as a row on the Launchpad, ready for anyone in the workspace to launch.

## Where to go next

- Back to the [basespace-copy overview](../) for the compliance-cascade explanation, parameter reference, and troubleshooting.
- For the equivalent flow launched from a Jupyter notebook on your Workbench, see [Launch from Vertex AI Workbench](../tutorial-vertex/).
- For a primer on Seqera Platform generally, including the launchpad concept and per-project workspace model, see the [Seqera Platform primer](../../../seqera/).
- To read the FASTQs you just transferred: [notebook 02 (`02-read-your-data`)](../../../notebook-templates/#02-read-your-data).
- To launch a pipeline against them: [notebook 03 (`03-launch-a-pipeline`)](../../../notebook-templates/#03-launch-a-pipeline).
