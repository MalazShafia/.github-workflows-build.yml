# ملف واحد بيبني تطبيق الأندرويد كامل: بيولّد المشروع من جواته وبيطلّع APK جاهز.
name: Build APK

on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 40
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.10.2'

      - name: Generate project files
        run: |
          set -e
          mkdir -p project && cd project
          mkdir -p "$(dirname 'app/build.gradle.kts')"
          cat > 'app/build.gradle.kts' <<'EOF_APP_BUILD_GRADLE_KTS'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }

          // رقم نسخة مكتبة yt-dlp للأندرويد (الـ workflow بيجرّب نسخة احتياطية إذا فشلت)
          val ytdl = (findProperty("ytdl") as String?) ?: "0.18.1"

          android {
              namespace = "app.videodl"
              compileSdk = 35

              defaultConfig {
                  applicationId = "app.videodl"
                  minSdk = 24
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
                  ndk {
                      abiFilters += listOf("arm64-v8a", "armeabi-v7a")
                  }
              }

              buildTypes {
                  release {
                      isMinifyEnabled = false
                  }
              }

              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions {
                  jvmTarget = "17"
              }
              packaging {
                  jniLibs {
                      useLegacyPackaging = true
                  }
              }
          }

          dependencies {
              implementation("androidx.core:core-ktx:1.13.1")
              implementation("io.github.junkfood02.youtubedl-android:library:$ytdl")
              implementation("io.github.junkfood02.youtubedl-android:ffmpeg:$ytdl")
          }
          EOF_APP_BUILD_GRADLE_KTS

          mkdir -p "$(dirname 'app/src/main/AndroidManifest.xml')"
          cat > 'app/src/main/AndroidManifest.xml' <<'EOF_APP_SRC_MAIN_ANDROIDMANIFEST_XML'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission
                  android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                  android:maxSdkVersion="28" />

              <application
                  android:allowBackup="false"
                  android:extractNativeLibs="true"
                  android:icon="@drawable/ic_launcher"
                  android:label="محمّل الفيديوهات"
                  android:supportsRtl="true"
                  android:theme="@style/AppTheme">

                  <activity
                      android:name=".MainActivity"
                      android:configChanges="orientation|screenSize|keyboardHidden|smallestScreenSize|screenLayout"
                      android:exported="true"
                      android:launchMode="singleTop">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                      <intent-filter>
                          <action android:name="android.intent.action.SEND" />
                          <category android:name="android.intent.category.DEFAULT" />
                          <data android:mimeType="text/plain" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".DownloadService"
                      android:exported="false"
                      android:foregroundServiceType="dataSync" />
              </application>
          </manifest>
          EOF_APP_SRC_MAIN_ANDROIDMANIFEST_XML

          mkdir -p "$(dirname 'app/src/main/assets/index.html')"
          cat > 'app/src/main/assets/index.html' <<'EOF_APP_SRC_MAIN_ASSETS_INDEX_HTML'
          <!doctype html>
          <html lang="ar" dir="rtl">
          <head>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width, initial-scale=1">
          <meta name="theme-color" content="#0e0f13">
          <title>محمّل الفيديوهات</title>
          <style>
            :root {
              --bg: #0e0f13; --card: #171921; --line: #262a36; --text: #eceef4;
              --muted: #8b90a0; --accent: #7c6cff; --accent2: #5ad1c3; --err: #ff6b7a;
            }
            * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
            html, body { background: var(--bg); }
            body {
              margin: 0; color: var(--text);
              font-family: system-ui, -apple-system, "Segoe UI", Tahoma, sans-serif;
              padding: 20px 16px 40px;
            }
            main { max-width: 520px; margin: 0 auto; }
            header { display: flex; align-items: center; justify-content: space-between; margin: 8px 0 4px; }
            h1 { font-size: 22px; margin: 0; }
            .pill {
              display: inline-flex; align-items: center; gap: 7px; font-size: 12px; color: var(--muted);
              background: var(--card); border: 1px solid var(--line); border-radius: 999px; padding: 6px 11px;
            }
            .dot { width: 8px; height: 8px; border-radius: 50%; background: var(--muted); }
            .dot.ok { background: var(--accent2); box-shadow: 0 0 8px var(--accent2); }
            .dot.bad { background: var(--err); }
            .sub { color: var(--muted); font-size: 14px; margin-bottom: 22px; }
            .row { display: flex; gap: 8px; }
            input {
              flex: 1; min-width: 0; background: var(--card); border: 1px solid var(--line);
              color: var(--text); border-radius: 14px; padding: 14px; font-size: 16px; direction: ltr; text-align: left;
            }
            input:focus { outline: none; border-color: var(--accent); }
            button {
              border: 0; border-radius: 14px; padding: 14px 18px; font-size: 16px; font-weight: 600;
              background: var(--accent); color: #fff; cursor: pointer; font-family: inherit;
            }
            button.ghost { background: var(--card); border: 1px solid var(--line); color: var(--text); }
            button:disabled { opacity: .5; }
            .card {
              background: var(--card); border: 1px solid var(--line); border-radius: 18px;
              margin-top: 18px; overflow: hidden;
            }
            .thumb { width: 100%; aspect-ratio: 16/9; object-fit: cover; display: block; background: #000; }
            .body { padding: 16px; }
            .title { font-weight: 700; line-height: 1.5; margin: 0 0 6px; }
            .meta { color: var(--muted); font-size: 13px; }
            .label { margin: 16px 0 8px; font-size: 13px; color: var(--muted); }
            .chips { display: flex; flex-wrap: wrap; gap: 8px; }
            .chip {
              padding: 9px 14px; border-radius: 999px; background: transparent; color: var(--text);
              border: 1px solid var(--line); font-size: 14px; font-weight: 500;
            }
            .chip.on { background: var(--accent); border-color: var(--accent); }
            .go { width: 100%; margin-top: 18px; }
            .bar { height: 8px; background: var(--line); border-radius: 99px; overflow: hidden; margin-top: 16px; }
            .bar > i { display: block; height: 100%; width: 0; background: linear-gradient(90deg, var(--accent), var(--accent2)); transition: width .3s; }
            .status { margin-top: 10px; font-size: 14px; color: var(--muted); display: flex; justify-content: space-between; }
            .error { color: var(--err); margin-top: 14px; font-size: 14px; line-height: 1.6; word-break: break-word; }
            .done {
              margin-top: 14px; background: var(--accent2); color: #06201c; border-radius: 14px;
              padding: 14px; font-weight: 700; line-height: 1.7; text-align: center;
            }
            .done small { display: block; font-weight: 500; opacity: .85; direction: ltr; word-break: break-all; }
            .foot { margin-top: 28px; text-align: center; }
            .foot button { font-size: 13px; padding: 10px 16px; }
            .foot .note { color: var(--muted); font-size: 12px; margin-top: 8px; min-height: 18px; }
            .hidden { display: none; }
          </style>
          </head>
          <body>
          <main>
            <header>
              <h1>محمّل الفيديوهات</h1>
              <span class="pill"><i class="dot" id="dot"></i><span id="pillText">جاري التجهيز…</span></span>
            </header>
            <div class="sub">الصق الرابط (أو شاركه من أي تطبيق)، اختار الجودة، ونزّل.</div>

            <div class="row">
              <input id="url" type="url" inputmode="url" placeholder="https://..." autocomplete="off">
              <button class="ghost" id="paste" aria-label="لصق">لصق</button>
            </div>
            <button class="go" id="analyze">تحليل الرابط</button>
            <div class="error hidden" id="err"></div>

            <section class="card hidden" id="card">
              <img class="thumb" id="thumb" alt="">
              <div class="body">
                <p class="title" id="title"></p>
                <div class="meta" id="meta"></div>

                <div class="label">الجودة</div>
                <div class="chips" id="chips"></div>

                <button class="go" id="download">تنزيل</button>

                <div id="prog" class="hidden">
                  <div class="bar"><i id="fill"></i></div>
                  <div class="status"><span id="stage"></span><span id="speed"></span></div>
                </div>
                <div class="done hidden" id="done"></div>
              </div>
            </section>

            <div class="foot">
              <button class="ghost" id="update">تحديث المحرّك</button>
              <div class="note" id="updNote">إذا منصة وقفت عن التنزيل، حدّث المحرّك.</div>
            </div>
          </main>

          <script>
          const $ = id => document.getElementById(id);
          const A = window.Android;
          let quality = "best", current = "";

          const sleep = ms => new Promise(r => setTimeout(r, ms));
          const parse = s => { try { return JSON.parse(s); } catch (e) { return { stage: "error", error: "رد غير مفهوم من التطبيق" }; } };

          async function waitFor(id) {
            for (;;) {
              const r = parse(A.result(id));
              if (r.stage !== "pending") return r;
              await sleep(350);
            }
          }

          function showErr(m) { $("err").textContent = m || ""; $("err").classList.toggle("hidden", !m); }
          function fmtDur(s) {
            if (!s) return "";
            const h = Math.floor(s / 3600), m = Math.floor(s % 3600 / 60), r = Math.floor(s % 60);
            return (h ? h + ":" + String(m).padStart(2, "0") : m) + ":" + String(r).padStart(2, "0");
          }

          if (!A) {
            $("dot").className = "dot bad"; $("pillText").textContent = "افتحها من التطبيق";
            showErr("هالصفحة بتشتغل بس جوّا تطبيق محمّل الفيديوهات.");
            ["analyze", "paste", "update"].forEach(i => $(i).disabled = true);
          } else {
            const tick = () => {
              const p = parse(A.ping());
              if (p.ready) { $("dot").className = "dot ok"; $("pillText").textContent = "المحرّك جاهز"; }
              else { $("dot").className = "dot"; $("pillText").textContent = "عم يجهّز المحرّك…"; setTimeout(tick, 800); }
            };
            tick();
          }

          $("paste").onclick = () => {
            const t = A ? A.clipboard() : "";
            if (t) $("url").value = t.trim(); else $("url").focus();
          };

          window.onShared = () => {
            if (!A) return;
            const u = A.pendingShare();
            if (u) { $("url").value = u; $("analyze").click(); }
          };

          $("analyze").onclick = async () => {
            const url = $("url").value.trim();
            showErr(""); $("card").classList.add("hidden");
            if (!/^https?:\/\//i.test(url)) return showErr("الصق رابط صالح أول شي");
            $("analyze").disabled = true; $("analyze").textContent = "عم يحلل…";
            try {
              const r = await waitFor(A.info(url));
              if (r.stage === "error") throw new Error(r.error || "صار خطأ");
              const d = r.data;
              current = url;
              $("thumb").src = d.thumbnail || "";
              $("thumb").classList.toggle("hidden", !d.thumbnail);
              $("title").textContent = d.title || "بدون عنوان";
              $("meta").textContent = [d.site, d.uploader, fmtDur(d.duration)].filter(Boolean).join(" · ");
              buildChips(d);
              $("prog").classList.add("hidden"); $("done").classList.add("hidden");
              $("card").classList.remove("hidden");
            } catch (e) { showErr(e.message); }
            $("analyze").disabled = false; $("analyze").textContent = "تحليل الرابط";
          };

          function buildChips(d) {
            const opts = [["best", "أعلى جودة"]];
            const hs = (d.heights || []).filter(h => [2160, 1440, 1080, 720, 480, 360].includes(h));
            hs.forEach(h => opts.push([String(h), h + "p"]));
            opts.push(["audio", d.ffmpeg ? "صوت MP3" : "صوت فقط"]);
            quality = "best";
            $("chips").innerHTML = "";
            opts.forEach(([v, t]) => {
              const b = document.createElement("button");
              b.className = "chip" + (v === quality ? " on" : "");
              b.textContent = t;
              b.onclick = () => { quality = v; [...$("chips").children].forEach(c => c.classList.remove("on")); b.classList.add("on"); };
              $("chips").appendChild(b);
            });
          }

          $("download").onclick = () => {
            showErr(""); $("done").classList.add("hidden");
            $("download").disabled = true;
            $("prog").classList.remove("hidden"); $("fill").style.width = "0%";
            $("stage").textContent = "عم يبلش…"; $("speed").textContent = "";
            const id = A.download(current, quality);
            poll(id);
          };

          function poll(id) {
            const t = setInterval(() => {
              const j = parse(A.result(id));
              const pct = j.percent || 0;
              $("fill").style.width = pct + "%";
              if (j.stage === "downloading") {
                $("stage").textContent = pct >= 99.5 ? "عم يعالج الملف…" : "عم ينزّل… " + Math.round(pct) + "%";
                $("speed").textContent = [j.speed, j.eta && "باقي " + j.eta].filter(Boolean).join(" · ");
              } else if (j.stage === "processing") {
                $("stage").textContent = "عم يحفظ الملف…"; $("speed").textContent = "";
              } else if (j.stage === "done") {
                clearInterval(t);
                $("fill").style.width = "100%"; $("stage").textContent = "خلص ✓"; $("speed").textContent = "";
                const box = $("done");
                box.textContent = "انحفظ بمجلد: " + j.folder;
                const s = document.createElement("small"); s.textContent = j.file; box.appendChild(s);
                box.classList.remove("hidden");
                $("download").disabled = false;
              } else if (j.stage === "error") {
                clearInterval(t); showErr(j.error || "فشل التنزيل");
                $("prog").classList.add("hidden"); $("download").disabled = false;
              }
            }, 600);
          }

          $("update").onclick = async () => {
            $("update").disabled = true; $("updNote").textContent = "عم يحدّث… (ممكن ياخد شوي)";
            try {
              const r = await waitFor(A.update());
              $("updNote").textContent = r.stage === "error" ? ("فشل التحديث: " + (r.error || "")) : (r.data && r.data.message) || "تم";
            } catch (e) { $("updNote").textContent = "فشل التحديث"; }
            $("update").disabled = false;
          };

          window.onShared();
          </script>
          </body>
          </html>
          EOF_APP_SRC_MAIN_ASSETS_INDEX_HTML

          mkdir -p "$(dirname 'app/src/main/java/app/videodl/Bridge.kt')"
          cat > 'app/src/main/java/app/videodl/Bridge.kt' <<'EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_BRIDGE_KT'
          package app.videodl

          import android.content.ClipboardManager
          import android.content.Context
          import android.webkit.JavascriptInterface
          import org.json.JSONObject

          /** الجسر بين صفحة HTML والمحرّك. كل الدوال بترجّع فوراً (النتيجة بتنجلب بـ result). */
          class Bridge(private val act: MainActivity) {

              private fun ctx(): Context = act.applicationContext

              @JavascriptInterface
              fun ping(): String =
                  JSONObject().put("ok", true).put("ready", Engine.ready()).toString()

              @JavascriptInterface
              fun info(url: String): String = Engine.startInfo(ctx(), url)

              @JavascriptInterface
              fun download(url: String, quality: String): String =
                  Engine.startDownload(ctx(), url, quality)

              @JavascriptInterface
              fun update(): String = Engine.startUpdate(ctx())

              @JavascriptInterface
              fun result(id: String): String = Engine.result(id)

              @JavascriptInterface
              fun clipboard(): String {
                  return try {
                      val cm = act.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
                      val clip = cm.primaryClip
                      if (clip != null && clip.itemCount > 0) {
                          clip.getItemAt(0).coerceToText(act).toString()
                      } else {
                          ""
                      }
                  } catch (e: Throwable) {
                      ""
                  }
              }

              @JavascriptInterface
              fun pendingShare(): String {
                  val s = act.pendingShare
                  act.pendingShare = ""
                  return s
              }
          }
          EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_BRIDGE_KT

          mkdir -p "$(dirname 'app/src/main/java/app/videodl/DownloadService.kt')"
          cat > 'app/src/main/java/app/videodl/DownloadService.kt' <<'EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_DOWNLOADSERVICE_KT'
          package app.videodl

          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.Service
          import android.content.Context
          import android.content.Intent
          import android.content.pm.ServiceInfo
          import android.os.Build
          import android.os.IBinder
          import androidx.core.app.NotificationCompat
          import androidx.core.content.ContextCompat

          /** خدمة أمامية بسيطة: بتخلي التنزيل شغّال لو طلعت من التطبيق أو قفلت الشاشة. */
          class DownloadService : Service() {

              override fun onBind(intent: Intent?): IBinder? = null

              override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
                  if (Build.VERSION.SDK_INT >= 26) {
                      val nm = getSystemService(NotificationManager::class.java)
                      nm.createNotificationChannel(
                          NotificationChannel(CHANNEL, "التنزيلات", NotificationManager.IMPORTANCE_LOW)
                      )
                  }
                  val n = NotificationCompat.Builder(this, CHANNEL)
                      .setContentTitle("محمّل الفيديوهات")
                      .setContentText("جاري التنزيل…")
                      .setSmallIcon(android.R.drawable.stat_sys_download)
                      .setOngoing(true)
                      .build()
                  if (Build.VERSION.SDK_INT >= 29) {
                      startForeground(1, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)
                  } else {
                      startForeground(1, n)
                  }
                  return START_NOT_STICKY
              }

              companion object {
                  private const val CHANNEL = "downloads"

                  fun start(ctx: Context) {
                      try {
                          ContextCompat.startForegroundService(ctx, Intent(ctx, DownloadService::class.java))
                      } catch (e: Throwable) {
                          // التنزيل بيكمّل حتى لو ما اشتغلت الخدمة
                      }
                  }

                  fun stop(ctx: Context) {
                      try {
                          ctx.stopService(Intent(ctx, DownloadService::class.java))
                      } catch (e: Throwable) {
                      }
                  }
              }
          }
          EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_DOWNLOADSERVICE_KT

          mkdir -p "$(dirname 'app/src/main/java/app/videodl/Engine.kt')"
          cat > 'app/src/main/java/app/videodl/Engine.kt' <<'EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_ENGINE_KT'
          package app.videodl

          import android.content.ContentValues
          import android.content.Context
          import android.media.MediaScannerConnection
          import android.os.Build
          import android.os.Environment
          import android.os.Handler
          import android.os.Looper
          import android.provider.MediaStore
          import android.webkit.MimeTypeMap
          import com.yausername.youtubedl_android.YoutubeDL
          import com.yausername.youtubedl_android.YoutubeDLRequest
          import org.json.JSONArray
          import org.json.JSONObject
          import java.io.File
          import java.io.IOException
          import java.lang.reflect.InvocationTargetException
          import java.util.UUID
          import java.util.concurrent.ConcurrentHashMap
          import java.util.concurrent.Executors
          import java.util.concurrent.atomic.AtomicInteger

          /** حالة طلب (معلومات / تنزيل / تحديث) بتنقرا من الواجهة عن طريق polling. */
          class Res {
              @Volatile var stage: String = "pending"
              @Volatile var data: JSONObject? = null
              @Volatile var error: String? = null
              @Volatile var percent: Double = 0.0
              @Volatile var speed: String = ""
              @Volatile var eta: String = ""
              @Volatile var file: String? = null
              @Volatile var folder: String? = null

              fun toJson(): String {
                  val j = JSONObject()
                  j.put("stage", stage)
                  val d = data
                  if (d != null) j.put("data", d)
                  val er = error
                  if (er != null) j.put("error", er)
                  j.put("percent", percent)
                  j.put("speed", speed)
                  j.put("eta", eta)
                  val f = file
                  if (f != null) j.put("file", f)
                  val fo = folder
                  if (fo != null) j.put("folder", fo)
                  return j.toString()
              }
          }

          object Engine {
              private val pool = Executors.newCachedThreadPool()
              private val jobs = ConcurrentHashMap<String, Res>()
              private val lock = Any()
              private val svcLock = Any()
              private val mainHandler = Handler(Looper.getMainLooper())
              private val active = AtomicInteger(0)
              private val qualities = setOf("best", "2160", "1440", "1080", "720", "480", "360", "audio")
              private val speedRegex = Regex("""at\s+(\S+/s)""")
              private val ansiRegex = Regex("\u001B\\[[0-9;]*m")

              @Volatile private var initDone = false
              @Volatile var hasFfmpeg = false
                  private set

              fun ready(): Boolean = initDone

              private fun newId(): String = UUID.randomUUID().toString().substring(0, 10)

              // الخدمة الأمامية: بتشتغل لما يكون في تنزيل نشط، وبتنوقف بعد ما يخلصوا كلهم بتأخير بسيط
              private fun serviceAcquire(ctx: Context) {
                  synchronized(svcLock) {
                      if (active.incrementAndGet() == 1) DownloadService.start(ctx)
                  }
              }

              private fun serviceRelease(ctx: Context) {
                  synchronized(svcLock) {
                      if (active.decrementAndGet() == 0) {
                          mainHandler.postDelayed({
                              synchronized(svcLock) {
                                  if (active.get() == 0) DownloadService.stop(ctx)
                              }
                          }, 2000)
                      }
                  }
              }

              fun warmUp(ctx: Context) {
                  pool.execute {
                      try {
                          ensureInit(ctx)
                      } catch (e: Throwable) {
                          // بنعيد المحاولة لما يجي أول طلب
                      }
                  }
              }

              private fun ensureInit(ctx: Context) {
                  synchronized(lock) {
                      if (initDone) return
                      YoutubeDL.getInstance().init(ctx)
                      hasFfmpeg = initFfmpeg(ctx)
                      initDone = true
                  }
              }

              // بنستدعي مكتبة ffmpeg بالـ reflection عشان ما نعتمد على اسم الباكيج بالضبط
              private fun initFfmpeg(ctx: Context): Boolean {
                  val names = listOf(
                      "com.yausername.ffmpeg.FFmpeg",
                      "com.yausername.youtubedl_android.ffmpeg.FFmpeg"
                  )
                  for (name in names) {
                      try {
                          val c = Class.forName(name)
                          val inst = try {
                              c.getMethod("getInstance").invoke(null)
                          } catch (e: Throwable) {
                              c.getField("INSTANCE").get(null)
                          }
                          c.getMethod("init", Context::class.java).invoke(inst, ctx)
                          return true
                      } catch (e: Throwable) {
                          // جرّب الاسم التاني
                      }
                  }
                  return false
              }

              fun result(id: String): String {
                  val r = jobs[id]
                  if (r != null) return r.toJson()
                  return JSONObject().put("stage", "error").put("error", "طلب غير معروف").toString()
              }

              private fun JSONObject.str(key: String): String =
                  if (isNull(key)) "" else optString(key, "")

              private fun cleanError(e: Throwable): String {
                  var t: Throwable = e
                  if (t is InvocationTargetException) {
                      val c = t.cause
                      if (c != null) t = c
                  }
                  val raw = (t.message ?: t.toString()).replace(ansiRegex, "").replace("ERROR: ", "").trim()
                  val line = raw.lines().lastOrNull { it.isNotBlank() } ?: raw
                  return line.trim().take(300)
              }

              private fun fmtEta(sec: Long): String {
                  if (sec < 0) return ""
                  val m = sec / 60
                  val s = sec % 60
                  return m.toString() + ":" + s.toString().padStart(2, '0')
              }

              // ---------- معلومات الفيديو ----------
              fun startInfo(ctx: Context, url: String): String {
                  val id = newId()
                  val res = Res()
                  jobs[id] = res
                  pool.execute {
                      try {
                          ensureInit(ctx)
                          val req = YoutubeDLRequest(url)
                          req.addOption("--dump-single-json")
                          req.addOption("--no-playlist")
                          req.addOption("--no-warnings")
                          val resp = YoutubeDL.getInstance().execute(req, id) { _, _, _ -> }
                          val d = JSONObject(resp.out)

                          val hs = mutableSetOf<Int>()
                          val fm = d.optJSONArray("formats")
                          if (fm != null) {
                              for (i in 0 until fm.length()) {
                                  val f = fm.optJSONObject(i) ?: continue
                                  val h = f.optInt("height", 0)
                                  val v = if (f.isNull("vcodec")) "unknown" else f.optString("vcodec", "unknown")
                                  if (h > 0 && v != "none") hs.add(h)
                              }
                          }
                          val heights = JSONArray()
                          for (h in hs.sortedDescending()) heights.put(h)

                          val duration = if (d.isNull("duration")) 0.0 else d.optDouble("duration", 0.0)
                          val data = JSONObject()
                          data.put("title", d.str("title"))
                          data.put("thumbnail", d.str("thumbnail"))
                          data.put("duration", duration)
                          data.put("uploader", d.str("uploader"))
                          data.put("site", d.str("extractor_key"))
                          data.put("heights", heights)
                          data.put("ffmpeg", hasFfmpeg)
                          res.data = data
                          res.stage = "done"
                      } catch (e: Throwable) {
                          res.error = cleanError(e)
                          res.stage = "error"
                      }
                  }
                  return id
              }

              // ---------- التنزيل ----------
              fun startDownload(ctx: Context, url: String, quality: String): String {
                  val id = newId()
                  val res = Res()
                  if (quality !in qualities) {
                      res.error = "جودة غير صالحة"
                      res.stage = "error"
                      jobs[id] = res
                      return id
                  }
                  res.stage = "downloading"
                  jobs[id] = res
                  serviceAcquire(ctx)

                  pool.execute {
                      val base = ctx.getExternalFilesDir(null) ?: ctx.cacheDir
                      val tmp = File(base, "tmp/" + id)
                      try {
                          ensureInit(ctx)
                          tmp.mkdirs()
                          val req = YoutubeDLRequest(url)
                          req.addOption("--no-playlist")
                          req.addOption("--no-mtime")
                          req.addOption("-o", tmp.absolutePath + "/%(title).80s [%(id)s].%(ext)s")

                          if (quality == "audio") {
                              req.addOption("-f", "bestaudio/best")
                              if (hasFfmpeg) {
                                  req.addOption("-x")
                                  req.addOption("--audio-format", "mp3")
                                  req.addOption("--audio-quality", "0")
                              }
                          } else {
                              val h = if (quality == "best") "" else "[height<=" + quality + "]"
                              if (hasFfmpeg) {
                                  req.addOption("-f", "bestvideo" + h + "+bestaudio/best" + h + "/best")
                                  req.addOption("--merge-output-format", "mp4")
                              } else {
                                  req.addOption("-f", "best" + h + "/best")
                              }
                          }

                          YoutubeDL.getInstance().execute(req, id) { progress, eta, line ->
                              res.percent = progress.toDouble().coerceAtLeast(0.0)
                              res.eta = fmtEta(eta)
                              val m = speedRegex.find(line ?: "")
                              if (m != null) res.speed = m.groupValues[1]
                          }

                          val files = (tmp.listFiles() ?: emptyArray<File>()).filter {
                              it.isFile && !it.name.endsWith(".part") &&
                                  !it.name.endsWith(".ytdl") && !it.name.endsWith(".temp")
                          }
                          val best = files.maxByOrNull { it.length() } ?: throw IOException("ما انحفظ أي ملف")
                          res.percent = 100.0
                          res.eta = ""
                          res.stage = "processing"
                          val folder = saveToDownloads(ctx, best)
                          res.file = best.name
                          res.folder = folder
                          res.stage = "done"
                      } catch (e: Throwable) {
                          res.error = cleanError(e)
                          res.stage = "error"
                      } finally {
                          tmp.deleteRecursively()
                          serviceRelease(ctx)
                      }
                  }
                  return id
              }

              // بينقل الملف لمجلد Download/VideoDownloader (بيظهر بمدير الملفات والمعرض)
              private fun saveToDownloads(ctx: Context, src: File): String {
                  val ext = src.extension.lowercase()
                  val mime = MimeTypeMap.getSingleton().getMimeTypeFromExtension(ext) ?: "application/octet-stream"
                  if (Build.VERSION.SDK_INT >= 29) {
                      val cv = ContentValues()
                      cv.put(MediaStore.MediaColumns.DISPLAY_NAME, src.name)
                      cv.put(MediaStore.MediaColumns.MIME_TYPE, mime)
                      cv.put(MediaStore.MediaColumns.RELATIVE_PATH, "Download/VideoDownloader")
                      cv.put(MediaStore.MediaColumns.IS_PENDING, 1)
                      val resolver = ctx.contentResolver
                      val uri = resolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, cv)
                          ?: throw IOException("ما قدرت أنشئ الملف بمجلد التنزيلات")
                      val out = resolver.openOutputStream(uri) ?: throw IOException("ما قدرت أفتح الملف للكتابة")
                      out.use { o -> src.inputStream().use { i -> i.copyTo(o) } }
                      val done = ContentValues()
                      done.put(MediaStore.MediaColumns.IS_PENDING, 0)
                      resolver.update(uri, done, null, null)
                  } else {
                      val dir = File(
                          Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS),
                          "VideoDownloader"
                      )
                      if (!dir.exists() && !dir.mkdirs()) {
                          throw IOException("ما قدرت أنشئ المجلد، تأكد من صلاحية التخزين")
                      }
                      var dest = File(dir, src.name)
                      var n = 1
                      while (dest.exists()) {
                          dest = File(dir, src.nameWithoutExtension + " (" + n + ")." + src.extension)
                          n++
                      }
                      src.copyTo(dest)
                      MediaScannerConnection.scanFile(ctx, arrayOf(dest.absolutePath), null, null)
                  }
                  src.delete()
                  return "Download/VideoDownloader"
              }

              // ---------- تحديث yt-dlp ----------
              fun startUpdate(ctx: Context): String {
                  val id = newId()
                  val res = Res()
                  jobs[id] = res
                  pool.execute {
                      try {
                          ensureInit(ctx)
                          res.data = JSONObject().put("message", updateYtDlp(ctx))
                          res.stage = "done"
                      } catch (e: Throwable) {
                          res.error = cleanError(e)
                          res.stage = "error"
                      }
                  }
                  return id
              }

              // بالـ reflection لأنو شكل قناة التحديث (UpdateChannel) بيختلف بين نسخ المكتبة
              private fun updateYtDlp(ctx: Context): String {
                  val ydl = YoutubeDL.getInstance()
                  val m = ydl.javaClass.methods.firstOrNull {
                      it.name == "updateYoutubeDL" && it.parameterTypes.size == 2
                  } ?: return "التحديث التلقائي مو متاح بهالنسخة من المكتبة"
                  val channel = pickStableChannel(m.parameterTypes[1])
                      ?: return "ما لقيت قناة التحديث بهالنسخة من المكتبة"
                  val r = m.invoke(ydl, ctx, channel)
                  val s = r?.toString() ?: ""
                  return if (s.contains("ALREADY", ignoreCase = true)) "المحرّك محدّث أصلاً ✓" else "تم تحديث المحرّك ✓"
              }

              private fun pickStableChannel(t: Class<*>): Any? {
                  for (n in listOf("STABLE", "_STABLE")) {
                      try {
                          return t.getField(n).get(null)
                      } catch (e: Throwable) {
                      }
                      try {
                          val c = Class.forName(t.name + "$" + n)
                          return c.getField("INSTANCE").get(null)
                      } catch (e: Throwable) {
                      }
                      try {
                          val comp = t.getField("Companion").get(null)
                          return comp.javaClass.getMethod("get" + n).invoke(comp)
                      } catch (e: Throwable) {
                      }
                  }
                  val consts = t.enumConstants
                  if (consts != null) {
                      for (c in consts) {
                          if (c.toString().contains("STABLE", ignoreCase = true)) return c
                      }
                  }
                  return null
              }
          }
          EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_ENGINE_KT

          mkdir -p "$(dirname 'app/src/main/java/app/videodl/MainActivity.kt')"
          cat > 'app/src/main/java/app/videodl/MainActivity.kt' <<'EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_MAINACTIVITY_KT'
          package app.videodl

          import android.Manifest
          import android.annotation.SuppressLint
          import android.app.Activity
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.graphics.Color
          import android.os.Build
          import android.os.Bundle
          import android.webkit.WebView

          class MainActivity : Activity() {

              private lateinit var web: WebView

              // رابط جاي من زر "مشاركة" بتطبيق تاني (يوتيوب، تيك توك...)
              @Volatile
              var pendingShare: String = ""

              @SuppressLint("SetJavaScriptEnabled")
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  web = WebView(this)
                  web.setBackgroundColor(Color.parseColor("#0E0F13"))
                  web.settings.javaScriptEnabled = true
                  web.settings.domStorageEnabled = true
                  web.addJavascriptInterface(Bridge(this), "Android")
                  setContentView(web)

                  handleIntent(intent)
                  web.loadUrl("file:///android_asset/index.html")

                  Engine.warmUp(applicationContext)
                  askPermissions()
              }

              override fun onNewIntent(intent: Intent) {
                  super.onNewIntent(intent)
                  setIntent(intent)
                  handleIntent(intent)
                  web.evaluateJavascript("window.onShared && window.onShared()", null)
              }

              private fun handleIntent(i: Intent?) {
                  if (i == null) return
                  val type = i.type
                  if (i.action == Intent.ACTION_SEND && type != null && type.startsWith("text/")) {
                      val text = i.getStringExtra(Intent.EXTRA_TEXT) ?: return
                      val m = Regex("https?://\\S+").find(text)
                      pendingShare = if (m != null) m.value else text.trim()
                  }
              }

              private fun askPermissions() {
                  val need = mutableListOf<String>()
                  if (Build.VERSION.SDK_INT >= 33) need.add(Manifest.permission.POST_NOTIFICATIONS)
                  if (Build.VERSION.SDK_INT <= 28) need.add(Manifest.permission.WRITE_EXTERNAL_STORAGE)
                  val missing = need.filter { checkSelfPermission(it) != PackageManager.PERMISSION_GRANTED }
                  if (missing.isNotEmpty()) requestPermissions(missing.toTypedArray(), 1)
              }
          }
          EOF_APP_SRC_MAIN_JAVA_APP_VIDEODL_MAINACTIVITY_KT

          mkdir -p "$(dirname 'app/src/main/res/drawable/ic_launcher.xml')"
          cat > 'app/src/main/res/drawable/ic_launcher.xml' <<'EOF_APP_SRC_MAIN_RES_DRAWABLE_IC_LAUNCHER_XML'
          <?xml version="1.0" encoding="utf-8"?>
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="108dp"
              android:height="108dp"
              android:viewportWidth="108"
              android:viewportHeight="108">
              <path
                  android:fillColor="#0E0F13"
                  android:pathData="M0,0h108v108h-108z" />
              <path
                  android:fillColor="#7C6CFF"
                  android:pathData="M48,26h12v26h14L54,74L34,52h14z" />
              <path
                  android:fillColor="#5AD1C3"
                  android:pathData="M30,82h48v6h-48z" />
          </vector>
          EOF_APP_SRC_MAIN_RES_DRAWABLE_IC_LAUNCHER_XML

          mkdir -p "$(dirname 'app/src/main/res/values/colors.xml')"
          cat > 'app/src/main/res/values/colors.xml' <<'EOF_APP_SRC_MAIN_RES_VALUES_COLORS_XML'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <color name="bg">#0E0F13</color>
          </resources>
          EOF_APP_SRC_MAIN_RES_VALUES_COLORS_XML

          mkdir -p "$(dirname 'app/src/main/res/values/styles.xml')"
          cat > 'app/src/main/res/values/styles.xml' <<'EOF_APP_SRC_MAIN_RES_VALUES_STYLES_XML'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <style name="AppTheme" parent="android:Theme.Material.NoActionBar">
                  <item name="android:windowBackground">@color/bg</item>
                  <item name="android:statusBarColor">@color/bg</item>
                  <item name="android:navigationBarColor">@color/bg</item>
              </style>
          </resources>
          EOF_APP_SRC_MAIN_RES_VALUES_STYLES_XML

          mkdir -p "$(dirname 'build.gradle.kts')"
          cat > 'build.gradle.kts' <<'EOF_BUILD_GRADLE_KTS'
          plugins {
              id("com.android.application") version "8.7.2" apply false
              id("org.jetbrains.kotlin.android") version "2.1.0" apply false
          }
          EOF_BUILD_GRADLE_KTS

          mkdir -p "$(dirname 'gradle.properties')"
          cat > 'gradle.properties' <<'EOF_GRADLE_PROPERTIES'
          org.gradle.jvmargs=-Xmx3g -Dfile.encoding=UTF-8
          android.useAndroidX=true
          android.nonTransitiveRClass=true
          kotlin.code.style=official
          EOF_GRADLE_PROPERTIES

          mkdir -p "$(dirname 'settings.gradle.kts')"
          cat > 'settings.gradle.kts' <<'EOF_SETTINGS_GRADLE_KTS'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "VideoDownloader"
          include(":app")
          EOF_SETTINGS_GRADLE_KTS

      - name: Build APK
        working-directory: project
        run: |
          gradle assembleDebug --no-daemon --stacktrace -Pytdl=0.18.1 || gradle assembleDebug --no-daemon --stacktrace -Pytdl=0.17.4

      - uses: actions/upload-artifact@v4
        with:
          name: video-downloader-apk
          path: project/app/build/outputs/apk/debug/app-debug.apk
