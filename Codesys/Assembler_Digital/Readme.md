# Factory I/O Assembler – PLC Ladder Logic (CODESYS)

Simulasi lini perakitan otomatis (lid-on-base) di **Factory I/O** yang dikendalikan program **ladder logic CODESYS** (IEC 61131-3). Dua conveyor membawa lid dan base ke posisi masing-masing, lengan pick-and-place 2 sumbu (X/Z) dengan gripper vakum memasang lid ke base, lalu produk jadi dihitung.

## Demo

video yt

Tekan tombol **Vacuum Release** saat demo untuk melepas benda dari gripper.

## Tools

| Item | Keterangan |
|---|---|
| Simulasi | Factory I/O (scene Assembler) |
| PLC / IDE | CODESYS (soft PLC) |
| Bahasa pemrograman | Ladder Diagram (LD) |
| Blok standar | TOF, CTU, RS, SR, F_TRIG |

## Alur Proses

1. **Start**: `Start` mengunci (latch) `System_Running`. `Stop` dan `E_Stop` dipasang tipe NC (TRUE = kondisi normal), sehingga kabel putus juga menghentikan sistem (fail-safe).
2. **Pengumpanan**: conveyor lid dan conveyor base berjalan selama sistem aktif.
3. **Positioning**: saat lid / base mencapai posisi ambil (`Sensor_Lid_atP`, `Sensor_Base_atP`), falling-edge trigger men-set latch clamp, dan conveyor terkait dihentikan melalui rangkaian seal-in.
4. **Pick** ⚠️: sumbu Z turun dan gripper (`Grab`) aktif saat `Sensor_item` mendeteksi lid (filter TOF 50 ms). Vakum ditahan oleh latch `Holding_Item` sampai operator menekan tombol **Vacuum Release** (`Stop_Grab`, NC) atau benda hilang dari sensor.
5. **Transfer** ⚠️: sumbu X menggerakkan lid ke atas base; `Arm_X_AtBase` mengonfirmasi posisi.
6. **Place** ⚠️: setelah sumbu Z selesai bergerak dan lengan berada di base, `Lid_Placed` menyala selama 500 ms (TOF dipakai sebagai pulse stretcher), lalu clamp base dilepas.
7. **Return dan release** ⚠️: `Pos_Raise_B` di-latch, lengan kembali di kedua sumbu (`Arm_Z_Returned`, `Arm_X_Return`), dan conveyor dilepas.
8. **Counting**: saat produk jadi meninggalkan area (`Part_Leave`), `Part_Cleared` dibangkitkan oleh falling edge dan counter CTU bertambah.

## Struktur Program

| Rung | Fungsi |
|---|---|
| 1 | Urutan start / stop dengan seal-in |
| 2–4 | Lid conveyor: run, clamp latch, kondisi stop |
| 5–7 | Base conveyor: run, clamp latch, kondisi stop |
| 8 | Kontrol sumbu Z |
| 9–10 | Aktivasi dan penahanan gripper (dilepas lewat tombol Vacuum Release) |
| 11–12 | Kontrol sumbu X dan konfirmasi posisi |
| 13 | Konfirmasi lid terpasang (pulsa 500 ms) |
| 14–16 | Base blockade latch, konfirmasi Z kembali, perintah X kembali |
| 17–18 | Deteksi part keluar dan counter produk jadi |

## Teknik yang Dipakai

- **Rangkaian stop fail-safe**: `E_Stop`, `Stop`, dan `Stop_Grab` (Vacuum Release) bertipe NC, sehingga kabel putus juga menghentikan fungsinya (TRUE = normal).
- **Sequencing berbasis edge**: blok `F_TRIG` mengubah perubahan sinyal sensor menjadi event satu scan, sehingga urutan bergerak selangkah demi selangkah.
- **Latching dengan RS/SR**: RS (reset dominan) untuk clamp dan perintah lengan; reset hanya diizinkan saat sistem berhenti dan tidak sedang menggenggam.
- **Filter timer**: TOF pada gripper dan konfirmasi lid terpasang untuk mencegah sinyal berkedip.
- **Rangkaian seal-in** untuk kondisi stop conveyor, dilepas oleh event kembalinya lengan.

## I/O Mapping

| Sinyal | Tipe | Keterangan |
|---|---|---|
| `Start`, `Stop`, `E_Stop`, `Reset` | Input | Kontrol operator (`Stop` dan `E_Stop` bertipe NC) |
| `Stop_Grab` | Input | Push button Vacuum Release (NC, TRUE = tidak ditekan) |
| `Sensor_Lid_atP`, `Sensor_Base_atP` | Input | Lid / base berada di posisi ambil |
| `Sensor_item` | Input | Benda terdeteksi oleh gripper |
| `Moving_x`, `Moving_Z` | Input | Umpan balik gerak sumbu |
| `Lim_Lid`, `Part_Leave` | Input | Limit sensor / sensor part keluar |
| `lid_conv`, `base_conv` | Output | Conveyor |
| `Move_x`, `Move_Z`, `Grab` | Output | Sumbu lengan dan gripper (`Holding_Item` = latch penahan internal) |
| `Clamp_Lid`, `Clamp_Base`, `Pos_Raise_B` | Output | Clamp dan base blockade |

## Cara Menjalankan

1. Buka scene Assembler di Factory I/O, lalu atur driver ke soft PLC CODESYS (Modbus TCP / OPC sesuai konfigurasi).
2. Buka file `.project` di CODESYS, download ke soft PLC, lalu jalankan.
3. Tekan `Start` di Factory I/O.

## Pengembangan Selanjutnya

- Ubah urutan menjadi state machine (SFC / GRAFCET) agar lebih mudah dikembangkan dan di-debug.
- Tambahkan penanganan kesalahan (timeout untuk gerak sumbu dan sensor).
- Buat sinyal berbasis pulsa tidak bergantung pada urutan eksekusi rung.
- Tambahkan tampilan HMI dengan penghitung produk.

## Penulis

**Izzuddin Alqassam**, Teknik Elektro, Universitas Hasanuddin
Fokus: sistem kontrol, PLC, otomasi industri
[LinkedIn](#) · [Email](#)
