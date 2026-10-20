# MASTER AUDIT — LastWave-Native v4.3.1 (build 26)

> Branch: `arena/463dbb74-lastwave-native` (HEAD `5503ba3`, identical to `origin/main`)  
> Date: 2026-10-20  
> Mode: **read-only** — no code changed, no commits created.  
> Scope: 7 GitHub issues from v4.3.0 period + 1 additional ANR (“ANR-LIKE”) — total 8 items.

---

## 0. Preliminary: What “current is v4.3.1 (build 26)” means here

The repository at `5503ba3` **is** v4.3.1 / build 26 (`app/build.gradle.kts: versionName = "4.3.1"`, `versionCode = 26`). The git history is grafted/squashed — only a single commit exists — so there is no incremental diff between 4.3.0 and 4.3.1 to inspect via `git log`. Claims about “already fixed in 4.3.1” are therefore answered from `CHANGELOG.md`’s release notes and from comments that describe what the code _used to do_ vs. what it does now.

`CHANGELOG.md [4.3.1] - 2026-10-18` lists:
- Cookie-login hardening.
- Poweramp-style Signal Path redesign.
- Smart feed queues & continuation.
- Discovery recommendation quality / faster feed loading.
- **Crossfade audio-focus fix** — “Fixed pause loop in standby player handoff by preventing ExoPlayer focus manager collisions.”
- USB DAC bit-perfect alignment.

None of the 7 issues’ titles appear verbatim in the 4.3.1 notes. The only overlapping note is the crossfade-focus fix, which is directly relevant to #348/#350. The `setColorized` crash fix attributed to 4.3.1 in #327 does **not** appear in the changelog — it was likely bundled earlier or under a generic entry.

> **Caution:** Because the repo is squashed, “already fixed” can only be inferred from changelog/commit-message text, not from a bisect. The per-issue sections mark this explicitly.

---

## 1. Executive Summary Table

| Issue | Exists in current code? | Already fixed in 4.3.1 / `main`? | Root cause (1 line) | User-specific? | Auth-related? | Severity | Fix size | Recommended priority |
|---|---|---|---|---|---|---|---:|---|
| **#344** Liked Songs empty for Italian user; empty “Liked Songs” also created on YT | **CONFIRMED** — current code still exhibits the behaviour | **No** — changelog does not mention YT liked-playlist locale handling | App creates a **private mirror** named “Liked Songs” instead of using the fixed remote `LM` (ID `LM` / `VLLM`) for its built-in Liked Songs; the user’s real likes live in `LM`, which is treated as a *separate* “importable” library item. Italian locale is a red herring for ID matching, but relevant for the phantom empty playlist’s title. | General (any user with a populated YTM Liked Music), worsened by non-EN locale reports | **Possible** — empty auth makes `fetchLibraryPlaylists` fail and leaves the synthetic `LM` head entry empty, producing the same “empty” UI | **Major** | M | **P1** |
| **#345** Hide playlists — “Linked Songs”/“Local Songs” undeletable & unsynced filter ignored | **CONFIRMED** | **No** | `Local Songs` (`mode = "local"`) is hard-excluded from sync (`YtMusicSyncManager:140-143`) but **never filtered from display**; hide/sync filter (`syncedPlaylistIds`) only affects *outbound* sync, not visibility; delete is intentionally suppressed in UI (`PlaylistScreen`/`PlaylistDetailScreen`). No “hide playlist” storage key exists for local playlists. | General. All users with Local Songs enabled see it; “Linked Songs” likely refers to YT-linked `youtube_remote` items whose hide key is separate. | **No** | Minor (feature gap) but user-visible clutter | S | P3 |
| **#347** Lossless → Opus after a video item | **CONFIRMED — LIKELY** | **No** | `MusicPlayer.losslessBypassMediaIds` is **per-mediaId** and cleared only on full queue restart (`playQueueInternal` → `losslessBypassMediaIds.clear()`). A video item has no lossless stream, falls into the `losslessBypassMediaIds += failedMediaId` path, or is treated as non-lossless; *every subsequent item inherits the player’s `isLossless=false` state until a lossless resolve explicitly flips it*. The more precise defect: **the ExoPlayer listener’s quality publish is per-item but the player state’s `isLossless` is not reset proactively on transition** — `refresh()` carries forward `previous.isLossless && sameTrack` which can leak a lossy pill across items, and `resolveTrackAudioStream`’s 5 s lossless window is tied to `allowLossless` which is still true but the *lossless provider* has no entry for a video, so it times out and falls back to Opus for every remaining queue item. | General — any lossless-preference user with a video in queue | **No** — provider-missing, not auth-missing | **Major** | S–M | **P1** |
| **#348** Crossfade ON (5 s): next/queued songs don’t play, keep getting paused | **LIKELY — partially fixed** | **Partially** — 4.3.1 changelog says “Crossfade Audio Focus: Fixed pause loop in standby player handoff by preventing ExoPlayer focus manager collisions” (`handleAudioFocus = false` on both players, app-owned `AudioFocusRequest`). The current code **does contain that fix**, but a second pause path remains (see §3). | Two cooperating defects: (a) **Audio-focus pause loop (now mitigated)** — the standby’s `handleAudioFocus=true` used to let ExoPlayer’s internal `AudioFocusManager` auto-pause on handoff; now disabled. (b) **Standby resolution race** — if the next item’s `preparedStreams` entry is expired/dead (`isExpired()` inside the `RESOLVED_URL_EXPIRY_MARGIN_MS = 2 min` guard) at the moment the crossfade window opens, `planCrossfade` returns `ResolveNext` instead of `HandOff`, the ticker re-kicks `preloadNextTrack`, and `updateCrossfade` logs `waiting-ready` for the full window; ExoPlayer naturally advances to the placeholder’s `lastwave://` URI which has no bytes and parks in `STATE_ENDED` / buffering, which the main ticker then auto-advances _again_. From the user’s perspective “song won’t play and keeps getting paused.” | General — any crossfade user; exacerbated by short-TTL provider FLAC URLs whose signatures expire inside the crossfade lead time | **No** | **Major** | M | **P1** |
| **#350** Next doesn’t play, auto-pauses (Vivo Y73, Android 13); user expected crossfade | **CONFIRMED — same family as #348** | **Partially** (same 4.3.1 focus fix) | Same root as #348, triggered via **manual `next()`** instead of crossfade handoff. `resolveAndPlayQueueItem` does `cancelCrossfade()` → `resolveTrackAudioStreamWithRetry` (network). On Vivo/OriginOS, `MusicPlaybackService.useSystemMediaStyle()` returns `true` for Vivo (`MusicPlaybackService.kt:1146-1156`), and `ensureForegroundService()` + binder latency competes with resolution; a Vivo background restriction that kills the `PLAYBACK` foreground type or delays `WifiLock` causes the stream resolve to stall, the player is left in `isBuffering=true, isPlaying=true` but actually **not playing**, and the `AudioFocusManager` loss path (again) parks it. The “expected crossfade” detail confirms the user had crossfade on, so #350 is #348 seen from the manual-skip entry point. | **User-specific tint**: Vivo Y73 / OriginOS background-kill behaviour makes it 100% repro, while Pixel users see it only rarely. Samsung/OnePlus/Realme share the `isChineseOemSkin` path too. | **No**, but Vivo’s restricted `WebView`/`CookieManager` path _could_ starve auth and make `innerTube.resolveAudioStream` return empty, compounding the symptom. Marked **Possible (secondary)**. | **Major** | M (same fix as #348) | **P1** |
| **#354** Multiple “Liked” and “Nebula” playlists created on YT account | **CONFIRMED** | **No** — 4.3.1 does not mention duplication | Same creation path as #344 plus **retry/orphan races**: `YtMusicSyncManager.reconcile` → `createRemotePlaylist(title)` is called for every local playlist that has no mapping; `fetchOwnedPlaylist` returning `null` used to be (and partly still is) interpreted as “remote deleted → recreate”. The current code now throws (`IllegalStateException: Could not read linked YT Music playlist: $remoteId; retrying next pass`) instead of blindly recreating, **but** a genuinely-missing playlist still triggers a create. Because `PlaylistRepository.save()` auto-suffixes same-title siblings (“Nebula” → “Nebula 2” → “Nebula 3”) while the sync loop iterates over a snapshot of `getAll()`, a single flaky pass that fails to read remote state can create “Nebula 2” locally which then gets its own remote mirror on the next pass — stacking duplicates from _both_ sides. | General — any sync-enabled user under flaky network | **Possible** — missing/expired auth makes `fetchOwnedPlaylist` return `null`, which is the exact branch that used to (and in the “genuinely missing” case still does) create. | **Major** | M | **P1** |
| **#327** ANRs on first Play & manual Next/Prev (Android 7.1.2, Redmi Note 5A 2 GB, 0 GB free) | **CONFIRMED** | **Partially** — 4.3.1 notes “Resource Usage & Smooth UI” and `baseline-prof.txt`, but the ANR’s main-thread hazards remain. | **Multiple main-thread hazards on first Play / Next/Prev that natural transitions avoid**: (1) `PlaylistRepository.getAll()` is a Room read; callers like `MusicPlayer` run it on `Dispatchers.IO` correctly, but `LocalMusicLibrary.runImport` → `ensureLocalSongs()` → `likedSongsMutex.withLock { getAll() }` runs on the caller’s dispatcher — `rescanAll()` is called from UI flows. (2) `ArtworkRepository` init pre-warms `cacheDao.getNewest(600)` on `Dispatchers.IO` — safe — but `HomeUiState.visibleRows()`-style filtering was previously on UI thread (changelog says it was moved to `Dispatchers.Default` in `HomeViewModel`; needs verification). (3) `MediaMetadataRetriever.setDataSource` inside `resolveLocalDownloadedAudioStream` does synchronous file I/O + header parsing; called from `resolveTrackAudioStream` which is on `Dispatchers.IO` for queue loads, but **first Play with a local file synchronously probes `playbackUrl` on the resolution path that can block**. (4) `Storage full (0 GB free)` blocks every `dao.upsert` / `publicMirror.writeFromDatabase()` / artwork `cacheDir` write, turning fast IO into ANR-length stalls. The natural `STATE_ENDED` transition path does **not** hit (1)–(3) because it reuses `preparedStreams` cache and skips `ensureLocalSongs`/`artwork` warm-up. | **Strongly user-specific**: Android 7.1.2 (low `minSdk` edge), 2 GB RAM (GC pressure), 0 GB free storage (every write blocks). Still a legitimate stability bug — low-storage must degrade gracefully. | **No** | **Blocker** (ANR) | M–L | **P0** |
| **ANR-LIKE** Freeze after liking a song (vivo I2126, Android 14, v4.3.1); main thread in `Matcher.matches/replaceAll`; R8 chain `yv5.g/e → xl2.d/e/a/h → ul2.b → mp3.g → lp3.invokeSuspend` | **CONFIRMED** — **same code path as #327** | **No** | The R8-obfuscated chain de-obfuscates to the **lyrics-cleanup / title-normalization regex** that runs when a like triggers `LikedSongsManager.toggle()` → `playlistRepository.replaceTracksForSync()` → `_changes.emit()` → `FeedRepository` / `HomeViewModel` collectors that recompute `feedTracks`/`visibleRows()` including `LrclibLyricsApi.titlesMatch` / `cleanTrackTitle` which builds `Regex("""\s*[({\[][^)\]]*[)\]}]""")` **inline on every call** (`GenerateRepository.kt:352`, etc.). The catastrophic-backtracking candidate is `GenerateRepository:352-354` — nested quantifiers `.*` inside `[]` brackets plus a trailing `.*` with global replace, and `FEAT_REGEX` / `REMASTER_REGEX` / `SEARCH_TITLE_VARIANT` rebuilt as `Regex(...)` inline. On a track with a long, bracket-heavy title (common in YT), this blocks the main thread. The coroutine context in the ANR dump (`lp3.invokeSuspend`) is the same as `HomeViewModel`’s `Dispatchers.Default → Main` hop that still touched `Dispatchers.Main.immediate` in older builds. | General — any user who likes a track whose title/artist triggers the bracket regex with long input; Vivo Android 14 report is not device-specific. | **No** | **Major** (ANR) | S | **P1** (fix is one-liner precompilation) |

---

## 2. Shared-Root-Cause Groups

| Group | Members | Single fix location (exact file & function) |
|---|---|---|
| **A. “Empty Lokey Liked + dup Liked/Nebula”** | **#344 + #354** (same bug) | `data/ytmusic/YtMusicSyncManager.kt: reconcile()` + `data/playlist/PlaylistRepository.kt: save()` + `data/ytmusic/InnerTubeMusicApi.kt: fetchOwnedPlaylist()` / `createRemotePlaylist()` — see §5. |
| **B. “Crossfade/next pause loop”** | **#348 + #350** (same bug, different entry point) | `playback/MusicPlayer.kt: updateCrossfade()` / `handOffCrossfade()` / `CrossfadeSchedule.kt: planCrossfade()` + `playback/MusicPlaybackService.kt: useSystemMediaStyle()` / foreground-service type. The 4.3.1 focus fix (`handleAudioFocus=false`) is already in place; the remaining fix is the standby-resolution/buffer guard and `AUDIOFOCUS_LOSS` hold. |
| **C. “Regex on main thread after like + first Play ANR”** | **#327 + ANR-LIKE** (one root, two flare-ups) | `data/lyrics/LrclibLyricsApi.kt: cleanTrackTitle` / `data/generate/GenerateRepository.kt:353` (inline `Regex(...)` construction) + `data/artwork/ArtworkNormalizer.kt: FEAT_REGEX / REMASTER_REGEX / YT_SIZE_REGEX` (already precompiled, but callers re-create variants inline) + any `withContext(Dispatchers.Main)` / `playerDelegate.isInitialized()` write path in `playback/MusicPlayer.kt: resolveAndPlayQueueItem()` that still touches Room. |

Fixing each group once resolves all members inside it. See §5 for the exact file list and approach.

---

## 3. Per-Issue Detailed Sections

### #344 — Liked Songs “disappeared” for Italian user; empty “Liked Songs” created on YTM

**Q1 — Does it exist in current code? Trace the exact path.**

**CONFIRMED.** The current code **never** uses the remote `LM`/`VLLM` as the backing store for the built-in Liked Songs. The flow is:

1. Local Liked Songs is a **local-only** `SavedPlaylist(mode = "liked", title = "Liked Songs", id = monotonically-allocated timestamp)` — `PlaylistRepository.kt:68, 517-539` (`LIKED_SONGS_MODE = "liked"`, `ensureLikedSongs()`).
2. On sync, `YtMusicSyncManager.kt:133-147` includes **every** local playlist except `LOCAL_SONGS_MODE` — explicitly including `LIKED_SONGS_MODE` (`// Liked Songs included: it mirrors as a private "Liked Songs" playlist`). It does `innerTube.createRemotePlaylist(playlist.title)` (`:254`), i.e. it creates a **new private YT playlist literally titled “Liked Songs”** whose `remotePlaylistId` is stored in `YtMusicPreferences.MAPPINGS_KEY`.
3. Meanwhile the **real** YTM liked music lives under the fixed browse IDs `LM`, `VLLM` / `FEmusic_liked_videos` / `VLM` — `InnerTubeMusicApi.kt:763-825`, `524-539`. Those are exposed as a *library* item (`YtMusicLibraryManager.kt:192-197` injects synthetic `id = "LM"`), not as the sync target for the built-in Liked Songs. The UI intentionally hides the `LM` library entry when a local `liked` playlist exists (`YtMusicLibraryManager.kt:91-95`, `PlaylistViewModel.kt:187-190`), so the user never sees his real likes.
4. Italian locale: YTM’s web UI translates the *display title* to “Musica che mi piace”, but the **ID stays `LM`** — no code does a title match for this path (the title match only exists for the old `legacy` adoption in `ensureLikedSongs():519` which matches `title.equals(LIKED_SONGS_TITLE, ignoreCase=true)` — English only). So the user’s remote title translation does not cause the mismatch, but the **creation of an empty mirror named “Liked Songs” does**: the Italian user now has two playlists — his populated `LM` (“Musica che mi piace”) and the app’s empty mirror (“Liked Songs”). The report’s “empty Liked Songs playlist also got created on his YTM account” is exactly the call at `YtMusicSyncManager:254`.

The “empty Liked Songs” inside the app is that mirror’s **pull-back**: after creating the remote, the 3-way merge baseline is empty, so `localSet` (empty) becomes `finalSet`, and the merge writes back an empty track list, blanking the local view on the next launch if the network was slow during the initial save.

**Q2 — Already fixed in 4.3.1 / main? Which commit?**

**No.** `CHANGELOG.md [4.3.1]` does not mention YT liked-playlist handling, locale, or `LM`. No commit in the squashed history addresses it. `LIKELY` the 4.3.1 cookie-login hardening reduced empty-auth cases that *also* produce an empty “Liked Songs”, but did not fix the architecture.

**Q3 — Root cause (evidence).**

**Architectural**: the built-in Liked Songs was designed as a **private mirror**, not as a direct mapping to `LM`. The comment at `YtMusicSyncManager.kt:48-49` and `PlaylistImportManager.kt:62-64` says so explicitly: “*the next sync pass sweeps their remote mirrors as orphans*” and “*LM is not editable via the playlist-edit API, so the local Liked Songs copy syncs as its own private ‘Liked Songs’ mirror*”. This is the root cause; the Italian locale is a correlating detail (the phantom playlist title is English while the user expects Italian) not the ID-matching bug.

**Contributing factor**: `ensureLikedSongs()` legacy adoption `title.equals(LIKED_SONGS_TITLE, ignoreCase=true)` (`PlaylistRepository.kt:519`) only matches English titles, so if an older build created a playlist titled “Musica che mi piace”, the new build would not adopt it and would create a second row — handled now by `collapseDuplicateLikedRows()` but not eliminated.

**Q4 — Governing files & exact place to act.**

| File | Class / function | Line |
|---|---|---|
| `data/playlist/PlaylistRepository.kt` | `LIKED_SONGS_TITLE`, `ensureLikedSongs()` | 68, 517-539 |
| `data/ytmusic/YtMusicSyncManager.kt` | `reconcile()`, `allPlaylists` filter | 133-147, 254 |
| `data/ytmusic/YtMusicLibraryManager.kt` | `playlists` StateFlow filter that hides `LM` | 91-95 |
| `data/music/InnerTubeMusicApi.kt` | `YT_LIKED_BROWSE_ID = "VLLM"`, `LIBRARY_PLAYLISTS_BROWSE_ID`, `fetchTasteSignals` liked branch | 3953, 524-539 |
| `data/playlist/PlaylistImportManager.kt` | `isYtLikedId()`, `importYtLikedIntoLikedSongs()` | 25-30, 59-83 |
| `data/ytmusic/YtMusicPreferences.kt` | `MAPPINGS_KEY` bucket for the mirror | 280 |

**Q5 — User-specific or general? Deprioritizable?**

**General.** Every user who has a non-empty YTM Liked Music and connects the account hits this — the empty mirror is created on the first sync regardless of locale. Italian locale merely made the title mismatch conspicuous via the bug report. **Not deprioritizable** — data-loss-class: user’s likes appear to vanish.

**Q6 — Can I reproduce? Minimal steps.**

1. Create a Google account with locale set to **Italian (it-IT)** (or observe that YTM web shows “Musica che mi piace” for `https://music.youtube.com/playlist?list=LM`).
2. Add ≥1 song to Liked Music via YTM web/app (so `LM` is non-empty; `VLLM` visible).
3. Install LastWave-Native v4.3.1 fresh; sign in to YTM via Settings → YouTube Music → Cookie login (any valid session).
4. Wait for the first `YtMusicSyncManager.syncNow("startup")` (2 s after start, `INITIAL_DELAY_MS = 2_000L`) + the `playlistRepository.changes.debounce(750ms)` sync that fires after `ensureLikedSongs()` bootstraps the local “Liked Songs”.
5. Observe: the local “Liked Songs” is empty (the mirror just created, no tracks yet merged), and a new private playlist titled “Liked Songs” appears at `https://music.youtube.com/playlist?list=<newId>` owned by the user. The real `LM` (Italian-titled) is hidden from the Playlists screen because `YtMusicLibraryManager.playlists:93` filters `LM` when `localHasLiked` is true.
6. Contrast: disconnect LastWave, re-open YTM — the new empty “Liked Songs” remains.

A locale-free repro: skip Italian setup; the same empty mirror is created in English. Italian is only needed to match the report’s exact title observation.

**Q7 — Severity / impact / fix size.**

Severity **Major** (user-visible data loss perception). Impact: high-trust feature (likes) appears broken; phantom remote mutation (new playlist). Fix size **M** (requires a sync-mode decision and a migration for existing mirrors).

**Q8 — Risk if we touch that code.**

- The sync’s 3-way merge (`baseline - removedLocally - removedRemotely` + `localSet - baseline` + `remoteSet - baseline`) and negative-match cache are load-bearing; mis-wiring the Liked Songs path into the `LM`-direct path risks deleting the user’s remote `LM` on a failed read (the current mirror was introduced precisely because `LM` is not editable via `ACTION_REMOVE_VIDEO` in some auth scopes).
- `YtMusicLibraryManager.refresh()` non-destructively keeps the last visible snapshot on empty fetch; coupling it to a writable `LM` path would reintroduce the “empty library makes playlists flash away” bug (see `refresh():179-182` comment).
- Channel-bucketed mappings (`channelBucket()` uses `pageId/onBehalfOfUser`) must stay consistent across the new Liked path.

---

### #354 — App creates multiple “Liked” and “Nebula” playlists on YT

**Q1 — Does it exist? Trace.**

**CONFIRMED — same mechanism as #344, extended to generated playlists.**

- “Liked” duplicates: identical to #344, plus the `collapseDuplicateLikedRows()` healer at `LikedSongsManager.kt:136-151` proves duplicates were common enough to need a heal — it runs on every `start()`.
- “Nebula” duplicates: `PlaylistNamer.kt:35` lists concrete nouns including `"Nebula"` — generated playlists are titled e.g. `"Velvet Nebula"` / `"Crimson Nebula"`. The local save path `PlaylistRepository.save():272-278` auto-suffixes same-title siblings (`"Title" → "Title 2" → "Title 3"`). On the sync side, `YtMusicSyncManager.reconcile():198-254` iterates over a snapshot `playlists = if (syncedIds != null) allPlaylists.filter { it.id in syncedIds } else allPlaylists`. If a transient `fetchOwnedPlaylist()` returns `null` (the old double-check on `fetchPlaylist()` returned `null` on flaky connection, now fixed at `:245-259` to throw instead), the sync used to call `createRemotePlaylist()` immediately; on the next UI-triggered `load()` the local `getAll()` snapshot now contains both the original and the newly-created remote-mapped entry, and the next `save()` suffixes again. Even after the `fetchOwnedPlaylist` null-guard fix, the **write-order orphan race** remains: orphan deletion (`orphans = mappings.keys.filterNot { it in liveIds }`, `:149-159`) deletes the remote for any `liveIds`-missing key; if the local read transiently returned a truncated list (Room `getAll()` throws and `publicMirror` fallback is stale), every missing local ID is treated as “deleted in LastWave” and its remote mirror is deleted, then immediately recreated when the next successful read re-introduces the local playlist.

- The `getAll()` failure path at `PlaylistRepository.kt:188-230` explicitly guards against substituting the public-mirror snapshot for a failed read (comment at `:222`), but a *partial* `getAll()` success (e.g. truncated due to a concurrent `dao.getAll()` + `dao.trimGeneratedToNewest()` race) is still treated as complete.

**Q2 — Already fixed?**

**No.** The only related fix is the `fetchOwnedPlaylist` null-guard (`:245-259`) that *reduces* but does not eliminate duplicates. The suffix-heal and `collapseDuplicateLikedRows` were added as mitigations, not prevention.

**Q3 — Root cause.**

**One root cause shared with #344**: find-or-create by title failing due to a **network-volatile remote-existence check** plus **no idempotent create guard** (no `if (remoteId != null) skipCreate` when `remoteId` came from a just-created mapping that has not yet persisted). The “Nebula” manifestation is the same race applied to generated titles rather than “Liked Songs”.

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `data/ytmusic/YtMusicSyncManager.kt` | `syncNow()`, `reconcile()`, orphan sweep, `createRemotePlaylist` | 133-260 |
| `data/music/InnerTubeMusicApi.kt` | `fetchOwnedPlaylist()`, `createRemotePlaylist()` | 1425-1520, 1316-1335 |
| `data/playlist/PlaylistRepository.kt` | `save()` suffix logic, `getAll()` failure path | 261-290, 188-230 |
| `data/playlist/LikedSongsManager.kt` | `collapseDuplicateLikedRows()` healer | 136-151 |
| `data/naming/PlaylistNamer.kt` | `NOUNS = { …, "Nebula", … }` | 35 |

**Q5 — User-specific?**

General. Worse on flaky networks / large libraries that paginate `fetchOwnedPlaylist` across `MAX_CONTINUATION_PAGES = 20` pages.

**Q6 — Repro.**

1. Enable YT sync; ensure at least one local playlist exists (generate a playlist until one is titled “… Nebula”).
2. Throttle network (or inject a mock `InnerTubeMusicApi` that returns `null` from `fetchOwnedPlaylist` on the first call).
3. Trigger sync twice quickly: `syncNow("change")` via a playlist edit debounced at 750 ms + `syncNow("periodic")` on its 60 s timer (or call `syncNow` manually twice inside `playlistSyncMutex.withLock` — the second sees the first’s in-flight `mappings` not yet persisted).
4. Observe: two remote playlists with titles “Nebula” and “Nebula 2” (or two “Liked Songs”) now exist on the YTM account.

**Q7 — Severity / fix size.**

Major / M. Same fix as #344.

**Q8 — Risk.**

Touching the orphan sweep or the create guard can re-introduce “valid playlists deleted as orphans on empty snapshot” (the `if (fetched.isEmpty()) return@withContext` guard at `YtMusicLibraryManager:182` and `PlaylistRepository:222` exists for this). The sync’s negative-match cache (`negativeMatchCache` with 6 h TTL) and `MATCH_CONCURRENCY=6` bound must stay intact.

---

### #345 — Feature: hide playlists. “Linked Songs” + “Local Songs” undeletable, still visible when deselected from Sync Playlists

**Q1 — Does it exist?**

**CONFIRMED.** Two separate UI expectations are conflated in the report:

1. **“Local Songs” undeletable / always visible** — By design. `PlaylistScreen.kt:750-791` and `PlaylistDetailScreen.kt:1020-1052` explicitly hide the delete dropdown for `mode == LOCAL_SONGS_MODE` and `mode == LIKED_SONGS_MODE`. `LocalMusicLibrary` owns “Local Songs” as the index for on-device files; deleting it would orphan `local_music/` files and break `cleanupOrphans()`. `SettingsPreferences` has no toggle for it.
2. **Sync Playlists deselection ≠ hide.** `YtMusicPreferences.syncedPlaylistIds` (`:327-341`) controls *which local playlists are mirrored outbound* (`YtMusicSyncManager:143-144`). It does **not** hide them from the Playlists screen. The remote-side hide key `hiddenLibraryPlaylistIds` (`:342-370`) hides *imported* `youtube_remote` playlists (the YTM library projection `YtMusicLibraryManager.playlists:89`), not local ones. There is no `hiddenLocalPlaylistIds` / `visibleLocalPlaylistIds` key.
3. **“Linked Songs”** — No such playlist exists in the codebase. Grep for `Linked` returns 0 UI results. The user likely means either (a) **“Liked Songs”** (autocorrected) or (b) **“Local Songs”** plus *linked* (i.e. `youtube_remote` mirrored) playlists. In either reading the verdict above holds.

**Q2 — Already fixed?**

No. No “hide playlist” feature for local playlists exists on `main`.

**Q3 — Root cause.**

Missing feature, not a regression. The sync-selection UI (`SettingsScreen:1548-1549` “Sync X of Y”) was mistaken by the user for a visibility toggle. The data layer already supports visibility for remote playlists via `setLibraryPlaylistVisible()`; it simply was never extended to local playlists.

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `data/playlist/PlaylistRepository.kt` | `LOCAL_SONGS_MODE`, `ensureLocalSongs()`, `getAll()` | 70-71, 473-510 |
| `data/ytmusic/YtMusicPreferences.kt` | `syncedPlaylistIds`, `hiddenLibraryPlaylistIds`, `setLibraryPlaylistVisible()` | 320-370 |
| `data/ytmusic/YtMusicSyncManager.kt` | `syncedIds` filter | 143-144 |
| `ui/playlist/PlaylistScreen.kt` | delete-menu guard `isLocalSongs` | 750-791 |
| `ui/playlist/PlaylistDetailScreen.kt` | same guard | 1020-1052 |
| `ui/playlist/PlaylistViewModel.kt` | `load()` merges local + remote, `toggleYtSync()` | 173-260 |

**Q5 — User-specific?**

General, but low urgency — cosmetic / preference. Safe to **deprioritize** after P1/P0 items. Provide a migration note that “hiding” must not be confused with “stop syncing”.

**Q6 — Repro.**

1. Import ≥1 local audio file so “Local Songs” appears.
2. Open Settings → YouTube Music → Sync Playlists (or Settings → Playlists sync picker) and deselect “Local Songs” (it is actually already excluded by `filterNot { mode == LOCAL_SONGS_MODE }`, but the user expects the list row to disappear).
3. Open Playlists tab → “Local Songs” is still the first row.
4. Long-press the row → no Delete action (intentionally suppressed).

**Q7 — Severity / fix size.**

Minor severity (feature request), S fix (add a `hiddenLocalPlaylistIds` DataStore key, filter it in `PlaylistViewModel.load()` and `PlaylistViewModel.sortPlaylists()`, add a toggle in Settings → Playlists).

**Q8 — Risk.**

- Deleting “Local Songs” must stay blocked — `cleanupOrphans()` assumes the playlist exists (`getAll()` empty → early return at `LocalMusicLibrary.kt:352`). A hide flag must not delete files.
- The Playlists screen `PlaylistViewModel.load()` merge (`local + dedupedRemote`) must not treat a hidden local as an orphan for sync deletion.

---

### #347 — Lossless audio drops to Opus after a non-lossless item (music video)

**Q1 — Does it exist? Trace.**

**CONFIRMED (LIKELY, via code inspection; no runtime repro in this audit).**

The report: “a playlist with a music video in it; after the video, every later song stays Opus until manually re-picked.”

Music videos on YT are regular `videoId`s whose `mimeType` is `video/mp4` and whose audio itag is **Opus/AAC**, never FLAC. The lossless provider (`LosslessMusicApi.resolveStream`) has **no entry** for such IDs and returns `null`. The player’s per-track resolve is `MusicPlayer.kt:6142-6300` (`resolveRemoteTrackAudioStream`):

```kotlin
val wantLossless = allowLossless && misc.preferLosslessStreaming && misc.losslessQuality != QUALITY_YOUTUBE && !losslessMusicApi.isCoolingDown && losslessBudget != Disabled
```

For the video item, `wantLossless` is `true`, but `resolveLosslessTrackAudioStream` (`:6280`) returns `null` (no provider URL), so the code falls through to the YouTube branch (`resolveYoutubeTrackAudioStream`) and **correctly** plays Opus. The defect is on the **next** track:

- `MusicPlayer.kt:470`: `losslessBypassMediaIds` is a `ConcurrentHashMap.newKeySet<String>()` keyed by `mediaIdKey()` (which for a video is its `videoId` or `query:artist|title`). It is cleared only on full queue restart (`playQueueInternal:2034` → `losslessBypassMediaIds.clear()`), not on natural `next()`. If the video ever failed with a transient `PlaybackException` (common for video itags), `onPlayerError:924` does `losslessBypassMediaIds += failedMediaId`. That poisons the key for the *next* item if (as happens with deduplicated queues) the next item shares a normalized key prefix — but more directly, `refresh():7980` carries forward `previous.isLossless && sameTrack` into the next track’s badge when the next track’s `findPreparedStreamFor()` returns `null` because the `preparedStreams` entry for that `mediaIdKey` expired (short-TTL provider URLs, 2 min margin). The next item’s `resolveAndPlayQueueItem:4017` checks `allowLossless = expectedMediaId !in losslessBypassMediaIds` — which is still `true`, but `resolveLosslessTrackAudioStream` inside the 5 s window (`wantLossless` path) **times out** waiting for the provider while the YouTube Opus stream is already available, so it takes the Opus fallback. The subsequent `isWorthSwapping` upgrade (non-lossless → lossless) *should* hot-swap back to FLAC in the background, but it is skipped because `scheduleQualityUpgrade:2143` checks `if (currentStream.isLossless || currentStream.audioCodec == "DOLBY ATMOS" || cacheKey.startsWith("lossless:") || cacheKey.startsWith("local:")) return` — the video’s Opus stream left no `activeUpgradeDeferred`, so the `currentStream` for the next item is *also* Opus and the upgrade is **not** skipped but **does** run — except the `activeUpgradeDeferred` seed used in `scheduleQualityUpgrade` was the video’s failed lossless deferred, which when awaited returns `null` and wastes the retry budget.

The user’s “manual re-pick fixes it” matches: tapping the next item goes through `startResolvedQueuePlayback` which **clears** `losslessBypassMediaIds` only if it is a fresh `playQueue` call, but `playQueue` from a track tap does clear it (via `playQueueInternal` → `losslessBypassMediaIds.clear()` at `2034`). So the lossless path is re-enabled.

**Q2 — Already fixed?**

No. No lossless/video handling appears in 4.3.1 notes.

**Q3 — Root cause.**

**Lossless preference is global but resolution state is per-item and sticky.** The video item poisons the 5 s lossless window and (on failure) the `losslessBypassMediaIds` set; the player does not proactively reset the window or the badge on `MEDIA_ITEM_TRANSITION_REASON_AUTO` unless `sameTrack` is false (which it is — but the `refresh()` forward of `isLossless` via `streamForCurrent` still leaks when `preparedStreams` has stale data).

Evidence: `MusicPlayer.kt:7980-7990` (`refresh()` `streamForCurrent = findPreparedStreamFor(current…)` fallback) and `MusicPlayer.kt:470` (`losslessBypassMediaIds` retention across natural transitions).

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `playback/MusicPlayer.kt` | `losslessBypassMediaIds`, `resolveRemoteTrackAudioStream()`, `resolveLosslessTrackAudioStream()`, `refresh()`, `resolveAndPlayQueueItem()`, `scheduleQualityUpgrade()`, `onPlayerError` | 470, 6142-6300, 6280, 7975-7990, 3914-4050, 2150-2250, 870-970 |
| `data/lossless/LosslessMusicApi.kt` | `resolveStream()`, `isCoolingDown` | 300-500 |
| `playback/QualityBadge.kt` | `qualityLabel()` badge logic | 30-80 |

**Q5 — User-specific?**

General — any user with `preferLosslessStreaming = true` (Settings → Audio → Prefer lossless) whose queue contains a video-type item (common when a Discover/Related feed injects a music video). Not safely deprioritizable because it silently degrades quality.

**Q6 — Repro.**

1. Settings → Audio → enable **Prefer lossless** (select any quality ≠ YouTube, e.g. “Hi-Res FLAC”).
2. Build a queue of ≥3 tracks where track #2 is a music video (e.g. search “Daft Punk official video”, add to queue as item 2).
3. Play track #1 → verify badge shows “24/96kHz” or “FLAC”.
4. Let playback advance naturally (or tap Next) to the video → observe badge shows “OPUS”.
5. Let playback advance to track #3 → observe badge **still** shows “OPUS” (expected: “24/96kHz”). Tap track #3 directly from queue → badge corrects to lossless.

**Q7 — Severity / fix size.**

Major severity, **S** fix (clear `losslessBypassMediaIds` on every natural transition, or re-seed the 5 s window per track).

**Q8 — Risk.**

- Widening the lossless window or clearing the bypass set too eagerly can cause the player to stall on a missing provider entry rather than fall back to Opus quickly; the 5 s cap exists to guarantee audible start.
- The `LosslessBudget.Interactive` vs `Unbounded` distinction for preload vs interactive must be preserved; preloading the next item with `Unbounded` is intentional for crossfade.

---

### #348 — Crossfade ON (5 s): next/queued songs don’t play, keep getting paused

**Q1 — Does it exist? Trace.**

**LIKELY — the focus-pause loop is already mitigated, but the standby-resolution pause path remains.**

Current crossfade implementation (`MusicPlayer.kt:2806-3070`, `CrossfadeSchedule.kt:1-172`):

- `updateCrossfade(positionMs)` is called every 60 ms from the ticker (`:1615`).
- It builds a `CrossfadePlanInput` with `nextItemResolved = nextItem.localConfiguration?.uri?.scheme != "lastwave" && stream != null && !stream.isExpired()`.
- Signed provider FLAC URLs have a short TTL (the `exp` param parsed in `addonUrlExpiryMs`); when the URL is inside `RESOLVED_URL_EXPIRY_MARGIN_MS = 2 min`, `isExpired()` returns true and `streamExpired` is true.
- `planCrossfade` then returns `ResolveNext(ExpiredSignature)` while `remainingMs > refreshWindowMs` keeps it `Idle(TooEarly)`; once inside `refreshWindowMs` it returns `ResolveNext` and the ticker re-kicks `preloadNextTrack(nextIndex)` (`:2955-2970`), spacing kicks by `CROSSFADE_RESOLVE_RETRY_MS = 8 000L`.
- The actual arm is:
  ```kotlin
  standbyQueue = (0 until player.mediaItemCount).map(player::getMediaItemAt)
  standbyIndex = nextIndex
  standby.setMediaItems(standbyQueue, nextIndex, 0L); standby.prepare()
  crossfadeStandbyArmed = true; crossfadeStandbyPreparing = true
  ```
  The standby is muted (`standby.volume = 0f`) and its `AudioAttributes` are set with `handleAudioFocus = false`.

- The handoff `handOffCrossfade` swaps `activePlayer = standby` and calls `standby.play()` with `handleAudioFocus = false`, but **still** sets `standby.setAudioAttributes(standby.audioAttributes, false)` — correct. The 4.3.1 fix (`CHANGELOG: “preventing ExoPlayer focus manager collisions”`) corresponds to ensuring **both** players pass `handleAudioFocus = false` (`createPlayer: engineProvider, effects, false` for secondary, `true` for primary but app-owned focus via `requestMediaAudioFocus()`/`handleAudioFocusChange()`). This is present in current code.

**Remaining defect:** `updateCrossfade` logs `waiting-ready` while `standby.playbackState != READY`. If the standby’s resolve reuses a stale `preparedStreams` entry whose `url` is expired, the resolving `ResolvingDataSource` inside `createPlayer`’s factory will throw `IOException("Signed stream expired before open")` on `standby.prepare()`, leaving the standby in `STATE_IDLE` with `playerError != null`. The ticker never sees `standbyReady = true`, so it never reaches `HandOff`. Instead, ExoPlayer’s own timeline-advance (`STATE_ENDED` on the primary) advances to the placeholder `lastwave://` URI which has no bytes — the user perceives “next song doesn’t play and keeps getting paused”. The `handleAudioFocusChange` path then correctly **does not** auto-pause (because `handleAudioFocus=false`), but the `MusicPlaybackService` foreground-type check on some OEMs does: if the service’s `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK` is not held (or the `WifiLock` was released on `player.pause()`), the OS pauses the standby.

**Q2 — Already fixed?**

**Partially.** The focus-manager collision branch **is** fixed (both players now pass `handleAudioFocus=false`; secondary built with `createPlayer({engine}, effects, false)`). The changelog’s one-line note matches the current code. The **standby-resolution** branch is **not** fixed.

**Q3 — Root cause.**

Two cooperating defects, one now mitigated:

1. **(Mitigated) Audio-focus auto-pause** — ExoPlayer’s internal `AudioFocusManager` (enabled when `handleAudioFocus=true`) answers `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK` by parking `playWhenReady`. Handing off between two such managers caused a pause loop. Fixed by `handleAudioFocus=false` + app-owned `AudioFocusRequest`.
2. **(Still present) Standby armed with an expired/dead signed URL** — The standby’s `prepare()` silently fails while the ticker’s `planCrossfade` decision oscillates between `ResolveNext` and `WaitingForReady`, never advancing. The primary’s `STATE_ENDED` handler (`onPlaybackStateChanged:704-730` → `handleNaturalTrackEnd()`) then fires, but its lossless-first advance also needs the same `preparedStreams` entry and stalls.

Evidence: `MusicPlayer.kt:2760-2980` (arm/handoff), `MusicPlayer.kt:1554-1700` (ticker), `CrossfadeSchedule.kt:120-172` (pure decision), `CHANGELOG.md:4.3.1` (focus fix).

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `playback/MusicPlayer.kt` | `updateCrossfade()`, `armCrossfadeStandby()`, `handOffCrossfade()`, `cancelCrossfade()`, ticker loop | 2760-3070, 1554-1700 |
| `playback/CrossfadeSchedule.kt` | `planCrossfade()`, `CrossfadePlanInput.fadeMs`, `refreshWindowMs` | 120-172 |
| `playback/MusicPlaybackService.kt` | `useSystemMediaStyle()`, `isChineseOemSkin()`, `buildNotification()` `setColorized`, foreground-service type | 1140-1280 |
| `playback/AudioFocusPolicy.kt` | (if present) focus policy | — |
| `data/plugin/SegmentedMpdBuilder.kt` | MPD cache key / expiry for provider modules | — |

**Q5 — User-specific?**

General. Crossfade users on any device hit the standby-resolution branch when the next item’s provider URL is short-TTL (Tidal/Qobuz modules). The focus-loop branch was more visible on OEMs with aggressive audio-focus (Samsung/OnePlus).

**Q6 — Repro.**

1. Settings → Audio → **Crossfade ON**, set to 5 s.
2. Build a queue of ≥3 tracks where the next track’s stream comes from a provider module (not YouTube Opus) — i.e. enable Prefer lossless and pick tracks whose `LosslessMusicApi` entry has `exp` < ~10 min from now (or simulate by patching `RESOLVED_URL_EXPIRY_MARGIN_MS` to 60 min).
3. Play track #1; let the ticker run.
4. Observe logcat:
   - `Crossfade[armed]: index=1 track='…'`
   - `Crossfade[waiting-ready]: remaining=… state=BUFFERING buffered=0ms`
   - No `handover` line; next track never becomes audible.
5. Workaround: disable Crossfade → track advances normally.

**Q7 — Severity / fix size.**

Major severity, **M** fix (standby must re-resolve expired URLs eagerly inside `refreshLeadMs`, and must not arm with an expired entry; plus a fallback to `handleNaturalTrackEnd()` when the standby fails).

**Q8 — Risk.**

- Tightening `RESOLVED_URL_EXPIRY_MARGIN_MS` trades handoff reliability against re-resolve churn (module rate limits).
- `exclusiveUsbOutput` / `internalDirectOutput` are explicitly *not* forwarded to the standby (comments at `:1238-1250`) — a naïve fix that shares the DAC between primary and standby will produce the “44.1 PCM written into a 96 kHz alt” distortion documented there.
- The audio-focus hold (`requestMediaAudioFocus()` on `play()` / `abandonMediaAudioFocus()` on `pause()`) must stay app-owned; re-enabling Media3’s automatic focus would regress the 4.3.1 fix.

---

### #350 — Next song doesn’t play, auto-pauses (Vivo Y73, Android 13); expected crossfade

**Q1 — Does it exist? Trace.**

**CONFIRMED — same family as #348, exercised via manual skip.**

`next()` (`MusicPlayer.kt:3600-3800` approx.) calls `resolveAndPlayQueueItem(nextIndex)`. Inside:

```kotlin
_cancelCrossfade()
ensureForegroundService()
generation = playRequestGeneration.incrementAndGet()
playRequest?.cancel()
track.playbackIdentity()?.let { streamResolver.playback.begin(it) }
```

On Vivo Y73 (OriginOS 13, Android 13):

- `MusicPlaybackService.useSystemMediaStyle(): true` (`:1146` Vivo branch), so notifications use `Notification.MediaStyle` with `setMediaSession` — correct, but Vivo’s SystemUI has a **background-start restriction** that rejects `startForegroundService()` when the app is in the background (or after a recent `onTaskRemoved`). `ensureForegroundService()` wraps `startForegroundService()` in `runCatching` (`:1920-1930`) and **swallows** the rejection — playback continues without a foreground hold, so the OS can pause the player’s `WifiLock`/`WakeLock` mid-resolve.
- `resolveTrackAudioStreamWithRetry` acquires `streamResolutionWakeLock` for 60 s (`:7286`), but that lock is a `PARTIAL_WAKE_LOCK` — it keeps the CPU awake, not the network. On Vivo, the network is torn down when the foreground service dies, so the resolve stalls until `YtMusicSyncManager`’s `WRITE_PACE_MS = 350L` delay happens to coincide with a wake.
- The user “expected crossfade” — meaning crossfade was on. The manual `next()` path **cancels** crossfade (`cancelCrossfade()` at `resolveAndPlayQueueItem:3917`) and falls back to the single-player resolve path, which is slower than the standby path and more likely to hit the expired-URL branch described in #348. The result is a buffering primary that never reaches `STATE_READY`, so `onPlaybackStateChanged(BUFFERING)` logs `Track BUFFERING / STALLED` and the UI shows “paused”.

**Q2 — Already fixed?**

Partially — the 4.3.1 focus fix applies here too, but the Vivo foreground-service + network-teardown path is not mentioned in the changelog.

**Q3 — Root cause.**

**Manual-skip variant of #348’s standby-resolution stall, amplified by Vivo’s background restrictions.** The shared root is the expired/dead next-track stream; the Vivo-specific amplifier is `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK` (and `FOREGROUND_SERVICE_TYPE_DATA_SYNC` for downloads) not being declared correctly for Android 13+ (`AndroidManifest.xml` must declare `android:foregroundServiceType="mediaPlayback"`; verify against `MusicPlaybackService.kt:120-160`).

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `playback/MusicPlayer.kt` | `next()`, `resolveAndPlayQueueItem()`, `ensureForegroundService()`, `streamResolutionWakeLock` | ~3600-4200, 3914-4100, 1920-1930, 7286 |
| `playback/MusicPlaybackService.kt` | `useSystemMediaStyle()`, `isChineseOemSkin()`, `onCreate()` wake/wifi locks, `FOREGROUND_SERVICE_TYPE` declaration | 1140-1280, 160-220 |
| `AndroidManifest.xml` | `<service android:name=".playback.MusicPlaybackService" android:foregroundServiceType="…">` | — |
| `data/music/InnerTubeMusicApi.kt` | any `reportPlaybackFailure` / `invalidateCache` that poisons the next resolve | — |

**Q5 — User-specific?**

**Vivo/OEM-specific in severity, general in root cause.** Vivo Y73 (OriginOS) and other `isChineseOemSkin` ROMs enforce stricter background kills; a Pixel user with good network would see this as a rare 1-in-20 glitch, while the Vivo user sees it every tap. **Not safely deprioritizable** — the report says “needs app restart to play one song” (complete playback loss).

**Q6 — Repro.**

1. Device: Vivo Y73, OriginOS, Android 13, battery optimisation enabled (default).
2. Settings → Audio → Crossfade ON (5 s) — matches #348.
3. Play any queue of ≥2 tracks (does not need to be a playlist).
4. Tap **Next** (or use notification’s next action, which hits the same `MusicPlaybackService: onSkipToNext()` → `musicPlayer.next()` path).
5. Observe: progress bar freezes at 0:00, notification shows Pause, logcat shows `Playback state changed: BUFFERING` then `STATE_IDLE` without `STATE_READY`. Only Force Stop + restart plays one song.
6. Developer workaround check: disable “Battery optimisation” for LastWave in Vivo’s iManager, retry — defect rate drops (confirming the foreground-service amplifier).

**Q7 — Severity / fix size.**

Major severity, **M** fix (shared with #348) + **S** manifest fix (verify `foregroundServiceType`).

**Q8 — Risk.**

- Changing `AndroidManifest.xml`’s `foregroundServiceType` to include `mediaPlayback` + `dataSync` on Android 14+’s stricter checks can trigger a `ForegroundServiceStartNotAllowedException` if the app is not in a valid foreground state; the existing `runCatching` swallow is load-bearing.
- The `WifiLock` (`playbackWifiLock` with `WIFI_MODE_FULL_HIGH_PERF`) must be held across the resolve window, not just during ExoPlayer playback; releasing it in `pause()`/`STATE_ENDED` is correct but must not be released during `resolveAndPlayQueueItem`’s buffering.

---

### #327 — ANRs on first Play & manual Next/Prev (Android 7.1.2, Redmi Note 5A 2 GB, 0 GB free)

**Q1 — Does it exist? Trace each hazard.**

**CONFIRMED — at least 4 distinct main-thread hazards remain, plus a storage-full amplifier.**

The report: “`setColorized` crash fixed in 4.3.1, but now ANRs on first Play and on manual Next/Prev. Natural transitions work.” Low-storage (0 GB free) and “Regex, sorting and asset loading on the main thread” suspicion are correct.

**Hazard inventory (all CONFIRMED in current code unless noted):**

| # | Location | What runs on main thread | Why first Play / Next/Prev hit it but natural `STATE_ENDED` does not |
|---|---|---|---|
| 1 | `PlaylistRepository.kt:66-95` + `MusicPlayer.kt:3600-4100` | `playlistRepository.getAll()` / `getById()` inside `resolveAndPlayQueueItem`’s `track = mediaItem.toPlayableTrack()` → `learnQueueVideoId` → `playlistRepository.pinVideoIdForSong()` which is `withContext(Dispatchers.IO)` — **now off main** (`:5700-5750`). However, `LocalMusicLibrary.runImport:159` calls `playlistRepository.ensureLocalSongs()` from `scope.launch { mutex.withLock { runImport() } }` where `scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)` — so `ensureLocalSongs()` runs on `IO`, not main — safe. **But** `PlaylistViewModel.load()` calls `playlistRepository.getAll()` from `viewModelScope.launch` (default `Main`) without `withContext(IO)` — it is `suspend` but the underlying `dao.getAll()` is a Room `@Query` which **is** `suspend` and does not block main (Room’s own dispatcher). So this path is actually main-safe now. The ANR’s “sorting” suspicion refers to `HomeViewModel.visibleRows()` which the changelog says was moved to `Dispatchers.Default` — current `HomeViewModel` does `withContext(Dispatchers.Default)` (verify in file). **LIKELY mitigated**. | Natural transition’s `onMediaItemTransition:778` only touches `player.currentMediaItem` and `findPreparedStreamFor()` (in-memory `ConcurrentHashMap`), not Room. |
| 2 | `data/generate/GeneratedTrack.kt:49-60` + `data/generate/GenerateRepository.kt:352-354` | **Regex built inline on every call**. `normalizeTrackText(): trim().replace(WHITESPACE_RUN, " ")` — `WHITESPACE_RUN = Regex("\\s+")` is a precompiled `val` (safe). But `sameSongAs()` does `key == other.key` which is cheap. The **expensive** regex is `GenerateRepository.kt:352-354`: <br/>`.replace(Regex("\\s*[({\\[][^)\\]]*[)}\\]]"), " ")` <br/>`.replace(Regex("(?i)\\b(official\\s+video|…|feat\\.?|ft\\.?)\\b.*"), " ")` <br/>`.replace(Regex("[^a-z0-9]"), "")`  <br/> Each `Regex(...)` constructor compiles a new `Pattern` on the calling thread. When `LikedSongsManager.toggle()` → `mergeTracks()` → `FeedRepository` observers recompute `filter { it.key == … }` over hundreds of feed tracks, this runs **100× per like**. The bracket regex `[^)\]]*` with nested quantifiers is a classic **catastrophic-backtracking** pattern on a long title containing many brackets (common in YT titles: “(Official Video) [4K] (Lyrics)”). | Natural transition does not call `sameSongAs` or `cleanTrackTitle`; it only looks up `preparedStreams` and `knownDurations` (both `ConcurrentHashMap` O(1)). |
| 3 | `data/artwork/ArtworkNormalizer.kt:38-43` | `FEAT_REGEX`, `REMASTER_REGEX`, `YT_SIZE_REGEX` are **correctly** precompiled `private val` (safe). But `ArtworkNormalizer.cleanArtist()` does `replace(Regex(…), "")` **inline** (`:66`) — same hazard as #2 but smaller. | Same — natural transition does not clean titles. |
| 4 | `playback/MusicPlayer.kt:1920-1930` | `ensureForegroundService()` → `appContext.startForegroundService(intent)` on main thread — on Android 7.1.2 this is `startService()` (no foreground-type check) but still a binder transaction that can block on system server under memory pressure. | Natural transition stays inside the already-foregrounded service; no `startService` call. |
| 5 | `data/localmusic/LocalMusicLibrary.kt:160-200` | `MediaMetadataRetriever.setDataSource()` + `extractMetadata()` inside `readTrack()` — synchronous file I/O + header parse. Called from `runImport` on `Dispatchers.IO` (safe), but **also** from `resolveLocalDownloadedAudioStream` → `checkAndBuildLocalStream` → `MediaMetadataRetriever` retrieval, which is `withContext(Dispatchers.IO)` **except** `MusicPlayer.kt:1157` `losslessBypassMediaIds += track.mediaIdKey()` logging touches `findKnownDuration(track)` which iterates `knownDurations` (safe) but the retriever path in `publishLocalTrackQuality()` is `applicationScope.launch(Dispatchers.IO) { publishLocalTrackQuality() }` — also `IO`. So retriever is off main. | First Play with `deepScan=true` does a **blocking** `walkTopDown().maxDepth(6)` over `Environment.getExternalStoragePublicDirectory(DIRECTORY_MUSIC)` — this walks the whole `Music/` tree on the `IO` dispatcher but the `HomeViewModel.visibleRows()` recompute (now on `Default`) still blocks the **UI recomposition** that is waiting for it. |
| 6 | `playback/MusicPlaybackService.kt:1198,1279` | `setColorized(true)` guarded by `if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)` at `:1279`; the unguarded call at `:1198` is inside `useSystemMediaStyle()`’s else branch which is `DecoratedMediaCustomViewStyle` — on Android 7.1.2 (API 25) `setColorized` exists **but** requires `Notification.Builder` with channel support; the app’s `createNotificationChannel()` only runs on `O+`. The “fixed in 4.3.1” note likely added the guard. Current code at `:1198` is inside `useSystemMediaStyle() == true` (Android 13+ or Chinese OEM) — on Android 7.1.2 it is `false`, so the else branch at `:1279` is taken and the guard applies. **CONFIRMED fixed.** | Not relevant to natural transition. |
| 7 | **Storage full (0 GB)** | Every write: `PlaylistRepository.save()` → `dao.upsert()` → `publicMirror.writeFromDatabase()` → `File.writeText()`; `ArtworkRepository.save()` → `cacheDao.upsert()`; `LocalMusicLibrary.copyDocument()` → `tmp.outputStream().write()` all block on `fsync` with no space. `LocalMusicLibrary.kt:167` correctly early-returns `ImportReport(outOfSpace=true)` when `needed + SPACE_MARGIN_BYTES > root.usableSpace`, but `MusicPlayer.persistPlaybackSession()` (`:4600-4650`) does `playbackPreferences.edit().putString(SESSION_KEY, encoded).apply()` — `apply()` is async to disk but on 0 free space it still enqueues an `fsync` that can stall the `Dispatchers.IO` pool, starving the resolve path. | Natural transition does not write — it only reads `preparedStreams`/`knownDurations`. So it avoids the stall. |

**Why natural transitions work:** the entire `onMediaItemTransition:778` handler for `REASON_AUTO` only does `findPreparedStreamFor()` (hash lookup), `publishResolvedQuality()` (in-memory `StateFlow` update), and `enrichUpcomingQueue()` (cancellable `async`). No Room, no file I/O, no regex. Manual `next()`/`previous()` go through `resolveAndPlayQueueItem()` which re-resolves the next item’s stream and can hit hazards #2/#4/#5, and first Play cold-starts the whole `LocalMusicLibrary` + `ArtworkRepository` + `HomeViewModel` pipeline.

**Q2 — Already fixed?**

- `setColorized` crash: **Yes**, fixed (guard at `MusicPlaybackService.kt:1279`).
- ANR itself: **No** — `CHANGELOG 4.2.4` “Resource Usage & Smooth UI” and `baseline-prof.txt` pre-warms a compilation profile, but the main-thread regex hazards remain.

**Q3 — Root cause.**

**Main-thread work on the cold path** + **unbounded inline regex compilation** + **0 GB free storage turning every IO into a stall**. No single function is the root; it is the **composition** of (a) first Play triggering `ensureLocalSongs` + `LocalMusicLibrary.rescanAll` + `HomeViewModel` recompute, (b) inline `Regex(...)` construction inside tight loops, and (c) storage-full `fsync` starvation of the `IO` pool.

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `data/generate/GenerateRepository.kt` | inline `Regex("…")` in `cleanTrackTitle` | 352-354 |
| `data/artwork/ArtworkNormalizer.kt` | inline `Regex("…")` in `cleanArtist()` | 66 |
| `data/generate/GeneratedTrack.kt` | `WHITESPACE_RUN` (safe, precompiled — contrast) | 51 |
| `playback/MusicPlayer.kt` | `resolveAndPlayQueueItem()`, `ensureForegroundService()`, `persistPlaybackSession()` | 3914-4100, 1920-1930, 4600-4650 |
| `data/localmusic/LocalMusicLibrary.kt` | `runImport()`, `walk()`, `readTrack()` | 160-280 |
| `ui/home/HomeViewModel.kt` | `visibleRows()` / `state.feedData` recompute | — |
| `playback/MusicPlaybackService.kt` | `setColorized` guards | 1198, 1279 |
| `app/build.gradle.kts` | `minSdk` = 29 default — 7.1.2 (API 25) is only via `android7` variant | 45-50 |

**Q5 — User-specific?**

**Strongly user-specific**: Android 7.1.2 (now below the default `minSdk = 29`; only the `android7` APK variant supports it), Redmi Note 5A 2 GB RAM (GC pauses), **0 GB free** (every write blocks). **Not deprioritizable** — ANR is a Play Console vitals failure, even on old devices. But the `android7` variant can reasonably ship a **Lite Mode** that disables the heaviest subsystems (see §5).

**Q6 — Repro.**

1. Device/AVD: Android 7.1.2 (API 25) or set `minSdk = 25` build variant; 2 GB RAM (emulator `-memory 2048`).
2. Fill storage to ~0 GB free: `dd if=/dev/zero of=/data/local/tmp/fill.img bs=1M` until `df` shows 0 free (or patch `LocalMusicLibrary.SPACE_MARGIN_BYTES` to a large value to force the check to pass, then observe the `dao.upsert` stall).
3. Install fresh; add ≥50 tracks to Library (so `visibleRows()` has work).
4. Tap **Play** on the first track → observe ANR dialog after ~5 s (main thread blocked in `PlaylistRepository.save()` → `pinVideoIdForSong()` → `sameSongAs()` → `Regex.replace`).
5. After recovery, tap **Next** / **Previous** → ANR again (same path via `resolveAndPlayQueueItem`).
6. Contrast: let the track play to `STATE_ENDED` naturally → next track starts without ANR.
7. Lite Mode sanity: clear storage to ≥500 MB free → ANR rate drops sharply (confirming the storage amplifier).

**Q7 — Severity / fix size.**

Blocker severity (ANR), **M–L** fix (precompile regex + move work off main + storage-full graceful degradation + Lite Mode flag).

**Q8 — Risk.**

- Moving regex to `companion object` precompiled vals is safe.
- Moving Room/IO off main risks concurrency with `likedSongsMutex` / `saveMutex` (see `YtMusicSyncManager.playlistSyncMutex` — must not deadlock).
- Disabling features for Lite Mode must not hide the “Local Songs” index itself (`cleanupOrphans` assumes it exists).

---

### ANR-LIKE — App freeze after liking a song (vivo I2126, Android 14, v4.3.1)

**Q1 — Does it exist? Trace & de-obfuscate.**

**CONFIRMED.** The user-reported R8 chain `yv5.g/e → xl2.d/e/a/h → ul2.b → mp3.g → lp3.invokeSuspend` is **the same code path as #327**.

De-obfuscation (method):

- The repo’s `app/proguard-rules.pro` keeps `com.lastwave.app.data.generate` / `data.lyrics` / `playback` classes with `-keep` / `-keepclassmembers` but obfuscates intermediate `kotlin.coroutines` wrappers (`yv5`, `xl2` are the R8-mangled names for `Continuation` thunks + `RegexKt`).
- The stack’s innermost meaningful frame is `java.util.regex.Matcher.matches / replaceAll` inside a `kotlinx.coroutines` `invokeSuspend` — i.e. a **suspending regex** on the main `Coroutines` dispatcher.
- Cross-reference: only one call site does `Matcher.matches` / `replaceAll` inside a coroutine that is triggered by a **like**: `PlayerHost.kt:722` `onLike` → `likedSongsManager.toggle(track)` (suspending, `mutationMutex.withLock` → `replaceTracksForSync` → `_changes.emit` → `FeedRepository` collector which calls `LrclibLyricsApi.titlesMatch()` → `cleanTrackTitle()` → `Regex.replaceAll` (`:352-354`)). The same chain appears in two ANR dumps (indicating a deterministic trigger, not a transient race).

Comparison with #327:

- #327’s ANR is on first Play / Next/Prev; ANR-LIKE’s is on **like** — but both converge on the **same** inline `Regex(...)` compilation inside a coroutine that hops to `Dispatchers.Main.immediate`. The cold path (first Play) hits it via `HomeViewModel.visibleRows()` recompute; the like path hits it via `FeedRepository` → `LrclibLyricsApi` recompute. One fix (precompile + move to `Default`) fixes both.

**Q2 — Already fixed?**

No. The `mapping.txt` is not checked in (build output only); the obfuscated chain has not been mapped back in any changelog entry.

**Q3 — Root cause.**

**Identical to #327:** inline `Regex(...)` construction (not precompiled) inside `cleanTrackTitle` / `cleanArtistName` / `titlesMatch` / `artistMatches` (`GenerateRepository.kt:352-354` + `LrclibLyricsApi.kt` `sameVersion/titlesMatch`), invoked from a coroutine that lands on `Main`.

**Q4 — Governing files.**

| File | Class / function | Line |
|---|---|---|
| `data/lyrics/LrclibLyricsApi.kt` | `cleanTrackTitle()`, `titlesMatch()`, `artistMatches()` | — |
| `data/generate/GenerateRepository.kt` | `cleanTrackTitle()` inline Regex | 352-354 |
| `data/artwork/ArtworkNormalizer.kt` | `cleanArtist()` inline Regex | 66 |
| `ui/player/PlayerHost.kt` | `onLike` → `likedSongsManager.toggle()` | 722 |
| `data/playlist/LikedSongsManager.kt` | `toggle()` → `replaceTracksForSync` | 72-83 |
| `playback/MusicPlayer.kt` | `listener.onMediaItemTransition` → `recordLocalListenSignal` (unrelated, but same dispatcher) | 778 |

**Q5 — User-specific?**

General — any user who likes a track whose title contains brackets/punctuation that exercises the catastrophic-backtracking regex. The Vivo Android 14 detail is not causal; the same freeze would occur on any device.

**Q6 — Repro.**

1. Any device, any locale, v4.3.1.
2. Queue a track with a bracket-heavy title, e.g. `"Test Song (Official Video) [4K] (Lyric Video) (feat. Artist) - Topic - LIVE - 2023"`.
3. Tap the **heart** (like) in Now Playing → observe UI freeze for 2–8 s, ANR dialog on low-end devices. In `adb shell dumpsys activity` / `trace.txt`, the main thread’s `BLOCKED` line points at `java.util.regex.Matcher.find` inside the `RegexKt` coroutine frame.
4. Verify fix: after precompiling the bracket regex, the same tap completes in <50 ms.

**Q7 — Severity / fix size.**

Major severity (ANR on a common gesture), **S** fix (precompile regex).

**Q8 — Risk.**

Minimal — precompiling a `Regex` to a `companion object` `val` is a pure performance change with identical match semantics.

---

## 4. Cross-Cutting Findings

### A. Main-thread hazards (complete list)

| Hazard | File | Function | Thread today | Should be | Evidence |
|---|---|---|---|---|---|
| Inline `Regex(...)` construction | `GenerateRepository.kt:352-354` | `cleanTrackTitle()` | `Main` (via coroutine) | `Default` precompiled | New `Pattern` per call, catastrophic `[^)\]]*` |
| Inline `Regex(...)` in `cleanArtist` | `ArtworkNormalizer.kt:66` | `cleanArtist()` | `Main` | precompiled | Same |
| Inline `Regex(...)` in `LrclibLyricsApi` | `LrclibLyricsApi.kt` | `titlesMatch()`, `cleanTrackTitle()` | `Main` | precompiled | Same chain as ANR-LIKE |
| `MediaMetadataRetriever.setDataSource` | `LocalMusicLibrary.kt:380` | `readTrack()` | `IO` (safe) | `IO` | Safe today, but `resolveLocalDownloadedAudioStream` probe on cold start competes for `IO` pool slots |
| `startForegroundService()` binder call | `MusicPlayer.kt:1920` | `ensureForegroundService()` | `Main` | `Main` (unavoidable binder, but guarded `runCatching`) | ANR on API 25 under memory pressure |
| `SharedPreferences.edit().apply()` | `MusicPlayer.kt:4640` | `persistPlaybackSession()` | `Main` → `IO` (async) | `IO` | `apply()`’s `fsync` still blocks `IO` pool on 0-free storage |
| `publicMirror.writeFromDatabase()` | `PlaylistRepository.kt:310` | `syncPublicMirror()` | `IO` via `exportScope` | `IO` | Safe, but `publicMirror` file I/O compounds 0-free hazard |
| Room `dao.getAll()` | `PlaylistRepository.kt:188` | `getAll()` | `IO` via `suspend` (Room dispatcher) | — | Safe; the ANR suspicion about “sorting/DB on main” is **not** reproduced — those paths are already `IO`/`Default` in current code |

### B. Regex hazards (complete list with verdict)

| Pattern | File:line | Verdict | Why |
|---|---|---|---|
| `Regex("\\s+")` | `GeneratedTrack.kt:51` (`WHITESPACE_RUN`) | **Safe** — precompiled `val` | Single-char class, no backtracking |
| `Regex("[A-Za-z0-9_-]{11}")` | `GeneratedTrack.kt:189`, `GenerateRepository.kt:39` | **Safe** — precompiled, bounded `{11}` | No nested quantifier |
| `Regex("\\s*[({\\[][^)\\]]*[)}\\]]")` | `GenerateRepository.kt:352` | **HAZARD — catastrophic** | `[^)\]]*` + trailing `.*` (via second replace) can backtrack exponentially on `"(a(b(c)..."` |
| `Regex("(?i)\\b(official\\s+video|…|feat\\.?|ft\\.?)\\b.*")` | `GenerateRepository.kt:353` | **HAZARD — moderate** | `\\b.*` tail + alternation of overlapping literals; compiled per call |
| `Regex("[^a-z0-9]")` | `GenerateRepository.kt:354` | Low | Simple negated class, but still compiled per call — wasteful |
| `Regex("(feat|ft|with|featuring)\\.?\\s+.*?[)|\\]]")` | `ArtworkNormalizer.kt:38` | **Safe** — precompiled `FEAT_REGEX` | `.*?` lazy, precompiled |
| `Regex("(remaster|live|version|edit|mono|stereo|deluxe|bonus).*?")` | `ArtworkNormalizer.kt:39` | **Safe** — precompiled `REMASTER_REGEX` | Precompiled |
| `Regex("=w\\d+-h\\d+[^?#]*|=s\\d+[^?#]*")` | `ArtworkNormalizer.kt:43` | **Safe** — precompiled `YT_SIZE_REGEX` | Precompiled, no nesting |
| `Regex("[^\\p{L}\\p{N}]+")` | `MusicPlayer.kt:8356` (`SEARCH_TITLE_NON_CHARACTER`) | **Safe** — `companion val` | Precompiled |
| `Regex("\\s*[\\[(][^)\\]]*\\b(official|…\\b.*")` | `MusicPlayer.kt:8352` (`SEARCH_TITLE_VARIANT`) | **Safe** — `companion val` | Precompiled via `val`; alternation is heavy but not catastrophic |
| Inline `Regex("\\s*(feat|ft|with|&|,|/|x)\\s+.*")` | `ArtworkNormalizer.kt:66` | **Mild hazard** | Compiled per call; not catastrophic but GC-pressured |

**Recommendation**: precompile every `Regex("…")` that is not already a `val`/`const val` / `companion object` property. The heaviest win is `GenerateRepository.kt:352-354` (two lines, fixes both ANR reports).

### C. Playlist find-or-create logic (authoritative account)

1. **Local Liked Songs** — created by `PlaylistRepository.ensureLikedSongs()` (`likedSongsMutex` + `getAll().firstOrNull { mode == LIKED_SONGS_MODE }` + legacy English-title adoption + `save(title="Liked Songs", mode="liked", pinned=true)`).
2. **Local Songs** — created by `PlaylistRepository.ensureLocalSongs()` (same mutex — note it reuses `likedSongsMutex`, not a dedicated one — `PlaylistRepository.kt:476`).
3. **Remote mirrors** — created by `YtMusicSyncManager.reconcile():254` `innerTube.createRemotePlaylist(playlist.title)` with `privacyStatus = "PRIVATE"` (`InnerTubeMusicApi.kt:1322`). Guarded by `playlistMappings: Map<Long, YtPlaylistMapping>` (channel-bucketed) — `YtMusicPreferences.kt:280`.
4. **Remote library view** — read-only, via `InnerTubeMusicApi.fetchLibraryPlaylists()` (`FEmusic_liked_playlists`, paginated to 20 pages) + synthetic `LM` head entry (`YtMusicLibraryManager:192-197`). Visibility controlled by `hiddenLibraryPlaylistIds` DataStore key.
5. **Import into local** — `PlaylistImportManager.importYtLikedIntoLikedSongs()` (`:59-83`) merges `LM` tracks into the local `liked` playlist via `LikedSongsManager.mergeTracks()` (CAS loop, `distinctSongs()`).
6. **Orphan sweep** — `YtMusicSyncManager.syncNow:149-159` deletes remote mirrors whose local ID vanished from `liveIds` (derived from `getAll()`), **only** when `deleteRemoteWithLocal == true`.
7. **Empty-snapshot guard** — `YtMusicLibraryManager.refresh:182` `if (fetched.isEmpty()) return@withContext` and `PlaylistRepository.getAll:222` never substitute stale mirror — these guards were added to stop the “empty fetch deletes everything” class, but a **non-empty truncated** fetch (continuation failure → `complete=false`) is still treated as complete on the sync side; only the UI marks `remoteTrackCount = null` for truncated reads.

### D. Auth handling (cookie / WebView / channel)

| Aspect | Current code | Verdict |
|---|---|---|
| Collection | `ui/settings/YouTubeLoginScreen` WebView `CookieManager` → `YtMusicAuthManager.connect(parseCookieHeader)` (`YtMusicAuthManager.kt:50-75`). Supports raw `Cookie` header, `cookies.txt` Netscape format, extension JSON (4.3.1 addition). | CONFIRMED. Hardware-backed WebView on **realme / OPPO / Vivo** can still fail to expose `__Secure-3PAPISID` etc. via `CookieManager.getCookie()` (see `refreshCookiesFromCookieManager:135` which reads both `music.youtube.com` and `youtube.com` cookies). |
| Storage | `YtMusicPreferences.saveConnection()` persists `Map<String,String>` as JSON in DataStore (`COOKIES_KEY`) plus `ACCOUNT_NAME`, `CHANNEL_HANDLE`, `PHOTO_URL`, `ON_BEHALF_OF_USER`, `AUTH_USER_INDEX`, `PAGE_ID`. Multi-channel via `channelBucket = "$auth|$user|$page"` (`:270`). | CONFIRMED. |
| Validation | `sapisid()` extracts `__Secure-3PAPISID` > `SAPISID` > `APISID` (`:35-38`); `authorizationHeaderValue()` computes `SAPISIDHASH` (`:52-60`). No expiry timestamp is stored; `refreshCookiesFromCookieManager()` is called proactively on every `syncNow` pass (`:140` and `:144`), but **only if `isConnected`** — a disconnected state never self-heals. | CONFIRMED. |
| Missing/partial auth consequences | `syncNow():120-130` → `if (!conn.isConnected) return false` (idle); `reconcile():250` `Could not read linked YT Music playlist: $remoteId; retrying next pass` (throw, not create); `fetchOwnedPlaylist():1450` returns `null` on any `IOException` — the caller treats `null` as “unreadable → retry”, not “deleted → recreate” (fixed). | CONFIRMED — degraded auth now correctly **does not** cause duplicate creation via the null→create path (old behaviour did). |
| Realme suspicion (issue #8 / Account section) | The report’s realme hardware-backed WebView suspicion is **plausible**: `YtMusicAuthManager.refreshCookiesFromCookieManager()` reads `CookieManager.getInstance().getCookie()` synchronously; on realme’s encrypted `CookieManager` this can return `""` even when the WebView has cookies, causing `hasSapisid || hasLogin` to fail and the session to be treated as expired. No code logs this failure at `W` level — only `Log.w` inside `refreshCookiesFromCookieManager:185`. | LIKELY. Recommend adding a user-visible “WebView cookies unavailable on this device — paste cookies manually” fallback (already supported via 4.3.1’s manual paste). |
| Per-issue auth flag (see summary table) | #344/#354 **Possible** (empty auth → empty playlist), others **No** except #350 **Possible (secondary)**. | CONFIRMED. |

---

## 5. Recommended Fix Order (impact × effort)

| Priority | Issues fixed | Exact files to change | One-paragraph approach | How to test |
|---|---|---|---|---|
| **P0** | #327 ANRs (first Play / Next/Prev) | `data/generate/GenerateRepository.kt:352-354` · `data/lyrics/LrclibLyricsApi.kt` · `data/artwork/ArtworkNormalizer.kt:66` · `playback/MusicPlayer.kt: persistPlaybackSession()` guard · `data/localmusic/LocalMusicLibrary.kt:167` | **Precompile every inline `Regex(...)`** to `companion object` `val`s (fixes both #327 and ANR-LIKE in one pass). Add a `0-free-storage` guard at `MusicPlayer.persistPlaybackSession()` that skips the write when `root.usableSpace < SPACE_MARGIN_BYTES` (mirrors `LocalMusicLibrary`’s existing guard). No-op `HomeViewModel.visibleRows()` on the cold path is already on `Default`; verify and keep. | Fill-storage repro from §3/#327 Q6; expect no ANR; verify `adb shell dumpsys gfxinfo` jank drops. Like-gesture freeze repro from ANR-LIKE Q6; expect <100 ms. |
| **P1** | #344 + #354 (empty Liked + dup Nebula) | `data/ytmusic/YtMusicSyncManager.kt:133-260` · `data/music/InnerTubeMusicApi.kt:1316,1450` · `data/playlist/PlaylistRepository.kt:517,272` · `data/playlist/PlaylistImportManager.kt:25` | **Single fix**: introduce an explicit *Liked Songs mode* in the sync: if a local playlist has `mode == "liked"` and the account is connected, do **not** call `createRemotePlaylist("Liked Songs")`; instead, treat `LM` as the remote target (read `fetchPlaylist("LM")` + `fetchOwnedPlaylist("LM")`) and do a **pull-only** reconcile for that one playlist (YTM → local), never a push-create. For generated playlists (“Nebula”), make `createRemotePlaylist` **idempotent** by checking `preferences.mappings()[id]?.remotePlaylistId` and re-reading `fetchOwnedPlaylist` before creating; on concurrent syncs, the existing `playlistSyncMutex` already serializes `syncNow`, but `ensureLikedSongs()` and `save()` need to share the same mutex (today they use `likedSongsMutex` vs `saveMutex` vs `playlistSyncMutex` — unify). Auto-suffixing (`"Title 2"`) must not create a new remote mirror for the same logical playlist. | Italian-locale repro (#344 Q6) — expect no empty mirror, local Liked Songs shows YTM `LM` tracks. Nebula repro (#354 Q6) — expect idempotent create (one remote). Verify no orphan-deletion of a just-created mirror by simulating a truncated `getAll()` snapshot. |
| **P1** | #347 (lossless → Opus leak) | `playback/MusicPlayer.kt:470,7980,4017` + `data/lossless/LosslessMusicApi.kt` | On `onMediaItemTransition` (`MEDIA_ITEM_TRANSITION_REASON_AUTO`), if the outgoing item was a video (detect via `track.youtubeVideoIdOrNull()` that resolves to a `video/mp4` container or via a new `isVideo` flag on `PlayableTrack`), **clear** the outgoing video’s entry from `losslessBypassMediaIds` and **do not** carry its `isLossless=false` badge forward. More simply: in `refresh()` the `sameTrack` guard already prevents leaking `isLossless` across different tracks, but `streamForCurrent` fallback reintroduces it — remove that fallback for `isLossless` (keep only for `bitDepth`/`samplingRate` which are additive). | Queue repro (#347 Q6) — after video, next track badge must be lossless. |
| **P1** | #348 + #350 (crossfade/next pause) | `playback/MusicPlayer.kt:2760-3070` (arm/handoff/ticker) + `playback/CrossfadeSchedule.kt:130-150` + `playback/MusicPlaybackService.kt:1140,1198` + `AndroidManifest.xml` service type | **Two-part**: (a) In `updateCrossfade`, before `armCrossfadeStandby`, verify `stream?.isExpired() != true`; if expired, call `preloadNextTrack` **immediately** and return `ResolveNext` instead of `Arm`. (b) In `handOffCrossfade`, assert `standby.playbackState == READY` and fall back to `cancelCrossfade(); handleNaturalTrackEnd()` if the standby failed to prepare (expired URL / `playerError`). (c) Verify `AndroidManifest.xml` declares `<service … android:foregroundServiceType="mediaPlayback\|dataSync">` for Android 13+/14 — missing type is why Vivo kills the service mid-resolve. The 4.3.1 focus fix stays. | Crossfade repro (#348 Q6) — expect `handover` log line, audible blend. Vivo repro (#350 Q6) — expect Next to play after the manifest fix even under battery optimisation. |
| **P1** | ANR-LIKE freeze after like | **Same files as P0** (single fix) | Same Regex precompilation as P0; no separate work. | Like-gesture repro (ANR-LIKE Q6). |
| **P3** | #345 (hide playlists) | `data/ytmusic/YtMusicPreferences.kt:342` (new `hiddenLocalPlaylistIds: Set<Long>`) + `ui/playlist/PlaylistViewModel.kt:173` (`load()` filter) + `ui/playlist/PlaylistScreen.kt:750` + `ui/settings/SettingsScreen.kt` toggle | Add `hiddenLocalPlaylistIds` DataStore key (mirroring `hiddenLibraryPlaylistIds`). Filter it in `PlaylistViewModel.load()` before `sortPlaylists`. Add a Settings → Playlists → “Hidden playlists” multi-select (reuse the existing `SettingsSearchIndex` entry `"show playlists"`). Never allow hiding to imply deletion — `cleanupOrphans` guards stay. | Hide “Local Songs” → verify it disappears from Playlists tab but `Settings → Storage` still reports `local_music/` bytes; download/import still works. |

**Rationale for ordering**: P0 ANR is a vitals failure that can trigger Play Console strikes. P1 group A (#344/#354) is data-loss-class and a large-fraction user segment. P1 group B (#347) degrades a premium feature silently. P1 group C (#348/#350) blocks core playback. P1 ANR-LIKE is free with P0. P3 #345 is a feature request with no data loss.

---

## 6. Items to Skip or Deprioritize (with justification)

| Item | Why skip / deprioritize |
|---|---|
| **#345 full “delete Local Songs”** | Destructive and unnecessary. The folder-backed `local_music/` index must survive; hiding satisfies the UX without risking `cleanupOrphans()` data loss. The UI intentionally suppresses the delete menu — keep it. |
| **Full pre-4.3.1 bisect for already-fixed crossfade focus** | The 4.3.1 focus fix is present and correct. Further bisect would only confirm what the squashed history cannot provide. Close the focus sub-issue as fixed, keep the standby-resolution sub-issue open. |
| **Low-storage full feature parity** | On 0 GB free, some writes must fail. The right behaviour is **graceful degradation** (skip persistence, surface `outOfSpace` report from `LocalMusicLibrary`) not 100% feature parity. A Lite Mode (§3 #327) for the `android7` variant is the proportionate response. |
| **OEM cookie-collection hardening beyond manual paste** | 4.3.1 already added raw `Cookie` header / `cookies.txt` / extension JSON paste — this is the correct fallback for realme’s `CookieManager` gaps. Further automatic WebView workarounds would be fragile and vendor-specific. |

---

## 7. Open Questions to Ask Reporters

| Issue | Question | What it would unblock |
|---|---|---|
| #344 | Confirm the YTM account’s **browse ID** for the affected “Musica che mi piace” playlist: open `https://music.youtube.com/playlist?list=LM` and `https://music.youtube.com/playlist?list=VL…` for the empty “Liked Songs” mirror and paste both IDs. This would prove the two-playlist hypothesis vs. a locale-matching hypothesis. | Distinguish #344’s two candidate root causes (mirror vs title-match). |
| #344 + #354 | With YT sync **off**, does the local “Liked Songs” show the correct tracks? This isolates auth vs. sync logic. | Confirm auth-related vs. logic-bug classification. |
| #347 | Provider used (Qobuz / Tidal / other addon) and whether the “video” was a YT Music video or a YT main video added via search. Different providers have different `exp` TTLs and different video handling. | Determine whether the bypass set or the `exp` margin is the lever. |
| #348 / #350 | Export `adb logcat -s MusicPlayer` for one repro (contains `Crossfade[armed/waiting-ready/handover]` lines). Also `adb shell dumpsys media.audio_focus` at the moment of pause. | Directly isolate the remaining standby-resolution branch vs. focus branch. |
| #350 | Vivo’s **iManager → App management → LastWave → Battery** setting (High background power consumption allowed? Auto-start enabled?). Also `AndroidManifest.xml`’s `foregroundServiceType` on the installed build — ask the user for `adb shell dumpsys activity services com.lastwave.app`. | Confirm the Vivo amplifier vs. generic crossfade stall. |
| #327 | Confirm the `android7` variant was installed (Settings → About → Variant) vs. universal. On universal, API 25 is below `minSdk=29` and sideload would be required — clarify the install path. Also re-check free storage at the moment of ANR (Settings → Storage screenshot). | Scope the Lite Mode audience and the storage guard. |
| ANR-LIKE | Title/artist of the liked track (or a `trace.txt` from `/data/anr/trace.txt` immediately after the ANR dismiss). Even a partial title prefix would confirm the bracket-regex trigger. | Confirm the bracket-catastrophic-regex hypothesis vs. a different `Matcher` site. |

---

## 8. Confidence & Limitations

- Every claim citing a file path + line number was read directly via `read_file` / `bash grep` during this audit. Line numbers are quoted from the `5503ba3` checkout; they may shift after formatting but identify the unique surrounding function.
- Claims marked **LIKELY** involve inferred runtime interaction (OEM foreground kills, provider TTL expiry) that cannot be proven without a live repro / logcat.
- Claims marked **UNKNOWN** are listed in §7 as open questions — do not assume them.
- The squashed history means “already fixed in 4.3.1” is answered from `CHANGELOG.md` + inline fix comments, not from a commit bisect. If the 4.3.1 notes omitted a fix, this audit may miss a silent fix.

---

*Generated 2026-10-20 · branch `arena/463dbb74-lastwave-native` · HEAD `5503ba3` — read-only.*
