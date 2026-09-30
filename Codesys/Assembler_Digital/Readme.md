# Factory I/O Assembler – PLC Ladder Logic (CODESYS)

Simulasi lini perakitan otomatis di **Factory I/O** yang dikendalikan program **ladder logic CODESYS** (IEC 61131-3).

> **Objective:** Assemble parts made of lids and bases using a two-axis pick and place.

Sistem merakit produk yang terdiri dari lid dan base. Dua conveyor membawa lid dan base ke posisi masing-masing, lalu lengan pick-and-place 2 sumbu (X/Z) dengan gripper vakum memasang lid ke base. Produk jadi kemudian dihitung.

## Demo

[Tonton video demo](https://drive.google.com/file/d/1CyKIppeB3qQCbHB4MFrak4dVzG6ohpfF/view?usp=sharing)

## Tools

| Item | Keterangan |
|---|---|
| Simulasi | Factory I/O (scene Assembler) |
| PLC / IDE | CODESYS (soft PLC) |
| Bahasa pemrograman | Ladder Diagram (LD) |
| Blok standar | TOF, CTU, RS, SR, F_TRIG |

## Fungsi Push Button

| Tombol | Fungsi |
|---|---|
| **Start** | Menjalankan sistem. |
| **Stop** | Menghentikan sistem tanpa mereset logika sekuensial yang tersimpan. Proses dapat dilanjutkan dengan menekan Start kembali. |
| **E-Stop** | Menghentikan sistem secara paksa. Selama E-Stop aktif, tombol Start, Stop, dan Reset tidak berfungsi. |
| **Vacuum Release** | Melepas vakum pada gripper, setara tombol pelepas *vacuum check valve* di sistem nyata. Hanya berfungsi ketika E-Stop sedang ditekan. |
| **Reset** | Mereset logika sekuensial yang tersimpan beserta counter. |

Ketentuan tombol Reset:
- Hanya dapat digunakan ketika sistem tidak berjalan dan gripper tidak sedang memegang lid.
- Urutan yang disarankan: tekan E-Stop, lepas (reset) E-Stop, baru tekan Reset.

## Kondisi Khusus: E-Stop Saat Gripper Memegang Lid

Jika E-Stop ditekan ketika gripper sedang memegang lid, gripper tetap menahan lid agar tidak jatuh. Lid baru dilepas setelah tombol Vacuum Release ditekan.

Perilaku ini meniru *vacuum check valve* pada sistem pneumatik nyata, yang menjaga vakum tetap tertahan saat suplai berhenti sampai dilepas secara manual.

### Catatan Simulasi

Factory I/O tidak menyediakan komponen check valve. Karena itu, penahanan vakum diimplementasikan lewat logika PLC berupa latch internal `Holding_Item` (rung 10). Latch ini aktif selama `Grab` aktif dan `Sensor_item` mendeteksi benda, dan terlepas saat Vacuum Release (`Stop_Grab`) ditekan atau benda hilang dari sensor.

## Alur Proses (Operasi Normal)

1. **Start**: `Start` mengunci (latch) `System_Running`. `Stop` dan `E_Stop` dipasang tipe NC (TRUE = kondisi normal), sehingga kabel putus juga menghentikan sistem (fail-safe).
2. **Pengumpanan**: conveyor lid dan conveyor base berjalan selama sistem aktif.
3. **Positioning**: saat lid / base mencapai posisi ambil (`Sensor_Lid_atP`, `Sensor_Base_atP`), falling-edge trigger men-set latch clamp, dan conveyor terkait dihentikan melalui rangkaian seal-in.
4. **Pick**: sumbu Z turun dan gripper (`Grab`) aktif saat `Sensor_item` mendeteksi lid (filter TOF 50 ms). Lid ditahan oleh latch `Holding_Item` yang meniru check valve, sampai Vacuum Release (`Stop_Grab`, NC) ditekan atau benda hilang dari sensor.
5. **Transfer**: sumbu X menggerakkan lid ke atas base; `Arm_X_AtBase` mengonfirmasi posisi.
6. **Place**: setelah sumbu Z selesai bergerak dan lengan berada di base, `Lid_Placed` menyala selama 500 ms (TOF dipakai sebagai pulse stretcher), lalu clamp base dilepas.
7. **Return dan release**: `Pos_Raise_B` di-latch, lengan kembali di kedua sumbu (`Arm_Z_Returned`, `Arm_X_Return`), dan conveyor dilepas.
8. **Counting**: saat produk jadi meninggalkan area (`Part_Leave`), `Part_Cleared` dibangkitkan oleh falling edge dan counter CTU bertambah.

## Struktur Program

| Rung | Fungsi |
|---|---|
| 1 | Urutan start / stop dengan seal-in |
| 2–4 | Lid conveyor: run, clamp latch, kondisi stop |
| 5–7 | Base conveyor: run, clamp latch, kondisi stop |
| 8 | Kontrol sumbu Z |
| 9–10 | Aktivasi gripper dan latch penahan vakum (`Holding_Item`) |
| 11–12 | Kontrol sumbu X dan konfirmasi posisi |
| 13 | Konfirmasi lid terpasang (pulsa 500 ms) |
| 14–16 | Base blockade latch, konfirmasi Z kembali, perintah X kembali |
| 17–18 | Deteksi part keluar dan counter produk jadi |

## Teknik yang Dipakai

- **Rangkaian stop fail-safe**: `E_Stop`, `Stop`, dan `Stop_Grab` (Vacuum Release) bertipe NC, sehingga kabel putus juga menghentikan fungsinya.
- **Emulasi vacuum check valve**: latch `Holding_Item` menahan benda saat E-Stop, menggantikan komponen yang tidak tersedia di Factory I/O.
- **Sequencing berbasis edge**: blok `F_TRIG` mengubah perubahan sinyal sensor menjadi event satu scan, sehingga urutan bergerak selangkah demi selangkah.
- **Latching dengan RS/SR**: kondisi sekuensial tetap tersimpan saat Stop ditekan dan hanya dihapus oleh Reset. Reset hanya diizinkan saat sistem berhenti dan gripper tidak memegang lid.
- **Filter timer**: TOF pada gripper dan konfirmasi lid terpasang untuk mencegah sinyal berkedip.
- **Rangkaian seal-in** untuk kondisi stop conveyor, dilepas oleh event kembalinya lengan.

## I/O Mapping

| Sinyal | Tipe | Keterangan |
|---|---|---|
| `Start`, `Stop`, `E_Stop`, `Reset` | Input | Kontrol operator (`Stop` dan `E_Stop` bertipe NC) |
| `Stop_Grab` | Input | Push button Vacuum Release (NC, TRUE = tidak ditekan), setara pelepas check valve |
| `Sensor_Lid_atP`, `Sensor_Base_atP` | Input | Lid / base berada di posisi ambil |
| `Sensor_item` | Input | Benda terdeteksi oleh gripper |
| `Moving_x`, `Moving_Z` | Input | Umpan balik gerak sumbu |
| `Lim_Lid`, `Part_Leave` | Input | Limit sensor / sensor part keluar |
| `lid_conv`, `base_conv` | Output | Conveyor |
| `Move_x`, `Move_Z`, `Grab` | Output | Sumbu lengan dan gripper (`Holding_Item` = latch internal pengganti check valve) |
| `Clamp_Lid`, `Clamp_Base`, `Pos_Raise_B` | Output | Clamp dan base blockade |

## Komunikasi

Program CODESYS terhubung ke Factory I/O melalui protokol **Modbus**. Sinyal input dan output pada tabel I/O Mapping dipetakan ke alamat Modbus antara softwaare PLC CODESYS dan Factory I/O.


## Penulis

**Izzuddin Alqassam**, Teknik Elektro, Universitas Hasanuddin
Fokus: sistem kontrol, PLC, otomasi industri
