# Ammora Season 2 Bot — LYID

Bot automation untuk **Ammora Season 2** (liquidity protocol di atas GIWA — L2 milik Dunamu/Upbit, chainId `91342`). Murni HTTP + transaksi on-chain ringan, **tanpa browser**.

> Status reward: $tAMM testnet + SBT (pool 20M $tAMM). Konversi $tAMM → $AMM direncanakan setelah mainnet GIWA. Belum ada TGE/date resmi. Airdrop = spekulatif; nilai nyata saat ini: kursi beta top-1.000 leaderboard.

## Fitur

| Aksi | Reward | Cara |
|---|---|---|
| Daily check-in | +100 AP | API signed action `checkin` |
| Social backlog (eco/baekho/channel/chat/repost/like/comment/follow) | +50/+100 AP sekali | API `claim-social` (identity X sudah verified via web sekali jalan) |
| Vault transfer harian | +50 AP | tx `deposit()` 0.001 test ETH → API `verify-transaction` |
| **Swap harian 5×** | **+20 AP × 5** | tx ke `nativeRouter` (bukan langsung ke pool) → `verify-transaction` mission `swap` |
| **Weekly post di X** | **+300 AP** | post otomatis pakai cookie `auth_token`+`ct0`, lalu `submit {kind:"weekly", url}` |
| **Stake $tAMM** | **+100 AP/hari** | auto: register identity (`registerAuthorized` pakai signature server) → `deposit(100 tAMM)` di kontrak staking → `verify-transaction` mission `stake` |
| Laporan status per akun | — | `--status` read-only (termasuk hitungan misi hari ini) |

## Humanisasi (anti-sybil)

Ammora menulis di pesan registrasinya: **"One verified X identity may participate with one wallet"** dan mengaktifkan kontrak identity. Jadi scaling yang benar = 1 identitas X : 1 wallet (bukan multi-wallet satu X). Yang di-randomisasi modul `src/humanize.py`:

- `run_jitter` — start acak 0–900 dtk, cron tidak pernah jalan di detik yang sama
- `account_stagger` 40–180 dtk antar akun; `mission_gap` 25–110 dtk antar aksi (+ kemungkinan "kesamber" lama)
- urutan misi di-shuffle tiap run; gate misi (checkin/vault) tidak pernah diskip
- `should_skip` 12% — tapi hanya untuk backlog non-rotten (sehari-harian tidak dibuang)
- satu wallet berbeda per akun — sink address bersama adalah link paling jelas di graph

## Kebutuhan

| Item | Nilai |
|---|---|
| Python | 3.10+ |
| Paket | `eth-account`, `pycryptodome` (`pip install -r requirements.txt`) |
| Wallet | EVM, sudah didaftarkan di ammora.org/waitlist + X verified (manual sekali per identitas) |
| Test ETH | Faucet: https://faucet.lambda256.io/giwa-sepolia (0.01/24j) atau https://faucet.giwa.io (0.005/24j, Turnstile) |
| Modal | **Rp 0** — semua testnet |

## Setup

1. Copy `config.json.example` → `config.json`, isi private key + label per akun.
2. Pastikan wallet sudah **registered & X-verified di website** (bot tidak bikin akun).
3. Klaim faucet untuk wallet (butuh ≥0.002 ETH test utk vault + gas).
4. Untuk weekly post: buat `x_sessions.json` (gitignored) berisi cookie sesi X akun yang X-nya sudah verified:

```json
{"nama_handle": {"auth_token": "...", "ct0": "..."}}
```

Ambil dari DevTools → Application → Cookies → `x.com` setelah login manual sekali. `x_handle` di `config.json` harus sama dengan key di `x_sessions.json`.

```bash
python -m src.main --status                      # cek AP/rank + hitungan misi hari ini
python -m src.main --daily                       # pass harian penuh (checkin, vault, 5 swap, weekly post)
python -m src.main --daily --no-jitter --swap-count 2   # run uji: 2 swap saja
python -m src.main --daily --no-x                # tanpa auto-post X
python -m src.main --daily --no-stake            # tanpa stake tAMM
python -m src.main --daily --stake-amount 100    # nominal stake (default 100 tAMM)
python -m src.main --status --accounts acc01     # satu akun saja
```

## Contoh output run nyata (18 Nov 2026, 1 wallet)

```
[abdulbinjai32] AP=840 rank=3420 reg=True
[abdulbinjai32] checkin already counted today
[abdulbinjai32] sending 0.001 ETH to vault via deposit()...
[abdulbinjai32] vault tx 0xe4dae538321bd292973fcea7a436648dff807919bb4b74af08e2fa8ca0533f16
[abdulbinjai32] swap 2/5 today -> 3 more
[abdulbinjai32] swap buy 229779166595738 -> 321634 tx 0x78719ab4...
[abdulbinjai32] swap verified +20 AP (buy)
[abdulbinjai32] swap sell 221786 -> 155367452895648 tx 0xeb16222c...
[abdulbinjai32] swap verified +20 AP (sell)
[abdulbinjai32] swap buy 180736503117202 -> 252818 tx 0xaa179c0a...
[abdulbinjai32] swap verified +20 AP (buy)
[abdulbinjai32] weekly post already pending for week 2026-11-16
=== SUMMARY ===
abdulbinjai32: AP=840->1050 done=['swap buy +20', 'swap sell +20', 'swap buy +20', 'identity registered', 'stake +100'] errors=[]
```

Total hari itu **+250 AP** (checkin 100 + vault 50 + swap 5×20 + stake 100) naik dari 800 → 1050, rank 3.688 → 2.354,
plus weekly post +300 menunggu payout senin. `stake accounts` on-chain: `active=100000000` (100 tAMM), `weight=5000000`.

## Catatan teknis

- Auth = `personal_sign` atas pesan `Ammora Season 2 action ...` — tidak ada cookie/sesi.
- Header `Origin` + `Referer` wajib; API menolak tanpa itu (`invalid_origin`).
- Reset harian mengikuti `period` di event ledger server (tanggal UTC), bukan jam lokal. Karena itu pengecekan "sudah hari ini" selalu dari `state.events`, bukan file lokal.
- Vault butuh nilai **tepat** 0.001 ETH — bukan lebih, bukan kurang (event dicek exact).
- **Swap wajib lewat `nativeRouter`** (`0x4FC91b90...`). Verifikasi server mengecek `tx.to == nativeRouter` + ada event `Transfer` tAMM. Swap langsung ke pool tetap sukses on-chain tapi **tidak dihitung AP**.
- Kurva tAMM sudah graduate (`curve.status() == 3`), jadi rute aktif = ARL V5 pool dari `curve.canonicalPool()`. Kalau status balik `1`, `src/swap.py` otomatis pakai `buySourceExactInNative` / `sellSourceExactInForNative`.
- `verify-transaction` menolak selama tx belum final (`TRANSACTION_NOT_FINAL`). Bot retry sampai ~2 menit; error transient (`database_overloaded`, `INTERNAL_ERROR`, `RATE_LIMITED`, `TIMEOUT`) juga di-retry — dulu satu error begini bikin AP hari itu hangus.
- Weekly post dibayar otomatis **Senin 00:00 UTC**, status tetap `pending` sampai jam itu. Satu post per minggu per wallet.
- **Stake butuh identity on-chain.** `deposit()` di kontrak staking revert (`0xb4fa3fb3`) selama `identity.eligible(wallet) == false` — nominal berapa pun gagal, dan `excluded`/`depositsPaused`/`periodEnd` semuanya normal. Alur yang benar (sama seperti situs): signed action `identity-authorization` → server balas `participant` + `deadline` + `signature` → kirim `registerAuthorized(address,bytes32,uint64,bytes)` ke kontrak identity → satu blok kemudian `eligible` jadi true → baru `deposit(amount)`. Bot melakukan ini otomatis dalam `do_identity()`.
- `deposit()` di staking butuh **allowance tAMM ke kontrak staking** (bukan ke router) dan saldo tAMM ≥ nominal. Kalau saldo kurang, bot mencatat `stake: tAMM X < 100, swap first` dan lanjut besok.
- `src/xpost.py` posting lewat GraphQL `CreateTweet` pakai cookie web. Kalau X merotasi queryId, gejala = HTTP 200 dengan `tweet_results` kosong; ambil queryId baru dari bundle `x.com` (lihat komentar di file).
- `tools/swap.mjs` = implementasi pembanding berbasis SDK resmi (`@ammora-protocol/sdk`), dipakai saat reverse-engineering. **Tidak dibutuhkan** untuk menjalankan bot — `src/swap.py` murni Python.

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `SEASON2_ONBOARDING_REQUIRED` | Wallet belum register/X verify di web — selesaikan onboarding manual |
| `database_overloaded` / `service_unavailable` saat verify | Indexer/D1 telat — bot retry otomatis; kalau tetap gagal, jalankan ulang, hash sama akan reconcile |
| `vault (no gas)` | Klaim faucet lagi (butuh 0.001 vault + cadangan gas) |
| `INVALID_MISSION` | Nama misi diganti server — cek daftar di state / bundle |
| `weekly post: no tweet id in response` | Feature flag X berubah — refresh queryId + `FEATURES` di `src/xpost.py` |
| Swap sukses tapi AP tidak nambah | Cek `tx.to` — harus `nativeRouter`, dan tx sudah final baru bisa diverify |
| `stake: identity not eligible` | `registerAuthorized` gagal/terlambat — jalankan ulang; signature server punya deadline ~15 menit, bot selalu minta yang baru |
| `stake: tAMM 0.00 < 100, swap first` | tAMM kurang (habis dipakai stake/swap) — biarkan swap harian jalan dulu, stake menyusul besok |
| `verify stake: service_unavailable` | API Ammora sedang sibuk; bot retry 10× — kalau run habis, jalankan ulang, hash sama akan reconcile |

## Tangkapan layar

![run](docs/screenshots/run.png)

## Keamanan
`config.json` (private key) di-gitignore. Jangan pernah share. Kode: org `layerairdrop` (private), showcase: github.com/rmndkyl/ammora-bot.

*Dibuat oleh LYID — Layer Airdrop ID. Update project lifecycle dipantau cron claim-tracker (TGE/mainnet signal).*
