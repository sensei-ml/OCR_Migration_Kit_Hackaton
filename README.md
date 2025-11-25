# 🚀 Android OCR Migration Kit (Google ML Kit)

![Status](https://img.shields.io/badge/Status-Ready_to_Deploy-success)
![Tech](https://img.shields.io/badge/Tech-Google_ML_Kit_On--Device-blue)
![Language](https://img.shields.io/badge/Language-Kotlin-purple)
![Architecture](https://img.shields.io/badge/Architecture-Edge_AI_%7C_Offline-orange)

## 📋 Descripción y Justificación
Este repositorio facilita la migración inmediata de R-CNN a **Google ML Kit (Vision API v2)**.

**Ventajas Técnicas:**
1.  **Latencia Cero:** Procesamiento en tiempo real (<50ms).
2.  **100% Offline:** Modelo *Bundled* (empaquetado). No requiere internet ni Keys de Cloud.
3.  **Privacidad:** Las imágenes nunca salen del dispositivo.

---

## 🛠️ Guía de Implementación "Copy-Paste"

Sigue estos 5 pasos exactos para tener el MVP corriendo.

### PASO 1: Crear Proyecto
1.  Android Studio > New Project.
2.  Template: **Empty Views Activity**.
3.  Language: **Kotlin**.
4.  Min SDK: **API 24**.

### PASO 2: Inyectar Dependencias
Abre `app/build.gradle` (Module: app) y agrega esto dentro del bloque `dependencies { ... }`:

```groovy
dependencies {
    // --- INICIO BLOQUE OCR ---
    // CameraX
    def camerax_version = "1.3.1"
    implementation "androidx.camera:camera-core:${camerax_version}"
    implementation "androidx.camera:camera-camera2:${camerax_version}"
    implementation "androidx.camera:camera-lifecycle:${camerax_version}"
    implementation "androidx.camera:camera-view:${camerax_version}"

    // ML Kit Text Recognition (Versión Offline)
    implementation 'com.google.android.gms:play-services-mlkit-text-recognition:19.0.0'

    // UI & Threading
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    // --- FIN BLOQUE OCR ---
    
    // ... (Mantén las otras dependencias que ya estaban) ...
}
```
**Dale clic a "Sync Now" arriba a la derecha.**

### PASO 3: Permisos de Cámara
Abre `app/src/main/AndroidManifest.xml`. Pega estas dos líneas justo antes de la etiqueta `<application ...>`:

```
<manifest xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)" ...>

    <uses-feature android:name="android.hardware.camera.any" />
    <uses-permission android:name="android.permission.CAMERA" />

    <application ... >
        </application>

</manifest>
```

### PASO 4: La Interfaz (Layout)

Abre `app/src/main/res/layout/activity_main.xml`. Borra todo y pega esto:

```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    xmlns:app="[http://schemas.android.com/apk/res-auto](http://schemas.android.com/apk/res-auto)"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#000000">

    <androidx.camera.view.PreviewView
        android:id="@+id/viewFinder"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="250dp"
        android:background="#99000000"
        app:layout_constraintBottom_toBottomOf="parent">

        <TextView
            android:id="@+id/tvResult"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:text="Apunta a un texto..."
            android:textColor="#00FF00"
            android:textSize="18sp"
            android:fontFamily="monospace"
            android:textStyle="bold" />
    </ScrollView>

</androidx.constraintlayout.widget.ConstraintLayout>
```

### PASO 5: La Lógica (Kotlin)

Abre `app/src/main/java/com/.../MainActivity.kt`. Borra todo y pega esto:

```
package com.hackathon.ocr // <--- CAMBIAR SI TU PROYECTO TIENE OTRO NOMBRE DE PAQUETE

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.util.Log
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import com.google.mlkit.vision.common.InputImage
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var viewFinder: PreviewView
    private lateinit var tvResult: TextView
    private lateinit var cameraExecutor: ExecutorService
    
    // Instancia del reconocedor (Modelo V2 Bundled)
    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewFinder = findViewById(R.id.viewFinder)
        tvResult = findViewById(R.id.tvResult)

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            ActivityCompat.requestPermissions(this, REQUIRED_PERMISSIONS, REQUEST_CODE_PERMISSIONS)
        }

        cameraExecutor = Executors.newSingleThreadExecutor()
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(viewFinder.surfaceProvider)
            }
            val imageAnalyzer = ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also {
                    it.setAnalyzer(cameraExecutor) { imageProxy ->
                        processImageProxy(imageProxy)
                    }
                }
            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, cameraSelector, preview, imageAnalyzer)
            } catch (exc: Exception) {
                Log.e(TAG, "Error al vincular cámara", exc)
            }
        }, ContextCompat.getMainExecutor(this))
    }

    @androidx.annotation.OptIn(ExperimentalGetImage::class)
    private fun processImageProxy(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image
        if (mediaImage != null) {
            val image = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)
            recognizer.process(image)
                .addOnSuccessListener { visionText ->
                    if (visionText.text.isNotBlank()) {
                         tvResult.text = visionText.text
                    }
                }
                .addOnFailureListener { e -> Log.e(TAG, "Error OCR", e) }
                .addOnCompleteListener { imageProxy.close() }
        } else {
            imageProxy.close()
        }
    }

    private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(baseContext, it) == PackageManager.PERMISSION_GRANTED
    }

    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            if (allPermissionsGranted()) {
                startCamera()
            } else {
                Toast.makeText(this, "Se requieren permisos.", Toast.LENGTH_SHORT).show()
                finish()
            }
        }
    }

    companion object {
        private const val TAG = "HackathonOCR"
        private const val REQUEST_CODE_PERMISSIONS = 10
        private val REQUIRED_PERMISSIONS = arrayOf(Manifest.permission.CAMERA)
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}
```
