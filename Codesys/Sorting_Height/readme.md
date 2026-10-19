# Factory I/O Sorting by Height – PLC Ladder Logic (CODESYS)

Simulasi penyortir box berdasarkan tinggi di **Factory I/O** (scene Sorting by Height) yang dikendalikan program **ladder logic CODESYS** (IEC 61131-3) dengan desain modular berbasis function block.

> **Objective:** Sort boxes by height onto two side conveyors (tall boxes to the left, short boxes to the right) and count the sorted boxes.

Sistem menerima box dari conveyor masuk, memuatnya ke posisi sensor tinggi, lalu mengklasifikasikannya sebagai box tinggi atau box pendek. Box tinggi dipindahkan ke conveyor kiri, box pendek ke conveyor kanan. Box yang keluar di ujung masing-masing conveyor dihitung, dan totalnya ditampilkan pada display.

## Demo

[TODO: Tonton video demo](link-video)

## Factory I/O Scene

![Tampilan scene Sorting by Height](assets/tampilan-scene.png)

*Tampilan scene: conveyor masuk, conveyor pemuat, dua transfer (kiri dan kanan), dua side conveyor, dan panel kontrol.*

### Control Panel

![Control panel](assets/panel-kontrol.png)

*Panel operator: Start, Stop, Reset, Emergency Stop, dan display jumlah box tersortir.*

### Sensor dan Actuator

| Sensor | Fungsi |
|---|---|
| `Pallet_Sensor` | Deteksi box di posisi pemuatan |
| `Loaded` | Konfirmasi box sudah termuat |
| `High_Sensor` | Deteksi box tinggi |
| `Low_Sensor` | Deteksi box pendek |
| `At_Entry_Left`, `At_Entry_Right` | Box sampai di pintu masuk side conveyor kiri/kanan |
| `At_left_Exit`, `At_right_Exit` | Box keluar dari side conveyor kiri/kanan |

| Actuator | Fungsi |
|---|---|
| `Conveyor_Entry` | Conveyor masuk |
| `Load_Conveyor` | Conveyor pemuat |
| `Transfer_Left`, `Transfer_Right` | Transfer box ke kiri/kanan |
| `Conveyor_Left`, `Conveyor_Right` | Side conveyor kiri/kanan |
| `Display_Dig` | Display jumlah box tersortir |

## Tools

| Item | Keterangan |
|---|---|
| Simulasi | Factory I/O (scene Sorting by Height) |
| PLC / IDE | CODESYS (soft PLC) |
| Bahasa pemrograman | Ladder Diagram (LD) |
| Blok standar | CTU, R_TRIG, ADD, TO_WORD, kontak edge (P/N) |

## Push Button Functions

| Tombol | Fungsi |
|---|---|
| **Start** | Menjalankan sistem. `System_Running` di-latch aktif (seal-in). |
| **Stop** | Menghentikan sistem. |
| **E-Stop** | Menghentikan sistem secara paksa dan di-latch. Sistem tidak dapat berjalan sampai latch dilepas dengan Reset. |
| **Reset** | Melepas latch E-Stop dan mereset counter (lihat ketentuan di bawah). |

**Ketentuan tombol Reset:**

Reset punya dua fungsi yang sengaja dipisahkan agar jumlah box tidak hilang tanpa disengaja:

1. **Setelah E-Stop:** tekan Reset sekali untuk melepas latch E-Stop. Counter tetap utuh. Lepas tombol, lalu tekan Reset sekali lagi jika ingin mengosongkan counter.
2. **Setelah Stop:** saat sistem tidak berjalan, tekan Reset untuk mengosongkan counter.

Counter tidak dapat direset saat sistem sedang berjalan.

## Process Flow (Normal Operation)

1. **Start**: Tombol `Start` mengaktifkan dan me-latch `System_Running`.
2. **Box masuk**: Conveyor masuk membawa box ke posisi pemuatan. Saat `Pallet_Sensor` mendeteksi box, conveyor masuk berhenti dan conveyor pemuat memuat box.
3. **Klasifikasi tinggi**: `High_Sensor` dan `Low_Sensor` terpasang pada dua ketinggian berbeda. Box tinggi terdeteksi oleh `High_Sensor`, sehingga dikunci pada latch `High_Item`. Box pendek tidak mengaktifkan `High_Sensor`, sehingga `High_Item` tetap tidak aktif dan box ditangani oleh jalur box pendek.
4. **Transfer**: Instance `Height_Sorter` untuk box tinggi memindahkan box ke kiri. Instance untuk box pendek (hanya aktif saat `High_Item` tidak aktif) memindahkan box ke kanan.
5. **Side conveyor**: Setelah box sampai di pintu masuk side conveyor, transfer berhenti dan side conveyor membawa box ke ujung conveyor.
6. **Menghitung box**: Saat box melewati sensor di ujung conveyor kiri atau kanan, rising edge menaikkan counter masing-masing sisi. Kedua counter dijumlahkan, dikonversi ke WORD, lalu dikirim ke display.
7. **E-Stop**: Saat E-Stop ditekan, sinyal dikonfirmasi lalu di-latch menjadi `E_StopLatched`. Semua blok berhenti sampai latch dilepas dengan Reset.

## Program Structure

Program terdiri dari satu program utama (`PLC_PRG`) dan tiga function block yang dipakai ulang.

### PLC_PRG

| Rung | Fungsi |
|---|---|
| 1–3 | E-Stop: konfirmasi sinyal dan latch |
| 4 | Reset melepas latch E-Stop |
| 5 | Start / Stop sistem (`System_Running`) |
| 6 | Izin conveyor masuk |
| 7 | Output conveyor masuk |
| 8 | Box terdeteksi: stop conveyor masuk |
| 9 | Latch proses pemuatan |
| 10 | Output conveyor pemuat |
| 11 | Latch box tinggi (`High_Item`), harus dievaluasi sebelum rung 14 |
| 12 | `Height_Sorter` HIGH: transfer kiri |
| 13 | Side conveyor kiri |
| 14 | `Height_Sorter` LOW: transfer kanan (bergantung pada rung 11) |
| 15 | Side conveyor kanan |
| 16 | Counter dan display |

### Function Block

| FB | Fungsi | Dipakai |
|---|---|---|
| `Height_Sorter` | Mendeteksi box, meminta transfer, dan menghentikan transfer | 2x (HIGH dan LOW) |
| `Side_Conveyor` | Menjalankan side conveyor dari pintu masuk sampai ujung | 2x (kiri dan kanan) |
| `Sorting_Counter` | Menghitung box kiri dan kanan, menjumlahkan, dan konversi ke display | 1x |

## Techniques Used

- **E-Stop latch**: sinyal E-Stop dikonfirmasi lalu dikunci menjadi satu sinyal `E_StopLatched` yang dipakai semua blok. Latch hanya dilepas oleh Reset saat tombol E-Stop sudah dilepas.
- **Desain modular**: tiap fungsi dibuat sebagai function block dan dipakai ulang (`Height_Sorter` dan `Side_Conveyor` masing-masing dua instance), sehingga sisi kiri dan kanan memakai logika yang sama.
- **Reset counter dua tahap**: tekanan Reset yang melepas latch E-Stop ditangkap dan diabaikan oleh counter, sehingga reset counter baru terjadi pada tekanan berikutnya. Ini mencegah jumlah box hilang tanpa disengaja.
- **Penghitungan berbasis edge**: counter dinaikkan oleh rising edge sensor ujung conveyor, sehingga satu box dihitung satu kali.
- **Latching bertahap**: kondisi seperti box terdeteksi dan izin transfer disimpan pada latch, lalu dilepas oleh event berikutnya (misalnya box sampai di pintu masuk side conveyor).

## Design Notes

- Urutan rung penting pada dua tempat: `High_Item` (rung 11) harus dievaluasi sebelum `Height_Sorter` LOW (rung 14), dan `Stop_Transfer` pada `Height_Sorter` dipakai oleh latch pada scan berikutnya.
- Fungsi Reset ganda (melepas latch E-Stop dan mereset counter) disengaja dan dipisahkan oleh logika dua tahap di `Sorting_Counter`.

## Communication

Program CODESYS terhubung ke Factory I/O melalui **Modbus TCP**. CODESYS berperan sebagai slave dan Factory I/O sebagai client yang terhubung ke `127.0.0.1:502`.

[TODO: sesuaikan Slave ID dan alamat dengan konfigurasi projectmu]

Arah data dilihat dari sisi PLC:

- **Coil** (Factory I/O ke PLC): sensor dan tombol, terbaca di CODESYS sebagai input.
- **Discrete Input dan Input Register** (PLC ke Factory I/O): aktuator dan counter, ditulis dari output CODESYS.

### PLC Inputs

| Coil | Label di Factory I/O | Variabel |
|---|---|---|
| [TODO] | [TODO] | [TODO] |

### PLC Outputs

| Alamat | Label di Factory I/O | Variabel |
|---|---|---|
| [TODO] | [TODO] | [TODO] |

## Testing

Sudah diuji di Factory I/O:

- Fungsi semua tombol (Start, Stop, E-Stop, Reset)
- Proses sortir normal (box tinggi ke kiri, box pendek ke kanan)
- Reset counter: setelah Stop, dan dua tahap setelah E-Stop

[TODO: tambahkan hasil tes tepi bila sudah dicoba: E-Stop saat box di tengah transfer, dua box berdekatan, Stop lalu Start]

## Future Improvements

- `Side_Conveyor` menerima `System_Running` dari program utama agar hanya ada satu sumber status sistem.
- Tambahan HMI untuk menampilkan status dan counter.
- Integrasi dengan TIA Portal.

## Author

**Izzuddin Alqassam**, Teknik Elektro, Universitas Hasanuddin
Fokus: sistem kontrol, PLC, otomasi industri