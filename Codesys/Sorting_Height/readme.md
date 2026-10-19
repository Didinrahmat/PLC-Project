Factory I/O Sorting by Height – PLC Ladder Logic (CODESYS)
Simulasi sistem penyortir barang berdasarkan tinggi di Factory I/O yang dikendalikan program ladder logic CODESYS (IEC 61131-3).

Objective: Sort items by height onto two side conveyors (high items to the left, low items to the right) and count the sorted items.

Sistem menerima barang dari conveyor masuk, memuatnya ke posisi sensor tinggi, lalu mengklasifikasikannya sebagai tinggi atau rendah. Barang tinggi dipindahkan ke conveyor kiri, barang rendah ke conveyor kanan. Barang yang keluar di ujung masing-masing conveyor dihitung dan totalnya ditampilkan pada display.

Demo
[TODO: tambahkan link video demo]

Scene Factory I/O
[TODO: tambahkan screenshot scene]

Tampilan scene: conveyor masuk, conveyor pemuat, dua transfer (kiri dan kanan), dua side conveyor, dan panel kontrol.

Panel Kontrol
[TODO: tambahkan screenshot panel]

Panel operator: Emergency Stop, Start, Stop, Reset, dan display jumlah barang tersortir.

Sensor dan Aktuator
Sensor:
- Pallet_Sensor: deteksi barang di posisi pemuatan
- Loaded: konfirmasi barang sudah termuat
- High_Sensor: deteksi barang tinggi
- Low_Sensor: deteksi barang rendah
- At_Entry_Left, At_Entry_Right: barang sampai di pintu masuk side conveyor kiri/kanan
- At_left_Exit, At_right_Exit: barang keluar dari side conveyor kiri/kanan

Aktuator:
- Conveyor_Entry: conveyor masuk
- Load_Conveyor: conveyor pemuat
- Transfer_Left, Transfer_Right: transfer barang ke kiri/kanan
- Conveyor_Left, Conveyor_Right: side conveyor kiri/kanan
- Display_Dig: display jumlah barang

Tools
Item                  | Keterangan
Simulasi              | Factory I/O (scene Sorting by Height (Basic))
PLC / IDE             | CODESYS (soft PLC)
Bahasa pemrograman    | Ladder Diagram (LD)
Blok standar          | CTU, R_TRIG, F_TRIG, ADD, TO_WORD

Fungsi Push Button
Tombol  | Fungsi
Start   | Menjalankan sistem.
Stop    | Menghentikan sistem.
E-Stop  | Menghentikan sistem secara paksa dan dikunci (latch). Sistem tidak dapat berjalan sampai latch dilepas dengan Reset.
Reset   | Melepas latch E-Stop dan mereset counter (lihat ketentuan di bawah).

Ketentuan tombol Reset:
Reset memiliki dua fungsi yang dipisahkan agar jumlah barang tidak hilang tanpa disengaja:
1. Setelah E-Stop: tekan Reset sekali untuk melepas latch E-Stop. Counter tetap utuh. Tekan Reset sekali lagi (setelah tombol dilepas) jika ingin mengosongkan counter.
2. Setelah Stop: ketika sistem tidak berjalan, tekan Reset untuk mengosongkan counter.
Counter tidak dapat direset saat sistem sedang berjalan.

Alur Proses (Operasi Normal)
1. Start: Tombol Start mengaktifkan dan mengunci (latch) System_Running.
2. Barang masuk: Conveyor masuk membawa barang ke posisi pemuatan. Saat Pallet_Sensor mendeteksi barang, conveyor masuk berhenti dan conveyor pemuat memuat barang.
3. Klasifikasi tinggi: Sensor tinggi (High_Sensor) menentukan jenis barang. Barang tinggi dikunci pada latch High_Item.
4. Transfer: Instance Height_Sorter untuk barang tinggi memindahkan barang ke kiri. Instance untuk barang rendah (aktif saat High_Item tidak aktif) memindahkan barang ke kanan.
5. Side conveyor: Setelah barang sampai di pintu masuk side conveyor, transfer berhenti dan side conveyor membawa barang ke ujung conveyor.
6. Menghitung barang: Saat barang melewati sensor di ujung conveyor kiri atau kanan, rising edge menaikkan counter masing-masing sisi. Jumlah kedua counter dijumlahkan dan dikirim ke display.

Struktur Program
Program dibagi menjadi satu program utama (PLC_PRG) dan tiga function block yang dipakai ulang.

PLC_PRG
Rung  | Fungsi
1–3   | E-Stop: konfirmasi sinyal dan latch
4     | Reset melepas latch E-Stop
5     | Start / Stop sistem (System_Running)
6     | Izin conveyor masuk
7     | Output conveyor masuk
8     | Barang terdeteksi: stop conveyor masuk
9     | Latch proses pemuatan
10    | Output conveyor pemuat
11    | Latch barang tinggi (High_Item), harus dievaluasi sebelum rung 14
12    | Height_Sorter HIGH: transfer kiri
13    | Side conveyor kiri
14    | Height_Sorter LOW: transfer kanan (bergantung pada rung 11)
15    | Side conveyor kanan
16    | Counter dan display

Function Block
FB               | Fungsi                                                                  | Dipakai
Height_Sorter    | Mendeteksi barang, meminta transfer, dan menghentikan transfer          | 2x (HIGH dan LOW)
Side_Conveyor    | Menjalankan side conveyor dari pintu masuk sampai ujung                 | 2x (kiri dan kanan)
Sorting_Counter  | Menghitung barang kiri dan kanan, menjumlahkan, dan konversi ke display | 1x

Teknik yang Dipakai
- Rangkaian E-Stop dengan latch: sinyal E-Stop dikonfirmasi lalu dikunci menjadi satu sinyal E_StopLatched yang dipakai semua blok. Latch hanya dilepas oleh Reset saat tombol E-Stop sudah dilepas.
- Desain modular: tiap fungsi dibuat sebagai function block dan dipakai ulang (Height_Sorter dan Side_Conveyor masing-masing dua instance), sehingga sisi kiri dan kanan memakai logika yang sama.
- Reset counter dua tahap: tekanan Reset yang melepas latch E-Stop ditangkap dan diabaikan oleh counter, sehingga reset counter baru terjadi pada tekanan berikutnya. Ini mencegah jumlah barang hilang tanpa disengaja.
- Penghitungan berbasis edge: counter dinaikkan oleh rising edge sensor ujung conveyor, sehingga satu barang dihitung satu kali.
- Latching bertahap: kondisi seperti barang terdeteksi dan izin transfer disimpan pada latch dan dilepas oleh event berikutnya (misalnya barang sampai di pintu masuk side conveyor).

Catatan Desain
- Urutan rung penting pada dua tempat: High_Item (rung 11) harus dievaluasi sebelum Height_Sorter LOW (rung 14), dan Stop_Transfer pada Height_Sorter dipakai oleh latch di scan berikutnya.
- Fungsi Reset ganda (melepas latch E-Stop dan mereset counter) disengaja dan dipisahkan oleh logika dua tahap di Sorting_Counter.

Komunikasi
Program CODESYS terhubung ke Factory I/O melalui Modbus TCP. CODESYS berperan sebagai slave dan Factory I/O sebagai client yang terhubung ke 127.0.0.1:502.
[TODO: sesuaikan Slave ID dan alamat dengan konfigurasi di projectmu]

Arah data dilihat dari sisi PLC:
- Coil (Factory I/O ke PLC): sensor dan tombol, terbaca di CODESYS sebagai input.
- Discrete Input dan Input Register (PLC ke Factory I/O): aktuator dan counter, ditulis dari output CODESYS.

Input PLC (Coil)
Coil  | Label di Factory I/O | Variabel
[TODO: isi dari konfigurasi Modbus]

Output PLC (Discrete Input dan Input Register)
Alamat | Label di Factory I/O | Variabel
[TODO: isi dari konfigurasi Modbus]

Pengujian
Sudah diuji di Factory I/O:
- Fungsi semua tombol (Start, Stop, E-Stop, Reset)
- Proses sortir normal (barang tinggi ke kiri, barang rendah ke kanan)
- Reset counter: setelah Stop, dan dua tahap setelah E-Stop
[TODO: tambahkan hasil tes tepi bila sudah dicoba: E-Stop saat barang di tengah transfer, dua barang berdekatan, Stop lalu Start]

Penulis
Izzuddin Alqassam, Teknik Elektro, Universitas Hasanuddin
Fokus: sistem kontrol, PLC, otomasi industri