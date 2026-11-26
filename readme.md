# Blue Iris → S3 Video Backup

Research notes and a plan for automatically backing up Blue Iris recordings to cloud storage (AWS S3), keeping them off-site for emergency use.

> Prices are approximate AWS list prices (us-east-1) as of late 2026. Check current pricing before you commit.

---

## 1. Where to run the upload

**Recommendation: run [rclone](https://rclone.org) on the Windows / Blue Iris machine as a Task Scheduler job.**

| | Script on Windows (Blue Iris box) | Mac mini over an SMB share |
|---|---|---|
| Moving parts | One machine | Two machines + the share + the network |
| Reliability | Runs whenever Blue Iris runs | Breaks if the Mac sleeps, the share drops or Windows reboots |
| Files still being recorded | Easy to skip (`--min-age`) | Harder; SMB locks can make uploads fail |
| Network | Disk → internet | Disk → LAN → Mac → internet (every file crosses the LAN twice) |

Why rclone:
- Free, a single `.exe`, supports S3 natively.
- Retries, bandwidth limits and skipping in-progress files.
- Blue Iris can't upload to S3 itself (its archiving supports FTP/SFTP only).
- `aws s3 sync` can't skip files that are still growing.

**Deleting old footage:** don't make the script delete anything. An **S3 Lifecycle rule** expires objects automatically and for free. Blue Iris already prunes local clips by age or size.

---

## 2. Cost at 76 GB/day

Measured recording volume: **~76 GB/day**.

### Upload bandwidth
76 GB/day needs a steady **~7 Mbps of upload**, 24/7. Aim for at least 10–15 Mbps of upload so rclone can catch up after outages.

### If you need fast access to the footage

| Keep for | Stored | S3 Standard ($0.023/GB) | S3 Standard-IA ($0.0125/GB, billed for at least 30 days) | Backblaze B2 (~$6–7/TB) |
|---|---|---|---|---|
| 30 days | 2,280 GB | $52 | **$28.50** | ~$14–16 |
| 14 days | 1,064 GB | **$24.50** | $28.50 (still billed for 30 days) | ~$7 |
| 7 days | 532 GB | **$12.25** | $28.50 (still billed for 30 days) | ~$3.50 |

Rule of thumb: Standard-IA is cheaper if you keep files **17 days or more**. Below that, use Standard.

### If you only need the footage in an emergency (chosen approach)

Each file is billed for the longer of its retention and the class's minimum duration.

| Class | Minimum billed | Cost if kept 30 days | Keep as long as the minimum | How fast you get footage back |
|---|---|---|---|---|
| S3 Standard-IA | 30 days | $28.50 | same (30 days) | Instant |
| Glacier Instant Retrieval | 90 days | $27.40 | 90 days for $27.40 | Instant |
| Glacier Flexible Retrieval | 90 days | $24.60 | 90 days for $24.60 | 1–5 min (expedited) / 3–5 hr (standard) |
| **Glacier Deep Archive** | 180 days | $13.50 | **180 days for $13.50** | ~12 hr (standard) / up to 48 hr (bulk) |
| Backblaze B2 (not AWS) | none | ~$15 | — | Instant |

**Pick: Glacier Deep Archive with a 180-day lifecycle expiry, at ~$13.50/month.** That gives 6 months of history for the same price as 30 days.

Emergency retrieval of one day of footage (76 GB):
- **Restore:** ~$1.50 at standard speed, ~$0.20 at bulk speed.
- **Download:** AWS's first 100 GB/month is free, so a typical emergency costs **about $2 or less**.

A 12-hour wait is fine for the main use case: the PC is stolen or destroyed and you need evidence for police or insurance. Local footage still covers day-to-day viewing.

Other costs:
- **Uploading:** free.
- **Upload requests:** pennies a month.

### Big saver
76 GB/day suggests 24/7 recording. Uploading only alert clips and keeping continuous footage local would likely cut cloud usage by 80–95%.

---

## 3. Planned setup

### AWS
1. Create a bucket and turn on **versioning**, so footage can't be wiped by uploading over it.
2. Add **Lifecycle rules**:
   - Expire current objects after 180 days.
   - Expire noncurrent (old) versions too.
3. Create an IAM user for the uploader with **`s3:PutObject` + `s3:ListBucket` only**:
   - No `DeleteObject`, so a stolen PC or ransomware can't delete the cloud copy.
   - rclone needs `ListBucket` to avoid uploading files twice.
   - Optional: S3 Object Lock for guaranteed immutability.

### Windows
1. Install rclone and run `rclone config` to add an S3 remote (e.g. `s3cam`).
2. Create a Task Scheduler job that runs every 5–10 minutes:

```bat
rclone copy "D:\BlueIris\New" s3cam:my-bi-bucket/recordings ^
  --min-age 2m ^
  --s3-storage-class DEEP_ARCHIVE ^
  --s3-no-check-bucket ^
  --bwlimit 4M --transfers 2 ^
  --log-file C:\rclone\upload.log --log-level INFO
```

- `--min-age 2m` skips files Blue Iris is still writing.
- Use `copy`, not `sync`, so local deletions never remove cloud copies.
- Adjust `--bwlimit` to your upload speed. It must still be above ~7 Mbps (≈ 0.9 MB/s) to keep up.

### Resource usage
- **rclone:** a few percent CPU and ~50–200 MB RAM. It's limited by the network, not the processor.
- **MP4 recording with direct-to-disc:** essentially no extra CPU. It only changes the container; nothing is re-encoded.

---

## 4. Alternative: Hetzner Storage Box

A [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box/) is storage you rent without a server. It's a flat monthly price with **unlimited traffic** and no retrieval fees, and it works with rclone over SFTP. There's no need to attach a disk to a server.

### Hetzner options compared (76 GB/day)

| Option | Price (excl. VAT) | Verdict |
|---|---|---|
| Cloud Volume attached to an existing server (e.g. the website box) | Roughly €0.04–0.05/GB-month historically (current rate unconfirmed), ≈ **€100/month for 30 days of footage** | ❌ Most expensive; mixes footage with a public web server |
| **Storage Box** | 5 TB **€10.90** · 10 TB **€20.80** · 20 TB **€40.60** | ✅ Best for ≤ ~60 days with instant access |
| Object Storage (S3-compatible) | €6.49 incl. 1 TB, then ≈ €5/TB | ≈ €13/month for 30 days |

### Storage Box vs AWS Deep Archive

| Keep for | Data | Storage Box | AWS Deep Archive |
|---|---|---|---|
| 30 days | 2.3 TB | €10.90 (5 TB) | $13.50 (billed for at least 180 days) |
| **60 days** | 4.6 TB | **€10.90 (5 TB)** ✅ | $13.50 |
| ~120 days | 9–10 TB | €20.80 (10 TB) | $13.50 |
| 180 days | 13.7 TB | €40.60 (20 TB) | **$13.50** ✅ |

- **Storage Box wins at ~60 days or less.** You can view or download footage instantly, with no restore wait and no fees.
- **Deep Archive wins for long, emergency-only history.**

### Downsides
1. **No lifecycle rules.** Old files have to be deleted by a scheduled cleanup command.
2. **Locations: Germany and Finland only.** A steady ~7 Mbps upload is fine from anywhere; use `--transfers 4` to keep up over a long-distance link.
3. **Weaker tamper protection than S3** with versioning and Object Lock. Mitigate with:
   - Automatic snapshots (they use space inside the box).
   - Separate logins (sub-accounts) for uploading and for cleanup, so a compromised home PC can't wipe the backups.

### Setup sketch
1. Order a **BX21 (5 TB)** and turn on SSH/SFTP access.
2. Create a sub-account (e.g. `uXXXXX-sub1`) for the Blue Iris uploader.
3. On Windows, `rclone config` → new **SFTP** remote (e.g. `hbox`):
   - host `uXXXXX-sub1.your-storagebox.de`
   - port `23`
   - key-based auth
4. Task Scheduler job every 5–10 minutes:

```bat
rclone copy "D:\BlueIris\New" hbox:recordings ^
  --min-age 2m --transfers 4 --bwlimit 4M ^
  --log-file C:\rclone\upload.log --log-level INFO
```

5. Nightly cleanup, run from a different machine (e.g. the website server, via cron) using the main account:

```sh
# delete footage older than 60 days, then remove empty folders
rclone delete hbox-main:recordings --min-age 60d
rclone rmdirs hbox-main:recordings --leave-root
```

6. Optional: turn on automatic snapshots as a safety net.

> Prices from Hetzner's site as of late 2026. Check current pricing and your VAT before ordering.

---

## 5. Recording format: BVR vs MP4

- **BVR** is Blue Iris's proprietary format. There's no reliable Mac player; VLC and ffmpeg are hit-or-miss.
- **Switch to MP4** (Camera settings → Record → file format) and keep **direct-to-disc** on.
  - Files then play without Blue Iris, which matters if the PC is stolen and only the cloud copy is left.
- Viewing old BVRs:
  - Export to MP4 inside Blue Iris, or
  - Run Windows + Blue Iris in a VM (Parallels / VMware Fusion / UTM).

### Playing MP4s on a Mac
QuickTime refuses many camera files:
- **H.265 tagged `hev1`:** QuickTime only plays HEVC tagged `hvc1`.
- **G.711 audio (PCMU/PCMA):** not supported by QuickTime in MP4.

Solutions:
- **Use [IINA](https://iina.io) or VLC.** Both play these files as-is. ✅ IINA confirmed working.
- **Fix the tag for QuickTime when needed** (no re-encode, takes seconds):
  ```sh
  ffmpeg -i in.mp4 -c copy -tag:v hvc1 out.mp4
  ```
- **Keep H.265.** It's ~30–50% smaller than H.264. Switching to H.264 could raise 76 GB/day to ~100–150 GB/day, with matching increases in storage, bandwidth and cost.

### Timestamp overlay missing in MP4
- **Why:** with direct-to-disc, Blue Iris saves the camera stream as-is. Its overlays only appear in Blue Iris's own viewer and are never added to the file.
- **Fix:** turn on the **camera's own OSD date/time** in its web interface (Hikvision: Image → OSD; Dahua: Camera → Overlay; Reolink: Display).
  - Turn on **NTP** and set the correct time zone so timestamps are accurate for evidence.
  - Hide the Blue Iris overlay for that camera to avoid a double timestamp.
- Don't turn off direct-to-disc just to get overlays. Re-encoding every camera uses a lot of CPU.
- Blue Iris filenames also include the date and time, as a backup.

---

## 6. Optional: hidden recorder / backup box with 10 TB

A 10 TB drive holds about **4 months** of local footage at 76 GB/day. Most mini PCs only fit 2.5"/M.2 drives, so:

1. **Refurbished business SFF PC (recommended).**
   - Models: Dell OptiPlex SFF, HP ProDesk/EliteDesk 600/800 SFF, Lenovo ThinkCentre M-series SFF. Choose **SFF**, not Micro/Tiny, which have no 3.5" bay.
   - Look for an Intel 8th–12th gen i5 (Quick Sync), 16 GB RAM, an SSD for the OS and the 10 TB drive in the internal 3.5" bay.
   - Can run Blue Iris directly. Roughly $150–250 refurbished, plus the drive.
2. **N100/N150 mini PC + powered USB 3.5" enclosure.**
   - Tiny and ~10–15 W.
   - USB drives sometimes disconnect, so it's better as a backup target than as the 24/7 recorder.
3. **Small NAS** (Synology DS124 / DS224+, UGREEN NASync).
   - Use it as a backup target, or replace Blue Iris with Synology Surveillance Station (2 free camera licenses, then paid).

**Drive:** a surveillance drive (WD Purple / Seagate SkyHawk) built for constant writing. WD Red Plus or IronWolf also work.

**Tips for the hidden location:**
- **Network:** wired Ethernet, not Wi-Fi.
- **Power:** a small UPS for the PC, PoE switch and router.
- **Heat:** leave airflow around the drive.
- **Auto-restart:** set the BIOS to "power on after AC loss" and start Blue Iris automatically.

---

## 7. Next steps
- [ ] Decide on the cloud target: AWS Deep Archive (180 days, emergency-only) or Hetzner Storage Box (≤ 60 days, instant access).
- [ ] Decide: is the hidden PC the recorder, or a backup target?
- [ ] Turn on camera OSD timestamps + NTP on all cameras.
- [ ] Switch all cameras to MP4 + direct-to-disc.
- [ ] Create the S3 bucket, versioning, lifecycle rules and the upload-only IAM user.
- [ ] Install rclone + Task Scheduler job on the recorder.
- [ ] Test an emergency restore from Deep Archive.
