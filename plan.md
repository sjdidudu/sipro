# Rencana Development Lanjutan — SIPRO (repo `hzjsjdychc/sipro`)

Problem statement (verbatim, sesi ini):
> "saya ingin anda lanjutkan development dari repo ini https://github.com/hzjsjdychc/sipro —
> sebelumnya development terhenti di: *Now fixing the gate's comment-aware regex and making the
> sync assertion non-flaky* → dua `search_replace` ke `/app/scripts/verify_41.py`."

Keputusan user sesi ini: **Tuntaskan Fase 41 + 42 saja** sampai seluruh gate (23 + 2 baru) PASS
+ E2E testing. Integrasi pihak ketiga tetap **simulasi** (tanpa kredensial nyata).

---

## STATUS AKHIR SESI — ✅ FASE 41 & 42 TUNTAS

| Kriteria | Hasil | Bukti |
|---|---|---|
| `bash scripts/run_all_gates.sh` | ✅ **OVERALL PASS (25 gates)** | dijalankan 3× berturut (termasuk setelah E2E) |
| `python3 scripts/mutasi_41_42.py` | ✅ **32/32 pemeriksaan** | 16 mutasi tertangkap + semuanya pulih hijau |
| E2E multi-peran (testing agent) | ✅ **0 bug kritis, 0 bug UI** | `test_reports/iteration_63.json`, backend 39/40 |
| Invarian keuangan & integritas data | ✅ PASS | `verify_business_invariants`, `verify_data_integrity` |

Gate baru terdaftar di `run_all_gates.sh`: **`verify_41.py`** (jam tahap & SLA) dan
**`verify_partner.py`** (mitra & fee). Uji-mutasinya: **`mutasi_41_42.py`**.

---

## 1) Titik berhenti sesi lalu — SUDAH DIPERBAIKI

`scripts/verify_41.py` rusak di tengah edit: docstring `code_only()` tertulis dengan escape
**mentah** (`\n` dan `\"\"\"`) → file jadi `unterminated triple-quoted string literal`, artinya
**gate 41 belum pernah benar-benar dijalankan**. Sudah ditulis ulang benar.

Dua niat asli edit itu tetap dipertahankan dan kini benar-benar bekerja:
1. **regex sadar-komentar** — `code_only()` membuang baris komentar supaya pemeriksaan
   "tidak ada ambang SLA hardcode" menilai KODE, bukan penjelasan sejarah di docstring
   komponen (AgingCell sengaja mendokumentasikan angka lama 72/48/168/336/720).
2. **assertion sinkron tidak rapuh** — gate memanggil `POST /aging/reconcile` lebih dulu
   (sweeper jalan tiap 60 detik) lalu menuntut invariannya: **tidak boleh ada satu pun baris
   yang tidak sinkron**.

Bonus temuan: satu assertion gate **salah sasaran** — mencari string `sla_state` di dalam
`AgingCell.js` padahal nama field itu hanya ada di KOMENTAR (kodenya menerima prop `state`).
Diganti dengan bukti yang sebenarnya: sel menerima keadaan sebagai prop + prop itu jadi sumber
utama, dan **ketujuh** berkas pemakai dibuktikan meneruskan `state={x.sla_state}`.

---

## 2) Cacat NYATA yang ditemukan & diperbaiki sesi ini

Semua di bawah adalah cacat asli (bukan penyesuaian gate agar hijau):

1. **`partner_tax_type` tidak terdaftar di SSOT** (`backend/reference_p41.py`).
   `models_p41.TaxType` memvalidasi lewat grup yang tak pernah ada → **setiap** pembuatan
   aturan fee dengan blok pajak mati **500** (`KeyError`) di lapisan validasi request, bukan
   400 berbahasa Indonesia. Grup didaftarkan (pph21/pph23/none, sama dengan
   `partner_fee.TAX_TYPES`). Dikunci mutasi **M11**.
2. **Uji idempotensi fee memakai peran salah** (`scripts/verify_partner.py`).
   Penerbitan manual diuji dengan header **finance**, padahal finance memang TIDAK boleh
   mengajukan (pemisahan tugas) → **403 menutupi hasil 400** yang sebenarnya diuji; idempotensi
   tidak pernah teruji. Diganti ke peran manajer + ditambah pemeriksaan eksplisit
   "finance TIDAK menerbitkan tagihannya sendiri".
3. **Regresi `audit_forms_deep` (baseline nyata 22/23, bukan 23/23)** — 5 temuan dari berkas
   Fase 41/42:
   - field **Bank** → dropdown SSOT `financing_bank` (grup ini memang dirancang dipakai
     bersama untuk rekening mitra);
   - **"Batasi ke tipe unit"** → dropdown SSOT `unit_type`; dulu teks bebas yang di-uppercase,
     sehingga salah ketik membuat aturan fee **tidak pernah cocok** tanpa ada yang sadar;
   - `SLA_FILTER_OPTIONS` tidak lagi menyalin 4 opsi + labelnya → dari grup SSOT `sla_state`
     (7 pemanggil `slaFilter()` disesuaikan; urutan "paling mendesak dulu" tetap dijaga sebagai
     keputusan UX, bukan kosakata);
   - input nominal aturan fee diberi `aria-label` dinamis;
   - **presisi gate**: input bertipe tanggal tidak mungkin jadi dropdown enum, jadi label
     rentang tanggal ("Lead dari") tidak lagi dituduh E1 — cacat palsu yang memaksa label
     dibuat kabur hanya demi menyenangkan gate.
4. **2 CTA MATI + duplikasi matriks RBAC di layar** (ditemukan lewat uji browser sendiri,
   bukan dari gate). Finance melihat tombol **"Ajukan Fee"** dan **"Terbitkan tagihan fee"**
   padahal server menjawab **403**. Penyebab: 6 layar menuliskan ULANG daftar peran
   (`[...].includes(user?.role)`) alih-alih memakai izin **efektif** `can(resource, action)`
   dari `GET /auth/me` — padahal matriks RBAC bisa diubah admin lewat Pusat Konfigurasi.
   Semua diganti ke `can()`; dikunci gate baru di `verify_partner.py` + mutasi **M15/M16**.

---

## 3) Yang sudah berdiri (diverifikasi lewat gate + browser)

**Fase 41 — jam tahap jadi field tersimpan**
- `stage_entered_at`, `stage_sla_hours`, `stage_due_at`, `stage_due2_at`, `stage_clock_stage`,
  `stage_clock_source` tersimpan pada **7 koleksi** (lead, deal, task, complaint, customer,
  ar_invoice, document) + index `org_id+stage_due_at` / `org_id+stage_entered_at`.
- Ambang SLA dari **Pusat Konfigurasi**; mengubah setting langsung berlaku ke baris yang ada
  (`resync_for_setting`), bukan hanya baris baru.
- Filter `?sla=over|over2|ok|none` dieksekusi **di database**; nilai tak dikenal →
  hasil KOSONG (bukan diabaikan diam-diam).
- Tab **"Umur Tahap & SLA"** di hub Kerja: 4 KPI, tabel per tahap (SLA/jumlah/lewat/rata-rata/
  median/P90/terlama), panel lintas domain 7 objek, semua angka punya tautan drill ke rute nyata.
- RBAC: semua peran MELIHAT; hanya owner/super_admin menjalankan `reconcile` (`aging:manage`).

**Fase 42 — Mitra & Fee**
- Menu **"Mitra & Fee"** terbuka (`/partners`, hub 5 tab: Master Mitra, Aturan Fee, Tagihan Fee,
  Sengketa Atribusi, Analitik Mitra) + halaman kanonik `/partners/:id`.
  Rute alias lama **`/marketing-fee` tetap hidup** dan ada di peta menu.
- Data mitra tetap di koleksi `agents` → invarian GL `marketing_fee.py` (6-1200 / 2-1500 /
  2-1300) tidak berubah.
- `partner_fee_rules` + mesin aturan murni (`partner_fee.py`): persen harga, nominal per
  transaksi/tipe unit, berjenjang per jumlah/nilai, per lead terkualifikasi, gabungan;
  aturan bentrok **DITOLAK** (tidak dipilih diam-diam).
- Fee otomatis dari **pemicu nyata** lewat event bus (bukan tombol karangan), idempoten per
  mitra × deal × pemicu; **INV-09**: tidak ada fee tanpa aturan berlaku, dengan alasan jelas.
  Terbukti di data nyata: `MF/2026/0003` = 2% × Rp 850.000.000 → bruto Rp 17.000.000,
  PPh Rp 425.000, netto Rp 16.575.000 (beban = netto + PPh, jurnal seimbang).
- Analitik mitra dihitung dari data (lead, terkualifikasi, closing, fee, sisa utang).

---

## 4) Utang teknis TERBUKA (jujur — belum dikerjakan, di luar lingkup sesi ini)

1. **Daftar peran hardcode di 25 berkas fase lama** (32 kemunculan `includes(user?.role)`) —
   sudah dibereskan hanya untuk berkas Fase 41/42. Idealnya semua pindah ke `can()` dan
   diberi gate global.
2. **3 peringatan eslint** `react-hooks/exhaustive-deps` (LeadsPage `counts`,
   AgingReportTab `totals`, FeeRulesTab `toggleStatus`) — peringatan lama, bukan error
   kompilasi; kompilasi frontend bersih.
3. **`/marketing-fee` masih me-render halaman lama** (dengan tab "Pengajuan Fee/Master Agen")
   alih-alih langsung mendarat di tab **"Tagihan Fee"** hub `/partners`. Alias hidup dan ada
   peta menu, jadi tidak ada tautan rusak — tapi masih ada dua pintu untuk satu urusan.
4. Menu masih "Segera Hadir": Kampanye & Biaya Iklan, Atribusi & CAPI, Analitik & BI.
5. Integrasi pihak ketiga tetap **mode simulasi**: WhatsApp Cloud API, e-sign, BI/SLIK, e-Faktur.

---

## 5) Cara menjalankan ulang verifikasi

```bash
bash scripts/run_all_gates.sh          # 25 gates, harus OVERALL PASS
python3 scripts/mutasi_41_42.py        # 32 pemeriksaan (16 mutasi), ±25 menit
python3 scripts/mutasi_41_42.py M7 M12 # hanya mutasi tertentu (mempercepat penyelidikan)
bash scripts/seed_reset.sh             # DB bersih + seed + seluruh gate
```

Catatan pemulihan lingkungan (WAJIB, lihat `memory/test_credentials.md`): `.env` tidak ada di
git — tanpa `JWT_SECRET` di `backend/.env`, setiap login mati 500.

---

## 6) Saran fase berikutnya (belum disepakati user)

- **Fase 43 — Kampanye & Biaya Iklan + Atribusi/CAPI**: satu-satunya menu bisnis besar yang
  masih "Segera Hadir" dan sudah punya spec (`docs/v2/30_MARKETING_INTEGRATION_SPEC.md`).
- **Analitik & BI** (`docs/v2/31_ANALYTICS_BI_SPEC.md`) — sekarang seluruh angka aging/fee sudah
  bisa diagregasi di database, jadi fondasinya sudah siap.
- **Bersihkan utang RBAC frontend** (butir 4.1) + gate global agar tidak terulang.
