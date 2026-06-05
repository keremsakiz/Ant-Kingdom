# DURUM.md — Ant Kingdom Devir Belgesi

> Bu belge kalıcı devir notudur. Yeni oturumlar projeyi sıfırdan doğru kavrasın
> diye var. Tüm değerler `index.html` ve `CONFIG`'ten DOĞRULANMIŞTIR (tahmin değil).
> **Bu dosya her önemli adımdan sonra güncellenmeli.**

---

## 1. PROJE

**Ant Kingdom**: izometrik 2.5D tower-defense + karınca kolonisi simülasyonu.
- HTML5 Canvas, **TEK dosya** (`index.html`, ~2613 satır), **vanilla JS**, harici bağımlılık **YOK**.
- GitHub: `keremsakiz/Ant-Kingdom` · Canlı: `keremsakiz.github.io/Ant-Kingdom/`
- Hedef: App Store (Capacitor), **dokunmatik öncelik**, 390×844 baz çözünürlük.
- Branch: `claude/gifted-planck-JSihd` · Repo: `C:\Users\Onur\Ant-Kingdom`
- Grid 15×15, tile 64×32 (izometrik). Yuva merkezde (NEST_COL/ROW = 7,7).

---

## 2. TAMAMLANAN (koddan doğrulanmış, gerçek değerlerle)

### Faz 0/1 — Çekirdek
- İzometrik render, kamera: pan + pinch zoom (anchor'lı), zoom aralığı **0.55–2.5** (başlangıç 1.0).
- Koloni: Worker/Soldier/Carrier. Başlangıç: **10 worker, 3 soldier, 2 carrier**, MAX karınca **80**.
- Feromon ızgarası (CELL 14, MAX 100, DECAY 0.992, DEPOSIT 3).
- Yem ekonomisi: carrier yem taşır, yuvaya bırakınca `food` artar; yavaş worker doğumu.

### Faz 2A — Düşmanlar & Dalga (CONFIG.ENEMIES / CONFIG.WAVE, gerçek değerler)
- **Kraliçe HP**: MAX_HP = **100** (`CONFIG.QUEEN.MAX_HP`).
- **Dalga** (`CONFIG.WAVE`): PREP 15000ms, DURATION 60000ms, BASE_ENEMIES 6, ENEMY_PER_WAVE 3, **HP_GROWTH 1.22**, BONUS_PER_WAVE 50.
- **Soldier savaşı** (`CONFIG.ANT`): SOLDIER_DETECT_RANGE 260, SOLDIER_RANGE 45, SOLDIER_DPS 6, GUARD_RADIUS 80.
- **5 düşman** (hp / speed / queenDmg / reward / unlockWave):

| Düşman | hp | speed | queenDmg | reward | unlockWave | bina yıkar? | buildingDmg |
|---|---|---|---|---|---|---|---|
| spider 🕷 | 25 | 1.6 | 5 | 8 | 1 | **HAYIR** | — |
| ladybug 🐞 | 55 | 1.0 | 8 | 12 | 1 | **EVET** | 8 |
| dungbeetle 🪲 | 140 | 0.6 | 15 | 25 | 5 | **EVET** | 15 |
| bird 🐦 | 45 | 1.4 | 12 | 20 | 10 | **HAYIR** (uçar) | — |
| enemyant 🐜 | 80 | 1.1 | 10 | 18 | 20 | **EVET** | 12 |

### 7 BİNA (CONFIG.BUILDINGS, gerçek cost / hp / etki / maxCount)

| Anahtar | İsim | cost | hp | maxCount | Etki (gerçek değerler) |
|---|---|---|---|---|---|
| moat | Hendek 🌊 | 60 | 50 | ∞ | Üstündeki düşmanı yavaşlatır, hız çarpanı seviyeye göre `slowByLevel` [0.4, 0.3, 0.2] |
| barricade | Barikat 🪨 | 90 | 120 | ∞ | Engel; yıkıcı-olmayan düşman etrafından dolaşır |
| barracks | Kışla ⚔️ | 150 | 80 | ∞ | `spawnInterval` 15000ms'de bir soldier üretir (karınca MAX 80'e kadar) |
| tower | Zehir Kulesi 🍄 | 320 | 60 | **4** | AOE zehir: range 150, cooldown 2000ms, poisonDmg 28 |
| alarm | Şifa Taşı 💚 | 75 | 40 | ∞ | Kraliçeyi iyileştirir: healRange 130, healInterval 15000ms, healAmount **+3/15sn** |
| ranger | Atış Kulesi 🏹 | 220 | 70 | **4** | TEK HEDEF mermi: range 185, cooldown 900ms, dmg 14. **Yuva-çevresi kısıtı** (aşağıda) |
| catapult | Mancınık 🏰 | 300 | 70 | **3** | AOE ağır gülle: range 160, cooldown 2500ms, dmg 45, **aoeRadius 70** |

- **Yuva-çevresi kısıtı (SADECE ranger)**: `canPlaceTypeAt` → `nearNest`, Chebyshev mesafe ≤ **NEAR_NEST_R = 3** olan tile'lara kurulabilir.
- `canBuild`: yeterli `food` VAR **ve** (maxCount yok ya da o tipten limit dolmadı).

### Upgrade sistemi (Faz 3C)
- **LVL_MUL = [1.0, 1.4, 1.9]** → **3 seviye** (sv1/sv2/sv3, MAX = 3).
- `applyLevel()`: maxHP = `hp × LVL_MUL[lvl-1]`, can dolar. `effectMul()` = LVL_MUL (tower/healer/barracks/moat etkisi seviyeyle ölçeklenir).
- **upgradeCostFor**: sv2 = base × **1.5**, sv3 = base × **2.5** (yuvarlanmış).
- Upgrade balonu: her bina için **"şu an → yükseltince"** etki satırı (sv3'te sadece mevcut değer, "MAX").
- **Seviye görseli** (LVL_TIER, metin yok — renk + boyut + halka):
  - sv1: bronz/nötr, scale 1.00, halka yok
  - sv2: **gümüş** #c0c0c0, scale 1.12, halka + glow 0.16
  - sv3: **altın** #ffd700, scale 1.25, halka + glow 0.24
- **Long-press %75 yıkma**: dolu tile'da ~**500ms** (LONG_PRESS_MS) basılı tutunca bina yıkılır, `investedTotal × 0.75` yem **iade** edilir (sürükleme iptal eder).

### Healer sınırı (alarmCanHeal)
- Yuva menzilindeki şifa taşlarından **EN FAZLA 3'ü** (en eski kurulan, uid sıralı) kraliçeyi iyileştirir; fazlası boşa gider. Sv1 zayıf (+3/15sn) olduğu için ölümsüzlük oluşmaz.

### Faz 2B Parça 1 — Düşman-bina savaşı (koddan doğrulandı)
- **Bina yıkan düşmanlar**: ladybug, dungbeetle, enemyant (`isBuildingBreaker:true`). **spider yıkmaz**, **bird yıkmaz** (uçar).
- buildingDmg: ladybug **8**, dungbeetle **15**, enemyant **12** (saniyede hasar; `Enemy.step` içinde önündeki tile'da bina varsa durup yıkar).
- **Bina HP barı** çizilir; ~0.5sn'de bir kırmızı "-X" float + kıvılcım.
- **Düşman yıkınca İADE YOK** (`destroyBuildingByEnemy` — ceza). Oyuncunun long-press %75 iadesi bundan AYRI.

### Yuva HP upgrade (C — TAMAMLANDI)
- Yuva (merkez tile) tıklanınca açılan ayrı menü: `nestMenu` / `drawNestMenu` / `doNestUpgrade` / `nestUpgradeCost`. Bina upgrade sisteminden TAMAMEN ayrı.
- **SINIRSIZ** yükseltme: her seferinde **+50** max HP (`CONFIG.QUEEN.MAX_HP += 50`) + anlık `hudQueenHP += 50`.
- **Artan maliyet**: `nestUpgradeCost() = 80 + nestUpgradeCount * 40` (80 → 120 → 160 → ...).
- Yükseltmede **pembe/kırmızı puls** (`#ff5a7a`, şifa taşının yeşil `#5aff8a` pulsundan ayrı) + yükselen **"+50 ❤️"** float text.
- **`QUEEN_BASE_HP`** ile baz HP saklanır; `startGame`'de `CONFIG.QUEEN.MAX_HP = QUEEN_BASE_HP` + `nestUpgradeCount = 0` → yeni oyunda sıfırlanır.
- Menü görseli bina upgrade balonuyla aynı stil (panel/font/buton/afford); yetersiz yemde buton soluk, tıklama no-op.

### Yuva görseli (koloni vibe'ı)
- **`drawNestEntrance`**: düz halkalar yerine izometrik **kubbe/höyük** (dikey toprak gradyanı `#9c6b3a`→`#5a3d20`, oturma gölgesi), tepesinde mevcut halkalı giriş ("ağız").
- **`drawNestHole`**: 3×3 nest alanının **dört köşesinde** küçük ikincil höyük + koyu delik (merkezin ~%50'si, sade).
- **Ortak toprak doku**: tüm 3×3 nest tile'larına (köşe/kenar/merkez) deterministik kahve serpme → alan "tek yuva tarlası" gibi birleşir.

### UI
- **Radyal menü 7 ikon**: yarım dairede eşit yayılır (RADIAL_R 108, RADIAL_IR 19), `radialIconPos` TEK formül → çizim ve hit-test asla kaymaz.
- **Alt panel 7 buton** (taşma/kesilme düzeltildi).

### Ses (NOT: temel Web Audio ZATEN VAR)
- `initAudio`/`beep` + efektler (sndFood, sndDeposit, sndBorn, sndCatch, sndPower, sndBird) + basit melodi döngüsü mevcut. Faz 4'teki "ses" işi = cila/genişletme.

---

## 3. SON COMMIT'LER

```
9cdf812 revert: yuva patikaları kaldırıldı — ortak doku korundu
80c508a fix: yuva patikaları uçları kısaltıldı + görünürlük artırıldı
b550ffd polish: yuva patikaları + ortak toprak doku — bağlı koloni hissi
f7b4dfa polish: yuva köşelerine ikincil giriş delikleri — daha heybetli yuva
1e03161 polish: yuva girişi kubbe/höyük görünümüne çevrildi
eb3f62f polish: yuva menüsü bina menüsü stiliyle uyumlu + yükseltme pulsu
368c406 feat: yuva HP upgrade — sınırsız +50, artan maliyet 80+40n
3f820e0 feat: yuva menüsü iskeleti — tıklama+panel, HP mantığı yok
8993be9 DURUM.md devir belgesi
6a73773 Radyal menü: 7 ikon aralığı açıldı + hit-test tek formüle bağlandı
```

---

## 4. BEKLEYEN İŞLER (kullanıcı onayladı, sırayla)

- **B) BİNA SEVİYESİ 3→6 — SIRADAKİ İŞ (en büyük iş).** `LVL_MUL`'u **6 seviyeye** çıkar (şu an 3),
  upgrade maliyetlerini (`upgradeCostFor` sadece sv2/sv3 biliyor), kabarcık MAX'ı 6'ya,
  `isMax` ve `level >= 3` kontrollerini 6'ya güncelle. Seviye görseli (`LVL_TIER` şu an
  3 kademe): sv4 bronz ışıma, sv5 gümüş, sv6 altın olacak şekilde 6 kademeye yeniden düzenle.

- **DENGE**: dalga ~22'de oyuncu ölüyor. Düşman büyümesi (HP_GROWTH 1.22) ZOR kalsın
  (dokunma); savunma tavanı (seviye 6 + yuva HP) ile karşılansın. Sonra ince ayar.

- **Sonra**: Faz 2B kalanı (boss dalgalar, düşman özel yetenekleri — spider ağ,
  dungbeetle itme, enemyant soldier avı vb.), Faz 4 (ana menü/HUD cila, ses genişletme,
  localStorage skor).

### Ertelenen / Notlar
- **Yuvalar arası koloni bağlantısı hissi**: statik toprak patika (`drawNestPaths`) denendi ama
  izometrik perspektifte höyüklerle çakıştığı için **iptal edildi** (revert: `9cdf812`). İleride
  **karınca hareketiyle** (deliklerden giriş/çıkış) yapılacak — karınca AI hareket işiyle birlikte.

---

## 5. TEST CHEAT'LERİ (tarayıcı Console — hepsi global `let`)

```js
food = 5000;        // bol yem
hudQueenHP = 40;    // kraliçe HP'sini düşür (test)
wave = N;           // dalga numarasını ayarla
phaseTimer = 999999;// fazı atla/zıpla (PREP veya WAVE biter)
```

---

## 6. DOKUNMA (çalışıyor — BOZMA)

- Render optimizasyonu: emoji sprite önbellek, gradient cache, baked gölge.
- Depth-sort: `wy + uid` ile stable sıralama (flicker düzeltmesi yapıldı).
- Kamera pinch anchor (parmak altındaki nokta sabit kalır).
- **Görsel hedef**: 2D sprite ile profesyonel görünüm (**3D DEĞİL**). Sprite geçişi tüm
  mekanik bitince **EN SONA** bırakılacak.

---

> **Hatırlatma**: Bu dosya her önemli adımdan sonra güncellenmeli ki yeni oturumlar
> projeyi doğru kavrasın. Hiçbir maddeyi tahmin etme — koddan/CONFIG'ten doğrula.
