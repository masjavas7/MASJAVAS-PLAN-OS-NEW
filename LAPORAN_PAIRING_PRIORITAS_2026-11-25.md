# Daftar Prioritas Pairing per Operator — MASJAVAS TUBE (Fase 3.1)

Dibuat: 2026-11-25 08:34 WIB | Sumber: DB Master (read-only) `/opt/masjavas-os/data/masjavas_tube.sqlite`
Generator: `scripts/pairing_priority.py` (dapat di-run ulang; output deterministik dari keadaan DB).

## Ringkasan

- Total slot: **707** | CONNECTED: **243** (34.4%) | UNPAIRED: **464** (65.6%)

| Operator | Total | CONNECTED | UNPAIRED | UNPAIRED tanpa vault | UNPAIRED tanpa email | Target/minggu |
|---|---:|---:|---:|---:|---:|---:|
| AZIZ | 251 | 11 | 240 | 240 | 240 | 10 |
| ARIEF | 143 | 17 | 126 | 126 | 122 | 5 |
| AGUS | 100 | 70 | 30 | 29 | 30 | 1 |
| HAMIDAH | 40 | 25 | 15 | 14 | 3 | 1 |
| SALSA | 40 | 25 | 15 | 15 | 5 | 1 |
| WULAN | 40 | 25 | 15 | 14 | 4 | 1 |
| BILA | 40 | 27 | 13 | 13 | 0 | 1 |
| LUTHFI | 40 | 30 | 10 | 9 | 5 | 0 |
| MONA | 13 | 13 | 0 | 0 | 0 | 0 |

## Kapasitas vault (penghambat utama)

- Vault terdaftar (`oauth_masters`): 60; terpakai: 59; cap 5 channel/vault (aturan 19 Nov 2026).
- Slot vault kosong pada cap-5: **55** — pada +20/minggu, kapasitas ini habis dalam ~2.8 minggu; setelah itu vault baru wajib dibuat (1 vault = 1 project GCP + OAuth client).
- Vault MELEBIHI cap-5: MV-252AB8E353B2421B (6 ch), MV-57D170A57C9BA7EE (6 ch) — peninggalan cap-6 lama; UI modal masih menulis 'Batas Kuota 6 Ch / Vault' (web/templates/modals/modal_channel_account.html:118).

## Aturan urutan

- Widget dashboard pairing (api/main.py:404-434) hanya menampilkan hitungan — TIDAK punya aturan urutan. Urutan di sini mengadopsi konvensi codebase: **slot ASC** (core/ingestion/fleet_audit.py §5, `build_worklist`).
- **P1 — siap pairing sekarang**: UNPAIRED yang sudah punya `master_vault_id` (modal pairing mewajibkan vault dipilih sebelum OAuth connect).
- **P2 — butuh penetapan vault dulu**: UNPAIRED tanpa `master_vault_id` (461 dari 464). Pairing P2 tidak bisa jalan sebelum vault ditetapkan — ini pekerjaan OWNER/koordinator, bukan operator.
- Operator diurutkan by UNPAIRED terbanyak dulu. Target mingguan dibagi proporsional terhadap backlog (largest remainder).

## AZIZ — UNPAIRED 240 (target minggu ini: 10)

### P2 — butuh vault dulu (240 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 356 | Happy Rewind | Tidak |
| 2 | 357 | Gentle Yesterday | Tidak |
| 3 | 358 | Faded Melody | Tidak |
| 4 | 359 | Dream Coast | Tidak |
| 5 | 360 | Amber Shore | Tidak |
| 6 | 361 | Calm Haven | Tidak |
| 7 | 362 | Silk Groove | Tidak |
| 8 | 363 | Electric Velvet | Tidak |
| 9 | 364 | Slow Ember | Tidak |
| 10 | 365 | Sunset Soul | Tidak |
| 11 | 366 | Crimson Night | Tidak |
| 12 | 367 | Tender Soul | Tidak |
| 13 | 368 | Smooth Serenade | Tidak |
| 14 | 369 | Retro Soul | Tidak |
| 15 | 370 | Golden Soul | Tidak |
| 16 | 371 | Starlit Soul | Tidak |
| 17 | 372 | Velvet Bedroom | Tidak |
| 18 | 373 | Tearful Soul | Tidak |
| 19 | 374 | Soulful Nights | Tidak |
| 20 | 375 | Neon Soul | Tidak |
| 21 | 376 | Soul Anthems | Tidak |
| 22 | 377 | Pop Horizon | Tidak |
| 23 | 378 | Pop Gold | Tidak |
| 24 | 379 | English Melody | Tidak |
| 25 | 380 | Summit Hits | Tidak |
| 26 | 381 | Pop Nova | Tidak |
| 27 | 382 | Viral Melody | Tidak |
| 28 | 383 | Global Echo | Tidak |
| 29 | 384 | English Gold | Tidak |
| 30 | 385 | Pop Blend | Tidak |
| 31 | 386 | Timeless Favorites | Tidak |
| 32 | 387 | Chart Nights | Tidak |
| 33 | 388 | Fresh Pop | Tidak |
| 34 | 389 | World Pop | Tidak |
| 35 | 390 | Sunny Pop | Tidak |
| 36 | 391 | Popular Waves | Tidak |
| 37 | 392 | Rainy Melody | Tidak |
| 38 | 393 | Shattered Heart | Tidak |
| 39 | 394 | Misty Tears | Tidak |
| 40 | 395 | Emotional Waves | Tidak |
| 41 | 396 | Fractured Heart | Tidak |
| 42 | 397 | Teardrop Melody | Tidak |
| 43 | 398 | Lonely Moon | Tidak |
| 44 | 399 | Deep Emotions | Tidak |
| 45 | 400 | Wistful Love | Tidak |
| 46 | 401 | Missing You | Tidak |
| 47 | 402 | Heavy Heart | Tidak |
| 48 | 403 | Hazy Reverie | Tidak |
| 49 | 404 | Mending Heart | Tidak |
| 50 | 405 | Aching Heart | Tidak |
| 51 | 406 | Faded Memories | Tidak |
| 52 | 407 | Midnight Highway | Tidak |
| 53 | 408 | Midnight Roads | Tidak |
| 54 | 409 | Open Highway | Tidak |
| 55 | 410 | After Dark | Tidak |
| 56 | 411 | Neon Streets | Tidak |
| 57 | 412 | Car Stereo | Tidak |
| 58 | 413 | Highway Lights | Tidak |
| 59 | 414 | Nightfall Vibes | Tidak |
| 60 | 415 | Metro Lights | Tidak |
| 61 | 416 | Night Rider | Tidak |
| 62 | 417 | Midnight Pop | Tidak |
| 63 | 418 | Retro Drive | Tidak |
| 64 | 419 | Cruise Control | Tidak |
| 65 | 420 | Dusk Drive | Tidak |
| 66 | 421 | Drive Anthems | Tidak |
| 67 | 422 | Coffee Corner | Tidak |
| 68 | 423 | Coffeehouse Tales | Tidak |
| 69 | 424 | Cafe Melody | Tidak |
| 70 | 425 | Warm Cup | Tidak |
| 71 | 426 | Morning Brew | Tidak |
| 72 | 427 | Sunday Calm | Tidak |
| 73 | 428 | Soft Melody | Tidak |
| 74 | 429 | Acoustic Brew | Tidak |
| 75 | 430 | Calm Cafe | Tidak |
| 76 | 431 | Quiet Background | Tidak |
| 77 | 432 | Cozy Glow | Tidak |
| 78 | 433 | Rainy Afternoon | Tidak |
| 79 | 434 | Aesthetic Mornings | Tidak |
| 80 | 435 | Study Brews | Tidak |
| 81 | 436 | Warm Vinyl | Tidak |
| 82 | 437 | Love Letters | Tidak |
| 83 | 438 | Romantic Echo | Tidak |
| 84 | 439 | Golden Love | Tidak |
| 85 | 440 | English Romance | Tidak |
| 86 | 441 | Two Souls | Tidak |
| 87 | 442 | Love Haze | Tidak |
| 88 | 443 | Moonlit Romance | Tidak |
| 89 | 444 | Love Anthology | Tidak |
| 90 | 445 | Honey Love | Tidak |
| 91 | 446 | Wedding Vows | Tidak |
| 92 | 547 | Son Habana | Tidak |
| 93 | 548 | Tres Habana | Tidak |
| 94 | 549 | Son Caribe | Tidak |
| 95 | 550 | Son Eterno | Tidak |
| 96 | 551 | Bolero Habana | Tidak |
| 97 | 552 | Bolero Luna | Tidak |
| 98 | 553 | Bolero Clasico | Tidak |
| 99 | 554 | Havana Jazz | Tidak |
| 100 | 555 | Jazz Caribe | Tidak |
| 101 | 556 | Latin Breeze | Tidak |
| 102 | 557 | Cafe Habana | Tidak |
| 103 | 558 | Havana Lounge | Tidak |
| 104 | 559 | Havana Nights | Tidak |
| 105 | 560 | Havana Moon | Tidak |
| 106 | 561 | Havana Nocturne | Tidak |
| 107 | 562 | Mambo Lounge | Tidak |
| 108 | 563 | Chacha Nights | Tidak |
| 109 | 564 | Tango Nocturno | Tidak |
| 110 | 565 | Grace Lofi | Tidak |
| 111 | 566 | Psalm Beats | Tidak |
| 112 | 567 | Faith Lofi | Tidak |
| 113 | 568 | Canaan Jazz | Tidak |
| 114 | 569 | Manna Jazz | Tidak |
| 115 | 570 | Gospel Keys | Tidak |
| 116 | 571 | Soul Rest | Tidak |
| 117 | 572 | Velvet Prayer | Tidak |
| 118 | 573 | Praise Flow | Tidak |
| 119 | 574 | Hymn Haven | Tidak |
| 120 | 575 | Old Hymns | Tidak |
| 121 | 576 | Verse Lofi | Tidak |
| 122 | 577 | Bible Beats | Tidak |
| 123 | 578 | Naija Praise | Tidak |
| 124 | 579 | Louvor Calmo | Tidak |
| 125 | 580 | Bossa Serena | Tidak |
| 126 | 581 | Rio Bossa | Tidak |
| 127 | 582 | Copacabana Nights | Tidak |
| 128 | 583 | Ipanema Sands | Tidak |
| 129 | 584 | Saudade Bossa | Tidak |
| 130 | 585 | Bossa Manha | Tidak |
| 131 | 586 | Delta Porch | Tidak |
| 132 | 587 | Mississippi Moon | Tidak |
| 133 | 588 | Whiskey Delta | Tidak |
| 134 | 589 | Harmonica Delta | Tidak |
| 135 | 590 | Neo Velvet | Tidak |
| 136 | 591 | Midnight Soul | Tidak |
| 137 | 592 | Quiet Storm | Tidak |
| 138 | 593 | Soul Nocturne | Tidak |
| 139 | 594 | Saxophone Jazz | Tidak |
| 140 | 595 | Hammond Groove | Tidak |
| 141 | 596 | West Coast | Tidak |
| 142 | 597 | Paris Swing | Tidak |
| 143 | 598 | Swing Era | Tidak |
| 144 | 599 | Samba Jazz | Tidak |
| 145 | 600 | Dekade Emas | Tidak |
| 146 | 601 | Radio Lawas | Tidak |
| 147 | 602 | Swara Lawas | Tidak |
| 148 | 603 | Dendang Tempo | Tidak |
| 149 | 604 | Serumpun Asmara | Tidak |
| 150 | 605 | Selat Melaka | Tidak |
| 151 | 606 | Semenanjung Kasih | Tidak |
| 152 | 607 | Tresno Loro | Tidak |
| 153 | 608 | Kangen Wengi | Tidak |
| 154 | 609 | Keroncong Kemayoran | Tidak |
| 155 | 610 | Cuk Cak | Tidak |
| 156 | 611 | Campursari Sragenan | Tidak |
| 157 | 612 | Kendang Kempul | Tidak |
| 158 | 613 | Langgam Jawa | Tidak |
| 159 | 614 | Laras Sragen | Tidak |
| 160 | 615 | Mojang Pasundan | Tidak |
| 161 | 616 | Kacapi Suling | Tidak |
| 162 | 617 | Gendang Seruling | Tidak |
| 163 | 618 | Seruling Bambu | Tidak |
| 164 | 619 | Mandolin Lawas | Tidak |
| 165 | 620 | Ampli Tua | Tidak |
| 166 | 621 | Vinyl Lawas | Tidak |
| 167 | 622 | Sajak Renungan | Tidak |
| 168 | 623 | Elegi Waktu | Tidak |
| 169 | 624 | Orkes Deli | Tidak |
| 170 | 625 | Gurindam Kasih | Tidak |
| 171 | 626 | Gambus Arab | Tidak |
| 172 | 627 | Qasidah Modern | Tidak |
| 173 | 628 | Gending Jawa | Tidak |
| 174 | 629 | Uyon Uyon | Tidak |
| 175 | 630 | Akustik Petik | Tidak |
| 176 | 631 | Akustik Senar | Tidak |
| 177 | 632 | Akustik Lirih | Tidak |
| 178 | 633 | Akustik Denting | Tidak |
| 179 | 634 | Indie Layar | Tidak |
| 180 | 635 | Indie Renjana | Tidak |
| 181 | 636 | Indie Kelana | Tidak |
| 182 | 637 | Indie Pesisir | Tidak |
| 183 | 638 | Koplo Kendang | Tidak |
| 184 | 639 | Koplo Senggol | Tidak |
| 185 | 640 | Koplo Gending | Tidak |
| 186 | 641 | Koplo Rancak | Tidak |
| 187 | 642 | Dangdut Jedug | Tidak |
| 188 | 643 | Dangdut Gebyar | Tidak |
| 189 | 644 | Dangdut Kempul | Tidak |
| 190 | 646 | Reggae Selancar | Tidak |
| 191 | 647 | Reggae Karibia | Tidak |
| 192 | 648 | Sholawat Qalbu | Tidak |
| 193 | 649 | Sholawat Hadroh | Tidak |
| 194 | 650 | Sholawat Rebana | Tidak |
| 195 | 651 | Sholawat Madinah | Tidak |
| 196 | 652 | Galau Mendung | Tidak |
| 197 | 653 | Galau Kelabu | Tidak |
| 198 | 654 | Galau Rintik | Tidak |
| 199 | 655 | Ambyar Tresno | Tidak |
| 200 | 656 | Ambyar Kangen | Tidak |
| 201 | 657 | Lofi Kamar | Tidak |
| 202 | 658 | Lofi Rebahan | Tidak |
| 203 | 659 | Funkot Gempita | Tidak |
| 204 | 660 | Rnb Kelam | Tidak |
| 205 | 661 | Hiphop Lorong | Tidak |
| 206 | 662 | Ivory Haven | Tidak |
| 207 | 663 | Felt Piano | Tidak |
| 208 | 664 | Moonlit Keys | Tidak |
| 209 | 665 | Piano Reverie | Tidak |
| 210 | 666 | Forest Murmur | Tidak |
| 211 | 667 | Ocean Hush | Tidak |
| 212 | 668 | Emerald Harp | Tidak |
| 213 | 669 | Glen Songs | Tidak |
| 214 | 670 | Dublin Hearth | Tidak |
| 215 | 671 | Tavern Tales | Tidak |
| 216 | 672 | Castle Hearth | Tidak |
| 217 | 673 | Elderwood Mist | Tidak |
| 218 | 674 | Jukebox Memories | Tidak |
| 219 | 675 | Golden Jukebox | Tidak |
| 220 | 676 | Timber Hollow | Tidak |
| 221 | 677 | Prairie Roads | Tidak |
| 222 | 678 | Velvet Strings | Tidak |
| 223 | 679 | Baroque Mornings | Tidak |
| 224 | 680 | Unplugged Hour | Tidak |
| 225 | 681 | Fireside Covers | Tidak |
| 226 | 682 | Balearic Sunset | Tidak |
| 227 | 683 | Azure Lounge | Tidak |
| 228 | 684 | Neon Dusk | Tidak |
| 229 | 685 | Chrome Sunset | Tidak |
| 230 | 686 | Sacred Voices | Tidak |
| 231 | 687 | Monastery Echo | Tidak |
| 232 | 688 | Seoul Breeze | Tidak |
| 233 | 689 | Han River | Tidak |
| 234 | 690 | Busan Nights | Tidak |
| 235 | 691 | Sakura Nights | Tidak |
| 236 | 692 | Kyoto Rain | Tidak |
| 237 | 693 | Andalusian Strings | Tidak |
| 238 | 694 | Patio Guitar | Tidak |
| 239 | 695 | Silver Harp | Tidak |
| 240 | 696 | Cradle Tunes | Tidak |

## ARIEF — UNPAIRED 126 (target minggu ini: 5)

### P2 — butuh vault dulu (126 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 107 | Bintang Saka | Tidak |
| 2 | 108 | Kisah Embun | Tidak |
| 3 | 109 | Pelabuhan Aruna | Tidak |
| 4 | 110 | Embun Jingga | Tidak |
| 5 | 111 | Gelombang Galau | Tidak |
| 6 | 115 | Senyum Lara | Ya |
| 7 | 117 | Tanpa Pamitan | Ya |
| 8 | 118 | Cinta Lara | Ya |
| 9 | 121 | Kapak Melayu | Tidak |
| 10 | 122 | Jiwang Kala | Tidak |
| 11 | 123 | Kenangan Kapak | Tidak |
| 12 | 124 | Gerimis Lara | Tidak |
| 13 | 125 | Ambang Senja | Tidak |
| 14 | 126 | Bintang Kapak | Tidak |
| 15 | 127 | Untuk Ibu | Tidak |
| 16 | 128 | Kenangan Lara | Tidak |
| 17 | 129 | Jiwang Aruna | Tidak |
| 18 | 130 | Melayu Saka | Tidak |
| 19 | 131 | Melodi Rindu | Tidak |
| 20 | 132 | Kaset Kenangan | Tidak |
| 21 | 133 | Tembang Aruna | Tidak |
| 22 | 134 | Lawas Embun | Tidak |
| 23 | 135 | Masa Saka | Tidak |
| 24 | 136 | Pop Nilam | Tidak |
| 25 | 137 | Balada Kala | Tidak |
| 26 | 138 | Suara Aruna | Tidak |
| 27 | 139 | Zaman Embun | Tidak |
| 28 | 140 | Kenangan Jingga | Tidak |
| 29 | 141 | Ambyar Rampak | Tidak |
| 30 | 142 | Blekuk Kala | Tidak |
| 31 | 143 | Ambyar Lara | Tidak |
| 32 | 144 | Rampak Aruna | Tidak |
| 33 | 145 | Sinden Nilam | Tidak |
| 34 | 146 | Galau Rampak | Tidak |
| 35 | 147 | Koplo Aruna | Tidak |
| 36 | 148 | Ambyar Desa | Tidak |
| 37 | 149 | Koplo Nilam | Tidak |
| 38 | 150 | Joget Embun | Tidak |
| 39 | 151 | Reggae Motivasi | Tidak |
| 40 | 152 | Rasta Damai | Tidak |
| 41 | 154 | Reggae Kala | Tidak |
| 42 | 155 | Reggae Aruna | Tidak |
| 43 | 156 | Reggae Sahabat | Tidak |
| 44 | 157 | Jalanan Reggae | Tidak |
| 45 | 158 | Damai Aruna | Tidak |
| 46 | 159 | Tongkrongan Saka | Tidak |
| 47 | 160 | Malam Reggae | Tidak |
| 48 | 161 | Lagu Santai | Tidak |
| 49 | 162 | Kopi Aruna | Tidak |
| 50 | 163 | Gitar Nilam | Tidak |
| 51 | 164 | Cover Embun | Tidak |
| 52 | 165 | Balkon Saka | Tidak |
| 53 | 166 | Hujan Lara | Tidak |
| 54 | 167 | Senja Nilam | Tidak |
| 55 | 168 | Kopi Embun | Tidak |
| 56 | 169 | Pelabuhan Nilam | Tidak |
| 57 | 170 | Akustik Aruna | Tidak |
| 58 | 171 | Kerja Aruna | Tidak |
| 59 | 172 | Tidur Embun | Tidak |
| 60 | 173 | Kafe Saka | Tidak |
| 61 | 174 | Hujan Nilam | Tidak |
| 62 | 175 | Fokus Saka | Tidak |
| 63 | 176 | Perjalanan Aruna | Tidak |
| 64 | 177 | Malam Nilam | Tidak |
| 65 | 178 | Taman Embun | Tidak |
| 66 | 179 | Rumah Aruna | Tidak |
| 67 | 180 | Instrumental Nilam | Tidak |
| 68 | 181 | Sholawat Aruna | Tidak |
| 69 | 182 | Sholawat Kala | Tidak |
| 70 | 183 | Religi Nilam | Tidak |
| 71 | 184 | Sholawat Embun | Tidak |
| 72 | 185 | Nasyid Aruna | Tidak |
| 73 | 186 | Sholawat Desa | Tidak |
| 74 | 187 | Religi Saka | Tidak |
| 75 | 188 | Anak Aruna | Tidak |
| 76 | 189 | Doa Embun | Tidak |
| 77 | 190 | Sholawat Rampak | Tidak |
| 78 | 191 | Afro Aruna | Tidak |
| 79 | 192 | Remix Embun | Tidak |
| 80 | 193 | Bali Saka | Tidak |
| 81 | 194 | Dance Aruna | Tidak |
| 82 | 195 | Afro Kala | Tidak |
| 83 | 196 | Remix Lara | Tidak |
| 84 | 197 | Latin Saka | Tidak |
| 85 | 198 | Remix Rampak | Tidak |
| 86 | 199 | Afro Nilam | Tidak |
| 87 | 200 | Pekan Aruna | Tidak |
| 88 | 201 | Sendu Rembulan | Tidak |
| 89 | 202 | Lara Jingga | Tidak |
| 90 | 203 | Sayu Jiwang | Tidak |
| 91 | 204 | Rindu Kapak | Tidak |
| 92 | 205 | Pilu Senja | Tidak |
| 93 | 206 | Alun Tenang | Tidak |
| 94 | 207 | Jiwang Melayu | Tidak |
| 95 | 208 | Blues Kenangan | Tidak |
| 96 | 209 | Senja Jiwang | Tidak |
| 97 | 210 | Badai Rindu | Tidak |
| 98 | 211 | Malam Syahdu | Tidak |
| 99 | 212 | Luka Senja | Tidak |
| 100 | 213 | Kenangan Senja | Tidak |
| 101 | 214 | Rindu Senja | Tidak |
| 102 | 215 | Embun Pagi | Tidak |
| 103 | 216 | Desa Kenangan | Tidak |
| 104 | 217 | Angkasa Hening | Tidak |
| 105 | 218 | Dangdut Senja | Tidak |
| 106 | 219 | Hening Malam | Tidak |
| 107 | 220 | Riang Sore | Tidak |
| 108 | 221 | Rindu Purnama | Tidak |
| 109 | 222 | Sendu Fajar | Tidak |
| 110 | 223 | Langit Jingga | Tidak |
| 111 | 224 | Koplo Rindu | Tidak |
| 112 | 225 | Pelita Malam | Tidak |
| 113 | 226 | Cahaya Senja | Tidak |
| 114 | 227 | Awan Senja | Tidak |
| 115 | 228 | Samudra Biru | Tidak |
| 116 | 229 | Teduh Jingga | Tidak |
| 117 | 230 | Rindu Jiwang | Tidak |
| 118 | 231 | Senja Akustik | Tidak |
| 119 | 232 | Luruh Jiwang | Tidak |
| 120 | 233 | Sholawat Senja | Tidak |
| 121 | 234 | Sholawat Teduh | Tidak |
| 122 | 235 | Nasyid Teduh | Tidak |
| 123 | 236 | Doa Teduh | Tidak |
| 124 | 237 | Remix Gema | Tidak |
| 125 | 238 | Afro Senja | Tidak |
| 126 | 241 | Nada Hati | Ya |

## AGUS — UNPAIRED 30 (target minggu ini: 1)

### P1 — siap pairing (sudah punya vault)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 317 | Kerja Fokus | Tidak |

### P2 — butuh vault dulu (29 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 318 | Tidur Nyenyak | Tidak |
| 2 | 319 | Lofi Pagi | Tidak |
| 3 | 320 | Piano Malam | Tidak |
| 4 | 321 | Fokus Belajar | Tidak |
| 5 | 322 | Drive Perjalanan | Tidak |
| 6 | 323 | Meditasi Tenang | Tidak |
| 7 | 324 | Taman Alam | Tidak |
| 8 | 325 | WFH Tenang | Tidak |
| 9 | 326 | Alam Senja | Tidak |
| 10 | 327 | Pagi Tenang | Tidak |
| 11 | 328 | Malam Ibadah | Tidak |
| 12 | 329 | Religi Santai | Tidak |
| 13 | 330 | Anak Ceria | Tidak |
| 14 | 331 | Jawa Damai | Tidak |
| 15 | 332 | Senja Syukur | Tidak |
| 16 | 333 | Koplo Syukur | Tidak |
| 17 | 334 | Melayu Religi | Tidak |
| 18 | 335 | Keluarga Bahagia | Tidak |
| 19 | 336 | Harapan Pagi | Tidak |
| 20 | 337 | Senja Kota | Tidak |
| 21 | 338 | Liburan Bali | Tidak |
| 22 | 339 | Semangat Pagi | Tidak |
| 23 | 340 | Malam Dansa | Tidak |
| 24 | 341 | Ritme Latin | Tidak |
| 25 | 342 | Malam Remix | Tidak |
| 26 | 343 | Remix Sedih | Tidak |
| 27 | 344 | Fusi Dangdut | Tidak |
| 28 | 345 | Akhir Pekan | Tidak |
| 29 | 346 | Perjalanan Senja | Tidak |

## HAMIDAH — UNPAIRED 15 (target minggu ini: 1)

### P1 — siap pairing (sudah punya vault)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 464 | Slow Ember | Ya |

### P2 — butuh vault dulu (14 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 479 | Lyrical Skies | Ya |
| 2 | 484 | Golden Lyrics | Ya |
| 3 | 489 | Atlas Pop | Ya |
| 4 | 494 | Grey Ballads | Ya |
| 5 | 499 | Deep Ache | Ya |
| 6 | 504 | Heartache Hymns | Ya |
| 7 | 509 | Asphalt Dreams | Ya |
| 8 | 514 | Nightfall Haze | Ya |
| 9 | 519 | Breezy Drive | Ya |
| 10 | 524 | Cafe Serenade | Ya |
| 11 | 529 | Acoustic Aroma | Ya |
| 12 | 534 | Pastel Cafe | Tidak |
| 13 | 539 | Eternal Love | Tidak |
| 14 | 544 | Beloved Collection | Tidak |

## SALSA — UNPAIRED 15 (target minggu ini: 1)

### P2 — butuh vault dulu (15 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 473 | Indigo Mood | Ya |
| 2 | 478 | Pop Radiance | Ya |
| 3 | 483 | World Chorus | Ya |
| 4 | 488 | Nova Pop | Ya |
| 5 | 493 | Broken Melody | Ya |
| 6 | 498 | Lonely Lantern | Ya |
| 7 | 503 | Blue Reverie | Ya |
| 8 | 508 | Midnight Cruise | Ya |
| 9 | 513 | Interstate Glow | Ya |
| 10 | 518 | Retro Sunset | Ya |
| 11 | 523 | Warm Mug | Tidak |
| 12 | 528 | Soft Whisper | Tidak |
| 13 | 533 | Rainy Window | Tidak |
| 14 | 538 | Rose Petals | Tidak |
| 15 | 543 | Candlelit Echo | Tidak |

## WULAN — UNPAIRED 15 (target minggu ini: 1)

### P1 — siap pairing (sudah punya vault)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 472 | LVVSTT | Ya |

### P2 — butuh vault dulu (14 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 477 | Hit Horizon | Ya |
| 2 | 482 | Viral Bloom | Ya |
| 3 | 487 | Chart Toppers | Ya |
| 4 | 492 | Rainy Echo | Ya |
| 5 | 497 | Teardrop Ballad | Ya |
| 6 | 502 | Grey Horizon | Ya |
| 7 | 507 | Neon Highway | Ya |
| 8 | 512 | Dashboard Beats | Ya |
| 9 | 517 | Shadow Pop | Ya |
| 10 | 522 | Coffee Chords | Ya |
| 11 | 527 | Sunday Glow | Tidak |
| 12 | 532 | Snug Vibes | Tidak |
| 13 | 537 | Tender Romance | Tidak |
| 14 | 542 | Sweetheart Serenade | Tidak |

## BILA — UNPAIRED 13 (target minggu ini: 1)

### P2 — butuh vault dulu (13 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 486 | Timeless Tunes | Ya |
| 2 | 491 | Crowd Favorites | Ya |
| 3 | 496 | Shattered Chords | Ya |
| 4 | 501 | Distant Hearts | Ya |
| 5 | 506 | Faded Photographs | Ya |
| 6 | 511 | Moonlit Drive | Ya |
| 7 | 516 | Evening Ride | Ya |
| 8 | 521 | Road Anthems | Ya |
| 9 | 526 | Andrew. | Ya |
| 10 | 531 | Afterhours Acoustics | Ya |
| 11 | 536 | Smoky Jazz | Ya |
| 12 | 541 | Two Hearts | Ya |
| 13 | 546 | Bridal Melody | Ya |

## LUTHFI — UNPAIRED 10 (target minggu ini: 0)

### P1 — siap pairing (sudah punya vault)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 485 | Pop Mosaic | Ya |

### P2 — butuh vault dulu (9 channel, slot ASC)

| # | NO | Channel | Email |
|---:|---:|---|:---:|
| 1 | 505 | Aching Soul | Ya |
| 2 | 510 | Starlit Drive | Ya |
| 3 | 515 | City Glow | Ya |
| 4 | 520 | Sunset Cruise | Ya |
| 5 | 525 | Cozy Corner | Tidak |
| 6 | 530 | Mellow Roast | Tidak |
| 7 | 535 | Focus Brew | Tidak |
| 8 | 540 | Lovely Verses | Tidak |
| 9 | 545 | Sweet Devotion | Tidak |

