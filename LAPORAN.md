# Laporan Praktikum Mandiri: MVC, MVVM, dan Pengujian Flutter

## Identitas Mahasiswa

- **Nama:** Endra Gustin
- **NIM:** 362558302145
- **Program Studi:** D4 Teknologi Rekayasa Perangkat Lunak — Politeknik Negeri Banyuwangi
- **Mata Kuliah:** Pemrograman Perangkat Bergerak
- **Tautan Repository:** https://www.google.com/search?q=https://github.com/EndraKunn/prak\_arsitektur\_tugas

## Cara Menjalankan Aplikasi dan Pengujian

Lingkungan pengujian lokal: **Flutter 3.47.1 (channel stable)** dan **Dart 3.13.1** pada Windows (PowerShell).

Jalankan perintah berikut dari dalam folder proyek `prak_arsitektur_tugas` (yang berisi `pubspec.yaml`):

```powershell
flutter --version
flutter pub get
flutter analyze
flutter test
flutter run -d chrome
```

## Tabel Perbandingan Arsitektur (MVC vs MVVM)

| Pertanyaan | Versi MVC | Versi MVVM |
| :--- | :--- | :--- |
| **Ketika filter diketuk, class mana yang menerima aksi dan menyimpan pilihan?** | `_LayarMvcState` (`lib/mvc/layar_mvc.dart`) menerima callback `onUbahFilter` melalui metode `_ubahFilter(bool aktif)`, lalu meneruskannya ke class `PengendaliTugas` (`lib/mvc/pengendali_tugas.dart`) melalui metode `ubahFilter(bool aktif)` yang menyimpan pilihan pada field `_hanyaBelumSelesai`. | `TugasViewModel` (`lib/mvvm/tugas_view_model.dart`) menerima aksi langsung dari callback `onUbahFilter: _viewModel.ubahFilter` pada metode `ubahFilter(bool aktif)` dan menyimpan pilihan pada field `_hanyaBelumSelesai`. |
| **Apa yang memicu rebuild? Apa akibatnya jika pemicu itu dihapus?** | **Pemicu:** Pemanggilan `setState(...)` di dalam `_LayarMvcState._ubahFilter(bool aktif)` (serta `_tambah` dan `_ubahStatus`).<br>**Hasil percobaan sebelum dipulihkan:** Ketika `setState(...)` pada `_ubahFilter` dihapus sementara, nilai `_hanyaBelumSelesai` di `PengendaliTugas` tetap berubah, tetapi `build()` tidak dijalankan ulang sehingga tugas yang sudah selesai (`Rancang UI Flutter`) tetap tampil dan `flutter test test/filter_test.dart` gagal pada skenario MVC (`+1 -1`, baris 34).<br>**Setelah dipulihkan:** Setelah `setState(...)` dikembalikan, tampilan tersaring sesuai chip dan seluruh 7 test lulus (`+7: All tests passed!`). | **Pemicu:** Pemanggilan `notifyListeners()` di dalam `TugasViewModel.ubahFilter(bool aktif)` (serta `tambah` dan `ubahStatus`) yang diamati oleh `ListenableBuilder` pada `_LayarMvvmState`.<br>**Hasil percobaan sebelum dipulihkan:** Ketika `notifyListeners()` pada `ubahFilter` dinonaktifkan sementara, state filter di `TugasViewModel` berubah, tetapi `ListenableBuilder` tidak memanggil ulang `builder` sehingga `Rancang UI Flutter` tetap tampil dan `flutter test test/filter_test.dart` gagal pada skenario MVVM (`+1 -1`, baris 34).<br>**Setelah dipulihkan:** Setelah `notifyListeners()` dikembalikan, `ListenableBuilder` memperbarui UI dengan benar dan seluruh 7 test lulus (`+7: All tests passed!`). |
| **Kapan Anda memilih masing-masing pola?** | **Skenario cocok:** Layar tunggal sederhana dengan sedikit aksi pengguna dan state lokal yang tidak memiliki banyak turunan data.<br>**Kelebihan:** Alur dari aksi UI (`_ubahFilter`) hingga pembaruan tampilan (`setState`) mudah ditelusuri langsung di satu berkas layar (`layar_mvc.dart`) tanpa boilerplate `ChangeNotifier` dan `dispose()`.<br>**Biaya/risiko:** Setiap aksi baru yang memengaruhi tampilan (seperti penambahan filter) wajib dibungkus `setState()` di View; jika terlewat, data di `PengendaliTugas` berubah diam-diam tanpa memperbarui layar seperti terbukti pada percobaan regresi. | **Skenario cocok:** Layar yang memiliki beberapa UI state turunan sekaligus (`daftarTerlihat` yang berbeda dari `daftar` lengkap, `totalTugas`, `jumlahSelesai`, dan `galat`), fitur penyaringan/pencarian, serta kebutuhan uji notifikasi dan state secara terisolasi.<br>**Kelebihan:** Seluruh UI state terpusat di `TugasViewModel`, View (`LayarMvvm`) cukup menghubungkan referensi metode (`_viewModel.ubahFilter`), dan unit test (`test/logika_test.dart`) dapat memverifikasi perubahan state sekaligus urutan notifikasi `[0, 0, 1]` tanpa merender widget.<br>**Biaya/risiko:** Membutuhkan pengelolaan lifecycle (`_viewModel.dispose()` di `State.dispose`) dan disiplin memanggil `notifyListeners()` di setiap metode pengubah state agar `ListenableBuilder` selalu sinkron. |




## Keluaran Terminal

### 1. Percobaan Hapus Pemicu Rebuild Sementara — MVC

`setState` dihapus sementara di `_LayarMvcState._ubahFilter(bool aktif)` lalu dijalankan `flutter test test/filter_test.dart`:

```text
PS D:\prak_arsitektur_tugas> flutter test test/filter_test.dart
00:04 +0: MVC menyaring tugas belum selesai dan menjaga ringkasan
══╡ EXCEPTION CAUGHT BY FLUTTER TEST FRAMEWORK ╞════════════════════════════════════════════════════
The following TestFailure was thrown running a test:
Expected: no matching candidates
  Actual: _TextWidgetFinder:<Found 1 widget with text "Rancang UI Flutter": [
            Text("Rancang UI Flutter", dependencies: [DefaultSelectionStyle, DefaultTextStyle,
MediaQuery]),
          ]>
   Which: means one was found but none were expected

This was caught by the test expectation on the following line:
  file:///D:/prak_arsitektur_tugas/test/filter_test.dart line 34
The test description was:
  MVC menyaring tugas belum selesai dan menjaga ringkasan
════════════════════════════════════════════════════════════════════════════════════════════════════
00:04 +0 -1: MVC menyaring tugas belum selesai dan menjaga ringkasan [E]
00:04 +1 -1: Some tests failed.
```

### 2. Percobaan Hapus Pemicu Rebuild Sementara — MVVM

`notifyListeners()` dinonaktifkan sementara di `TugasViewModel.ubahFilter(bool aktif)` lalu dijalankan `flutter test test/filter_test.dart`:

```text
PS D:\prak_arsitektur_tugas> flutter test test/filter_test.dart
00:04 +1: MVVM menyaring tugas belum selesai dan menjaga ringkasan
══╡ EXCEPTION CAUGHT BY FLUTTER TEST FRAMEWORK ╞════════════════════════════════════════════════════
The following TestFailure was thrown running a test:
Expected: no matching candidates
  Actual: _TextWidgetFinder:<Found 1 widget with text "Rancang UI Flutter": [
            Text("Rancang UI Flutter", dependencies: [DefaultSelectionStyle, DefaultTextStyle,
MediaQuery]),
          ]>
   Which: means one was found but none were expected

This was caught by the test expectation on the following line:
  file:///D:/prak_arsitektur_tugas/test/filter_test.dart line 34
The test description was:
  MVVM menyaring tugas belum selesai dan menjaga ringkasan
════════════════════════════════════════════════════════════════════════════════════════════════════
00:04 +1 -1: MVVM menyaring tugas belum selesai dan menjaga ringkasan [E]
00:04 +1 -1: Some tests failed.
```

### 3. Hasil Akhir Setelah Kode Dipulihkan (`flutter analyze` dan `flutter test`)

```text
PS D:\prak_arsitektur_tugas> flutter analyze
Analyzing prak_arsitektur_tugas...
No issues found! (ran in 2.0s)

PS D:\prak_arsitektur_tugas> flutter test
00:05 +7: All tests passed!
```

Ringkasan 7 pengujian yang lulus:
1. `test/smoke_test.dart` — 1 widget test navigasi beranda ke MVC dan MVVM.
2. `test/logika_test.dart` — 2 unit test (`PengendaliTugas` dan `TugasViewModel` termasuk urutan notifikasi `[0, 0, 1]`).
3. `test/layar_test.dart` — 2 widget test form, validasi, dan ringkasan untuk MVC dan MVVM.
4. `test/filter_test.dart` — 2 widget test filter **Belum selesai**, pemulihan daftar, ringkasan, dan *empty state* untuk MVC dan MVVM.
