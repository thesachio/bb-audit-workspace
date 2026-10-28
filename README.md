# BBounty Audit Workspace — Multi-Chain Smart Contract Audit System

Konstitusi kerja, aturan operasional, dan skill pipeline audit kontrak pintar lintas-rantai untuk agent Hermes (profil `bb`).

## Struktur

```
AGENTS.md                 Konstitusi operasional (mengikat Parent & Subagent)
SOUL.md                   Identitas & gaya komunikasi Parent agent
config/config.yaml        Konfigurasi profil (tanpa kredensial)
skills/security/
├── sc-phase1-recon/      Fase 0 & 1 — intake adaptif + data-first recon
├── sc-phase2-hunt/       Fase 2 — inspeksi mikro-ke-makro (Radius 0 → 2)
│   └── references/       7 modul checklist (4 rantai + 3 ekonomi universal)
├── sc-phase3-verify/     Fase 3 — verifikasi sandbox + laporan remediasi
└── sc-autopilot/         Orkestrator otonom Fase 1 → 3 via delegasi subagent
```

## Tiga Paradigma Eksekusi Rantai

- **Paradigma A** — Atomic Account/Storage (EVM, Tron/TVM, Solana)
- **Paradigma B** — Object/Capability & PTB (Sui, Aptos / Move)
- **Paradigma C** — Asynchronous Actor & Message Passing (TON / FunC / Tact / Tolk)

## Prinsip Inti

- Peran fungsi dikenali dari **mutasi state** (`+=`, `-=`, `delete`, mutasi field objek) dan sintaks bahasa — bukan dari namanya.
- Analisis **data-first**: pahami storage/struct/objek sebelum logika fungsi.
- Ekspansi **Radius 0 → 1 → 2** hanya jika ada bukti anomali.
- Setiap hipotesis wajib lolos **Devil's Advocate** sebelum diklaim.
- Klaim bug wajib didukung eksekusi test nyata di sandbox lokal, bukan asumsi.

## Artefak Audit (per proyek, di `.audit/`)

| File | Isi |
|---|---|
| `00-project-map.md` | Identitas, paradigma, in-scope, core invariants |
| `01-intake-and-env.md` | Scope, git history, klaim protokol, matriks lingkungan |
| `02-storage-map.md` | Peta storage/objek dalam 5 kategori state |
| `03-micro-queue.md` | Antrean fungsi + tag `[LOAD: ...]` + status |
| `04-hypotheses.md` | Hipotesis berstatus `[FLAGGED]` |
| `05-dead-ends.md` | Hipotesis gugur (false positive) |
| `FINAL-REPORT.md` | Temuan terverifikasi + patch remediasi |

## Lisensi

Materi metodologi audit. Gunakan dengan bijak — pengujian hanya di lingkungan lokal/terisolasi.
