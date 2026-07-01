# High CPU and constant camera drop-outs after upgrading 0.17.1 → 0.17.2 (go2rtc `#hardware` VAAPI sub-streams on Intel iGPU)

> Ready-to-paste bug report. Credentials/stream keys are redacted.
> Paste into https://github.com/blakeblackshear/frigate (issues or discussions).

## Describe the bug

After upgrading from **0.17.1 → 0.17.2** (HA OS add-on), CPU load went through the
roof and the system became effectively unusable: my two Wi-Fi cameras drop out
constantly and alternate between online/offline. **Nothing in my config
changed** across the upgrade. Restarting Frigate makes the streams come back to
normal **temporarily**, then it degrades again over time.

I have not been able to do a full deep-dive/debug, so this is mostly
observations + a working theory. I'm reverting to 0.17.1 to confirm it's a
regression and will update. **Looking for ideas on what to try / whether this is
known.**

## Version

- Frigate `0.17.2-3d4dd3a` (upgraded from `0.17.1`)
- Config version: `0.18-0`
- go2rtc `1.9.10`

## Environment / hardware

- Home Assistant OS 17.3, Core 2026.6.4; official Frigate add-on (full access)
- Intel iGPU, VAAPI decode/encode via `/dev/dri/renderD128`
- 2× OpenVINO detectors on the **GPU**

## Setup / relevant config (redacted)

Sub-stream pattern: go2rtc pulls each camera's main once and produces a
1280×720 H.264 sub via **`#hardware` (VAAPI)**; Frigate `detect` reads the sub
(software decode, `hwaccel_args: []`); live view uses the sub.

```yaml
go2rtc:
  streams:
    garten2:                       # EUFY, Wi-Fi, 192.168.1.19, H.264, NO native sub
      - rtsp://<redacted>@192.168.1.19:554/live0
      - ffmpeg:garten2#audio=opus
    garten2_sub:
      - ffmpeg:garten2#video=h264#width=1280#height=720#hardware#raw=-r 10 -g 10 -keyint_min 10
    door:                          # EUFY, Wi-Fi, 192.168.1.131, H.264, NO native sub
      - rtsp://<redacted>@192.168.1.131:554/live0
      - ffmpeg:door#audio=opus
    door_sub:
      - ffmpeg:door#video=h264#width=1280#height=720#hardware#raw=-r 10 -g 10 -keyint_min 10
    wall_stream:                   # UniFi, LAN, HEVC
      - rtspx://192.168.1.1:7441/<redacted>
      - ffmpeg:wall_stream#audio=opus
    wall_stream_sub:
      - ffmpeg:wall_stream#video=h264#width=1280#height=720#hardware#raw=-r 10 -g 10 -keyint_min 10

cameras:
  door:      # garten2/wall_stream analogous
    ffmpeg:
      hwaccel_args: []             # detect decodes in SW (HW on chained subs crashes)
      inputs:
        - path: rtsp://127.0.0.1:8554/door        # main -> record/audio
          input_args: preset-rtsp-restream
          roles: [record, audio]
        - path: rtsp://127.0.0.1:8554/door_sub    # sub  -> detect
          input_args: preset-rtsp-restream
          roles: [detect]
    detect: { width: 1280, height: 720, fps: 5 }  # garten2 uses fps: 10
```

## Symptoms

- CPU pegged after the upgrade (per-camera `detect`/`record` ~80–90%).
- `door` and `garten2` alternate online/offline; **Frigate restart fixes it
  temporarily**.
- **Frigate System → Cameras** shows the ingested "camera" FPS is ~20×/10× too
  high — but only for the two `#hardware`-transcoded EUFY subs:

  | camera | reported "camera" fps | configured `detect.fps` |
  |---|---|---|
  | door | **~104** | 5 |
  | garten2 | **~102** | 10 |
  | wall_stream (native sub) | **5.1** ✅ | 5 |

## Logs (representative, redacted)

**Frigate** — watchdog restart storm:
```
watchdog.garten2  INFO : garten2 exceeded fps limit. Exiting ffmpeg...
watchdog.garten2  INFO : FFmpeg did not exit. Force killing...        # ~30s later
frigate.video     ERROR: garten2: Unable to read frames from ffmpeg process.
ffmpeg.garten2.detect ERROR: [rtsp @ ...] RTP: PT=60: bad cseq 04b7 expected=2937
ffmpeg.door.detect    ERROR: [rtsp @ ...] DTS discontinuity in stream 0: packet 10 with DTS 90000, packet 11 with DTS 226918263
ffmpeg.door.detect    ERROR: method DESCRIBE failed: 404 Not Found     # door_sub gone
```

**go2rtc** (`log: { exec: trace }`) — the smoking gun on the `#hardware` subs:
```
[exec] [vf#0:0] Error while filtering: Cannot allocate memory
[exec] [vf#0:0] Task finished with error code: -12 (Cannot allocate memory)
producer.go:170 > error=EOF url="ffmpeg:garten2#video=h264#width=1280#height=720#hardware#raw=-r 10 -g 10 -keyint_min 10"
producer.go:170 > error="read tcp ...->192.168.1.19:554: i/o timeout" url=rtsp://<redacted>@192.168.1.19:554/live0
[exec] [h264_vaapi] A hardware frames reference is required to associate the encoding device.
[exec] run rtsp launch=2.8s      # <-- transcode start time climbs over minutes...
[exec] run rtsp launch=13.2s
[exec] run rtsp launch=17.0s     # <-- ...until the GPU is exhausted
```
The full go2rtc command it runs (ffmpeg pinned to 7.0 in these logs):
```
ffmpeg -vaapi_device /dev/dri/renderD128 -hwaccel vaapi -hwaccel_output_format vaapi
  -hwaccel_flags allow_profile_mismatch -fflags nobuffer -flags low_delay -timeout 5000000
  -rtsp_flags prefer_tcp -i "rtsp://127.0.0.1:8554/garten2?video&source=ffmpeg:garten2#...#hardware..."
  -r 10 -g 10 -keyint_min 10 -c:v h264_vaapi -g 50 -bf 0 -profile:v high -level:v 4.1 -sei:v 0 -an
  -vf "format=vaapi|nv12,hwupload,scale_vaapi=1280:720:out_color_matrix=bt709:out_range=tv:format=nv12"
  -rtsp_transport tcp -f rtsp rtsp://127.0.0.1:8554/<id>
```

## Working theory

The Intel iGPU appears to **run out of VAAPI surfaces**:

1. An EUFY Wi-Fi camera times out briefly (`read tcp ...:554: i/o timeout` — a
   known quirk of these cameras, present on 0.17.1 too).
2. go2rtc kills + respawns the `#hardware` transcode. Dying/overlapping
   transcodes don't release GPU surfaces fast enough → `Cannot allocate memory
   (-12)`; transcode `launch=` time climbs from ~2s to ~17s.
3. The sub stream dies → Frigate `detect` gets `404`/no frames → watchdog
   restarts detect → go2rtc re-kicks the transcode → **self-reinforcing loop**.
4. The constantly-restarting transcode emits **broken timestamps**, so Frigate's
   `-r/-vf fps` cap floods to ~100 fps → CPU pegged.
5. Only a **Frigate restart** frees the surfaces (matches "restart fixes it").

The native-sub camera (`wall_stream`, no VAAPI transcode) is unaffected → the
issue is specific to the `#hardware` VAAPI transcode path under source flapping.

## What I already tried (and result)

- Add `go2rtc.ffmpeg.global: -vaapi_device /dev/dri/renderD128` (FFmpeg 8
  `hwupload` requirement) — **did not fix it**.
- Pin `ffmpeg.path: "7.0"` — **still OOMs** → not an FFmpeg-8-only issue.
- Reduce GPU AI load (one detector to CPU, `face_recognition: small`) — **no
  effect** on the core problem.
- Switch `wall_stream` to its **native H.264 sub** (no VAAPI transcode) — that
  camera is now **stable**. (Note: the native UniFi "high" stream was
  *multi-layer HEVC*, which ffmpeg can't decode — had to pick an H.264 sub.)

## Why I think it's a 0.17.x regression

The trigger (EUFY flapping) existed on 0.17.1, but back then a flap
**self-recovered**; on 0.17.2 the same flap **cascades** into the GPU-exhaustion
spiral. I suspect the threshold was lowered by either the rewritten
restart/watchdog path in the new `frigate/video/ffmpeg.py` (the
`exceeded fps limit` restart + 30s force-kill re-kick the transcode more
aggressively) and/or a bundled media-stack change (ffmpeg default 8.0, go2rtc,
Intel VAAPI driver). I can't pin the exact commit myself.

## Questions / what to try

- Is this a known 0.17.x regression with `#hardware` go2rtc sub-streams on Intel
  VAAPI when the source RTSP is flaky?
- Is the `exceeded fps limit` watchdog restart (or the 30s force-kill) too
  aggressive when the sub stream has broken timestamps, creating a feedback loop?
- Any way to make Frigate's detect fps cap robust to broken source timestamps
  (e.g. `-use_wallclock_as_timestamps 1`)?
- Recommended pattern for cameras **without a native sub** on Intel iGPU to avoid
  VAAPI surface exhaustion under source flapping? (Software encode? Let detect
  read the main and downscale in Frigate instead of a separate transcode?)

I'll follow up with a 0.17.1 vs 0.17.2 A/B (identical config) to confirm the
regression. Happy to provide full logs / config on request.
