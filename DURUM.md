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
- **Dalga** (`CONFIG.WAVE`): PREP 15000ms, DURATION 60000ms, BASE_ENEMIES 6, ENEMY_PER_WAVE 3, **HP_GROWTH 1.22**, **BONUS_PER_WAVE 80** (balans pass'inde 50→80).
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

### Upgrade sistemi (Faz 3C + Faz3B 3→6)
- **LVL_MUL = [1.0, 1.4, 1.9, 2.3, 2.6, 2.9]** → **6 seviye** (sv1..sv6, MAX = 6).
- `applyLevel()`: maxHP = `hp × LVL_MUL[lvl-1]`, can dolar. `effectMul()` = LVL_MUL (tower/healer/barracks/moat etkisi seviyeyle ölçeklenir).
- **upgradeCostFor**: tablo `MUL = {2:1.5, 3:2.5, 4:3.5, 5:4.5, 6:6}` → base × çarpan (yuvarlanmış). Tablo dışı seviye = 0. **Balans pass'inde sv4-6 çarpanları düşürüldü** (eski {4:4, 5:6, 6:8.5}) → üst seviye binalar artık erişilebilir.
- Upgrade balonu: her bina için **"şu an → yükseltince"** etki satırı (sv6'da sadece mevcut değer, "MAX"). Etiket **"Sv X/6"**.
- **Beş seviye-tavanı kontrolü 3→6 yükseltildi**: `effectLine` isMax (`lvl >= 6`), drawUpgradeMenu "/6" etiketi, drawUpgradeMenu MAX kapısı (`b.level >= 6`), `upgradeBtnHit` (`>= 6`), `doUpgrade` guard (`>= 6`).
- **Seviye görseli** (LVL_TIER, metin yok — renk + boyut + halka). sv1-3 metal, sv4-6 enerji renkleri (daha güçlü glow):
  - sv1: bronz #cd7f32, scale 1.00, halka yok, glow 0.00
  - sv2: **gümüş** #c0c0c0, scale 1.12, halka + glow 0.18
  - sv3: **altın** #ffd700, scale 1.24, halka + glow 0.26
  - sv4: **turkuaz** #1fd6c4, scale 1.36, halka + glow 0.42
  - sv5: **mor** #a855f7, scale 1.48, halka + glow 0.55
  - sv6: **kızıl-altın** #ff5a3c, scale 1.62, halka + glow 0.70
- **sv4+ nabız parıltısı**: bina draw bloğunda `this.level >= 4` için, `performance.now()` + `(gridX+gridY)` faz kaymasıyla nabız gibi atan dış parıltı halkası (her bina ayrı fazda).
- **Mermi rengi seviyeye göre**: ranger/catapult mermileri yeni `lvl` alanı taşır; `drawBullets` rengi `LVL_TIER[lvl-1].color`'dan alır (lvl yoksa eski sabit renkler #c8783c / #ffd24a). Mancınık güllesi hep kalın, kalınlık `6 + (lvl-1)*0.7` ile seviyeyle hafif artar; ranger sabit 3.
- **Long-press %75 yıkma**: dolu tile'da ~**500ms** (LONG_PRESS_MS) basılı tutunca bina yıkılır, `investedTotal × 0.75` yem **iade** edilir (sürükleme iptal eder).

### Healer sınırı (alarmCanHeal)
- Yuva menzilindeki şifa taşlarından **EN FAZLA 3'ü** (en eski kurulan, uid sıralı) kraliçeyi iyileştirir; fazlası boşa gider. Sv1 zayıf (+3/15sn) olduğu için ölümsüzlük oluşmaz.

### Balans Pass'i — Ekonomi Dengesi (TAMAMLANDI, main'e merge+push)
Sorun: oyuncu hilesiz ~dalga 12'de tıkanıyordu. Kök neden **ekonomi**: ~12. dalgaya
kadar toplanan food, tek bina bile sv6'ya çıkmaya yetmiyordu (sv6 DPS tavanı kâğıt
üstünde vardı ama erişilemezdi). Dört değişiklik (hepsi koddan doğrulandı):
1. **`BONUS_PER_WAVE` 50 → 80** (dalga sonu bonus food; `endWave`).
2. **Öldürme food ödülü**: `killEnemy` içinde `food += Math.round(e.maxHp * 0.04)` —
   düşmanın **dalga-ölçekli** max HP'siyle orantılı **aktif gelir** (geç dalgalarda
   güçlü düşman = daha çok food). Skor (`score += e.reward`) ayrı, korundu.
3. **upgradeCostFor MUL** sv4-6: `{4:4,5:6,6:8.5}` → `{4:3.5,5:4.5,6:6}` → üst seviye erişilebilir.
4. **Başlangıç food 0 → 100** (`startGame`) — açılış darboğazı giderildi.

Sonuç: oyuncu artık hilesiz **15. dalgayı belirgin şekilde aşıyor**; ekonomi
açılış (başlangıç food) + aktif gelir (öldürme + dalga bonusu) + ulaşılabilir sv6
tavanı olarak dengelendi. **Balans pass'i kapandı.**

### Faz 2B Parça 1 — Düşman-bina savaşı (koddan doğrulandı)
- **Bina yıkan düşmanlar**: ladybug, dungbeetle, enemyant (`isBuildingBreaker:true`). **spider yıkmaz**, **bird yıkmaz** (uçar).
- buildingDmg: ladybug **8**, dungbeetle **15**, enemyant **12** (saniyede hasar; `Enemy.step` içinde önündeki tile'da bina varsa durup yıkar).
- **Bina HP barı** çizilir; ~0.5sn'de bir kırmızı "-X" float + kıvılcım.
- **Düşman yıkınca İADE YOK** (`destroyBuildingByEnemy` — ceza). Oyuncunun long-press %75 iadesi bundan AYRI.

### Faz 2B Boss — TAMAMLANDI (4 parça, koddan doğrulandı)
**Boss = `Enemy` (e.isBoss). Her 5. dalgada 1 devasa zehirli akrep.**

**P1 — Boss spawn altyapısı:**
- `isBossWave(w)` = `wave % 5 === 0` → o dalga boss dalgası.
- `startWave`: boss dalgasında normal düşman sayısı **yarıya** iner (`Math.ceil(count/2)`); boss bu sayıya dahil **DEĞİL** (ayrı spawn). `bossPending = isBossWave(wave)` set edilir.
- `updateWave` WAVE bloğu: faza girince **ilk frame'de** `bossPending` ise `spawnEnemy('dungbeetle', waveHpScale, true)` → tam **1 boss**, bayrak temizlenir.
- `spawnEnemy(type, hpScale, isBoss=false)` imzası genişletildi; `e.init`'e geçer.
- `Enemy.init(type, hpScale, isBoss=false)` boss çarpanları (dungbeetle tipinden türer): **maxHp = dungbeetle.hp × hpScale × 6**, **queenDmg × 2**, **reward × 5**, **speed aynı**; `e.isBoss = true`. Normal düşman init davranışı HİÇ değişmedi (default `false`).

**P2 — `drawBoss(x, y, angle, wig, cam)` (kod-çizimli devasa OYNAK akrep, emoji DEĞİL):**
- Koyu mor/siyah zehirli tema: gövde `#3a1a4d`, kenar/segment `#1f0d2e`, vurgu `#6a3a8a`, iğne ucu parlak `#b070e0`.
- Anatomi: **3 oval gövde** (sefalotoraks+abdomen) + **2 çatallı kıskaç** (pedipalp) + **8 bacak** (yanlarda 4'er) + **5 segmentli kıvrık kuyruk** + uçta parlak **iğne** üçgeni + sırt vurgu noktaları + parlak gözler.
- Boyut: karıncanın ~**3 katı** (`BS = cam × 3`); kendi büyük gölgesi.
- Animasyon karıncayla **AYNI teknik** (tek `Math.sin(wig)`, yeni zaman kaynağı YOK): gövde sallanması + bacaklar **zıt faz** (yürüyüş) + kıskaç açılıp kapanma + kuyruk ucu salınımı. `wig` zaten `Enemy.step`'te artar.
- `Enemy.draw`: `isBoss` ise `drawBoss(...)` çağrılır; **P1'deki geçici "emoji ×2.5" kaldırıldı**. Normal düşman emoji çizimi `else` dalında aynen.
- HP barı boss'a göre büyütüldü: `bw 26→64`, `bh 4→5`, `by -16→-40`. Normal düşman barı değişmedi.

**P3 — Akrep alan saldırısı (SADECE bina) + ekran sarsıntısı:**
- `bossAreaAttack(e)`: boss yürürken `Enemy.update` içinde **~2sn'de bir** (`atkTimer >= 120` dts; `this.atkTimer` boss değilse kullanılmaz) tetiklenir. `R = 70` dünya yarıçapındaki **TÜM binalara** `AOE_DMG = 40` hasar (`worldFromGrid` mesafesiyle); yeşil `-40` floatText + burst; `hp <= 0` → `destroyBuildingByEnemy(b)` (geri-iterasyon, splice güvenli).
- Görsel/işitsel: **yeşil genişleyen halka** (`buildPulses` `c:'#3aff6a'`, `maxR 90`) + `burst ×14` + `sndBoom()` + `shakeMag = 12`.
- `sndBoom = () => { beep(90,0.18,"sawtooth",0.2); beep(60,0.3,"square",0.16); }`.
- **Ekran sarsıntısı altyapısı (yeni, minimal):** global `shakeMag/shakeX/shakeY`; `loop()` başında ilk çizimden önce dt-ölçekli sönüm (`shakeMag *= Math.pow(0.9, dts)`, eşik altında 0). `toScreen` + `screenFromWorld` çıktısına **AYNI** `+shakeX/+shakeY` offset eklenir. **Ters dönüşümlere (toGrid/dokunma girişi) ve `camera.x/y`'ye DOKUNULMADI → pan/pinch bozulmaz.** `shakeMag=0` iken offset 0 (mevcut görüntü değişmez).
- Boss'un yürüme/`queenDmg` mantığı AYRI korundu; alan saldırısı ona EK. **Normal düşmanlar bu davranıştan etkilenmez** (sadece `isBoss`).

**P4 — Boss yuvada durup kraliçeyi döver (intihar yok):**
- Boss yuvaya (`NEST_WORLD_R`) varınca **ÖLMEZ**, tek seferlik `queenDmg` de vermez; `e.atNest = true` set edilir.
- `atNest` iken hareket yok (`step` çağrılmaz), sallanma animasyonu sürer (`wig` elle artar). Mevcut `atkTimer >= 120` (~2sn) döngüsü işlemeye devam eder; her tetiklenmede `bossAreaAttack` (çevre binalara AOE aynen) + kraliçeye **`BOSS_NEST_DMG = 15`** hasar + `queenFlash` + kırmızı "-15" floatText; HP ≤ 0 → mevcut `endGame` akışı.
- Boss SADECE HP'si bitince ölür (mevcut ölüm/ödül akışı değişmedi). Normal düşmanın yuva davranışı `else` dalında aynen korundu. `atNest` `init`'te sıfırlanır (pool güvenliği).

### Faz 2B Spider — TAMAMLANDI (2 parça, koddan doğrulandı)

**P1 — Spider ölünce öldüğü yerde 2 mini spider doğar:**
- `killEnemy` içinde `e.type === 'spider' && !e.isMini` ise 2× `spawnEnemy('spider', 1)` (hpScale=1, dalga ölçeği yok; pool dolu/null ise sessizce atlanır).
- Mini statları: **hp = maxHp = 8, speed = 2.0, queenDmg = 2, reward = 2**; `isMini = true` → mini ölünce TEKRAR doğurmaz (zincir kesik); `sizeMul = 0.6` → `Enemy.draw` emoji dalında sprite + gölge bu çarpanla küçülür (mul=1'de çıktı birebir eski).
- Konum: ölüm noktası ± offset (1. mini +12/+8, 2. mini −12/−8), açı init formülüyle yuvaya doğru yeniden hesaplanır.
- **Aliasing fix (`549173a`)**: `spawnEnemy` ölen spider'ın kendi objesini geri dönüştürebilir (m===e) ve `init` wx/wy'yi kenar-spawn ile ezer → ölüm konumu döngüden ÖNCE `ox/oy`'a saklanır; offset'ler oradan okunur.
- `isMini`/`sizeMul` `init`'te sıfırlanır (pool güvenliği).

**P2 — Ağ otoyolu (kalıcı tile izi + spider hızlanması):**
- `spiderWebs` dizisi (`{gx,gy,wx,wy}` tile merkezli) + `webSet` (lookup, anahtar `gx+','+gy`) + **`WEB_MAX = 120` FIFO** (doluysa en eski shift + anahtar silinir). `addSpiderWeb(wx,wy)` ekler (tile zaten ağlıysa no-op).
- Bırakma: `Enemy.update`'te SADECE `type==='spider'`, `webTimer >= 45` dts'de bir (~0.75sn) bulunduğu tile'a iz. `webTimer` init'te sıfırlanır.
- Hızlanma: `webMulAt(wx,wy,type)` → spider değilse 1; ağlı tile'da **`WEB_SPEED_MUL = 1.4`**. `update`'teki çarpan satırı: `slowMul = moatSlowAt(...) * webMulAt(...)` — **moat ile çarpılır**, `step`'e dokunulmadı (hız zaten `speed × slowMul`). Mini'ler de spider tipi olduğundan otomatik hızlanır.
- Çizim: `drawSpiderWebs()` — loop'ta `drawPheromone` sonrası, depth-sort ÖNCESİ ("Layer 2.5": zemin üstü, bina/birim altı). Tile merkezinde yarı saydam izometrik ağ (6 ışın + 2 basık elips halka, alpha 0.35, `#e8e8f0`, R = 11×scale). 120 ağın hepsi **tek path + tek stroke**; ekran dışı atlanır.
- Reset: `startGame`'de `spiderWebs.length = 0; webSet.clear()`.

### Faz 2B Bird — TAMAMLANDI (gerçek uçuş; dalış denendi ve bilinçli kaldırıldı)

**Kalan davranış (koddan doğrulanmış):** bird artık gerçek uçan düşman.
- **Moat muafiyeti**: `Enemy.update`'te `slowMul` ataması — `type === 'bird'` ise 1 sabitlenir (moat yavaşlatmaz; ağ çarpanı spider-dışında zaten 1, davranış kaybı yok).
- **Barikat muafiyeti**: `step`'teki `isBarricadeTile` dalına `this.type !== 'bird' &&` ön koşulu — bird barikatın üstünden düz uçar, `_side` dolaşması yok. Bina hasar bloğuna girmez (`isBuildingBreaker` değil, o dal zaten kapalı).
- **Uçuş görseli**: `Enemy.draw`'da `lift = (18 + Math.sin(this.wig * 0.1) * 4) * s` (`type === 'bird'`, diğerlerinde 0 → çıktı birebir aynı). Gövde sprite + HP barı `lift` kadar yukarıda; **gölge elipsi yerde** kalır. Salınım mevcut `wig`'den, yeni zaman kaynağı yok.

**Dalış mekaniği — DENENDİ, KALDIRILDI (tasarım kararı):** P2/P3'te periyodik dalış
eklendi (süzül → ~1sn hız çarpanı + alçalma görseli + sndBird), ayar geçişi de yapıldı
(150 dts periyot, ×1.6, lerp 0.06) ama **bird'ün kısa ömrüne uymadı** — kuleler bird'ü
dalış sayısı anlam kazanamadan düşürüyor. `94f6daa` ile dalış kodu komple temizlendi
(diveTimer/diveState/liftBase, update bloğu, sndBird çağrısı); uçuş P1 haliyle korundu.
- **Kalıntı:** `sndBird` tanımı (satır ~80, iki tonlu sawtooth) **ölü kod olarak bilinçli bırakıldı** — çağıran yok, zararsız; ileride bird sesi gerekirse hazır.

### Faz 2B Ladybug — TAMAMLANDI (P1 + P2, koddan doğrulanmış)

**P1 — 2 mini refakatçi (`84de777`, test edildi):**
- `spawnEnemy` içinde `type === 'ladybug' && !isBoss` ise ana ladybug'ın iki yanına
  (±18/±12 px) 2 mini spawn olur. Mini spider deseninin spawn-zamanlı uyarlaması.
- Mini statlar ananın **~%40'ı, aşağı yuvarlı** (dalga ölçeği otomatik taşınır):
  `hp = floor(maxHp×0.4)` (min 1), reward floor(×0.4), queenDmg floor(×0.4),
  buildingDmg floor(×0.4) — mini de bina-yıkıcı kalır, orantılı zayıf. `sizeMul = 0.6`.
- **Zincir kesme**: modül seviyesi `_spawningEscort` bayrağı — refakatçi spawn'ı bloğa
  tekrar girmez (mini, mini doğurmaz). Pool dolarsa sessizce atlanır.
- `warmEmojiSprites` değişikliği GEREKMEDİ (×0.6 mini boyut tüm düşman emojileri için zaten ısıtılıyor).

**P2 — Şifa pulsu SADECE ana ladybug (`c1ed745` + `294a2f5`, doğrulandı + kapatıldı):**
- Kod: `Enemy.update`'te `type === 'ladybug' && !this.isMini` bloğu — `healTimer >= 180`
  (~3sn), R=60px içindeki **kendisi hariç**, canı eksik aktif düşmanlara `+min(6, eksik)` hp
  (maxHp clamp); iyileşen başına yeşil `+N` floatText; en az 1 iyileşme varsa pembe halka
  (`#ff6b9d`, maxR 75) + `sndHeal`.
- Doğrulama sonucu: hedef filtresinde TİP filtresi yok (`!o.active || o === this || o.hp >= o.maxHp`)
  → diğer ladybug'lar dahil menzildeki tüm düşmanlar şifa alır; bu BİLİNÇLİ bırakıldı.
  Fullken efekt/ses çıkmaz (`healed === 0` → halka/ses yok) — kodda zaten çözülüydü.
- Kerem'in "ana kendine +6 basıyor" gözleminin kaynağı minilerin pulsuydu → `isMini` guard
  (`294a2f5`) ile kapatıldı: artık SADECE ana ladybug pulse atar.
- **+6 / 180 dts balansı ONAYLI** — ek düzeltme gerekmedi.

### Faz 2B Dungbeetle Topu — TAMAMLANDI (P1+P2+P3, koddan doğrulanmış)
**Tasarım: itme DEĞİL** (grid riski nedeniyle iptal edilmişti) — büyüyen top + fırlatma.

**P1 (`847ebf1`) — büyüyen top görseli:**
- `ballSize` (0..1) / `ballGrow` (dts sayacı) alanları `Enemy.init`'te sıfırlanır (pool güvenli).
- `Enemy.update`'te `type === 'dungbeetle' && !isMini && !isBoss` — `ballGrow += dts`,
  `ballSize = min(1, ballGrow / 240)` → **~4sn'de full**. **`!isBoss` guard ŞART**: boss
  dungbeetle tipinden türer, kendi alan saldırısı var, top atmamalı.
- Çizim (`Enemy.draw`, emoji dalı içinde): gövdenin **ÖNÜNDE** (`angle` yönünde 16px,
  gerçek bok böceği gibi iterek) kahverengi daire `#6b4a2a`, yarıçap
  `(4 + 8×ballSize) × camera.scale × sizeMul`; `wig` ile dönen 2 koyu leke `#4a3018`
  (yuvarlanma hissi). `ballSize > 0.05` altında çizilmez.

**P2 (`ab90d34`) — fırlatma + `enemyShots`:**
- Top dolunca (`ballSize >= 1`) **240px** (kare mesafe) içindeki **EN YAKIN binaya** fırlatılır.
- **`enemyShots` dizisi `bullets`'tan TAMAMEN AYRI** (bullets `target`=Enemy varsayar, dokunulmadı):
  `{ wx, wy, vx, vy, dmg: 25, tgtB (Building|null), isNest, size: 12, life: 600 }`. Hız ~3.5 px/dts.
- `updateEnemyShots(dts)`: hedef bina yıkılmış/listeden çıkmışsa veya `life` (~10sn) bitmişse
  top düşer; ~14px temasta `hp -= 25` + turuncu `-25` floatText + kahverengi burst,
  `hp <= 0 → destroyBuildingByEnemy` (iade YOK). Geri-iterasyon, splice güvenli.
- `drawEnemyShots()`: kahverengi daire + konuma bağlı (`(wx+wy)*0.1`) dönen leke.
- Loop çağrıları `updateBullets`/`drawBullets`'ın hemen yanında; `startGame`'de `enemyShots.length = 0`.

**P3 — yuva fallback + üçlü kalıp:**
- Hedef seçimi: **önce bina** (240px); menzilde bina YOKSA **yuva** (`nestW`, 240px) hedeflenir
  (`tgtB: null, isNest: true`); ikisi de menzil dışıysa top **dolu bekler** (her frame yeniden dener).
- Yuva çarpmasında **ÜÇLÜ KALIP** (boss yuvada dövme deseniyle aynı): `hudQueenHP -= 25` +
  `queenFlash = 1` + kırmızı `-25` floatText + `hudQueenHP <= 0 → endGame(score, wave)`.
- Karınca hedefleme HP workstream'ine ertelenmiş durumda (değişmedi).

### Perf — Emoji sprite ön-ısıtma (spawn/zoom stutter fix)
- Teşhis: yeni emoji+fontPx kombinasyonu ilk çizimde rasterize edilir (cache miss) → spawn anında frame takılması (mini ×0.6 boyutu, yeni düşman tipinin ilk spawn'ı, zoom değişimi).
- `warmEmojiSprites()`: `Enemy.draw`'daki GERÇEK formülle (`base = max(14, round(22×camera.scale))`) tüm `CONFIG.ENEMIES` emojilerini **normal + mini (×0.6)** boyutta `getEmojiSprite`'a pişirtir. SADECE düşman emojileri (boss kod-çizimli, karınca/bina dahil değil). `getEmojiSprite`'ın içine DOKUNULMADI.
- Çağrı: (a) `startGame`'de bir kez; (b) `loop()`'ta debounce — ölçek 10 frame sabit kalınca (`scaleStableFrames === 10`) ve `lastWarmScale`'den farklıysa bir kez ısıt. Pinch SIRASINDA asla çalışmaz (ölçek değiştikçe sayaç sıfırlanır).

> **NOT — kalan mikro stutter (izlemede):** spawn anlarında gözlemlenen mikro stutter
> profiler ile incelendi; takılma anında **Main ve GPU şeritleri boş** — oyun kodu
> kaynaklı DEĞİL, sistemsel/tarayıcı katmanı. İzlemede; **mobil cihaz testi yapılacak**.

> **ÖNEMLİ NOT — karınca hp/ölüm:** Karıncalarda **hp/ölüm mekaniği YOK** (koddan doğrulandı — `Ant`'ta hp alanı yok, hiçbir yer `ant.active=false` yapmıyor). Akrep alan saldırısı **şimdilik SADECE binaya** vuruyor. Karınca HP+ölüm **bilinçli ertelendi** — ertelenen **karınca AI/çok-tile workstream'iyle BİRLİKTE** yapılacak; o zaman akrep saldırısına "karıncaya da vur" eklemek **tek satır** (aynı yarıçap döngüsüne `ants` taraması).

### Faz 4 — Ana Menü Cila P1 (TAMAMLANDI, mobil doğrulandı)
- **Skor kartı arkası**: `drawMenu`'da "🏆 En İyi Skorlar" + 3 skor satırının arkasına yarı
  saydam yuvarlak köşeli panel (`rrect`, r=12, genişlik `W*0.62`, W/2 merkezli). Dolgu
  `rgba(28,19,10,0.55)`, kenarlık `rgba(255,224,143,0.18)` lineWidth 1 (sıcak çerçeve). Panel
  ÖNCE, metin SONRA. Skor yoksa panel hiç çizilmez (boş-menü davranışı korundu).
- **Dikey denge**: skor başlığı `H*0.46 → H*0.50` (tek kaynak `scoreY`, satırlar+panel onunla
  kayar), BAŞLA butonu `by: H*0.65 → H*0.62` (P1-fix'te 0.72'den geri çekildi). Lejant `H*0.82` sabit.
- **Dekoratif karıncalar**: zaten en alt katmandaydı; `globalAlpha 0.13 → 0.08` düşürüldü
  (metin/panel okunabilirliği). `menuStartBtn` hit-test `by`'yi doğrudan kullandığı için otomatik tutarlı.

### Faz 4 — Power-up Sistemi (YENİ — tam çalışır, koddan doğrulandı)
**Tap-to-collect power-up'lar. `foods`'tan TAMAMEN AYRI paralel sistem.**
- **Dizi**: `powerups = []`, öğe `{ wx, wy, kind, age, uid }`, `kind ∈ 'speed' | 'egg'`.
- **Spawn**: `spawnPowerup()` — `spawnFood` ile aynı iç-bölge tile seçimi (kenar/nest hariç),
  %50 speed / %50 egg, **harita CAP 3**. Loop'ta accumulator deseni (`powerupTimer += dt;
  >= 12000 → spawn`), yani **~12sn periyot**; `startGame`'de `powerups.length=0; powerupTimer=0`.
  (Countdown yerine accumulator: startGame=0 + ilk spawn ~12sn + foods-paralel direktifini birlikte sağlar.)
- **Çizim**: `drawPowerup(p)` — **programatik, emoji DEĞİL**. Zemin gölgesi + bob salınımı
  (`sin(age*0.005)`) + nabızlı parlak halka. **speed** = altın disk (`#ffd24a`) + şimşek poligonu;
  **egg** = krem oval (`#f5ead0`) + kontur + parlama. **Görsel r=11**, dünya px. Depth-sort'a `t:2`
  ile entegre (dispatch ÜÇ dallı: `t===0 food / t===2 powerup / else .draw()`).
- **Tap-collect**: `onPlayingTap` EN BAŞINA guard'lı blok — `!radialMenu && !upgradeMenu &&
  !nestMenu` iken ekran→dünya çevir, **tap HIT_R=26** (görünmez geniş alan, parmak-dostu), denk
  gelen power-up'ı topla + **early return**. Bina kurma/yükseltme/yuva menü mantığı BOZULMADI.
  Pointer listener'lara DOKUNULMADI.
- **Etkiler** (`collectPowerup(p)`):
  - **⚡ speed**: `speedTimer = 6000` (ms). Çarpan zaten hazırdı (`Ant.get speed`, `SPEED_BOOST=1.7`).
    **YENİ decrement loop**: loop'ta karınca update'inden ÖNCE `if (speedTimer > 0) speedTimer -= dt;`
    → boost **~6sn sonra biter** (kalıcı değil).
  - **🥚 egg**: `for 3: if (antCount() < CONFIG.ANT.MAX) spawnAnt('worker')` — MAX 80 kontrollü,
    `spawnAnt` null dönerse sessiz. Ant AI'ya dokunmaz.
  - Geri bildirim: yerinde `buildPulses` halkası (speed altın / egg krem) + `sndPowerup(p.kind)`.
- **Ses**: `sndPowerup(kind)` — 3-notalı **yükselen arpej** (`784 → 988 → tepe`, ~65ms ardışık
  `setTimeout`). speed = square / tepe 1319 (E6, parlak); egg = triangle / tepe 1175 (D6, yumuşak).
  `beep` helper'ı üzerinden (mute/`soundOn` otomatik). Frekans boşluğu ~1150 üstü + 3 ardışık nota
  hiçbir seste yok → diğerlerinden net ayrışır.

### Faz 4 — Lejant Dürüstlük (TAMAMLANDI)
- `drawMenu` lejant satırları gerçeğe çekildi. **🛡 Kalkan ÇIKARILDI** (implement edilmedi —
  aşağıdaki nota bak). Düşman listesi `unlockWave` sırasına göre tamamlandı: **🐜 enemyant + 🦂 boss
  eklendi**. 6 düşman 390px'e tek satıra sığmadığı için tehditler **3+3 iki satıra** bölündü
  (H*0.82+20 ve +40); font/renk (12px) aynı.

> **NOT — 🛡 Kalkan bilinçle ATLANDI:** `shieldTimer` sadece görsel halka çiziyor; karıncalarda
> HP/ölüm olmadığı için koruyacak bir şey yok. **`shieldTimer` için decrement loop EKLENMEDİ**
> (sadece `speedTimer`'a eklendi). Kalkan, ertelenen **karınca HP/ölüm workstream'i** gelince
> implement edilip lejanta geri eklenecek.

### localStorage Skor Sistemi — TAMAMLANDI (kod-doğrulandı, test edildi)
localStorage skor sistemi zaten tam implement edilmiş ve test edildi (Live Server'da F5
sonrası top skor listesi kalıcı). `saveScore`/`getTopScores`/`getBest` + `drawGameover` +
`drawMenu` skor listesi çalışıyor. Anahtarlar: **`antKingdomScores`** (top-5), **`antKingdomBest`**.
**Faz 4 kapsamından düşüldü.**
- `endGame(score, wave)` → `prevBest`/`lastScore`/`lastWave` saklanır, `saveScore` çağrılır, `state='GAMEOVER'`.
- `drawGameover` overlay: "Süre Doldu!" + yeni rekorsa "YENİ REKOR!" yoksa "En İyi", Puan/Dalga, "TEKRAR OYNA" butonu.
- `drawMenu` ana menüde top-3 skor listesi gösterir.

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
7deedf1 Faz4 Menu P2c: lejant durust (kalkan cikti, enemyant+boss eklendi)
b610894 Faz4 Powerup: ayirt edici sndPowerup arpeji
8cd334a Faz4 Powerup P2b: hiz+yumurta etki, boyut rotus
a710be4 Faz4 Powerup P2a: powerups dizisi + cizim + tap-collect (etki yok)
dc0a256 Faz4 Menu P1-fix: buton yukari + dekor karinca arka katman
4593b2c Faz4 Menu P1: skor karti + dikey denge
82fed05 docs: DURUM.md localStorage skor TAMAMLANDI, Faz4 kapsam guncel
4a619b1 Faz2B Dungbeetle P3: yuva hedefi (uclu kalip) + DURUM.md guncel
```
> **Branch durumu:** Branch `claude/gifted-planck-JSihd`. Bird + Ladybug P1 dahil
> `e9c921b`'ye kadar her şey merge + push edilmiş durumda (eski "origin'den ileride"
> uyarısı eskidi, silindi). Bu oturumun commitleri (`294a2f5` ve sonrası: Ladybug P2 +
> Dungbeetle topu) branch'te — push/merge durumu için `git status -sb` ile doğrula.

---

## 4. BEKLEYEN İŞLER (kullanıcı onayladı, sırayla)

**Öncelik sırası:**

1. ~~**DENGE**~~ — **TAMAMLANDI** (balans pass'i, bkz. Bölüm 2 "Balans Pass'i"). Oyuncu artık
   hilesiz 15. dalgayı belirgin şekilde aşıyor. Ekonomi açılış + aktif gelir + ulaşılabilir
   sv6 tavanı olarak dengelendi. (İleride yeni içerik eklenince ince ayar gerekebilir.)

2. **Faz 2B** — kısmen tamamlandı:
   - ~~**Boss dalgalar**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 2B Boss — 4 parça"). Her 5. dalgada devasa akrep boss; yuvada durup kraliçeyi döver.
   - ~~**Spider yetenekleri**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 2B Spider — 2 parça"): mini spider bölünmesi + ağ otoyolu.
   - ~~**Bird**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 2B Bird"): gerçek uçan düşman —
     moat/barikat muafiyeti + havada süzülme görseli (gölge yerde). Dalış mekaniği denendi,
     bird'ün kısa ömrüne uymadığı için **bilinçli kaldırıldı** (tasarım kararı — tekrar açma,
     gerekçe Bölüm 2'de).
   - ~~**Ladybug**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 2B Ladybug"): P1 2 mini
     refakatçi + P2 şifa pulsu isMini guard'ı; +6/3sn balansı onaylı.
   - ~~**Dungbeetle topu**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 2B Dungbeetle Topu"):
     P1 büyüyen top görseli + P2 fırlatma/`enemyShots` + P3 yuva fallback (üçlü kalıp).
   - Karınca HP gerektirenler (enemyant soldier avı, akrep "karıncaya vur", dungbeetle
     topunun karınca hedeflemesi) HÂLÂ ertelenmiş — karınca hp/ölüm mekaniğiyle,
     karınca AI workstream'iyle birlikte gelecek.

3. **Faz 4** — kısmen tamamlandı:
   - ~~**Ana menü cila**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 4 — Ana Menü Cila P1": skor kartı +
     dikey denge + dekor karınca arka katman) ve lejant dürüstlük güncellemesi.
   - ~~**Power-up sistemi**~~ — **✓ TAMAMLANDI** (YENİ, bkz. Bölüm 2 "Faz 4 — Power-up Sistemi":
     ⚡ hız + 🥚 yumurta, tap-to-collect, programatik çizim, sndPowerup arpeji).
   - **localStorage skor** — DÜŞÜLDÜ (zaten tam implement + test edildi).
   - **Kalan**: (a) **HUD cila** (oyun-içi üst panel/balon/buton görsel iyileştirme),
     (b) **ses genişletme** (temel Web Audio + sndPowerup var; ek efekt/karışım/cila).

> **SIRADAKİ ÖNCELİK — Kerem karar verecek (iki aday):**
> 1. **Enemyant soldier avı** — DİKKAT: karınca hp/ölüm mekaniği gerektirir (şu an
>    karıncalarda hp YOK, bilinçli ertelenmişti). Bu seçilirse önce karınca HP temeli
>    kurulmalı; akrep "karıncaya vur" ve dungbeetle topunun karınca hedeflemesi de
>    aynı temelin üstüne tek satırlık eklemeler olur.
> 2. **Faz 4** — ana menü cila + power-up sistemi + lejant **✓ TAMAMLANDI**. Kalan: (a) **HUD cila**,
>    (b) **ses genişletme** (temel Web Audio + sndPowerup var). localStorage skor düşüldü.

### Ertelenen / Notlar
- **Çok-tile bina ayak izi (footprint) — DÜŞÜK ÖNCELİK / YÜKSEK RİSK.** Binalar şu an
  seviyeden bağımsız HEP 1 tile kaplıyor; bu yüzden büyüyen sv4-6 sprite'ı komşu tile'lara
  taşıyor ve oyuncu bitişiğe hâlâ bina kurabiliyor. İstenen: ayak izi seviyeyle büyüsün,
  bitişik bina kurmayı engellesin, etki/AOE yarıçapı footprint'le ölçeklensin, düşmanlar
  kaplanan tile'ları **dolu/katı** algılasın. RİSKLİ — grid/yerleştirme, pathfinding, düşman
  çarpışması ve depth-sort'a (DOKUNMA bölgesi) dokunur. GEÇ ve **ertelenen karınca-hareket AI
  işiyle BİRLİKTE** yapılacak (ikisi de düşman/karınca pathfinding'i değiştirir).
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
