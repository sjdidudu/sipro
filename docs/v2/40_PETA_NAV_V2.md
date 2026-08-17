# 40 — PETA NAVIGASI V2 (lama → baru)

> Dibuat pada Fase 40c. **Tidak ada fitur yang dihapus** — tujuh pintu menu dilebur menjadi hub
> bertab, dan seluruh rute lama tetap hidup sebagai alias. Peta ini juga tersedia **di dalam
> aplikasi**: sidebar → “Peta menu baru (menu saya ke mana?)”
> (`frontend/src/config/navMigrationMap.js` + `components/layout/NavMigrationDialog.js`),
> sehingga pemakai lama tidak perlu membuka dokumentasi ini.

## 1. Angka yang bisa diverifikasi

| Ukuran | Sebelum | Sesudah |
|---|---|---|
| Item menu non-admin (rute unik yang bisa diklik) | 31 | **26** |
| Baris sidebar non-admin (termasuk duplikat) | 32 | 26 |
| Item “Segera Hadir” (terkunci, **tanpa route**) | 0 | 4 |
| Rute aplikasi | 36 | 37 (`/build` baru; tidak ada yang dihapus) |

Gate `scripts/verify_ia_v2.py` menegakkan angka & aturannya (mis. item “Segera Hadir” tidak
boleh punya `path`, dan menu yang dilebur tidak boleh muncul lagi di sidebar sementara rutenya
WAJIB tetap ada).

## 2. Peta menu

| Menu lama | Lokasi baru | Rute | Alasan |
|---|---|---|---|
| Beranda | Beranda (Control Tower) | `/` | KPI kini bisa di-drill-down |
| Work Hub | Kerja › **Tugas & Papan Divisi** | `/tasks` | nama menyebut isinya; daftar jadi tabel pro |
| Notifikasi | Kerja › Notifikasi | `/notifications` | tetap |
| Lead | CRM › **Pipeline Lead** | `/leads` | profil kanonik `/leads/:id` |
| Agenda & Survey | CRM › Agenda & Survey | `/appointments` | tetap |
| Inbox WA | CRM › **Percakapan (WA)** | `/inbox` | istilah seragam dengan playbook WA |
| **Deal & Unit** | CRM › Customer & Kontrak → tab **Deal & Unit** | `/customers?hub=deal` | satu alur: unit → deal → pembeli → dokumen → bayar |
| **Customer & KPR** | CRM › Customer & Kontrak → tab **Pembeli** | `/customers?hub=pembeli` | idem; profil kanonik `/customers/:id` |
| Automasi & Channel | Marketing › Automasi & Channel | `/automation` | domain marketing dipisah dari CRM |
| Proyek & Unit | Proyek › **Master Proyek** | `/projects` | struktur proyek→cluster→blok→unit |
| **Progres & Mutu** | Proyek › Pembangunan → tab Progres & Mutu | `/build?hub=progres` | 4 menu pembangunan jadi 1 hub |
| **Kalender Jadwal** | Proyek › Pembangunan → tab Kalender Jadwal | `/build?hub=kalender` | idem |
| **Buku Harian & Punch** | Proyek › Pembangunan → tab Buku Harian & Punch | `/build?hub=lapangan` | idem |
| **Kalibrasi Jadwal** | Proyek › Pembangunan → tab Kalibrasi Jadwal | `/build?hub=kalibrasi` | idem |
| — (baru) | Proyek › Pembangunan → tab **Papan Unit** | `/build?hub=unit` | tabel unit LINTAS proyek (mis. semua unit QC hold) |
| Material & Opname | Proyek › Material & Opname | `/materials` | tetap |
| **Site Plan & Showroom** (muncul 2×) | Proyek › **Site Plan** (satu item, terlihat utk penjualan & proyek) | `/site-plan` | hapus duplikasi baris menu |
| **Perizinan & Dokumen** | **Dokumen** → tab **Perizinan** | `/documents?hub=perizinan` | daftar global izin masuk Dokumen; izin per objek tetap di Unit 360 & Proyek |
| RAB/BoQ · Subkon & SPK · Pengadaan | Pengadaan (3 item) | `/boq` `/subcon` `/procurement` | tetap |
| Keuangan | Keuangan › AR / AP / Komisi | `/finance` | tab kini hidup di URL (`?tab=ar`) |
| Marketing Fee | **CRM › Mitra & Fee → tab “Tagihan Fee”** | `/partners?hub=tagihan` | **Fase 42 SELESAI**: keluar dari sidebar. Rute `/marketing-fee` tetap terdaftar tetapi **MENGALIHKAN** ke tab Tagihan Fee (satu pintu); halaman lama + tab “Master Agen” dihapus karena kembar dengan “Master Mitra” |
| **Kas Bon** | Keuangan › Kas Bon | `/petty-cash` | pindah dari grup terpisah “Kas & Pengeluaran” |
| Akuntansi (5 item) | Akuntansi | `/accounting` `/accounting/reports` `/fixed-assets` `/corporate-financing` `/tax` | tetap |
| Komplain & CS | Layanan › Komplain & CS | `/complaints` | daftar jadi tabel pro + KPI drill-down |
| Dokumen | **Dokumen & Perizinan** | `/documents` | lihat baris Perizinan di atas |
| Pusat Konfigurasi | Konfigurasi | `/config` | tetap |
| Admin (5 item) | Admin | `/admin/*` | tetap |

## 3. Rute alias yang SENGAJA dipertahankan

`/deals` · `/construction` · `/build-calendar` · `/build-calibration` · `/field` · `/permits` · `/marketing-fee` (Fase 42)

Alasan: notifikasi & tugas yang sudah terbit menyimpan tautan ke rute tersebut, dan pemakai
menyimpan bookmark. Menghapusnya berarti “fitur hilang” dari sudut pandang pemakai walau
kodenya masih ada. Rute-rute ini tidak lagi muncul di sidebar (pintu masuknya di hub).

## 4. “Segera Hadir” (terkunci di sidebar, tanpa route)

| Item | Grup | Fase |
|---|---|---|
| ~~Mitra & Fee~~ | ~~CRM~~ | **42 — SUDAH DIBUKA** (`/partners`, hub 5 tab) |
| Kampanye & Biaya Iklan | Marketing | 44 |
| Atribusi & CAPI | Marketing | 44 |
| Analitik & BI | Analitik & BI | 45 |

Aturan: item “Segera Hadir” **tidak boleh punya `path`** sehingga mustahil menjadi halaman
kosong; `check_nav_map.py` CHECK 2 + `verify_ia_v2.py` menjaganya.

## 5. Drill-down KPI (US-40-4)

Tautan drill dibentuk di **backend** (`routers/work_router.py::_kpis`) supaya definisi angka
dan definisi filter daftar tidak bisa berbeda. Contoh yang sudah terbukti di layar:

| KPI | Tautan | Hasil |
|---|---|---|
| Tugas Terlambat (org) = 4 | `/tasks?tab=tasks&scope=all&bucket=overdue` | tabel berisi tepat 4 baris |
| AR Outstanding | `/finance?tab=ar&status=unpaid,partial` | tab Piutang + chip filter status |
| QC Hold | `/build?hub=unit&construction_status=qc_hold` | Papan Unit terfilter |
| Lewat SLA (komplain) | `/complaints?sla=breached` | daftar komplain lewat SLA |

## 6. Tambahan Fase 41 — tab “Umur Tahap & SLA”

| Permukaan | Lokasi | Rute | Catatan |
|---|---|---|---|
| Umur Tahap & SLA | Kerja › Tugas & Papan Divisi → tab **Umur Tahap & SLA** | `/tasks?tab=aging` | laporan umur tahap 7 objek (lead, deal, tugas, komplain, pembeli, tagihan AR, dokumen); setiap angka punya tautan drill ke daftar terfilter |
| Ubah ambang SLA | tautan “Ubah SLA” di tab itu | `/config?group=sla` | ambang SLA tinggal di Pusat Konfigurasi, BUKAN di komponen |

Filter **“Umur / SLA”** (`?sla=over|over2|ok|none`) kini seragam di daftar Lead, Tugas, Komplain,
Deal, Pembeli, Tagihan (AR), dan Dokumen — dieksekusi di database atas field tersimpan
`stage_due_at`, sehingga angka laporan = angka daftar. Nilai filter tak dikenal menghasilkan
daftar KOSONG (bukan diabaikan diam-diam) supaya pemakai sadar filternya tidak berlaku.
