# nvid (876) - Writeup Lengkap

> **Kategori:** Rev / CUDA  
> **Judul:** `nvid`  
> **Deskripsi:** *"Plz donate to help me buy it. Anyway here's (another) flag checker."*  
> **Flag:** `CBC{Cc_uV_dAa___GPU!}`

---

## 1. Overview

Dari challenge kita dapat folder `chall/` berisi:

- `checker.exe`
- `info.txt`

Programnya adalah **flag checker lokal** berbasis CUDA. Target kita: recover input yang membuat checker output `:)`.

---

## 2. Recon Awal

### Jalankan binary

```powershell
.\checker.exe
```

Output:

```text
Cek flag >> : C:\projekan\hack-web2\chall\checker.exe <flag>
```

Kalau input salah:

```powershell
.\checker.exe TESTFLAG
```

Output:

```text
:(
```

---

## 3. Reverse Host Logic (x64)

Dengan disassembly `main`, ditemukan validasi format sebelum kernel CUDA dipanggil:

1. `argc` harus `2`
2. Panjang input harus `0x15` (21)
3. Prefix harus `CBC{`
4. Karakter terakhir harus `}`

Jadi format flag:

```text
CBC{################}
```

Bagian tengah (`#`) panjangnya **16 karakter**.

---

## 4. Ekstraksi Kernel CUDA

`checker.exe` menyimpan kode GPU pada section `.nv_fatb`.

Untuk disassembly SASS, aku pakai tool resmi NVIDIA dari PyPI:

```powershell
python -m pip install nvidia-cuda-nvdisasm nvidia-cuda-cuobjdump
```

Dump SASS:

```powershell
& 'C:\laragon\bin\python\python-3.10\Lib\site-packages\nvidia\cu13\bin\cuobjdump.exe' -sass checker.exe > cuobjdump_sass.txt
```

Dump ELF CUDA sections:

```powershell
& 'C:\laragon\bin\python\python-3.10\Lib\site-packages\nvidia\cu13\bin\cuobjdump.exe' -elf checker.exe > cuobjdump_elf.txt
```

Kernel target:

- `_Z9check_keyPKhPj`

Data konstanta penting ada di:

- `.nv.constant3` (berisi tabel konstanta)

---

## 5. Inti Algoritma Kernel

Kernel membaca 16 byte input (isi `CBC{...}`), lalu melakukan transform campuran:

- `PRMT` (byte permutation)
- `LOP3.LUT` (dipakai sebagai operasi bitwise; pada pola ini efektifnya `AND`/`XOR`)
- `SHF.R.W.U32` dan `SHF.L.W.U32.HI` (funnel shift)

Di akhir, kernel membentuk 4 nilai `u32` dan membandingkan dengan konstanta dari `c[0x3]`:

- compare ke offset `0x0`, `0x4`, `0x8`, `0xc`

Jika semua cocok, kernel menulis hasil sukses; host lalu print `:)`.

---

## 6. Detail Penting Saat Modeling

Ada jebakan utama: **urutan operand `SHF` di SASS**.

Pada SASS:

```text
SHF.R.W.U32 Rd, a, c, b
SHF.L.W.U32.HI Rd, a, c, b
```

Semantik PTX `shf` bekerja atas konkatenasi `[b, a]`, shift amount = `c & 31`.

Kalau urutan ini salah dimodelkan, constraint jadi `unsat`.

---

## 7. Solver Z3

Strategi:

1. Definisikan 16 byte simbolik (`b0..b15`)
2. Batasi printable ASCII (`0x20..0x7e`)
3. Reimplement seluruh jalur transform kernel secara bit-precise (`BitVec(32)`)
4. Tambah constraint akhir: semua pembanding harus true
5. `s.check()`

Hasil solver:

```text
b'Cc_uV_dAa___GPU!'
```

Kandidat flag:

```text
CBC{Cc_uV_dAa___GPU!}
```

Validasi:

```powershell
.\checker.exe "CBC{Cc_uV_dAa___GPU!}"
```

Output:

```text
:)
```

---

## 8. Flag

```text
CBC{Cc_uV_dAa___GPU!}
```

---

## 9. Ringkasan Singkat

1. Reverse host checker -> format diketahui: `CBC{16 chars}`.
2. Disassemble kernel CUDA `_Z9check_keyPKhPj` dari `.nv_fatb`.
3. Ambil konstanta `.nv.constant3`, modelkan instruksi inti (`PRMT`, `SHF`, `LOP3`) ke Z3.
4. Solve constraint dan verifikasi ke binary.
5. Dapat flag final.

