# Panduan Dokumentasi Integrasi AxManager

AxManager memungkinkan aplikasi klien (pihak ketiga) untuk mengeksekusi perintah shell dan skrip dengan hak akses tinggi (Privilege - ADB/Root) melalui layanan AxManager.

Ada dua cara utama yang bisa Anda gunakan untuk mengintegrasikan layanan ini agar bisa menjalankan perintah seperti `uname -r` dan sebagainya:

1. **Menggunakan Axerish dengan libsu** (Direkomendasikan untuk aplikasi yang mengandalkan penggunaan shell standar)
2. **Menggunakan Axeron API Langsung** (Lebih mendasar, tanpa harus bergantung pada `libsu`)

Berikut ini adalah langkah-langkah implementasinya:

---

## 1. Menggunakan Axerish + libsu

Axerish adalah *shell wrapper* bawaan AxManager yang akan otomatis menghubungkan aplikasi Anda ke antarmuka populer `libsu` buatan *topjohnwu*. Metode ini sangat mudah karena Anda bisa menggunakan gaya eksekusi `libsu` secara transparan namun berjalan di konteks AxManager.

### Langkah 1: Persiapan Dependensi

Tambahkan pustaka `libsu` dan Axeron API ke dalam file `build.gradle.kts` (pada tingkat *app*):

```kotlin
dependencies {
    // Pustaka libsu
    implementation("com.github.topjohnwu.libsu:core:<versi_terbaru>")

    // Axeron API
    implementation("com.github.fahrez182.AxManager:api:<versi_terbaru>")
}
```

### Langkah 2: Inisialisasi di dalam `Application`

Buat atau modifikasi kelas `Application` kustom Anda untuk melakukan inisialisasi `Axerish` dan mengubah konfigurasi bawaan `Shell.Builder` milik `libsu` supaya diarahkan menggunakan Axerish.

```kotlin
import android.app.Application
import com.topjohnwu.superuser.Shell
import frb.axeron.Axerish

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 1. Inisialisasi Axerish untuk aplikasi Anda
        Axerish.initialize(packageName)

        // 2. Atur libsu untuk menggunakan eksekutor Axerish sebagai ganti 'su' atau 'sh'
        Shell.setDefaultBuilder(
            Shell.Builder.create()
                .setCommands("sh", Axerish.axrun_path.absolutePath)
        )
    }
}
```

### Langkah 3: Eksekusi Perintah

Setelah terkonfigurasi, jalankan saja fungsi `Shell.cmd()` seperti biasa. Segala perintah akan diteruskan dan dieksekusi dengan *privilege* tinggi milik AxManager!

```kotlin
import com.topjohnwu.superuser.Shell

// Menjalankan perintah dengan hak akses tinggi (misal: cek kernel)
val result = Shell.cmd("uname -r").exec()

if (result.isSuccess) {
    println("Versi Kernel: ${result.out.joinToString(\"\n\")}")
}
```

---

## 2. Menggunakan Axeron API Langsung

Apabila Anda membutuhkan tingkat integrasi yang lebih mendalam, atau jika Anda memang tidak menggunakan pustaka `libsu`, Anda dapat langsung terhubung (*bind*) ke layanan IPC milik AxManager.

### Langkah 1: Persiapan Dependensi

Tambahkan pustaka Axeron API dan Axeron Provider di dalam file `build.gradle.kts` Anda:

```kotlin
dependencies {
    implementation("com.github.fahrez182.AxManager:api:<versi_terbaru>")
    implementation("com.github.fahrez182.AxManager:provider:<versi_terbaru>")
}
```

### Langkah 2: Deklarasi Provider

Tambahkan *provider* ini di dalam file `AndroidManifest.xml` aplikasi Anda. Hal ini dilakukan agar AxManager dapat mendeteksi aplikasi Anda dan mengirimkan koneksi (*IPC binder*) secara langsung.

```xml
<provider
    android:name="frb.axeron.provider.AxeronProvider"
    android:authorities="${applicationId}.axeron"
    android:exported="true"
    android:multiprocess="false" />
```

### Langkah 3: Dengarkan Koneksi & Eksekusi Perintah

Tunggu hingga layanan AxManager terkoneksi dengan memantau fungsi `Axeron.addBinderReceivedListenerSticky`, kemudian gunakan kelas `Axeron.newProcess` untuk menjalankan perintah yang diinginkan.

```kotlin
import frb.axeron.api.Axeron

// Dengarkan status koneksi ke layanan AxManager
Axeron.addBinderReceivedListenerSticky {
    // Di sini aplikasi kita telah berhasil terhubung ke AxManager!

    // Eksekusi sebuah perintah secara langsung
    val process = Axeron.newProcess("uname -r")

    // Membaca luaran/output secara stream
    process.inputStream.bufferedReader().useLines { lines ->
        lines.forEach { println("Output: $it") }
    }

    // Tunggu proses perintah hingga tuntas
    val exitCode = process.waitFor()
    println("Perintah selesai dengan kode keluaran (exit code): $exitCode")
}
```


---
*Catatan:* Pastikan untuk memperbarui kode `<versi_terbaru>` pada bagian dependensi dengan versi rilis API yang mutakhir.
