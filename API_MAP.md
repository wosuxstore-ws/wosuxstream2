# API_MAP (upload this file with every phase; the AI must return it updated)

Project: Wosux Stream (currently still named FreeTV in code). Live TV app, M3U playlists from iptv-org or a custom link.
All code: package `com.freetv.app`. ViewBinding ON. Theme: AppCompat (NO Material components). No Compose.
Paths: `K` = `app/src/main/java/com/freetv/app`, `R` = `app/src/main/res`

## Build facts
- Kotlin 1.9.24, AGP 8.5.2, Gradle 8.7 (installed by the GitHub Actions workflow, no wrapper in repo), Java 17
- compileSdk 34, targetSdk 34, minSdk 21
- namespace `com.freetv.app` (code package, do not change). applicationId `com.wosux.stream`; debug build adds suffix `.test` (installs as `com.wosux.stream.test`)
- Built only by GitHub Actions (Actions tab > Build APK > Run workflow, or by pushing a tag `v*`; pushing to a branch does NOT build). Release is signed with `app/freetv.keystore`.
- CI outputs: `Wosux-Stream.apk` (release) and `Wosux-Stream-test.apk` (debug), uploaded as artifact `Wosux-Stream-apk`; a `v*` tag also attaches both to a GitHub Release.
- Dependencies: core-ktx 1.13.1, appcompat 1.7.0, recyclerview 1.3.2, activity-ktx 1.9.1, lifecycle (viewmodel, livedata, runtime) 2.8.4,
  kotlinx-coroutines-android 1.8.1, media3 1.4.1 (exoplayer, exoplayer-hls, exoplayer-dash, ui), coil 2.7.0, core-splashscreen 1.0.1, okhttp 4.12.0, media3-datasource-okhttp 1.4.1

## K/Models.kt
```kotlin
data class Channel(val name: String, val url: String, val logo: String? = null, val group: String? = null,
                   val referrer: String? = null, val userAgent: String? = null)
data class Source(val name: String, val url: String)
object ChannelStore { var playlist: List<Channel>; var index: Int }   // hand-off list from the list screen to PlayerActivity
object Sources { val ALL: Source; val categories: List<Source>; val languages: List<Source>; val countries: List<Source> }
```

## K/Prefs.kt (object; `Prefs.init(ctx)` is called in FreeTVApp.onCreate; SharedPreferences file "freetv")
```kotlin
var source: Source            // selected source, persisted
var customUrl: String
fun favorites(): List<Channel>
fun isFavorite(url: String): Boolean
fun toggleFavorite(c: Channel): Boolean   // true = now a favorite, false = removed
```
Favorites are stored as JSON in key "favs" (fields n, u, l, g, r, a), cached in memory after first load.

## K/M3uParser.kt
```kotlin
object M3uParser { fun parse(reader: BufferedReader): List<Channel> }   // dedupes by url; reads #EXTINF attrs + #EXTVLCOPT referrer/user-agent (no Regex since Phase 13)
```

## K/MainViewModel.kt (AndroidViewModel)
```kotlin
val channels: MutableLiveData<List<Channel>>
val loading: MutableLiveData<Boolean>
val message: MutableLiveData<String?>
val source: MutableLiveData<Source>
var query: String
var favoritesOnly: Boolean
fun setSource(s: Source)
fun load(force: Boolean)   // shows cached playlist first, refreshes if cache older than 6h or force; cache file filesDir/pl_<hash>.m3u (+ ETag sidecar pl_<hash>.m3u.etag)
```
Private: cacheFile(), parseFile(), download(urlStr: String, dest: File): Boolean (OkHttp via Net.client + ETag sidecar; true = new file saved, false = 304 unchanged).

## K/ChannelAdapter.kt
```kotlin
class ChannelAdapter(isFavorite: (Channel) -> Boolean,
                     onClick: (Channel, Int) -> Unit,
                     onLongClick: (Channel, Int) -> Unit) : ListAdapter<Channel, ChannelAdapter.VH>
class VH(val b: ItemChannelBinding)   // focus: scales 1.06 + translationZ; logo via Coil .load(url) with ic_tv placeholder/error (served by the app-wide loader from FreeTVApp.newImageLoader(), Phase 08)
```

## K/FreeTVApp.kt
`class FreeTVApp : Application, coil.ImageLoaderFactory` : onCreate calls `CrashLog.install(this)`, then `Prefs.init(this)`, then `HealthStore.init(this)` (Phase 10).
`override fun newImageLoader(): ImageLoader` : the app-wide Coil loader (shared `Net.client`, memory cache 25%, disk cache `cacheDir/img` 100 MB,
`respectCacheHeaders(false)`, `crossfade(120)`). Coil calls it once, on the first image request. Details in Phase 08.

## K/MainActivity.kt (grid screen)
Grid of channels (span = screen dp / 150, clamp 2..10), buttons Source / Search / Favorites / Refresh, dialogs (source menu, country/category/language lists,
search text, custom M3U link). Click = set ChannelStore.playlist/index then start PlayerActivity. onResume after player: refresh favorites + restore focus.
onCreate calls `installSplashScreen()` as its FIRST line (before `super.onCreate`) and `CrashLog.showIfAny(this)` right after setContentView (keep both lines if the screen is ever moved).

## K/PlayerActivity.kt (full-screen live player)
ExoPlayer (Media3) created in onStart, released in onStop. `playCurrent()`, `zap(delta)`, overlay auto-hide 5 s, one retry on error,
BEHIND_LIVE_WINDOW handling, remote keys (UP/CH+ next, DOWN/CH- previous, OK toggles info, LEFT/RIGHT show overlay), hides system bars, keeps screen on.
Reads ChannelStore.playlist/index. Data source: `OkHttpDataSource.Factory(Net.client)` with the channel's own user-agent (fallback `DEFAULT_UA`, a Chrome-on-Android string) and `Referer` (Phase 09).
The player is built with a custom `DefaultLoadControl`, `DefaultRenderersFactory` (decoder fallback), audio focus and pause-on-unplug (Phase 09).
Since Phase 12 the player reports to `HealthStore` (READY = alive, final error = failed) and `zap(delta)` skips channels that are DEAD (`SKIP_DEAD`).

## Layout ids
- activity_main: progress, count, btnSource, btnSearch, btnFav, btnRefresh, list, empty
- item_channel: logo, star, name
- activity_player: playerView, touchLayer, buffering, error, overlay, chName, chInfo, btnPrev, btnFav, btnNext

## Resources
- colors: bg, surface, surface_focus, accent, text, text_dim, logo_bg, brand_blue, brand_blue_mid, brand_cyan, silver, silver_dim, live_red, ok_green, dead_gray, scrim, ic_launcher_background
- styles: Theme.FreeTV (AppCompat.NoActionBar), Theme.FreeTV.Player, Theme.Wosux.Starting (parent Theme.SplashScreen; splash theme of MainActivity), Btn
- drawables: bg_button, bg_card, bg_logo, bg_overlay, ic_tv, banner (drawable-xhdpi)
- mipmaps: ic_launcher (PNG in mipmap-mdpi/hdpi/xhdpi/xxhdpi/xxxhdpi + adaptive XML in mipmap-anydpi-v26), ic_launcher_foreground (PNG in the same five density folders; also the splash icon),
  ic_launcher_round (adaptive XML in mipmap-anydpi-v26 only)
- strings: app_name, source_btn, search, search_active, favorites_off, favorites_on, refresh, channels_count, empty_list, empty_favorites,
  fav_added, fav_removed, choose_source, src_all, src_country, src_category, src_language, src_custom, search_title, search_hint,
  custom_hint, custom_name, custom_invalid, load, clear, err_empty, err_refresh, err_load, prev, next, fav_off, fav_on, player_error

## Manifest
Permissions INTERNET, ACCESS_NETWORK_STATE. Leanback + touchscreen not required. usesCleartextTraffic=true. App class .FreeTVApp, theme Theme.FreeTV, icon @mipmap/ic_launcher, roundIcon @mipmap/ic_launcher_round, TV banner @drawable/banner.
MainActivity = LAUNCHER + LEANBACK_LAUNCHER, theme Theme.Wosux.Starting (splash, switches to Theme.FreeTV after launch). PlayerActivity = sensorLandscape, theme Theme.FreeTV.Player, not exported.

## Added by phases (the AI appends below)

### Phase 01: CrashLog
- `K/CrashLog.kt`, `object CrashLog`:
  - `fun install(app: Application)`: sets the default uncaught-exception handler; writes device, SDK, versionName, thread and stack trace to `filesDir/last_crash.txt`, then calls the previous handler.
  - `fun pending(ctx: Context): String?`: saved crash text or null.
  - `fun clear(ctx: Context)`: deletes the saved crash.
  - `fun showIfAny(activity: AppCompatActivity)`: dialog "Wosux Stream crashed last time" (first 4000 chars, selectable); "Copy" copies the FULL text and keeps the dialog open; "Close" deletes the file.
- Hooks: `FreeTVApp.onCreate` -> `CrashLog.install(this)` first; `MainActivity.onCreate` -> `CrashLog.showIfAny(this)` after setContentView.
- Limitation: a crash inside Application.onCreate repeats before the dialog can appear.

### Phase 02: Brand colors
- `R/values/colors.xml` values changed: bg #0A0D12, surface #131923, surface_focus #1C2533, accent #02E1FE (cyan, was yellow), text #FFFFFF, text_dim #9AA4B2, logo_bg #E9ECF2.
- New colors: brand_blue #0088FF, brand_blue_mid #02AEF9, brand_cyan #02E1FE, silver #E2E3E9, silver_dim #9AA0AA, live_red #FF4D5A, ok_green #2EE59D, dead_gray #5B6475, scrim #CC0A0D12.
- Still hard-coded elsewhere (to be replaced in later phases): item_channel.xml star color #F59E0B, activity_player.xml error background #AA000000, bg_overlay.xml gradient.

### Phase 03: App name, applicationId, CI file names
- `app/build.gradle.kts`: `applicationId` = `"com.wosux.stream"` (was `com.freetv.app`). `namespace` stays `"com.freetv.app"`. Signing config `freetv`, `app/freetv.keystore` and the debug `applicationIdSuffix = ".test"` are unchanged.
- `R/values/strings.xml`: `app_name` = `Wosux Stream` (was `FreeTV`). The launcher label comes from `@string/app_name` in the manifest.
- `.github/workflows/build-apk.yml`: `on:` is now only `push` with `tags: [ 'v*' ]` plus `workflow_dispatch:` (the branch-push trigger was removed). `cp` lines, the artifact (`Wosux-Stream-apk`) and the release `files:` list use `Wosux-Stream.apk` and `Wosux-Stream-test.apk`. Nothing else in the workflow changed.
- New applicationId = new app on the device: the old FreeTV must be uninstalled by hand and its favorites do not carry over.
- Still says FreeTV (left alone on purpose, not part of this phase): class `FreeTVApp`, styles `Theme.FreeTV` / `Theme.FreeTV.Player`, `rootProject.name` in `settings.gradle.kts`, the `.keystore` file name, and the text in `README.md` (updated in Phase 40).

### Phase 04: Launcher icon + TV banner
- New `R/mipmap-anydpi-v26/ic_launcher.xml` and `R/mipmap-anydpi-v26/ic_launcher_round.xml`: identical `<adaptive-icon>` files, `<background android:drawable="@color/ic_launcher_background" />` and `<foreground android:drawable="@mipmap/ic_launcher_foreground" />`. Android 8.0+ (API 26+) uses these; older Android uses the PNG `ic_launcher` from the density folders.
- New `R/values/ic_launcher_background.xml`: color `ic_launcher_background` = `#0A0D12` (same value as `bg`, separate name on purpose for the launcher).
- `app/src/main/AndroidManifest.xml`: `<application>` gained `android:roundIcon="@mipmap/ic_launcher_round"`. Nothing else changed (`android:icon="@mipmap/ic_launcher"` and `android:banner="@drawable/banner"` were already there).
- Art files made and placed by the user, not by the AI (sizes as the Phase 04 plan specifies, not checked by the AI): `ic_launcher.png` (48/72/96/144/192 px) and `ic_launcher_foreground.png` (108/162/216/324/432 px) in `R/mipmap-mdpi`, `-hdpi`, `-xhdpi`, `-xxhdpi`, `-xxxhdpi`, and the new TV banner `R/drawable-xhdpi/banner.png` (same name as before, so no manifest change was needed for it).
- Resource names now in use: mipmap `ic_launcher`, `ic_launcher_round`, `ic_launcher_foreground`; color `ic_launcher_background`. Phase 05 (splash) may reference `@mipmap/ic_launcher_foreground`.
- Limitation: `ic_launcher_round` exists only in `mipmap-anydpi-v26` (no round PNGs in the density folders), so it resolves on Android 8.0+ only.

### Phase 05: Splash screen
- `app/build.gradle.kts`: new dependency `implementation("androidx.core:core-splashscreen:1.0.1")` (the only new library; no other version changed).
- `R/values/themes.xml`: new style `Theme.Wosux.Starting` (parent `Theme.SplashScreen`, supplied by the library): `windowSplashScreenBackground` = `@color/bg`, `windowSplashScreenAnimatedIcon` = `@mipmap/ic_launcher_foreground`, `postSplashScreenTheme` = `@style/Theme.FreeTV`. Existing styles `Theme.FreeTV`, `Theme.FreeTV.Player` and `Btn` are untouched.
- `app/src/main/AndroidManifest.xml`: `MainActivity` now has `android:theme="@style/Theme.Wosux.Starting"`. The `<application>` theme stays `Theme.FreeTV`; `PlayerActivity` keeps `Theme.FreeTV.Player`.
- `K/MainActivity.kt`: new import `androidx.core.splashscreen.SplashScreen.Companion.installSplashScreen`; `onCreate` now begins with `installSplashScreen()` BEFORE `super.onCreate(savedInstanceState)`. Every other line is unchanged (including `CrashLog.showIfAny(this)` after setContentView).
- No `setKeepOnScreenCondition` and no exit animation: the splash goes away as soon as MainActivity draws its first frame.
- Rule for later phases: `installSplashScreen()` must stay the first call in `MainActivity.onCreate`, before `super.onCreate`, wherever that activity is edited (Phase 17 keeps it). Other activities do not use the splash theme.
- Needs `@mipmap/ic_launcher_foreground` (user-supplied art from Phase 04); the build fails if that PNG is missing.

### Phase 06: Net.kt (one shared network client)
- `app/build.gradle.kts`: new dependencies `implementation("com.squareup.okhttp3:okhttp:4.12.0")` and `implementation("androidx.media3:media3-datasource-okhttp:1.4.1")`. No other version changed.
- New `K/Net.kt`, `object Net`:
  - `const val UA = "WosuxStream/1.0 (Android)"`
  - `val client: OkHttpClient` (lazy): connect 10 s, read 20 s, retryOnConnectionFailure true, followRedirects + followSslRedirects true, `Dispatcher` maxRequests 64 / maxRequestsPerHost 16, application interceptor that sets `User-Agent = UA` only when the request has no User-Agent header. No `Cache` (live segments must not fill the disk).
  - `val probe: OkHttpClient` (lazy) = `client.newBuilder()` with connect 4 s, read 4 s, callTimeout 6 s. Shares the connection pool and dispatcher of `client`.
- In Phase 06 `Net` was not used anywhere yet. Since Phase 07 `MainViewModel.download()` uses `Net.client`; since Phase 08 Coil (through `FreeTVApp.newImageLoader()`) uses it too; since Phase 09 so does the player (`OkHttpDataSource.Factory(Net.client)` in PlayerActivity); `Net.probe` is first used in Phase 11.

### Phase 07: Playlist download through Net, with ETag
- Only `K/MainViewModel.kt` changed (plus this file). Its public API is unchanged (`channels`, `loading`, `message`, `source`, `query`, `favoritesOnly`, `setSource`, `load(force)`). `Net.kt` and `app/build.gradle.kts` are untouched; no new library, no new resource, no new string.
- Imports in MainViewModel: removed `java.net.HttpURLConnection` and `java.net.URL`, added `okhttp3.Request`.
- `private fun download(urlStr: String, dest: File): Boolean` (it returned nothing before) now uses `Net.client` (OkHttp). It still runs only inside `withContext(Dispatchers.IO)`. OkHttp follows the redirects itself (the manual 5-redirect loop is gone) and `Net` adds the `User-Agent` (`Net.UA`; was `FreeTV/1.0 (Android)`). Timeouts are now the ones of `Net.client` (connect 10 s, read 20 s; were 15 s / 30 s).
  - ETag sidecar: `dest.name + ".etag"` in the same folder (`filesDir/pl_<hash>.m3u.etag`), holds the raw `ETag` header text. `If-None-Match` is sent only when the sidecar is readable (printable ASCII) AND `dest` still exists and is not empty, so a 304 can never leave the app without a playlist file.
  - HTTP 304 (only possible when `If-None-Match` was sent): `dest.setLastModified(now)` (restarts the 6 h stale timer), nothing is written, returns `false` = unchanged.
  - HTTP 200: the body is written to `dest.name + ".tmp"`. An empty file deletes the tmp and throws `IOException("Empty playlist")` (as before; old cache and old ETag stay untouched). Otherwise the old sidecar is deleted, the tmp is renamed over `dest` (copy fallback as before), then the new `ETag` header (when the server sent one) is saved to the sidecar. Returns `true`. A kill between the rename and the sidecar write only means "no ETag", so the next refresh downloads in full.
  - Any other status throws `IOException("HTTP <code>")` (as before).
- `load()`: only the lines that call `download()` changed. The cached list is still parsed and shown first. After the download, the file is parsed and `channels` is emitted only when `download()` returned `true` OR nothing was shown from the cache (`shown == false`). If it returned `false` (304) and the cached list is already on screen, nothing is re-parsed and nothing is re-emitted. An empty parse result still sets the `err_empty` message; the catch blocks (`err_refresh` / `err_load`) are unchanged.
- Unchanged on purpose: `setSource()`, `cacheFile()`, `parseFile()`, `STALE_MS` (6 h), the `.tmp` file name and every field.
- Limitations: a server that sends no ETag is downloaded in full on every refresh (same as before Phase 07). There is no OkHttp `Cache` on purpose, so the sidecar file is the only revalidation. Cancelling a `load()` does not stop its blocking download, and two overlapping downloads of the same source still share the same `.tmp` name (existing behaviour, not touched in this phase).

### Phase 08: Fast image loader (logos)
- Only `K/FreeTVApp.kt` changed (plus this file). `Net.kt`, `app/build.gradle.kts`, every layout/resource and every other Kotlin file are untouched. No new library (Coil 2.7.0 and OkHttp 4.12.0 were already there), no new resource, no new string.
- `class FreeTVApp : Application(), ImageLoaderFactory`. New imports: `coil.ImageLoader`, `coil.ImageLoaderFactory`, `coil.disk.DiskCache`, `coil.memory.MemoryCache`, `java.io.File`. `onCreate` is unchanged (`super.onCreate()`, `CrashLog.install(this)`, `Prefs.init(this)`, in that order).
- New `override fun newImageLoader(): ImageLoader` = `ImageLoader.Builder(this)` with:
  - `okHttpClient { Net.client }`: the shared client from Phase 06 (no OkHttp `Cache`; its `User-Agent` interceptor also applies to logo requests). The lambda overload is used on purpose: Coil then creates the client lazily on its background thread when the first logo is downloaded, so this code never builds `Net.client` on the main thread.
  - `memoryCache { MemoryCache.Builder(this@FreeTVApp).maxSizePercent(0.25).build() }`: 25% of the app's available memory.
  - `diskCache { DiskCache.Builder().directory(File(cacheDir, "img")).maxSizeBytes(100L * 1024 * 1024).build() }`: folder `cacheDir/img`, 100 MB, least recently used entries are removed first. It lives in the cache dir, so Android's "Clear cache" empties it.
  - `respectCacheHeaders(false)`: a logo that is in the disk cache is used as is, without asking the server again, whatever the server's cache headers say. This is what makes a logo that was seen once show instantly, also offline.
  - `crossfade(120)`: 120 ms fade-in. Coil skips the fade for images found in the memory cache.
- Coil calls `newImageLoader()` once, on the first image request, and keeps the result as the app-wide default loader (`ImageView.load(...)`, `Coil.imageLoader(context)`). `ChannelAdapter` (`logo.load(url) { placeholder(R.drawable.ic_tv); error(R.drawable.ic_tv) }`) is NOT changed and now uses it. Later screens (hero banner, rows) that call `.load(...)` get the same loader automatically: do not create another `ImageLoader`.
- Limitations: because cache headers are ignored, a channel that replaces its logo file under the same URL keeps showing the old picture until it is evicted (100 MB, least recently used first) or the cache is cleared. A logo is only available offline after it was downloaded once.

### Phase 09: Fast-start player
- Only `K/PlayerActivity.kt` changed (plus this file). `Net.kt`, `app/build.gradle.kts` (`media3-datasource-okhttp:1.4.1` and `okhttp:4.12.0` came in Phase 06), layouts, resources and every other Kotlin file are untouched. No new library, no new resource, no new string.
- Imports: removed `androidx.media3.datasource.DefaultHttpDataSource`; added `androidx.media3.common.AudioAttributes`, `androidx.media3.datasource.okhttp.OkHttpDataSource`, `androidx.media3.exoplayer.DefaultLoadControl`, `androidx.media3.exoplayer.DefaultRenderersFactory`.
- `onStart()`: the player is now built with `ExoPlayer.Builder(this, DefaultRenderersFactory(this).setEnableDecoderFallback(true))`, then `.setLoadControl(DefaultLoadControl.Builder().setBufferDurationsMs(15000, 40000, 1500, 3000).build())`, `.setHandleAudioBecomingNoisy(true)`, `.setAudioAttributes(AudioAttributes.DEFAULT, true)`, `.build()`.
  - Buffer durations in ms: min 15000, max 40000, start playback after 1500, restart after a rebuffer after 3000 (ExoPlayer defaults are 50000 / 50000 / 2500 / 5000), so a channel starts and recovers sooner.
  - `setEnableDecoderFallback(true)`: if a decoder fails to start, ExoPlayer tries the next decoder the device offers.
  - `setHandleAudioBecomingNoisy(true)`: playback pauses (`playWhenReady = false`) when headphones are unplugged or a Bluetooth audio device disconnects. The screen has no play button yet: resume with the Play/Pause key of the remote or headset, or by zapping (`playCurrent()` sets `playWhenReady = true`).
  - `setAudioAttributes(AudioAttributes.DEFAULT, true)`: media usage; the player takes audio focus and reacts when another app takes it (pauses or lowers the volume).
- `playCurrent()`: `DefaultHttpDataSource.Factory()` is replaced by `OkHttpDataSource.Factory(Net.client)` with `.setUserAgent(ch.userAgent ?: DEFAULT_UA)` (same rule as before: the channel's own user-agent, else `DEFAULT_UA`) and, when the channel has a referrer, `setDefaultRequestProperties(mapOf("Referer" to it))` (unchanged). The factory still goes to `DefaultMediaSourceFactory(this).setDataSourceFactory(http)`. The .m3u8 / .mpd MIME hints, `setMediaSource`, `prepare()` and `playWhenReady = true` are unchanged.
- Removed three calls that only `DefaultHttpDataSource.Factory` has: `setAllowCrossProtocolRedirects(true)`, `setConnectTimeoutMs(15_000)`, `setReadTimeoutMs(20_000)`. `Net.client` takes over: it follows redirects including http <-> https (`followRedirects` / `followSslRedirects` true), connect timeout 10 s (was 15 s), read timeout 20 s (same). The player uses `Net.client`, never `Net.probe` (its call timeout would cut live streams).
- Unchanged on purpose: `zap()`, the overlay and its 5 s auto-hide, the one quiet retry, BEHIND_LIVE_WINDOW handling, remote key handling, `onStop()` releasing the player.
- Limitations: no OkHttp `Cache` (on purpose, see Phase 06). The player's requests share the dispatcher of `Net.client` (64 requests in total, 16 per host) with playlist downloads and logo downloads.

### Phase 10: HealthStore (remember which channels work)
- New `K/HealthStore.kt`, `object HealthStore`. Changed: `K/FreeTVApp.kt` (one line, see below). No layout, resource, string, library or version change. Nothing in the UI uses it yet (`StreamChecker` = Phase 11, player reporting = Phase 12).
- Public API:
```kotlin
enum class State { UNKNOWN, ALIVE, DEAD }                    // HealthStore.State
val version: kotlinx.coroutines.flow.MutableStateFlow<Int>   // starts at 0; +1 (at most once per second) when some channel's State changed
fun init(ctx: Context)               // once, from FreeTVApp.onCreate; later calls do nothing; loads filesDir/health.txt on Dispatchers.IO
fun state(url: String): State        // UNKNOWN until the file is loaded
fun isFresh(url: String): Boolean    // state(url) != UNKNOWN (non-expired ALIVE or DEAD result)
fun mark(url: String, alive: Boolean)// record a check result; callable from any thread; debounced save
```
- Rules: key = `url.hashCode()` (a rare hash collision makes two urls share one entry). ALIVE counts for 6 h after the last success. DEAD needs 2 or more failures in a row (fail counter) and counts for 24 h after the last failure. One failure = UNKNOWN (so `isFresh` is false and it should be checked again). After expiry the entry is UNKNOWN but its fail counter stays, so one more failure right after a long DEAD period makes it DEAD again. `mark(url, true)` resets the counter to 0, `mark(url, false)` adds 1. A system clock set back makes entries UNKNOWN.
- `version` is raised only when the State of the marked channel really changes (not on every mark), plus once after the file finished loading (if it had entries). The first change goes out at once, further changes within 1 s are merged into one delayed bump, so the last change is always published.
- Saving: every `mark` restarts a 3 s timer; when it runs (background thread) the whole map is written to `filesDir/health.txt.tmp` and renamed over `health.txt`. Nothing is saved before the load has finished (it would overwrite the file). Marks made while loading are kept and win over the file. Save errors are swallowed and retried on the next `mark`.
- File format `filesDir/health.txt`: one line per entry `hash,alive,fails,timestamp` (`alive` 0/1, timestamp = `System.currentTimeMillis()`), oldest first. Bad lines are skipped. Cap 20,000 entries; when over the cap the entry with the oldest last update is dropped.
- Threads: all state is behind locks; `state()`, `isFresh()`, `mark()` are cheap (map lookup) and safe on the main thread. Disk work only on `Dispatchers.IO` (private scope, `SupervisorJob`).
- `K/FreeTVApp.kt`: `onCreate` is now `super.onCreate()`, `CrashLog.install(this)`, `Prefs.init(this)`, `HealthStore.init(this)`. `newImageLoader()` is unchanged.
- Limitations: while checks keep running with pauses shorter than 3 s, nothing is written until they pause (a kill in that time loses those results). Nothing saves on app exit. Callers should not mark channels as failed while the device is offline (two offline checks make a channel DEAD for 24 h).

### Phase 11: StreamChecker (test a channel without playing it)
- New `K/StreamChecker.kt`, `object StreamChecker`. Nothing else changed: no layout, resource, string, library, version, manifest or other Kotlin file. Nothing calls it yet (`MainViewModel.checkAhead` = Phase 29, player reporting = Phase 12). First user of `Net.probe`.
- Public API:
```kotlin
suspend fun check(ch: Channel): Boolean                      // true = the url answers like a stream; any exception = false; CancellationException is rethrown
suspend fun checkAll(list: List<Channel>, limit: Int = 40)   // returns Unit; results go to HealthStore.mark
```
- Imports used: kotlinx.coroutines (`CancellationException`, `Dispatchers`, `ensureActive`, `launch`, `suspendCancellableCoroutine`, `withContext`), `kotlinx.coroutines.sync.Semaphore`, `kotlin.coroutines.resume`, `okhttp3.Call`, `okhttp3.Request`, `okhttp3.Response`. Needs no new dependency.
- `check(ch)`: runs on `Dispatchers.IO` with `Net.probe` (connect 4 s, read 4 s, whole call 6 s). A GET request is built with `User-Agent` = `ch.userAgent`, or (when null/blank) a private `DEFAULT_UA` (the same Chrome-on-Android string as `PlayerActivity.DEFAULT_UA`, copied because that one is private, so the check sees what the player would see), and `Referer` = `ch.referrer` when not null/blank.
  - URL contains `.m3u8` (case-insensitive): plain GET, success only if HTTP 200 and the first 1024 bytes (read in a loop until 1024 bytes or end of stream) contain `#EXTM3U` (case-insensitive).
  - any other URL: GET with `Range: bytes=0-1023`, success if HTTP 200 or 206 and the `Content-Type` header is missing or does not contain `text/html` (case-insensitive). The body is not read.
  - The response is always closed (`use`), after at most 1024 bytes were read. Cancelling the coroutine calls `Call.cancel()`, so a blocked read ends at once.
  - Invalid URL or invalid header text (`IllegalArgumentException`), network errors, timeouts, anything else = `false`.
- `checkAll(list, limit = 40)`: runs on `Dispatchers.IO`. Walks the list in order; skips a url already seen in this call (duplicates are checked once) and every channel where `HealthStore.isFresh(url)` is true; stops after `limit` real checks have been started (`limit <= 0` or an empty list does nothing). At most 6 checks run at the same time (`Semaphore(6)`, acquired before each check starts). Each finished check calls `HealthStore.mark(url, result)`. Returns when all started checks are finished. If the caller's coroutine is cancelled, running checks are cancelled and their results are NOT marked.
- Limitations: one failed check is not yet a result (`HealthStore` needs 2 failures in a row for DEAD), so a channel with one failure is checked again by the next `checkAll`. `checkAll` does not look at the network state: callers should not run it while the device is offline (two offline checks make a channel DEAD for 24 h). A server that answers 200 with a playlist-less body to the `.m3u8` GET, or that blocks the app's requests, counts as dead. For a non-`.m3u8` url only the status code and the content type are tested, not the content.

### Phase 12: Player reports health, zapping skips dead channels
- Only `K/PlayerActivity.kt` changed (plus this file). `HealthStore.kt`, `StreamChecker.kt`, `app/build.gradle.kts`, layouts, resources, strings, manifest and every other Kotlin file are untouched. No new library, no new import (`HealthStore` is in the same package), no new resource, no new string. No new public API.
- New private members of `PlayerActivity`:
  - `private var healthMarked = false`: true once `HealthStore.mark(url, true)` was sent for the current `prepare()`. Reset to `false` in every place that calls `prepare()`: `playCurrent()`, `retryTask` (the one quiet retry) and the BEHIND_LIVE_WINDOW branch of `onPlayerError`.
  - `private companion object { const val SKIP_DEAD = true }` (next to `DEFAULT_UA`): `false` turns the skipping off, zap then behaves exactly as before Phase 12.
- Reporting (both on the main thread; `HealthStore.mark` is only a map update, the file is saved later on `Dispatchers.IO`). The url is that of `ChannelStore.playlist[index]`, the channel being played:
  - `onPlaybackStateChanged(STATE_READY)`: `HealthStore.mark(url, true)` the first time READY is reached after a `prepare()` (`healthMarked` stops repeat marks on every rebuffer). Also still hides the error text, as before.
  - `onPlayerError`, FINAL error only (the branch after the existing single retry, `retried == true`): `HealthStore.mark(url, false)`, then the error text / overlay as before. The first error (quiet retry) and BEHIND_LIVE_WINDOW errors do NOT mark anything.
- `zap(delta: Int)` (private; called by btnPrev / btnNext and the remote keys, signature unchanged):
  - `target` = the plain neighbour `((index + delta) % n + n) % n` (what Phase 11 did).
  - If `SKIP_DEAD`: starting at that neighbour and walking in the zapping direction (wrapping around the list), at most `n` (= playlist.size) candidates are looked at; the first one with `HealthStore.state(url) != HealthStore.State.DEAD` becomes `target`. If every candidate is DEAD, `target` stays the plain neighbour (= old behaviour).
  - Then `index = target; playCurrent()` (unchanged). Empty playlist: returns, as before.
  - The `n` candidates end with the current channel itself, so when every OTHER channel is DEAD and the current one is not, zapping restarts the current channel (the same thing a one-channel list always did).
  - Cost: up to `n` cheap `HealthStore.state` calls on the main thread only when most channels are DEAD (a stand-in test with 20,000 all-DEAD channels took about 4 ms on a desktop JVM).
- Effect on `HealthStore` rules (Phase 10): one failed final error = UNKNOWN (zap does not skip yet); the 2nd final error in a row for the same url = DEAD for 24 h (zap skips it); the next READY (`mark(url, true)`) makes it ALIVE and clears the counter. `HealthStore.state` is UNKNOWN until `health.txt` has loaded, so nothing is skipped in the first moments after app start.
- Not changed on purpose: `retried` is still reset only in `playCurrent()` (a channel that played for a long time and then errors twice counts the second error as FINAL, as before). Selecting a channel from the grid still plays it even if it is DEAD (only zapping skips). `MainActivity` / `ChannelAdapter` do not show health yet.
- Limitations: a final error while the device is OFFLINE is marked like any other failure (the player does not look at the network state): zapping through channels with the network down can mark them failed, and two such errors on the same channel make it DEAD (skipped by zapping) for 24 h. A later phase should skip the marking when offline. A final error caused by the device (e.g. decoder problem) is also marked as a channel failure. A stale player event arriving right after a zap could be booked to the new channel (not guarded; ExoPlayer clears errors on `prepare()`).

### Phase 13: Faster M3U parser
- Only `K/M3uParser.kt` changed (plus this file). `Models.kt`, `MainViewModel.kt` (its call `M3uParser.parse(it)` is untouched), `app/build.gradle.kts`, layouts, resources, strings, manifest and every other Kotlin file are untouched. No new library, no new import (still only `java.io.BufferedReader`), no new resource, no new string.
- Public API is unchanged: `object M3uParser { fun parse(reader: BufferedReader): List<Channel> }`. New members are all `private`:
  - `private const val EXPECTED_CHANNELS = 12_000` (initial capacity of the result `ArrayList`) and `private const val EXPECTED_URLS = 16_384` (initial capacity of the dedupe `HashSet`, resizes only above 12,288 entries). Starting sizes only: a bigger playlist still grows normally, a small one wastes about 100 KB for a moment.
  - `private class Attrs` (one instance per `parse()` call, so parse stays thread-safe): `scan(line)` finds the LAST `tvg-logo`, `group-title` and `tvg-name` values of an `#EXTINF` line with `indexOf("=\"")` scanning and keeps them as index ranges (`logoS/logoE`, `groupS/groupE`, `nameS/nameE`, -1 = not present). No Regex, no `Map`, no per-attribute substring.
  - `private fun isBlankRange(s: String, from: Int, to: Int): Boolean`.
- The old `private val ATTR = Regex(...)` is removed. Scan rules are the same as the old Regex `([A-Za-z0-9_-]+)="([^"]*)"`: key = run of letters/digits/`_`/`-` right before `="`, key compared ignoring case, value = up to the next `"`, matches never overlap, the later of two equal keys wins, the whole line is scanned (also the part after the last comma), a `="` with no closing quote ends the scan.
- Same output as before: title = text after the LAST comma of the line, trimmed; if the title is blank the `tvg-name` value (not trimmed) is used, if that is blank/missing the name is `Channel`. `logo` only when the value starts with `http` (case-sensitive). `group` only when the value is not blank (not trimmed). `#EXTVLCOPT:` handling (`http-referrer`, `http-referer`, `http-user-agent`), the reset of name/logo/group/referrer/userAgent after every non-comment line, `http://`/`https://`-only URLs, dedupe by exact URL string and the output order are unchanged. The `#EXTVLCOPT:` branch was NOT touched.
- How to verify: load the same source with the Phase 12 build and this build; the channel count on the main screen must be identical (same note is in the comment on top of `M3uParser`).
- Checked before delivery: the new attribute scan and the name/logo/group derivation were ported 1:1 to Java and compared with the old Regex logic on 3 million (scan) and 2 million (derivation) random lines with tricky characters (quotes, `=`, case changes, missing/duplicate keys, whitespace): 0 differences. The Kotlin file itself was not compiled here (no Kotlin compiler available); it uses only plain `String`/`Char` calls that exist in Kotlin 1.9.24.
- Limitations: the speed-up is in the per-line work (no Regex, no Map, fewer substrings, pre-sized collections); `BufferedReader.readLine()` still creates one `String` per line, and the playlist is still parsed on `Dispatchers.IO` by `MainViewModel.parseFile()` as before.

