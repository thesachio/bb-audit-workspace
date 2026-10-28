---
name: sc-autopilot
description: Master Orchestrator Audit Lintas-Rantai - Jalankan seluruh siklus audit (Fase 1 Recon -> Fase 2 Micro-to-Macro Hunt -> Fase 3 Sandbox Verify) secara otonom penuh menggunakan delegasi subagent berkonteks bersih
version: 3.0.0
metadata:
  hermes:
    tags: [security, web3, audit, orchestrator, autopilot, delegation, multichain]
    category: security
    requires_toolsets: [terminal]
---

# Autonomous Multi-Chain Smart Contract Audit Orchestrator (sc-autopilot)

## When to Use
Gunakan skill ini saat pengguna meminta audit menyeluruh dari awal hingga akhir tanpa intervensi manual (*Full Autopilot*), atau saat pengguna meminta orkestrasi otomatis atas banyak fungsi di dalam .audit/03-micro-queue.md.

## Pitfalls (Hukum Mutlak Orchestrator)
- JANGAN KERJAKAN SEMUA FASE SENDIRI DI PROSES UTAMA: Sebagai Parent Agent (Orchestrator), tugasmu adalah menjaga konteks utamamu tetap ringan dan mendelegasikan pekerjaan berat ke Subagent menggunakan delegate_task.
- INGAT BATASAN SUBAGENT (`skip_soul=True` & `cwd=workspace`): Subagent tidak tahu riwayat obrolanmu dan mulai dari /home/ubuntu/.hermes/profiles/bb/workspace. Setiap panggilan delegate_task WAJIB menyertakan Blok Konteks Wajib sesuai Bagian 3 di workspace/AGENTS.md!
- JANGAN LEWATI ANTREAN: Pastikan seluruh fungsi prioritas di .audit/03-micro-queue.md diperiksa secara sistematis oleh Subagent sebelum menyusun laporan akhir.
- DILARANG MEMANGGIL `skill_manage` ATAU `memory`.

---

## Procedure (Autonomous End-to-End Pipeline)

Saat pengguna memicu mode otonom penuh untuk sebuah proyek di /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/, jalankan 4 tahapan orkestrasi berikut secara berurutan dari awal hingga selesai:

### Tahap Orkestrasi 1: Eksekusi Fase 1 (Intake & Data-First Recon)
1. Periksa apakah /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit/00-project-map.md dan .audit/03-micro-queue.md sudah ada.
2. Jika BELUM ADA: Panggil delegate_task untuk menugaskan Subagent Recon:
   - Goal: "Jalankan prosedur penuh skill sc-phase1-recon pada proyek <nama-proyek> dan hasilkan 00-project-map.md, 01-intake-and-env.md, 02-storage-map.md, serta 03-micro-queue.md."
   - Context Wajib:
     * Path Absolut Proyek: /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>
     * Instruksi Eksplisit: "Panggil skill_view('sc-phase1-recon') terlebih dahulu, ikuti ke-5 langkahnya secara ketat, jangan buat AGENTS.md di dalam folder proyek, dan simpan seluruh output di dalam folder .audit/ proyek."
3. Setelah Subagent Recon selesai, baca .audit/00-project-map.md dan .audit/03-micro-queue.md menggunakan read_file untuk mengetahui Paradigma Rantai proyek dan daftar antrean fungsi yang harus dibedah.

### Tahap Orkestrasi 2: Eksekusi Fase 2 (Delegasi Bedah Mikro Per Fungsi / Pasangan Siklus Hidup)
1. Ambil daftar fungsi yang masih berstatus belum diperiksa dari .audit/03-micro-queue.md.
2. Kelompokkan secara cerdas: 1 panggilan `delegate_task` menangani 1 fungsi kompleks tunggal ATAU 1 pasang fungsi siklus hidup yang memodifikasi state yang sama (misal fungsi penambah state += dipasangkan dengan fungsi pengurang state -=).
3. Luncurkan Subagent Pemburu (delegate_task) untuk setiap unit antrean tersebut dengan menyertakan:
   - Goal: "Bedah fungsi [NamaFungsi/Pasangan] di [NamaFile:Baris] dari Radius 0 hingga Radius 2 menggunakan prosedur sc-phase2-hunt."
   - Context Wajib (Sesuai Konstitusi `workspace/AGENTS.md`):
     1. Path Absolut Proyek: /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>
     2. Paradigma & Rantai Proyek: Sebutkan paradigma rantai dari 00-project-map.md.
     3. File Konteks `.audit/` & Target: Sebutkan path .audit/00-project-map.md, .audit/02-storage-map.md, dan .audit/03-micro-queue.md, serta nama file dan baris fungsi target yang harus dibedah.
     4. Instruksi Pemuatan Skill & Referensi: "Wajib panggil skill_view('sc-phase2-hunt') untuk membaca prosedur Radius 0-2, DAN wajib panggil skill_view('sc-phase2-hunt', 'references/<nama-modul>.md') sesuai tag [LOAD: ...] pada fungsi target! Perbarui status fungsi di .audit/03-micro-queue.md dan tulis hipotesis yang lolos Devil's Advocate ke .audit/04-hypotheses.md."
4. Ulangi hingga seluruh antrean prioritas di .audit/03-micro-queue.md selesai diperiksa.

### Tahap Orkestrasi 3: Eksekusi Fase 3 (Delegasi Verifikasi Sandbox Lokal)
1. Baca .audit/04-hypotheses.md.
2. Jika tidak ada satu pun hipotesis berstatus [FLAGGED], tulis ringkasan di .audit/FINAL-REPORT.md bahwa seluruh antrean fungsi telah diperiksa dan tidak ditemukan pelanggaran invarian.
3. Jika terdapat hipotesis berstatus [FLAGGED], panggil delegate_task untuk menugaskan Subagent Verifier (1 Subagent per hipotesis agar fokus):
   - Goal: "Verifikasi hipotesis [NamaHipotesis] di sandbox pengujian lokal menggunakan prosedur sc-phase3-verify."
   - Context Wajib:
     * Path Absolut Proyek: /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>
     * Detail Hipotesis & File .audit/04-hypotheses.md serta .audit/00-project-map.md.
     * Instruksi Eksplisit: "Wajib panggil skill_view('sc-phase3-verify'), jalankan pengujian lokal sesuai toolchain rantai proyek, pindahkan ke .audit/05-dead-ends.md jika gugur (false positive), atau tulis laporan lengkap beserta diff remediasi ke .audit/FINAL-REPORT.md jika terbukti mematahkan invarian."

### Tahap Orkestrasi 4: Konsolidasi & Pelaporan ke Pengguna
Setelah seluruh Subagent menyelesaikan tugasnya, baca .audit/FINAL-REPORT.md, .audit/03-micro-queue.md, dan .audit/05-dead-ends.md, lalu sampaikan ringkasan eksekutif hasil audit secara tajam dan terstruktur kepada pengguna.

---

## Verification
Pastikan di akhir eksekusi otonom seluruh item prioritas di .audit/03-micro-queue.md telah berstatus [DONE] atau [VERIFIED], dan laporan akhir telah tersedia di .audit/FINAL-REPORT.md.
