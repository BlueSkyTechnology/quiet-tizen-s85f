# Deploying Quiet Tizen on 2025 S85F OLEDs (QN55S85FAFXZA, QN65S85FAFXZA)

Deployment records for two TVs using this project's generalized tooling, 2026-11.
The original case study targets the UN75U8000FFXZA on T-KSU2EFAKUC-1300.9; these
notes document what matched, what diverged, and what was adapted. Home-network
addresses and device-specific identifiers are intentionally omitted.

## Devices

| | 55-inch | 65-inch |
|---|---|---|
| Model | QN55S85FAFXZA | QN65S85FAFXZA |
| Firmware | T-PTMFAKUC-0090-1130.3 | T-PTMFAKUC-0090-1203.0 |
| Profile | `profiles/qn55s85fafxza-1130.3.json` | `profiles/qn65s85fafxza-1203.0.json` |
| Hero HDMI card | Apple TV box, source 15 | Roku box, source 16 |
| Side cards | Live TV (0), DLNA (56) | Live TV (0), DLNA (56) |
| App catalog | 13 apps + tile | 16 apps + tile (2 pages) |

`launcher/src/Home.cs` is a per-device template: the committed variant carries the
latest deployment's cards (Roku on the 65-inch). Each deployment edits the hero
card labels/actions, `lastSource`, the `RefreshSource` acceptance window and the
dock entries, then rebuilds. Read source IDs with Probe on each TV — never copy
them between devices.

## Compatibility findings

Both S85Fs landed on the same platform family as the tested U8000F (verified
per-device before any mutation):

- Runtime: .NET 6.0.9 at `/usr/share/dotnet/shared/Microsoft.NETCore.App/6.0.9`
- Kernel: Linux 5.4.261 armv7l
- Capability-bearing `/usr/bin/dotnet` + `sdbd-tarlauncher` (the archive-root route
  works unchanged; `tv doctor` and `root-check` both pass)
- `package_app_info` and `key_filter` schemas identical
- All 54 app IDs across the six profile groups present; all 12 stop-marker units
  exist with the exact negated `ConditionPathExists`

## Divergences that required adaptation

1. **Startup selector**: `db/appfw/startup_appid` was `org.tizen.tv-viewer` on this
   unit (the U8000F had `com.samsung.tv.csfs` for all three selectors). The
   restore path must use the values actually recorded from the device.
2. **Extra settings-key row**: this firmware carries a second
   `com.samsung.tv.quicklaunch` / `XF86More` `key_filter` row conditioned on
   `on_selector = '~alive-package:org.tizen.function-osd'` (empty-string
   `on_selector` on the unconditional row, not NULL). The stock bridge recipe
   expects exactly one row and refuses. Adapted install/restore scripts re-route
   only the two direct equivalents and leave the conditional row untouched; they
   match rows with `ifnull(on_selector,'')` instead of `IS NULL`.
3. **Source IDs differ per wiring**: on this deployment the connected sources were
   HDMI (source 15, auto-detected Apple TV box), live TV (source 0) and a DLNA
   renderer (source 56). `Home.cs` was adapted accordingly (cards, `lastSource`,
   `SourceIndex`, a `source:` action prefix alongside `hdmi:`, and a widened
   `RefreshSource` acceptance window). Do not copy these numbers to another TV —
   read them with Probe first.

## Local fixes to the shared tooling (upstreamable)

- `tools/tv.py` `--script` path piped base64 text straight into `/bin/sh -s`
  without decoding (`… | base64 -d | /bin/sh -s --` is required); the recipe path
  was broken on every platform, not just Windows.
- `tools/tv.py` now normalizes CRLF in uploaded scripts (Windows checkouts) and
  accepts `--profile` so a second device profile can be used without editing the
  default out.
- `profiles/qn55s85fafxza-1130.3.json`: second device profile. Groups and
  stop-markers were copied from the U8000F profile only after every app ID and
  unit condition was verified present on the S85F itself.

## Applied phases (all reversible)

Each phase took a fresh snapshot (`recipes/snapshot.sh`) with restore scripts kept
both on the TV under `/opt/usr/share/quiet-tizen-backups/<phase>/` and locally:

advertising-01 (10 rows, pisa/canalysis markers, adagent stop, four runtime-only
masks) → voice-01 (17 rows, 4 markers, voice prefs zeroed) → context-01 (13 rows,
3 markers) → smart-extras-01 (11 rows, 2 markers) → tv-plus-01 (1 row, slive
marker, autorun/preload prefs, WAS `RESERVED_18 $.rsp.preloadByCountry` guarded
JSON update) → Quiet Home install in the `com.samsung.tv.bixby-promotion`
container → Home/startup/main-operation assignment → settings bridge (adapted) →
light-shell (sidebar + browser) from the shell-01 snapshot.

Post-restart verification: 54 disabled rows retained, selectors intact, all
stop-marker units inactive, launcher healthy.

## Restore order for this deployment

```
tv exec --script recipes/restore-launcher.sh --arg home-01 \
  --arg com.samsung.tv.csfs --arg com.samsung.tv.csfs --arg org.tizen.tv-viewer
# then the adapted settings-bridge restore (shell-01), then light-shell restore,
# then each cleanup group's restore-registry.sh + stop-marker.sh restore,
# then preferences from each snapshot's preferences-before.txt.
```

## Far-field voice: the listener the U8000F group missed (2026-11-12)

The "AI hearing us" report was real: `com.samsung.tv.farfieldvoice`
(ffv-service, the always-listening far-field mic daemon) runs on BOTH S85Fs
and is **not in the U8000F voice group**. Its registry row had
`app_autorestart=true`, and a Samsung catalog refresh had silently restored
`db/was/preload_by_country=true` (both the vconf key and WAS
`RESERVED_18.$.rsp.preloadByCountry`) — the documented relapse — which
force-preloads voice apps at boot **regardless of `app_disable`** (proof:
`org.tizen.voice-app` and `com.samsung.tv.vif` were registry-disabled yet
running, even after a post-cleanup reboot). `app_launcher -k` alone does not
stick: the live app-loader respawns them within seconds.

The `voice-02` phase that holds (verify after each reboot):
1. registry-disable farfieldvoice (disable/onboot/autorestart → false/false/false)
2. stop markers on `com.samsung.tv.farfieldvoice.service` and
   `org.tizen.voice-app.service` (both carry `ConditionPathExists=!` —
   vif has no unit; with the other two gated it stays down)
3. preload flags set false again (BOTH layers)
4. reboot — the only thing that clears the respawner; then verify
   `ps -ef | grep -E 'farfieldvoice|voice-interaction|voice-app/bin'`
   is empty and the flags are still false

Both S85F profiles now include farfieldvoice in the voice group and its
stop marker. Recheck the preload flags after any firmware/catalog event.

## Auto-update disabled on both (2026-11-12)

`swu-auto-upgrade` + `was-background-update` stopped and runtime-masked
(this boot), plus the boot-durable block: the vendor unit's own
`ConditionPathExists=!/mnt/backup/OS_UPGRADE_FLAG` against the persistent
`/mnt/systemrw` mount. Manual update still works by removing the flag
first. Persistent unit masking is blocked by SMACK (read-only
`/etc/systemd/system`), matching the docs' note on firmware-backed units.



- Runtime systemd masks do not survive a restart; Samsung catalog refreshes can
  restore the cached/live preload flags. Recheck after firmware updates.
- On the 65-inch (1203.0), one hard power cycle reverted `db/appfw/startup_appid`
  to the viewer; boot still landed on Quiet Home via the home handler. Re-set it
  and verify after any ungraceful shutdown.
- `app_launcher -s com.samsung.tv.store` starts the store process without its
  Smart Hub environment and it rendered blank/froze one unit; opening the store
  through the launcher's "Get more apps" tile is the reliable path.
- The "Get more apps" tile is the final entry of the shipped
  `launcher/assets/apps.json` (`Id: org.volt.apps`, `Utility: true`). Replacing
  the catalog wholesale with `apps.local.json` silently drops it — re-append the
  utility entry to every custom catalog, and keep per-device catalogs separate
  when the installed-app sets differ (an entry whose app is not installed fails
  at launch with the on-screen error toast).
- The project never claimed a decrypted-traffic audit; "advertising/ACR components
  disabled" is the accurate statement, not "zero telemetry proven".
- `tv.py pull`/absolute-path `exec` commands need MSYS path-conversion disabled on
  Git Bash (`MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*'`).

Credits: quiet-tizen by jss8649 (MIT), visual design per Free My TV, temporary
access method per samsung-tv-root by chris-ritsen. This deployment's changes are
kept under the same MIT license with upstream attribution.
