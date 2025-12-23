#!/usr/bin/env bash
set -e

# Jalankan dari dalam folder repo yang sudah diclone: mimim1234/Drawing-studio
# Pastikan anda telah konfigurasi git credentials (HTTPS PAT or SSH) supaya push berjaya.

# Buat struktur direktori
mkdir -p app/src/main/java/com/kids/drawingstudio
mkdir -p app/src/main/assets/www
mkdir -p app/src/main/res/layout
mkdir -p app/src/main/res/values
mkdir -p .github/workflows

# settings.gradle
cat > settings.gradle <<'EOF'
rootProject.name = "KidsDrawingStudio"
include ':app'
EOF

# build.gradle (project)
cat > build.gradle <<'EOF'
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:8.1.1"
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}
EOF

# app/build.gradle
cat > app/build.gradle <<'EOF'
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}

android {
    namespace 'com.kids.drawingstudio'
    compileSdk 34

    defaultConfig {
        applicationId "com.kids.drawingstudio"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions { sourceCompatibility JavaVersion.VERSION_17; targetCompatibility JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.9.0"
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
}
EOF

# AndroidManifest.xml
cat > app/src/main/AndroidManifest.xml <<'EOF'
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.kids.drawingstudio">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="28" />

    <application
        android:allowBackup="true"
        android:label="Kids Drawing Studio"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        <activity android:name=".MainActivity"
            android:exported="true"
            android:configChanges="orientation|screenSize|keyboardHidden">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
EOF

# layout
cat > app/src/main/res/layout/activity_main.xml <<'EOF'
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <WebView
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>
EOF

# strings.xml
cat > app/src/main/res/values/strings.xml <<'EOF'
<resources>
    <string name="app_name">Kids Drawing Studio</string>
</resources>
EOF

# MainActivity.kt
cat > app/src/main/java/com/kids/drawingstudio/MainActivity.kt <<'EOF'
package com.kids.drawingstudio

import android.Manifest
import android.content.ContentValues
import android.content.pm.PackageManager
import android.graphics.BitmapFactory
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.MediaStore
import android.util.Base64
import android.webkit.JavascriptInterface
import android.webkit.WebChromeClient
import android.webkit.WebSettings
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import java.io.OutputStream
import java.text.SimpleDateFormat
import java.util.*

class MainActivity : AppCompatActivity() {
    private lateinit var web: WebView

    private val requestWritePermission =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            if (granted) {
                Toast.makeText(this, "Kebenaran diterima. Cuba simpan semula.", Toast.LENGTH_SHORT).show()
            } else {
                Toast.makeText(this, "Kebenaran diperlukan untuk simpan pada beberapa peranti lama.", Toast.LENGTH_LONG).show()
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        web = findViewById(R.id.webview)
        web.webViewClient = WebViewClient()
        web.webChromeClient = WebChromeClient()
        val settings: WebSettings = web.settings
        settings.javaScriptEnabled = true
        settings.allowFileAccess = true
        settings.domStorageEnabled = true
        settings.setSupportZoom(true)
        settings.builtInZoomControls = true
        settings.displayZoomControls = false

        web.addJavascriptInterface(AndroidBridge(), "Android")

        web.loadUrl("file:///android_asset/www/index.html")

        if (Build.VERSION.SDK_INT <= 28) {
            if (ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
                requestWritePermission.launch(Manifest.permission.WRITE_EXTERNAL_STORAGE)
            }
        }
    }

    inner class AndroidBridge {
        @JavascriptInterface
        fun saveImage(base64Data: String) {
            runOnUiThread {
                try {
                    val bytes = Base64.decode(base64Data, Base64.DEFAULT)
                    val filename = "kids_drawing_${SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(Date())}.png"

                    val fos: OutputStream?
                    var imageUri: Uri? = null
                    val resolver = contentResolver

                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                        val contentValues = ContentValues().apply {
                            put(MediaStore.MediaColumns.DISPLAY_NAME, filename)
                            put(MediaStore.MediaColumns.MIME_TYPE, "image/png")
                            put(MediaStore.MediaColumns.RELATIVE_PATH, "DCIM/KidsDrawingStudio")
                            put(MediaStore.Images.Media.IS_PENDING, 1)
                        }
                        val uri = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
                        if (uri == null) {
                            Toast.makeText(this@MainActivity, "Gagal simpan imej.", Toast.LENGTH_SHORT).show()
                            return@runOnUiThread
                        }
                        fos = resolver.openOutputStream(uri)
                        imageUri = uri
                        fos?.write(bytes)
                        fos?.close()
                        contentValues.clear()
                        contentValues.put(MediaStore.Images.Media.IS_PENDING, 0)
                        resolver.update(imageUri, contentValues, null, null)
                    } else {
                        val imagesDir = MediaStore.Images.Media.insertImage(resolver, BitmapFactory.decodeByteArray(bytes,0,bytes.size), filename, "Kids Drawing Studio")
                        if (imagesDir == null) {
                            Toast.makeText(this@MainActivity, "Gagal simpan imej pada peranti lama.", Toast.LENGTH_SHORT).show()
                            return@runOnUiThread
                        }
                        imageUri = Uri.parse(imagesDir)
                    }

                    Toast.makeText(this@MainActivity, "Imej disimpan ke Galeri.", Toast.LENGTH_SHORT).show()

                    imageUri?.let {
                        web.evaluateJavascript("window.onImageSaved && window.onImageSaved('${it.toString()}');", null)
                    }
                } catch (e: Exception) {
                    e.printStackTrace()
                    Toast.makeText(this@MainActivity, "Ralat semasa menyimpan imej.", Toast.LENGTH_LONG).show()
                }
            }
        }
    }

    override fun onBackPressed() {
        if (web.canGoBack()) web.goBack() else super.onBackPressed()
    }
}
EOF

# index.html (app web)
cat > app/src/main/assets/www/index.html <<'EOF'
<!doctype html>
<html lang="ms">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Kids Drawing Studio (App)</title>
  <style>
    body{font-family:Arial,Helvetica,sans-serif;margin:0;background:#f7fbff;color:#111}
    header{background:#ff6ea1;color:#fff;padding:12px;text-align:center}
    .controls{display:flex;gap:8px;padding:10px;align-items:center;flex-wrap:wrap}
    button,input[type=color],input[type=range]{padding:8px;border-radius:8px;border:1px solid #ddd}
    #canvasWrap{padding:12px;display:flex;flex-direction:column;align-items:center}
    canvas{background:#fff;border:1px solid #e6e6e6;border-radius:6px;max-width:100%}
    footer{padding:10px;text-align:center;font-size:12px;color:#666}
  </style>
</head>
<body>
  <header><h1>Kids Drawing Studio</h1><p style="margin:4px 0 0;font-size:14px">App versi mudah — lukis, rujuk contoh & simpan ke peranti</p></header>

  <div class="controls">
    <label>Warna <input id="color" type="color" value="#000000"></label>
    <label>Saiz <input id="size" type="range" min="1" max="40" value="4"></label>
    <button id="eraser">Pemadam</button>
    <button id="undo">Undo</button>
    <button id="clear">Padam</button>
    <button id="save">Simpan ke Peranti</button>
  </div>

  <div id="canvasWrap">
    <canvas id="draw" width="720" height="480" aria-label="Area lukisan"></canvas>
    <p style="font-size:13px;color:#666;margin-top:8px">Petua: Gunakan jari atau stylus. Tekan Simpan untuk simpan ke Galeri.</p>
  </div>

  <section style="padding:12px">
    <h3>Contoh pantas (ketik untuk muat panduan)</h3>
    <div style="display:flex;gap:8px;flex-wrap:wrap">
      <button class="example" data-type="pikachu">Pikachu</button>
      <button class="example" data-type="mickey">Mickey</button>
      <button class="example" data-type="anime">Anime Boy</button>
    </div>
  </section>

  <footer>&copy; 2025 Kids Drawing Studio — Buat karya sendiri & kongsi bersama keluarga</footer>

  <script>
    // Basic drawing logic
    const canvas = document.getElementById('draw');
    const ctx = canvas.getContext('2d');
    let drawing=false, lastX=0, lastY=0;
    let color = document.getElementById('color').value;
    let size = parseInt(document.getElementById('size').value,10);
    let isEraser = false;
    const undoStack = [];

    function saveState(){ try{ if(undoStack.length>20) undoStack.shift(); undoStack.push(canvas.toDataURL()); }catch(e){} }
    function restoreState(){ if(!undoStack.length) return; const img = new Image(); img.src = undoStack.pop(); img.onload = ()=>{ ctx.clearRect(0,0,canvas.width,canvas.height); ctx.drawImage(img,0,0); } }

    function setBrush(){
      ctx.lineJoin='round'; ctx.lineCap='round'; ctx.lineWidth = size; ctx.strokeStyle = isEraser ? '#ffffff' : color;
    }

    function getPos(e){
      const rect = canvas.getBoundingClientRect();
      if(e.touches && e.touches[0]) return {x: e.touches[0].clientX - rect.left, y: e.touches[0].clientY - rect.top};
      return {x: e.clientX - rect.left, y: e.clientY - rect.top};
    }

    function start(e){ e.preventDefault(); saveState(); drawing=true; const p=getPos(e); lastX=p.x; lastY=p.y; }
    function draw(e){ if(!drawing) return; e.preventDefault(); const p=getPos(e); setBrush(); ctx.beginPath(); ctx.moveTo(lastX,lastY); ctx.lineTo(p.x,p.y); ctx.stroke(); lastX=p.x; lastY=p.y; }
    function stop(e){ drawing=false; }

    canvas.addEventListener('mousedown', start); canvas.addEventListener('touchstart', start,{passive:false});
    canvas.addEventListener('mousemove', draw); canvas.addEventListener('touchmove', draw,{passive:false});
    canvas.addEventListener('mouseup', stop); canvas.addEventListener('mouseout', stop); canvas.addEventListener('touchend', stop);

    document.getElementById('color').addEventListener('input', (e)=>{ color=e.target.value; isEraser=false; });
    document.getElementById('size').addEventListener('input', (e)=>{ size=parseInt(e.target.value,10); });
    document.getElementById('eraser').addEventListener('click', ()=>{ isEraser=true; });
    document.getElementById('undo').addEventListener('click', ()=>{ restoreState(); });
    document.getElementById('clear').addEventListener('click', ()=>{ if(confirm('Padam semua lukisan?')){ ctx.clearRect(0,0,canvas.width,canvas.height); undoStack.length=0; } });

    ctx.fillStyle = '#ffffff'; ctx.fillRect(0,0,canvas.width,canvas.height);

    document.getElementById('save').addEventListener('click', ()=> {
      const dataUrl = canvas.toDataURL('image/png');
      const base64 = dataUrl.split(',')[1];
      if (window.Android && window.Android.saveImage) {
        window.Android.saveImage(base64);
      } else {
        const w = window.open();
        if (w) {
          w.document.body.style.margin = '0';
          const img = new Image();
          img.src = dataUrl;
          img.style.maxWidth = '100%';
          img.onload = ()=> w.document.body.appendChild(img);
        } else {
          alert('Tidak dapat akses fungsi simpan pada peranti ini.');
        }
      }
    });

    document.querySelectorAll('.example').forEach(btn=>{
      btn.addEventListener('click', ()=> {
        const t = btn.getAttribute('data-type');
        drawGuide(t);
      });
    });

    function drawGuide(type){
      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle = '#ffffff'; ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.strokeStyle = '#222'; ctx.lineWidth = 3;
      if(type==='pikachu'){
        ctx.beginPath(); ctx.ellipse(360,160,80,70,0,0,Math.PI*2); ctx.stroke();
        ctx.beginPath(); ctx.arc(330,180,10,0,Math.PI*2); ctx.fillStyle='red'; ctx.fill();
        ctx.beginPath(); ctx.arc(390,180,10,0,Math.PI*2); ctx.fillStyle='red'; ctx.fill();
      } else if(type==='mickey'){
        ctx.beginPath(); ctx.ellipse(360,160,80,80,0,0,Math.PI*2); ctx.stroke();
        ctx.beginPath(); ctx.ellipse(300,100,40,40,0,0,Math.PI*2); ctx.stroke();
        ctx.beginPath(); ctx.ellipse(420,100,40,40,0,0,Math.PI*2); ctx.stroke();
      } else if(type==='anime'){
        ctx.beginPath(); ctx.ellipse(360,160,80,100,0,0,Math.PI*2); ctx.stroke();
        ctx.beginPath(); ctx.ellipse(330,150,12,18,0,0,Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.ellipse(390,150,12,18,0,0,Math.PI*2); ctx.fill();
      }
    }

    window.onImageSaved = function(uri){ alert('Imej disimpan: ' + uri); }
  </script>
</body>
</html>
EOF

# README.md
cat > README.md <<'EOF'
# Kids Drawing Studio — Android APK (WebView app)

Ringkasan:
- Aplikasi ini membungkus laman web studio lukisan (HTML5 canvas) dalam sebuah WebView Android.
- Menyokong simpan lukisan ke galeri menggunakan JavaScript bridge `Android.saveImage(base64)`.

Cara bina APK:
1. Pasang Android Studio (sokong gradle/AGP versi yang sesuai).
2. Clone repo dan buka projek di Android Studio.
3. Sambungkan peranti Android atau gunakan emulator.
4. Klik Run -> pilih device -> Android Studio akan bina & pasang APK.
5. Untuk release APK:
   - Build -> Generate Signed Bundle / APK -> ikut arahan (perlu keystore).

Nota penting:
- Saya set `minSdk 21` dan `targetSdk 34`.
- Untuk peranti Android < 10 (API <= 28) mungkin memerlukan kebenaran storan runtime.
EOF

# GitHub Actions workflow
cat > .github/workflows/android-build.yml <<'EOF'
name: Android CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '17'

    - name: Cache Gradle
      uses: actions/cache@v4
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
        restore-keys: |
          ${{ runner.os }}-gradle-

    - name: Build Debug APK
      run: |
        chmod +x gradlew
        ./gradlew assembleDebug --no-daemon

    - name: Upload APK
      uses: actions/upload-artifact@v4
      with:
        name: app-debug-apk
        path: app/build/outputs/apk/debug/*.apk
EOF

# Git add, commit and push
git add .
git commit -m "Initial commit: Android WebView Kids Drawing Studio app + CI build workflow"
git branch -M main
git push -u origin main

echo "Done. Files created, committed and pushed to origin/main."
