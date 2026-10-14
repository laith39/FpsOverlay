name: Build APK
on: [push, workflow_dispatch]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.7
      - name: Generate project
        run: |
          mkdir -p app/src/main/java/com/laith/fpsoverlay
          cat > settings.gradle.kts <<'EOF'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement { repositories { google(); mavenCentral() } }
          rootProject.name = "FpsOverlay"
          include(":app")
          EOF
          cat > build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF
          cat > gradle.properties <<'EOF'
          android.useAndroidX=true
          org.gradle.jvmargs=-Xmx2g
          EOF
          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.laith.fpsoverlay"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.laith.fpsoverlay"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 2
                  versionName = "1.1"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
              <application
                  android:label="FPS Overlay"
                  android:theme="@android:style/Theme.Material.NoActionBar">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN"/>
                          <category android:name="android.intent.category.LAUNCHER"/>
                      </intent-filter>
                  </activity>
                  <service
                      android:name=".OverlayService"
                      android:exported="false"
                      android:foregroundServiceType="specialUse">
                      <property
                          android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                          android:value="overlay"/>
                  </service>
              </application>
          </manifest>
          EOF
          cat > app/src/main/java/com/laith/fpsoverlay/MainActivity.kt <<'EOF'
          package com.laith.fpsoverlay

          import android.app.Activity
          import android.content.Intent
          import android.net.Uri
          import android.os.Build
          import android.os.Bundle
          import android.provider.Settings
          import android.widget.Button
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.Switch

          class MainActivity : Activity() {
              private val tools = listOf(
                  "FPS" to "FPS - الفريمات",
                  "RAM" to "RAM - الذاكرة",
                  "BAT" to "BAT - نسبة البطارية",
                  "TMP" to "TMP - حرارة البطارية",
                  "CPU" to "CPU - حرارة المعالج",
                  "GPU" to "GPU - حرارة الرسوميات"
              )

              override fun onCreate(s: Bundle?) {
                  super.onCreate(s)
                  val prefs = getSharedPreferences("p", MODE_PRIVATE)
                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(48, 120, 48, 48)
                  }
                  root.addView(Button(this).apply {
                      text = "تشغيل القياسات"
                      setOnClickListener { startOverlay() }
                  })
                  root.addView(Button(this).apply {
                      text = "إيقاف"
                      setOnClickListener {
                          stopService(Intent(this@MainActivity, OverlayService::class.java))
                      }
                  })
                  for ((key, label) in tools) {
                      root.addView(Switch(this).apply {
                          text = label
                          textSize = 16f
                          setPadding(0, 28, 0, 28)
                          isChecked = prefs.getBoolean(key, true)
                          setOnCheckedChangeListener { _, c ->
                              prefs.edit().putBoolean(key, c).apply()
                          }
                      })
                  }
                  setContentView(ScrollView(this).apply { addView(root) })
                  if (Build.VERSION.SDK_INT >= 33) {
                      requestPermissions(arrayOf(android.Manifest.permission.POST_NOTIFICATIONS), 1)
                  }
              }

              private fun startOverlay() {
                  if (!Settings.canDrawOverlays(this)) {
                      startActivity(
                          Intent(
                              Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                              Uri.parse("package:$packageName")
                          )
                      )
                      return
                  }
                  startForegroundService(Intent(this, OverlayService::class.java))
              }
          }
          EOF
          cat > app/src/main/java/com/laith/fpsoverlay/OverlayService.kt <<'EOF'
          package com.laith.fpsoverlay

          import android.app.*
          import android.content.Context
          import android.content.Intent
          import android.content.IntentFilter
          import android.graphics.Color
          import android.graphics.PixelFormat
          import android.graphics.Typeface
          import android.graphics.drawable.GradientDrawable
          import android.os.BatteryManager
          import android.os.Handler
          import android.os.IBinder
          import android.os.Looper
          import android.view.Gravity
          import android.view.View
          import android.view.WindowManager
          import android.widget.LinearLayout
          import android.widget.TextView
          import java.io.File

          class OverlayService : Service() {
              private lateinit var wm: WindowManager
              private var panel: LinearLayout? = null
              private val h = Handler(Looper.getMainLooper())
              private val rows = LinkedHashMap<String, Pair<LinearLayout, TextView>>()
              private val defs = listOf(
                  "FPS" to "#FF5A1F",
                  "RAM" to "#3B5BFF",
                  "BAT" to "#22C55E",
                  "TMP" to "#EF4444",
                  "CPU" to "#A855F7",
                  "GPU" to "#EAB308"
              )
              private val tick = object : Runnable {
                  override fun run() { update(); h.postDelayed(this, 1000) }
              }

              override fun onBind(i: Intent?): IBinder? = null
              override fun onStartCommand(i: Intent?, f: Int, id: Int) = START_STICKY

              private fun dp(v: Int): Int = (v * resources.displayMetrics.density).toInt()

              override fun onCreate() {
                  super.onCreate()
                  val nm = getSystemService(NotificationManager::class.java)
                  nm.createNotificationChannel(
                      NotificationChannel("ov", "Overlay", NotificationManager.IMPORTANCE_LOW)
                  )
                  val n = Notification.Builder(this, "ov")
                      .setContentTitle("FPS Overlay يعمل")
                      .setSmallIcon(android.R.drawable.ic_menu_info_details)
                      .build()
                  startForeground(1, n)

                  wm = getSystemService(Context.WINDOW_SERVICE) as WindowManager
                  panel = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(dp(10), dp(8), dp(12), dp(8))
                      background = GradientDrawable().apply {
                          setColor(Color.parseColor("#CC0B1226"))
                          cornerRadius = dp(8).toFloat()
                      }
                  }
                  for ((key, color) in defs) {
                      val row = LinearLayout(this).apply {
                          orientation = LinearLayout.HORIZONTAL
                          gravity = Gravity.CENTER_VERTICAL
                          setPadding(0, dp(4), 0, dp(4))
                      }
                      val bar = View(this).apply {
                          setBackgroundColor(Color.parseColor(color))
                      }
                      row.addView(bar, LinearLayout.LayoutParams(dp(5), dp(34)))
                      val col = LinearLayout(this).apply {
                          orientation = LinearLayout.VERTICAL
                          setPadding(dp(8), 0, 0, 0)
                      }
                      col.addView(TextView(this).apply {
                          text = key
                          textSize = 8f
                          typeface = Typeface.MONOSPACE
                          setTextColor(Color.parseColor("#9AA4C0"))
                      })
                      val value = TextView(this).apply {
                          text = "--"
                          textSize = 17f
                          typeface = Typeface.create(Typeface.MONOSPACE, Typeface.BOLD)
                          setTextColor(Color.WHITE)
                      }
                      col.addView(value)
                      row.addView(col)
                      panel!!.addView(row)
                      rows[key] = Pair(row, value)
                  }
                  val lp = WindowManager.LayoutParams(
                      WindowManager.LayoutParams.WRAP_CONTENT,
                      WindowManager.LayoutParams.WRAP_CONTENT,
                      WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
                      WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                          WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE,
                      PixelFormat.TRANSLUCENT
                  )
                  lp.gravity = Gravity.TOP or Gravity.START
                  lp.x = 20
                  lp.y = 80
                  wm.addView(panel, lp)
                  h.post(tick)
              }

              private fun zoneTemp(keys: List<String>): String {
                  val dirs = File("/sys/class/thermal").listFiles() ?: return "--"
                  for (d in dirs) {
                      if (!d.name.startsWith("thermal_zone")) continue
                      try {
                          val type = File(d, "type").readText().trim().lowercase()
                          if (keys.any { type.contains(it) }) {
                              val v = File(d, "temp").readText().trim().toFloat()
                              val c = if (v > 1000f) v / 1000f else v
                              return c.toInt().toString()
                          }
                      } catch (e: Exception) { }
                  }
                  return "--"
              }

              private fun update() {
                  val prefs = getSharedPreferences("p", MODE_PRIVATE)
                  val b = registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
                  val t = (b?.getIntExtra(BatteryManager.EXTRA_TEMPERATURE, 0) ?: 0) / 10
                  val lvl = b?.getIntExtra(BatteryManager.EXTRA_LEVEL, 0) ?: 0
                  val mi = ActivityManager.MemoryInfo()
                  getSystemService(ActivityManager::class.java).getMemoryInfo(mi)
                  val ramPct = ((mi.totalMem - mi.availMem) * 100 / mi.totalMem).toInt()
                  for ((key, pair) in rows) {
                      val on = prefs.getBoolean(key, true)
                      pair.first.visibility = if (on) View.VISIBLE else View.GONE
                      if (!on) continue
                      pair.second.text = when (key) {
                          "FPS" -> "--"
                          "RAM" -> ramPct.toString()
                          "BAT" -> lvl.toString()
                          "TMP" -> t.toString()
                          "CPU" -> zoneTemp(listOf("cpu", "cpuss", "soc", "mtktscpu"))
                          "GPU" -> zoneTemp(listOf("gpu", "gpuss"))
                          else -> "--"
                      }
                  }
              }

              override fun onDestroy() {
                  h.removeCallbacks(tick)
                  panel?.let { wm.removeView(it) }
                  super.onDestroy()
              }
          }
          EOF
      - run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: FpsOverlay-APK
          path: app/build/outputs/apk/debug/app-debug.apk
