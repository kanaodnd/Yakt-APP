# Panduan Dokumentasi Integrasi Eksekusi Command (YAKT & AxManager)

Dokumen ini menjelaskan berbagai cara yang dapat digunakan untuk mengeksekusi perintah shell (seperti `uname -r`) dan skrip dengan hak akses tinggi (Root/Privilege). Terdapat metode asli (Native) yang digunakan oleh YAKT dan metode integrasi melalui layanan AxManager.

---

## 1. Metode Native YAKT (Menggunakan `Runtime.getRuntime()`)

Secara *default*, YAKT mengeksekusi perintah shell langsung menggunakan *built-in library* Java/Kotlin tanpa bergantung pada *library* pihak ketiga seperti `libsu`. Pendekatan ini sangat ringan dan menggunakan pemanggilan `su` langsung melalui kelas `Runtime`.

### Eksekusi Perintah Secara Langsung (Contoh: `uname -r`)

Untuk menjalankan perintah yang memerlukan *root* dan membaca hasilnya (*output*), Anda dapat menggunakan cara berikut:

```kotlin
fun executeRootCommand(command: String): String {
    return try {
        // Mengeksekusi perintah melalui binary root (su)
        val process = Runtime.getRuntime().exec(arrayOf("su", "-c", command))

        // Membaca luaran/output secara langsung
        val output = process.inputStream.bufferedReader().use { it.readText() }

        // Tunggu proses perintah hingga tuntas
        process.waitFor()

        output.trim()
    } catch (e: Exception) {
        e.printStackTrace()
        "Error: ${e.message}"
    }
}

// Cara menggunakan:
// val kernelVersion = executeRootCommand("uname -r")
// println("Versi Kernel: $kernelVersion")
```

---

## 2. Menggunakan Axerish + libsu (Layanan AxManager)

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

## 3. Menggunakan Axeron API Langsung (Layanan AxManager)

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

## 4. Deteksi Environment (Root vs Non-Root / AxManager)

Terkadang skrip atau aplikasi perlu mengetahui apakah ia dieksekusi dalam environment **Root** atau **Non-Root** (seperti ADB shell via AxManager). Hal ini bisa dilakukan dengan menjalankan perintah `id` dan mengecek *output*-nya.

### Non-Root / AxManager (Shell)
Jika output dari perintah `id` mengandung `shell`, ini menandakan environment berjalan menggunakan *privilege* ADB (Non-Root).

Contoh output:
```text
uid=2000(shell) gid=2000(shell) groups=2000(shell),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid),3012(readtracefs) context=u:r:shell:s0
```

### Root User
Jika output dari perintah `id` mengandung `root`, ini menandakan environment memiliki akses Superuser/Root secara penuh.

Contoh output:
```text
uid=0(root) gid=0(root) groups=0(root) context=u:r:su:s0
```

---
*Catatan:* Pastikan untuk memperbarui kode `<versi_terbaru>` pada bagian dependensi dengan versi rilis API yang mutakhir.