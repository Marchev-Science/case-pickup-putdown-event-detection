# Data access

The footage lives in a cloud object-storage bucket (S3-compatible), roughly
**80 GB** in total. Your access is **read-only**: you can list and download, but
you cannot write back. Anything you produce (labels, clips, features, models,
results) goes into *your own* storage — see
[`deliverables-and-repos.md`](deliverables-and-repos.md).

---

## 0. Fill these in first

The bucket location and any credentials are provided to you separately. Put them
in one place your team agrees on and **never commit them to a public repo**.

```
BUCKET_URI   = s3://REPLACE-WITH-BUCKET-NAME            # e.g. s3://store-footage-2026
REGION       = REPLACE-WITH-REGION                      # e.g. eu-central-1
ENDPOINT_URL = (only if non-AWS S3-compatible storage)  # e.g. https://...
```

Access comes in one of two forms; you will be told which:

- **Public / anonymous read** — no keys needed. Add `--no-sign-request` (CLI) or
  the equivalent "anonymous" flag (Python) shown below.
- **Credentialed read-only** — you receive an *access key id* and *secret access
  key*. Store them via the standard mechanism (below), not in code.

---

## 1. The golden rule: don't pull 80 GB

You almost certainly do **not** need the whole dataset on your laptop. Work like
this:

1. **List first**, understand the layout, sizes, and naming.
2. **Sample** a small, representative subset (a few GB) to develop on.
3. **Sync only what you need**, and only once — mirror a prefix locally so you
   don't re-download.
4. Where possible, **stream** frames on demand instead of downloading whole
   files.

A motion-triggered camera produces many empty clips; an early win is to filter
those out (see Layer 0) so you never download most of them.

---

## 2. AWS CLI — works on Windows, macOS, Linux

Install: <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>
(or `pipx install awscli` / `brew install awscli` / `winget install Amazon.AWSCLI`).

**Credentialed:** run `aws configure` once and paste your key id, secret, and
region. **Anonymous:** skip configure and add `--no-sign-request` to every
command.

```bash
# List the top level
aws s3 ls s3://REPLACE-WITH-BUCKET-NAME/                 # add --no-sign-request if public

# List a sub-folder, human-readable sizes, recursively
aws s3 ls s3://REPLACE-WITH-BUCKET-NAME/raw/ --recursive --human-readable --summarize

# Copy ONE file to look at it
aws s3 cp s3://REPLACE-WITH-BUCKET-NAME/raw/clip_00012.mp4 ./samples/

# Mirror just one prefix locally (re-runs only fetch new/changed files)
aws s3 sync s3://REPLACE-WITH-BUCKET-NAME/raw/2026-w1/ ./data/raw/2026-w1/

# Download a sample using a pattern (exclude everything, then include some)
aws s3 cp s3://REPLACE-WITH-BUCKET-NAME/raw/ ./data/sample/ --recursive \
    --exclude "*" --include "clip_000*.mp4"
```

For S3-compatible (non-AWS) storage, add `--endpoint-url $ENDPOINT_URL`.

---

## 3. Python — boto3, s3fs, or fsspec

Best for building your curation pipeline and for streaming.

```bash
pip install boto3 s3fs fsspec
```

**boto3 (explicit download):**
```python
import boto3
from botocore import UNSIGNED
from botocore.client import Config

# Credentialed: omit the two config/UNSIGNED lines and rely on env/credentials file.
s3 = boto3.client(
    "s3",
    region_name="REPLACE-WITH-REGION",
    config=Config(signature_version=UNSIGNED),   # <-- anonymous/public access only
)

bucket = "REPLACE-WITH-BUCKET-NAME"
# List keys
for page in s3.get_paginator("list_objects_v2").paginate(Bucket=bucket, Prefix="raw/"):
    for obj in page.get("Contents", []):
        print(obj["Key"], obj["Size"])

# Download one object
s3.download_file(bucket, "raw/clip_00012.mp4", "samples/clip_00012.mp4")
```

**s3fs / fsspec (treat S3 like a filesystem, good for streaming):**
```python
import s3fs
fs = s3fs.S3FileSystem(anon=True)             # anon=False + key/secret if credentialed
print(fs.ls("REPLACE-WITH-BUCKET-NAME/raw/")[:10])

# Open a video lazily and hand the file object to a reader (e.g. PyAV/imageio)
with fs.open("REPLACE-WITH-BUCKET-NAME/raw/clip_00012.mp4", "rb") as f:
    data = f.read()                            # or stream in chunks
```

Tip: pair `fsspec`/`s3fs` with **`smart_open`** if you want a one-liner that
opens local or S3 paths transparently.

---

## 4. rclone — the cross-platform power tool (sync + mount)

Great for fast, resumable syncing and for **mounting the bucket as a drive** so
non-coders can browse it in their file explorer.
Install: <https://rclone.org/downloads/>

```bash
rclone config            # interactive: create a remote named e.g. "store" of type "s3"
rclone ls store:REPLACE-WITH-BUCKET-NAME/raw/
rclone copy store:REPLACE-WITH-BUCKET-NAME/raw/2026-w1/ ./data/raw/2026-w1/ -P
rclone mount store:REPLACE-WITH-BUCKET-NAME/ ~/store-mount --read-only   # browse like a folder
```

---

## 5. GUI apps — for team members who prefer clicking

- **Cyberduck** (Windows/macOS) — <https://cyberduck.io/> — connect with an "Amazon
  S3" bookmark; supports anonymous connections.
- **S3 Browser** (Windows) — <https://s3browser.com/>.
- **WinSCP** (Windows) — supports S3.
- **Mountain Duck** (Windows/macOS) — mounts S3 as a disk.

These let the curation/labeling members preview and pull clips without the
command line.

---

## 6. Mounting as a filesystem (advanced)

- **Mountpoint for Amazon S3** — <https://github.com/awslabs/mountpoint-s3>
  (read-heavy workloads, official AWS).
- **s3fs-fuse** (Linux/macOS) — <https://github.com/s3fs-fuse/s3fs-fuse>.
- **rclone mount** (see above) — the most portable option.

Mounting is convenient but every read is a network call; for training/feature
extraction it is faster to `sync` your working subset to local disk first.

---

## 7. Google Colab

If you do heavy steps (feature extraction, model training) on Colab for the GPU:

```python
!pip -q install boto3 s3fs
# then use the boto3 / s3fs snippets above; sync a small subset to the Colab disk,
# or stream directly. Save outputs to your own Drive / your own bucket, not here.
```

Colab disks are ephemeral — persist anything you care about to your own storage.

---

## 8. Sanity checks before you trust the data

- Confirm total object count and size match expectations (`--summarize`).
- Open a handful of clips and check fps, resolution, duration, and codec
  (`ffprobe clip.mp4` is your friend).
- Note the **naming convention** of files — you will key your reference table on
  it (see [`manifest-schema.md`](manifest-schema.md)).
- Verify a download is intact (size/byte count) before processing in bulk.

---

## 9. Reminders

- The bucket is **read-only**. Do not attempt to write to it.
- Keep credentials out of code and out of git (use `aws configure`, environment
  variables, or a git-ignored secrets file).
- Mirror, don't hoard: keep your local working copy to the subset you actually
  use.
