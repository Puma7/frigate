# Frigate CPU-Spike nach Update 0.17.1 → 0.17.2 — Diagnose & Lösung

> Stand dieser Doku: laufende Sitzung, Build `0.17.2-3d4dd3a`.
> **Alle Zugangsdaten (RTSP-User/Passwörter, MQTT-Passwort, UniFi-Stream-Keys)
> sind in diesem Dokument bewusst durch `<redacted>` ersetzt.**

---

## TL;DR

Nach dem Update von 0.17.1 auf 0.17.2 schießt die CPU-Last hoch und die beiden
EUFY-Kameras (`door`, `garten2`) setzen wechselweise aus.

**Ursache:** Eine **VAAPI-Surface-Erschöpfung auf der Intel-iGPU**, ausgelöst
durch den **go2rtc-`#hardware`-Transcode-Restart-Sturm**. Getriggert wird der
Sturm von den **zeitweise wegtimeoutenden EUFY-WLAN-Kameras**. Die sterbenden
Transcodes geben ihre GPU-Surfaces nicht schnell genug frei → `Cannot allocate
memory` → immer mehr Neustarts → die iGPU läuft voll. Ein Frigate-Neustart
räumt alles frei (deshalb „Neustart hilft").

Als Folge liefert der kaputte Transcode einen Strom mit zerschossenen
Zeitstempeln, auf dem Frigates FPS-Begrenzung „durchdreht" und **~100 fps statt
5/10 fps** einliest — das ist der eigentliche CPU-Fresser.

**Lösung:** `#hardware` aus den beiden EUFY-Sub-Streams entfernen
(Software-Encode). Das nimmt der ganzen Spirale den Angriffspunkt: kein VAAPI →
keine Surface-Erschöpfung → kein 100-fps-Flood.

---

## Umgebung

| Komponente | Wert |
|---|---|
| Host | Home Assistant OS 17.3, Core 2026.6.4, `192.168.1.182` |
| Add-on | `ccab4aaf_frigate-fa`, Build `0.17.2-3d4dd3a` |
| GPU | Intel iGPU, VAAPI/QSV via `/dev/dri/renderD128` |
| Detektoren | 2× OpenVINO auf GPU (`ov_0`, `ov_1`) |
| go2rtc | `version=1.9.10` (Image bündelt bundle-abhängig neuere) |
| ffmpeg | Image-Default **8.0**; während der Diagnose auf **7.0** gepinnt |

### Kameras

| Name | Typ / Netz | IP | Codec | Nativer Sub? |
|---|---|---|---|---|
| `door` | EUFY, WLAN | `192.168.1.131` | H.264 | **nein** |
| `garten2` | EUFY, WLAN | `192.168.1.19` | H.264 | **nein** |
| `wall_stream` | UniFi, LAN | `192.168.1.1:7441` | HEVC | ja (H.264-Sub) |

### Architektur (Sub-Stream-Pattern)

- go2rtc zieht den **Main einmal** pro Kamera.
- Daraus erzeugt go2rtc per **ffmpeg + VAAPI (`#hardware`)** einen
  1280×720-H.264-Sub (`<cam>_sub`), 10 fps.
- Frigate-Rollen: `main` → record/audio (Stream-Copy), `sub` → detect
  (SW-Decode, `hwaccel_args: []`).
- Live-View nutzt den Sub.

Für die EUFYs ist der Transcode **nötig** (kein nativer Sub). Für `wall_stream`
gibt es einen nativen UniFi-Sub.

---

## Symptome

- CPU **nach dem Update** dauerhaft hoch; war auf 0.17.1 unauffällig.
- `door` und `garten2` **setzen wechselweise aus** (flip-flop); Neustart von
  Frigate stellt den Stream **temporär** wieder her.
- Frigate-Log (wiederkehrend):
  - `watchdog.<cam> INFO: <cam> exceeded fps limit. Exiting ffmpeg...`
  - 30 s später `FFmpeg did not exit. Force killing...`
  - `ffmpeg.<cam>.detect ERROR: [rtsp @ ...] RTP: PT=60: bad cseq ...`
  - `DTS discontinuity in stream 0: packet N with DTS ...`
  - `method DESCRIBE failed: 404 Not Found` auf `rtsp://127.0.0.1:8554/<cam>_sub`
- go2rtc-Log (wiederkehrend):
  - `producer.go:170 > error=EOF url="ffmpeg:<cam>#...#hardware..."`
  - `[vf#0:0] Error while filtering: Cannot allocate memory` → `error code: -12`
  - `error="read tcp ...->192.168.1.19:554: i/o timeout"` (bzw. `.131`)
  - `[h264_vaapi] A hardware frames reference is required to associate the encoding device`
  - Transcode-Startzeit `launch=` klettert von ~2 s auf **17 s**
- Frigate System → Kameras (der Schlüssel-Befund):

  | Kamera | „Kamera"-FPS (Ist) | Soll (`detect.fps`) |
  |---|---|---|
  | `door` | **103,9** | 5 |
  | `garten2` | **101,5** | 10 |
  | `wall_stream` | **5,1** ✅ | 5 |

  Nur die beiden **transkodierten** EUFY-Subs zeigen ~100 fps; der native
  `wall_stream`-Sub ist korrekt.

---

## Diagnose-Verlauf (geprüft & verworfen)

| # | Hypothese | Ergebnis |
|---|---|---|
| 1 | fps-Overflow-Watchdog (`camera_fps >= detect.fps+10`) ist selbst der Bug | **Verworfen** — reagiert korrekt auf real ~100 fps |
| 2 | `EventsPerSecond`-Änderung (Commit `d982b3a`, monotonic clock) | **Verworfen** — Diff ist output-äquivalent |
| 3 | FFmpeg 8 `hwupload` braucht `-vaapi_device` | **Teil-Effekt** — Device gesetzt, Problem blieb |
| 4 | ffmpeg-Version (8.0) | **Verworfen** — auf 7.0 gepinnt, OOM blieb |
| 5 | `wall_stream` nativer Sub kaputt | **Behoben** — gewählter UniFi-Stream war *multi-layer HEVC* (undecodbar), Umstieg auf H.264-Sub → `wall_stream` stabil |
| 6 | GPU-KI-Last (OpenVINO/face) zu hoch | **Kein Effekt** auf das Kernproblem |
| 7 | „Leak" via `shm`/`ffmpeg`/`drm`-Counter messen | **Untauglich** — VAAPI-Surfaces sind GPU-Speicher, keine FDs; tauchen in den Countern nicht auf |

---

## Root Cause (belegt)

**VAAPI-Surface-Erschöpfung durch den go2rtc-`#hardware`-Transcode-Sturm.**

Kausalkette:

1. EUFY-WLAN-Kamera timeoutet kurz → go2rtc: `read tcp ...:554: i/o timeout`.
2. go2rtc killt+startet den `#hardware`-Transcode neu. Die sterbenden Prozesse
   geben ihre VAAPI-Surfaces nicht schnell genug frei; es überlappen sich immer
   mehr → **Surfaces erschöpft** → `Cannot allocate memory (-12)`.
3. Dadurch bricht der `<cam>_sub` weg → Frigate-Detect bekommt `404` / keine
   Frames → Frigate-Watchdog startet Detect neu → go2rtc kickt den Transcode
   erneut → **selbstverstärkende Spirale**.
4. Der ständig neustartende Transcode liefert **kaputte Zeitstempel** → Frigates
   `-r/-vf fps`-Begrenzung dupliziert massiv → **~100 fps** in Detect → CPU
   pegged (`erkennen 88 %`, `aufnehmen 82 %`).
5. Nur ein **Frigate-Neustart** killt alle Prozesse und gibt die GPU frei.

Belege (alle direkt in den Logs):
- `vf#0:0 Error while filtering: Cannot allocate memory` (VAAPI-Surfaces alle)
- `launch=` steigt 2 s → 17 s (GPU läuft progressiv voll)
- „Kamera 100 fps" nur bei den transkodierten Subs, `wall_stream` (nativ) = 5 fps
- „Neustart hilft" = Prozesse werden gekillt, Surfaces frei

### Warum kam das erst mit 0.17.2?

- Der **Trigger** (flatternde EUFYs, `i/o timeout`) existierte auf 0.17.1
  **schon** — laut Nutzer-Config eine bekannte EUFY-WLAN-Eigenheit.
- Das Update hat **die Schwelle gesenkt**, ab der ein Aussetzer katastrophal
  wird. Wahrscheinlichste Verstärker (im Zusammenspiel):
  1. **Neu geschriebene Restart-/Watchdog-Maschinerie** (`frigate/video/ffmpeg.py`
     ist im 0.17-Refactor neu): `exceeded fps limit`-Restart + 30-s-Force-Kill
     kicken bei jedem fps-Blip die Detect-ffmpeg neu → mehr Transcode-Neustarts
     → mehr überlappende VAAPI-Prozesse.
  2. **Gebündelter Media-Stack** (ffmpeg-Default 8.0, go2rtc-Version, **Intel-
     VAAPI-Treiber** im Image): kann Surfaces knapper verwalten / langsamer
     freigeben.
- **Ehrliche Grenze:** Der exakte Commit ist aus diesem Repo **nicht**
  benennbar — der Klon ist „shallow" (keine 0.17.1 zum Byte-Diff, keine Tags).
  Beweisbar wäre die Regression nur per **A/B**: 0.17.1 mit identischer Config.

---

## Aktueller Stand (was bereits gemacht wurde)

- ✅ `wall_stream` auf nativen **H.264**-UniFi-Sub umgestellt → **stabil**.
- ✅ `go2rtc.ffmpeg.global: -vaapi_device /dev/dri/renderD128` gesetzt.
- ✅ `go2rtc.log.exec: trace` aktiviert (für die Diagnose).
- ✅ ffmpeg via `ffmpeg.path: "7.0"` gepinnt (kann später zurück auf Default).
- ✅ GPU-KI-Last testweise reduziert (Detektor auf CPU, `face_recognition: small`).
- ⚠️ **`door` und `garten2` weiterhin instabil** — der eigentliche Fix
  (`#hardware` raus) ist **noch nicht angewandt**.

---

## Empfohlener Fix (noch anzuwenden)

`#hardware` aus den beiden EUFY-Sub-Streams entfernen → go2rtc encodiert sie in
**Software** (libx264). Das behebt **OOM und 100-fps-Flood in einem Schnitt**.

```yaml
go2rtc:
  streams:
    garten2:
      - rtsp://<redacted>@192.168.1.19:554/live0
      - ffmpeg:garten2#audio=opus
    garten2_sub:
      # #hardware entfernt -> Software-Encode (sauberes CFR, keine VAAPI-Surfaces)
      - ffmpeg:garten2#video=h264#width=1280#height=720#raw=-r 10 -g 10 -keyint_min 10

    door:
      - rtsp://<redacted>@192.168.1.131:554/live0
      - ffmpeg:door#audio=opus
    door_sub:
      - ffmpeg:door#video=h264#width=1280#height=720#raw=-r 10 -g 10 -keyint_min 10
```

Warum das der richtige Hebel ist:
- **Kein VAAPI → keine Surfaces → kein OOM** (Spirale strukturell weg).
- **libx264 = sauberes CFR** → kein 100-fps-Flood → kein `exceeded fps limit`.
- **EUFY-Aussetzer werden überlebbar**: reconnect statt GPU-Spirale.
- **Netto vermutlich weniger CPU** als der jetzige Sturm — und vor allem stabil.
- Bewusster Tausch: etwas konstante CPU statt eines fragilen GPU-Pfads.

---

## Verifikation (nach dem Fix)

Frigate neu starten, dann prüfen:

1. **go2rtc-Trace** (2–3 Min): **keine** `Cannot allocate memory`, **kein**
   `launch=10s+` mehr.
2. **System → Kameras**: „Kamera"-FPS bei `door`/`garten2` wieder **~5/10**
   (statt ~100).
3. **Frigate-Log**: keine `exceeded fps limit`-Neustarts mehr.

> Hinweis: Der frühere `leakwatch`-Logger (`shm`/`ffmpeg`/`drm`-Counter) ist für
> dieses Leck **nicht** aussagekräftig — VAAPI-Surfaces sind GPU-Speicher und
> tauchen in diesen Zählern nicht auf. Verifikation über go2rtc-Trace +
> System-Seite.

---

## Sekundär / offene Punkte

- **EUFY-WLAN stabilisieren** (reduziert die Trigger-Frequenz): RTSP-Bitrate in
  der EUFY-App senken (~2 Mbit/s), Empfang prüfen, ggf. 5 GHz / anderer Kanal.
- **ffmpeg-Pin zurücknehmen**: `ffmpeg.path: "7.0"` kann nach Stabilisierung
  entfernt werden (Default 8.0). `-vaapi_device` darf bleiben.
- **`garten2` `detect.fps: 10` → `5`** (optional): halbiert die OpenVINO-Last
  für garten2; verstößt aktuell gegen die eigene „DON'T"-Regel in der Config.
- **`go2rtc.log.exec` wieder auf Default** setzen, sobald verifiziert (weniger
  Log-Rauschen).
- **Upstream-Meldung** (optional): Mit 0.17.1-A/B-Beweis + den
  `Cannot allocate memory` / `launch=17s`-Logs als reproduzierbare Regression.
- **Optionaler Code-Watchdog-Patch** (nicht empfohlen als Primärfix): in
  `frigate/video/ffmpeg.py:377` die Overflow-Schwelle von `== 3` auf z. B.
  `== 10` anheben, damit transiente Blips keinen 30-s-Force-Kill-Zyklus
  auslösen. Wird beim nächsten Upgrade überschrieben — Config-Fix ist die echte
  Lösung.

---

## Referenzen (Code-Fundstellen)

| Datei:Zeile | Bedeutung |
|---|---|
| `frigate/video/ffmpeg.py:374` | fps-Overflow-Watchdog (`camera_fps >= detect.fps + 10`) |
| `frigate/video/ffmpeg.py:212` | `reset_capture_thread` (terminate → wait 30 s → kill) |
| `frigate/video/ffmpeg.py:40-107` | Capture-Loop; `camera_fps = EventsPerSecond.eps()` |
| `frigate/util/builtin.py:31-64` | `EventsPerSecond` (Commit `d982b3a`, output-äquivalent) |
| `frigate/ffmpeg_presets.py:123,130` | Detect-Scale-Presets (`-r {fps} -vf fps={fps},scale=...`) |
| `docker/main/rootfs/usr/local/go2rtc/create_config.py:92-99` | go2rtc bekommt Frigates ffmpeg-Binary |
| `docker/main/Dockerfile:268` | `DEFAULT_FFMPEG_VERSION="8.0"` (amd64) |
| `docs/docs/troubleshooting/go2rtc.md:200` | FFmpeg-8-`hwupload`-Geräteanforderung |
