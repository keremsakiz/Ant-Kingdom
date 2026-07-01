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
- Branch: çalışma `claude/gifted-planck-JSihd` üzerinde; `main` ile senkron, faz tamamlanınca merge ediliyor. · Repo: `C:\Users\Onur\Ant-Kingdom`
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

### Faz 4 — Boss Reward + Şifa Alan Etkisi (TAMAMLANDI, bu oturum, test edildi)

**A) Boss reward — dalga-ölçekli yem + yüzen yazı (`763ad4e`):**
- `killEnemy` içine `if (e.isBoss)` dalı eklendi. Boss food ödülü artık
  **`bossReward = 800 + Math.floor(wave/5)*400`** (dalga5=1200, 10=1600, 15=2000).
- Boss ölünce konumunda **"+N 🍖"** altın (`#ffd24a`) `floatTexts` yazısı.
- Tiered ödül eski **`maxHp*0.04`'ün YERİNE** geçer (üstüne değil). **Normal düşman ödülü
  HİÇ değişmedi** (`else` dalında `maxHp*0.04` aynen). `floatTexts`/`wave` zaten scope'ta.

**B) Şifa Taşı (alarm) alan etkisi — çevredeki binaları onarır (`37096fc` + `71579ee`):**
- Kraliçe şifa mantığına DOKUNULMADI; aynı `healTimer` tikinde (15sn) **AYRI bina-onarım
  döngüsü** eklendi. Kraliçe if bloğu aynen korundu, altına yeni blok.
- **`repairRange: 120`** yeni config alanı (`CONFIG.BUILDINGS.alarm`, `healRange` 130 değişmedi).
- Healer çevresinde `repairRange` kare-mesafe içindeki, canı eksik (`o.hp < o.maxHP`) binalara
  onarım: **`rep = round(o.maxHP * 0.15 * eff)`** → **healer seviyesine bağlı** (eff = LVL_MUL,
  sv1→sv6 artar). `o.hp = min(o.maxHP, o.hp + rep)`.
- **Yuva kısıtından bağımsız** (kraliçe şifasındaki `inRange` yuva-yakınlığı buna uygulanmaz),
  ama **3-healer cap'i paylaşır** (`alarmCanHeal(this)` kapısı). Her onarımda yeşil `burst` +
  **"+N 🔧"** yeşil (`#5aff8a`) `floatTexts`.
- **KRİTİK:** binalarda alan adı **`maxHP` (büyük H)** — düşmanlardaki `maxHp` DEĞİL.
- Kerem'in Bölüm 4'teki A/B karar soruları bu implementasyonla kapandı (A: yerine; B: yeni
  repairRange + yuvadan bağımsız + 3-cap paylaşımı + eff-ölçekli).

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

### Faz 4 — Mute + Pause/Resume + drawScene refactor (TAMAMLANDI, bu oturum, test edildi)

**Mute tuşu (`86a02d1`):**
- **Menü + oyun-içi HUD** sağ üst köşede, **ikon-only toggle** (🔊/🔇). 44px, 16px margin.
- `toggleSound()`: `soundOn`'u çevirir + `soundOn ? startMusic() : stopMusic()`. Altyapı zaten
  hazırdı (`soundOn` flag [index.html:65], `beep` flag'e saygılı). `drawMuteButton()` ortak helper,
  `drawMenu` ve `drawHUD` sonunda en son çizilir (üstte). `muteBtn` global hit-rect; `onMenuTap` +
  `onPlayingTap` en başında AABB testi. **localStorage YOK** — her açılışta ses açık.

**Pause/Resume (`004d075`):**
- **Yeni `PAUSED` state.** Tuş mute'un solunda (8px boşluk), **SADECE oyun içi** (⏸/▶).
- `togglePause()` PLAYING↔PAUSED çevirir. PAUSED dalında **dünya tamamen donar** — hiçbir
  update/timer ilerlemez (`drawScene(0, 0)` ile son kare yeniden çizilir) + `rgba(0,0,0,0.5)`
  karartma + "DURAKLADI" yazısı, butonlar karartmanın üstünde parlak kalır.
- `onTap` dağıtıcısı **PAUSED'da da `onPlayingTap`'e** yönlendirir (▶ ile resume + mute çalışsın);
  `onPlayingTap`'te mute→pause hit-test'inden sonra **`if (state==='PAUSED') return;`** guard'ı
  → pause ekranında buton dışı tap bina kurma/power-up toplamayı tetiklemez.

**REFACTOR — `drawScene(dts, dt)` (ÖNEMLİ):**
- PLAYING'in inline çizim bloğu **`drawScene()` fonksiyonuna çıkarıldı**; hem PLAYING (gerçek
  dts/dt) hem PAUSED (`0, 0` → donmuş) çağırır. Entity update'leri PLAYING'de update bölümüne
  alındı; zemin katmanları entity pozisyonundan bağımsız + entity'ler post-update `drawables`'tan
  çizildiği için **davranış/görsel BİREBİR AYNI**. Render optimizasyonları (sprite cache,
  depth-sort `wy+uid` flicker fix, baked gölge) korundu. **Tek doğruluk kaynağı** — ileride render
  değişikliği tek yerde. Doğrulama: braces 579/579 dengeli (node yok, tarayıcı görsel testi önerilir).

### Faz 4 — Ses Genişletme + Game Over Polish (TAMAMLANDI, main'e merge edildi)

> **Not:** Bu commitler main'e çoktan merge edildi (eski "merge bekliyor" durumu geçti).

**Ses P1 — yeni SFX (`1fc0471`):** mevcut `beep()` + `snd*` kalıbına 3 helper (master bus YOK,
`actx.destination` kalır):
- `sndShoot()` — kule ateşi (ranger + catapult `bullets.push` ardına). **880Hz triangle**, 0.04s,
  vol 0.09 (ayar sonrası; ilk değer 440Hz square 0.05'ti). Kısa kuru orta-tiz tık.
- `sndWave()` — `startWave` sonunda. 2-nota alçaktan yükselen uyarı (330→440 sawtooth, 120ms ofset).
- `sndBossSpawn()` — boss spawn (`updateWave`, `spawnEnemy('dungbeetle',...,true)` yanı). Derin
  tehditkâr 2-ton (160 sawtooth → 120 square, 200ms ofset).

**Ses ayarları — yem gürültüsü çözüldü (`61568e2` → `cc31760`):**
- `sndShoot` duyulurluk: 440→**880Hz**, square→**triangle**, vol 0.05→**0.09** (vuruş/müzikten ayrışır).
- `sndFood` + `sndDeposit` ok-fonksiyondan normal `function`'a çevrildi; **vol %60'a indirildi**
  (sndFood 0.06→0.036; sndDeposit 0.1→0.06 ve 0.08→0.048), freq/dur/type AYNI (karakter korundu).
- **Throttle pattern**: her birine ayrı guard (`_lastFood` / `_lastDeposit`) + `performance.now()`;
  pencere kademeli açıldı 80→400→**600ms**. Toplu yem toplamada/bırakmada artık tek "ara sıra tık".

**Ses P2 — yuva hasar + game over (`2d30d08`):** 2 helper daha:
- `sndQueenHurt()` — `_lastHurt` guard + **500ms throttle** (akında tek alarm), alçalan 2-nota
  tehlike (220→165 square, 90ms ofset). Üç queen-hasar noktasının HER BİRİNE `hudQueenHP -=`
  ardına eklendi (dungbeetle topu / boss yuvada / normal düşman) — üçü aynı `_lastHurt`'ü paylaşır.
- `sndGameOver()` — düşen 3-nota kapanış (330→247→165 sawtooth, 200/420ms ofset). `endGame`'de
  `state='GAMEOVER'`'dan ÖNCE. ESC test-cheat'i de `endGame` üzerinden bunu çalar.

**Game over fix (`c6a6a6f`):** başlık **"⏱ Süre Doldu!" → "🐜 Yuva Düştü!"**. Doğrulandı: oyunun
**süre/zaman limiti diye bir bitişi YOK** — sadece kraliçe HP ≤ 0 ile biter (3 hasar noktası →
`endGame`). "Süre Doldu" tamamen yanlıştı.

**Game over polish (`df02f88`):** `drawGameover`'da Dalga satırı altına (H*0.63, 15px, soluk
#b89a7a) `lastWave`'e göre **dinamik teşvik mesajı**: ≤2 "Daha yeni başladın, pes etme!", ≤5 "İyi
dayandın...", ≤9 "Güçlü bir savunmaydı!", 10+ "Efsanevi bir koloni! Rekoru zorla!".
- **Doğrulandı (keşif):** `lastWave` ← `endGame`'in `wave` param'ı ← çağrı anındaki global `wave`
  (= `hudWave` = HUD'da gösterilen). "Dalga:" satırı ve mesaj **AYNI `lastWave`'i** okur → çelişemezler.
  Veri akışı doğru; "wave 8'de wave-1 mesajı" gözlemi koddan üretilemiyor → muhtemelen tarayıcı cache
  (sert yenileme önerildi).

### Faz 4 — HUD Cila (TAMAMLANDI, main'e merge edildi) ✅ FAZ 4'ÜN SON PARÇASI

> Faz 4'ün tek kalan parçasıydı; bittiğinde **Faz 4 TAM KAPANDI**. Beş alt parça, hepsi koddan
> doğrulandı. Ortak tasarım dili: afford/non-afford netliği + drop shadow derinliği + ince altın
> iç çizgi + (gerekli yerde) tam opak kırmızı uyarı.

**P1 — Merkezi HUD tema sabiti (`780b4bf`, saf refactor — görsel AYNI):**
- Dağınık inline renk/font/opaklık/radius string'leri tek `const HUD` objesine alındı
  (`drawHUD` öncesi tanımlı). Alanlar: `font`, `panelFill`, `pillFill`, `barFill`, `btnFill`,
  `gold`, `danger`, `affordText`, `affordText2`, `cantAfford`, `effectGreen`, `r{panel,binaBtn,menuBtn}`.
- Değerler koddaki gerçek string'lerle birebir; `drawHUD`/`drawUpgradeMenu`/`drawNestMenu`/
  `drawRadialMenu`/`drawMuteButton`/`drawPauseButton` bu sabitten okur. Ternary'lerin disabled
  dalları + emoji fontları (`NNpx sans-serif`) bilinçle inline bırakıldı.

**P2a — Pill görsel yenileme + r.* isim düzeltme (`1daa5ee` + `7660fcc`):**
- Üst pill'ler: ikon (15px) solda + değer (bold 13px) sağda, ikisi de dikey ortalı. Arka dolguya
  **SADECE** drop shadow (`save→shadow→fill→restore`, ikon/metne sızmaz), üst kenar highlight
  (`rgba(255,255,255,0.12)`), ince iç altın kenarlık (lineWidth 1.5, gold @ alpha 0.7).
- **Queen HP uyarısı:** ❤️ pill'inde `hudQueenHP/MAX < 0.30` ise iç çizgi altın yerine
  **`HUD.danger` (#ff6b6b)** @ alpha 0.9. Diğer pill'ler hep altın.
- `r.*` isimleri kullanımla örtüşecek şekilde düzeltildi: `{ panel:10, binaBtn:8, menuBtn:7 }`
  (eski gevşek `pill/btn` adları kaldırıldı; tüm referanslar güncellendi).

**P2b — Alt bina paneli afford netliği + derinlik (`2c102ec` + fix `0580929`):**
- Panel üst kenarına altın ayraç çizgisi (alpha 0.5). Her bina butonunda: dolguya drop shadow
  (afford belirgin / soluk hafif), afford kenarlık `HUD.gold` lineWidth 1.5, yetmeyen buton
  `globalAlpha 0.45` soluk + kenarlık `#6a4520`.
- **Fix:** yetmeyen fiyat metni `globalAlpha=1`'e çekilip **tam opak `HUD.cantAfford` (#ff8888)**
  çizilir — gövde soluk kalsa da fiyat uyarısı net kırmızı parlar (emoji/isim soluk).

**P3a — Radyal menü fiyat + ikon derinliği (`4a89145`):**
- İkon dairesi dolgusuna drop shadow (enabled belirgin / disabled hafif), kenarlık+emoji+etiket
  ayrı gölgesiz blokta. Disabled etiket (`DOLU`/`YUVA`/fiyat) **tam opak `HUD.cantAfford`** —
  daire soluk kalır ama uyarı net. `RADIAL_R`/`RADIAL_IR`/`radialIconPos` geometrisine dokunulmadı.
- **Not (açık konu):** radyale özgü 3 inline renk (`rgba(30,22,10,0.93)` enabled dolgu /
  `rgba(50,15,15,0.88)` disabled dolgu / `#7a3030` disabled kenarlık) HUD'a alınmadı (sadece radyalde).

**P3b — Upgrade + yuva menü cila (`eeaad44`):**
- İki menü artık **gerçekten ortaklaştı**: yeni `drawMenuPanel(L)` + `drawMenuButton(L,afford,label,cx)`
  helper'larını HEM `drawUpgradeMenu` HEM `drawNestMenu` çağırır (kasıtlı birebir eşleşme tek kaynaktan).
- Panel: alt drop shadow (`rgba(0,0,0,0.45)`, blur 6, offsetY 2) + altın kenarlık + üst highlight
  (`rgba(255,255,255,0.10)`). Buton: afford'da gölge+üst highlight (basılabilir his); yetmeyende
  gövde soluk ama metin **tam opak `HUD.cantAfford`** (eski disabled metin `#ffb0a0` değişti).
- **MAX SEVİYE** (sadece upgrade): `★ MAX SEVİYE ★` koyu gölgeyle (`rgba(0,0,0,0.5)`, blur 4)
  pop eder; renk `#7fff8a` aynen korundu.
- **Disabled renk raporu:** menü butonu disabled tonları (`rgba(70,40,20,0.9)` / `#7a5530`)
  radyalinkinden **FARKLI** → ortak `disabledFill`/`disabledBorder` sabitine alınmadı (birebir eşleşmiyor).

> **HUD cila tasarım disiplini (referans):** drop shadow HER ZAMAN `save→shadow→fill→restore`
> ile **sadece arka dolguya** uygulanır; ikon/metin gölgesiz ayrı blokta. `globalAlpha` daima
> save/restore içinde. Yetmeyen/uyarı metni `globalAlpha=1`'e çekilip tam opak kırmızı çizilir.
> Yeni sık-çalan görsel eklerken bu deseni taklit et.

### iOS/Safari Emoji Dikey Hizalama Bug'ı (✓ TAMAMLANDI — iPad Safari'de doğrulandı)

**Bug:** `ctx.textBaseline = 'middle'` WebKit'te (Safari) Chrome'dan **farklı oturuyor** —
emojiler dikey olarak kayık çiziliyor. **Chrome'da sorun YOK.** Capacitor iOS build'inde de
görünecek (WKWebView = Safari motoru), yani App Store hedefi için kritik.

**Çözüm deseni:** `textBaseline = 'middle'` yerine **`'alphabetic'` + elle dikey offset** — iki
tarayıcıda da tutarlı oturur.

**✓ DÜZELTİLDİ (koda yazıldı, merge edildi, GitHub Pages'te canlı, iPad Safari'de doğrulandı):**
- **Radyal menü emoji** (`drawRadialMenu`): `'middle'` → `'alphabetic'`, offset `iy + 2`. Alttaki
  fiyat/etiket satırı zaten `alphabetic`'ti, dokunulmadı.
- **Alt bina paneli emoji** (`drawHUD`): `'middle'` → `'alphabetic'`, `by + 15` → `by + 22`. Emoji
  artık `middle` bırakmadığı için, hemen sonraki **isim satırına `textBaseline = 'middle'` koruması
  eklendi** (isim baseline'ı emoji'den miras alıyordu, kaymasın diye). Fiyat satırı (alphabetic) aynen.
- **Yükselt paneli** (`drawUpgradeMenu`): emoji/metin hizalaması da `alphabetic` deseniyle düzeltildi.

**Test altyapısı (yeni — arkadaşa bağımlılık kalktı):** GitHub Pages zaten aktif:
**https://keremsakiz.github.io/Ant-Kingdom/** — Kerem artık **iPad Safari'den** doğrudan buradan
kendi testini yapıyor (uzaktan arkadaş testine bağımlı değil). Emoji düzeltmeleri burada doğrulandı.

> **Not — 🗡️ Kışla emojisi:** hafif **yatay (sağa) kayması** var ama bu **sprite geçişinde
> çözülecek**; şimdilik offset avı yapılmadı (yatay hizalama ayrı konu, dikey bug'dan bağımsız).

### 3 Safari Render Bug'ı (✓ TAMAMLANDI — main'e merge edildi)

iPad Safari testinde (GitHub Pages) ortaya çıkan 3 bug çözüldü; tek commit'le (`Fix: 3 Safari
render bug — menu emoji + title glow + HUD pill jitter`) düzeltildi ve **main'e merge edildi**:

1. **Ana menü 🐜 emojisi görünmüyordu (Safari):** menü tepesindeki büyük 🐜 (`drawMenu`) Chrome'da
   vardı, Safari'de boştu. **Kök:** font `'52px sans-serif'` + explicit baseline yok → miras kalan
   canvas state (`fillStyle`/`globalAlpha`/`shadow`) emojiyi görünmez bırakıyordu. **Çözüm:** font
   `'-apple-system, sans-serif'`'e çevrildi + çizimden önce **explicit state reset** (`globalAlpha=1`,
   `fillStyle='#000'`, `shadowBlur=0`, `shadowColor='transparent'`) + `textBaseline='alphabetic'` +
   dikey offset (`H*0.22 + 18`). Yerel IP'de doğrulandı.
2. **Ana menü başlık glow'u Safari'de düz görünüyordu:** "ANT KINGDOM" başlığı Chrome'da parıltılı,
   Safari'de düz. **Kök:** `ctx.shadowBlur` + `fillText` WebKit'te zayıf/kayıp. **Çözüm:** shadowBlur
   satırı KORUNDU (Chrome'da çalışıyor) + **manuel glow katmanı fallback** — her harf, asıl harften
   önce `#c8840a` altınla `alpha = c.alpha * 0.22` çarpanıyla 5 offset'te çizilir (poor-man's glow).
   `c.alpha` fade-in çarpanı manuel katmanlara da uygulandı → harf-harf fade-in mantığı/`measureText`
   ox hesabı BOZULMADI.
3. **HUD pill "twitch" (layout titremesi):** sol üst pill'ler (yem/karınca/dalga/timer/yuva canı)
   sayı basamağı değişince yana esniyordu. **Kök:** `pw` her frame `measureText(p.val)`'e göre
   değişiyordu → komşu pill'ler kayıyordu. **Çözüm:** her pill'e `minSample` (food/HP `'9999'`,
   diğerleri `'88'`); modül seviyesi `hudPillMinW` cache'iyle min-genişlik **bir kez** hesaplanır
   (her frame değil); loop'ta tek değişiklik `pw = Math.max(20+iw+6+vw, hudPillMinW[p.icon])`. Akış
   `px += pw + gap` / `rrect` / `HUD.*` sabitlerine dokunulmadı.

> **Not — 🗡️ Kışla yatay kayması ve genel emoji dikey hizası** önceki bölümde (iOS/Safari Emoji
> Dikey Hizalama) çözülmüştü; bu 3 bug ondan AYRI (görünürlük + glow + layout).

### Faz 4+ — PAUSED ekranı "Ana Menüye Dön" (✓ TAMAMLANDI, iki aşamalı onay)

Mobilde oyundan menüye dönüş yolu yoktu (Escape sadece masaüstü + test-only). PAUSED ekranına
**iki aşamalı onaylı "Ana Menüye Dön"** eklendi:

- **Aşama 1:** "DURAKLADI" altında **▶ Devam Et** (resume) + **🏠 Ana Menü** butonları.
- **Aşama 2** (Ana Menü'ye basınca): **"Çıkarsan skorun kaydedilmez!"** uyarısı + **Evet, Çık** /
  **Vazgeç**. Vazgeç → aşama 1'e döner.
- **KRİTİK — skor kaydı:** "Evet, Çık" `endGame()` ÇAĞIRMAZ; doğrudan `initMenu()` (state='MENU').
  Skor kaydı SADECE `endGame`'de (`saveScore`) olduğu için **çıkışta skor kaydedilmez** (bilinçli).
- **İzole eklendi:** mevcut `drawScene`/overlay/"DURAKLADI" çizimine + sağ üst mute/pause(▶) butonlarına
  DOKUNULMADI. Yeni `pauseConfirmExit` flag'i + 4 buton rect'i (`pauseResumeBtn`/`pauseExitBtn`/
  `pauseYesBtn`/`pauseNoBtn`); `togglePause` PLAYING'e dönerken flag'i sıfırlar; hit-test
  `onPlayingTap`'te pause butonundan SONRA, `if(state==='PAUSED')return;`'den ÖNCE (aynı AABB deseni).
  PAUSED ekranı dikey ortalandı (DURAKLADI `H/2-70`, butonlar merkez civarı).

> **Not — Escape (keydown):** hâlâ **test-only `endGame()`** çağırıyor (`Escape && state==='PLAYING'`),
> gerçek menü dönüşü DEĞİL — dokunulmadı. Gerçek menü dönüşü artık PAUSED ekranı butonlarıyla.
>
> **Not — localStorage origin'e özel:** skorlar origin başına ayrı saklanır (`antKingdomScores`/
> `antKingdomBest`). GitHub Pages, `127.0.0.1` ve yerel IP (telefon testi) **ayrı skor kutuları**
> tutar — aynı skor listesi beklenmemeli.

### Faz 5 ÖNCESİ — Küçük İzole Ön İşler (✓ TAMAMLANDI, bu oturum)

Faz 5 (karınca HP) büyük workstream'ine başlamadan önce yapılan üç bağımsız, düşük-riskli
denge/görsel iyileştirme. Hepsi koddan doğrulandı:

1. **Feromon izi görsel inceltme (`4d86ec7`):** `drawPheromone` içinde tek satır —
   `ctx.fillStyle = 'rgba(120,200,110,' + Math.min(v/60, 0.35) * 0.25 + ')'` (max alpha
   0.35 → 0.0875, çok daha soluk). **Mekanik AYNEN korundu:** `addP` (deposit), `senseP`
   (yön bulma), `fillRect` ve **decay satırları** (`PH.grid[i] *= DECAY`) hiç değişmedi —
   sadece çizim alpha çarpanı. Pathfinding bozulmadı.

2. **Düşman hızı erken dalgalarda yavaş (`7906c02`):** `Enemy.init`'teki İKİ `this.speed`
   ataması (boss + normal dal) → `this.speed = cfg.speed * Math.min(0.5 + (wave-1)*0.125, 1.0)`.
   Dalga 1 ×0.5 → her dalga +0.125 → **dalga 5'te ×1.0** (tavan, orada kalır). İlk dalgalarda
   oyuncuya kurulum zamanı tanır. `step()` hareket satırları / `slowMul` / `dts` / mini spider
   override (`m.speed = 2.0`) dokunulmadı.

3. **Zehir kulesi AOE halkası belirginleştirildi (`bc2289c`):** `drawBuildPulses`'a **geriye
   uyumlu opsiyonel** `p.a` (alpha) + `p.w` (lineWidth) alanları eklendi (`p.a || 0.75`,
   `p.w || 2.5`). Tower ateş push'u tek silik halka yerine **çift dalga** (life 1.0 + 0.7,
   mor `#b060ff`/`#d090ff`, w 5/3, a 1.0/0.9). **Diğer hiçbir pulse ETKİLENMEDİ** — yeşil
   yerleştirme / kırmızı yıkım / şifa / boss AOE / ladybug heal push'ları `a`/`w` taşımadığı
   için birebir eski davranış.

### Ranger (Atış Kulesi) sprite pilotu — TAMAMLANDI (ilk PNG sprite sistemi)
- **`assets/` klasörü + `loadSprite(path)` image loader** (`_imgCache`). İlk harici asset:
  `assets/ranger.png` (karıncasız boş taş kule, Google AI Studio üretimi, şeffaf PNG, ~673KB).
- **`Building.draw()` içinde `this.type==='ranger'`** için emoji yerine PNG çizimi + güvenli
  fallback. `imageSmoothingQuality='high'` (save/restore ile izole, kenar tırtıklılığı giderildi).
- **Kule SABİT** (nefes yok), atış geri tepme (`recoilT`) + atış parlaması korundu.
- **`drawTowerArcher()`**: oyunun karınca stilinde (`drawAntFull` dili) DİK/kompakt okçu karınca.
  Gövde sabit, yay/kollar `aimAngle` ile en yakın düşmana döner; karınca nefes (`breath`) alır.
  Asker rengi (`#8b1a1a`/`#5e0f0f`), okçu kaskı (anten yerine metalik başlık), ateşte yay gerilir.
- **Ölçek tavanı**: PNG kule + karınca `towerScale = Math.min(tier.scale, 1.24)` → sv4+ tile
  taşması önlendi. Halka/glow `tier.scale` ile büyümeye devam (seviye hissi korundu).
- **Konum/boyut**: `archSc` 0.85, dikey konum `drawH*0.35`.
- Seviye halkası/glow ve HP barı korundu. Render optimizasyonu (sprite cache, depth-sort,
  dispatch) değişmedi.

### Bina PNG Sprite Geçişi — DURUM (bu oturum, hepsi main'de)

Ranger pilotu kalıbı (`loadSprite` + `SPR.ready` PNG / `else` güvenli emoji fallback + ölçek
tavanı + `imageSmoothingQuality='high'` save/restore izole) **5 binaya** uygulandı. Draw
dispatch else-if zinciri: `ranger → moat → barricade → alarm → catapult → else{generic emoji}`
(çift çizim imkânsız). Loader'lar modül seviyesinde (`RANGER/MOAT/BARRICADE/ALARM/CATAPULT_SPR`).

- ✅ **ranger** (🏹 Atış Kulesi) — PNG + `drawTowerArcher` okçu karınca (pilot).
- ✅ **moat** (🌊 Hendek) — PNG, yassı; `drawW=46*s`, dikey offset `0.47`.
- ✅ **barricade** (🪨 Barikat) — PNG; `drawW=50*s`, offset `0.55`, yatay `+3*s`.
- ✅ **alarm** (💚 Şifa Taşı) — PNG (dikey kaya + jade kristal); `drawW=42*s`, offset `0.62`.
  **+ sürekli shimmering** (aşağıda).
- ✅ **catapult** (🏰 Mancınık) — PNG + `drawCatapultCommander` komutan karınca + ateş jesti
  (P1/P2/P3, aşağıda); `drawW=54*s`, offset `0.55`.
- ⬜ **barracks** (🗡️ Kışla) — HÂLÂ EMOJI. Hedef: **ÇADIR** (büyükçe, başlangıç sade).
- ⬜ **tower** (🍄 Zehir Kulesi) — HÂLÂ EMOJI. Hedef: PNG + canlı çizim + **DUMAN SALINIMI**
  efekti (pulse değil, yükselen duman hayali).

> Seviye halkası/glow + HP barı **generic ortak kod** — her PNG dalı ondan faydalanmaya devam
> ediyor (draw dallarında sadece PNG + kendi özel çizimi var). Emoji sprite cache/depth-sort/
> render opt değişmedi. Assetler git'te takipli (`git ls-files assets/`): ranger, moat, barricade,
> alarm, catapult (.png).

### alarm (Şifa Taşı) Shimmering — TAMAMLANDI (Yol B: deterministik, havuz kullanmaz)
- Alarm draw dalında, PNG çiziminden SONRA (save/restore İÇİNDE): `ctx.filter='none'` +
  `performance.now()/1000` + grid-faz (`this.gridX+this.gridY` → her taş farklı faz).
- **1) Nabız glow**: `createRadialGradient` yeşil (`rgba(90,255,138,...)`) hale, alpha+yarıçap
  sinüsle salınır (yoğunluk `glowPulse*0.75`). **2) Kıvılcımlar**: 3 açık-yeşil (`#aaffcc`) nokta,
  faz kaydırmalı yukarı süzülüp yanıp söner (`sparkAlpha=sin(rise*π)`).
- **Parçacık havuzu/`buildPulses`/`burst` KULLANMAZ** — salt yerel deterministik çizim, her frame
  ucuz. `ctx.globalAlpha=1` ile biter (leak yok). Commit `1c1c94c`.

### catapult (Mancınık) — TAMAMLANDI (P1+P2+P3 + jest)
**P1 — PNG sprite (`d79497a`):** moat kalıbı, salt draw. Ateş/nişan/recoil'e dokunulmadı.

**P2 — aimAngle nişan + recoilT geri-tepme (`e0c3a85`, davranışsal):**
- Catapult update'ine, `shootTimer += dt`'den ÖNCE **her-frame en yakın düşman taraması** (ranger
  deseni, kendi `{ }` blok-scope'u `wA/aimBest` → alttaki ateş `w/best` ile çakışmaz) → `this.aimAngle`.
- Ateşte (`bullets.push` + `sndShoot` sonrası) `this.recoilT = 0.28` (ranger 0.18'den ağır).
- **recoilT decay RANGER İLE ORTAK**: `Building.update` başında, tip-dallarından ÖNCE, satır ~1714
  `if (this.recoilT>0) this.recoilT = max(0, recoilT - dt/1000)` — tüm binalar paylaşır, ekstra kod yok.
- Mevcut cooldown/hedefleme/gülle/AOE mantığına dokunulmadı.

**P3 — `drawCatapultCommander(cx,cy,ang,sc,recoil,breathe)` komutan karınca (`5598674` + zırh `607588a`):**
- `drawTowerArcher`'dan AYRI yeni fonksiyon (ranger'a dokunulmadı). Aynı imza/desen (gövde+kafa SABIT,
  sadece kol grubu `ang` ile döner; `recoil` ile kollar geriye/yukarı savrulur).
- **Mor/mavi gövde** (`#5a3a9a`/`#3a2470`) — glow/seviye sistemine uygun **sv1 sade tasarım**.
- **Katmanlı altın zırh** (elit komutan): lorica lamel karın bandı (3 yatay şerit) + göğüs plakası +
  **komutan madalyonu** + hacimli omuzluk (pauldron, 2+ katman + beyaz ışık). Katmanlar:
  highlight `#f5d97a` → altın `#e8c14a` → koyu altın `#8a6a15`/`#b8901e` → koyu metal `#6a5518`.
- **Tek uzun kırmızı Romalı tüy** (crest, `#d43a3a`) — kask tepesinden yukarı-arkaya kavisli, iki
  kenar bezier + koyu orta çizgi + highlight. (4 dik tüy denemesi bununla değişti.)
- Çağrı catapult draw dalında, PNG'den sonra: `cmdSc=1.05*s*min(tier.scale,1.24)`, konum
  `x+drawW*0.16, cy+drawH*0.10` (sağ çıkrık/tekerlek yanı), `recoilN=recoilT/0.28`. `breath` lokal
  hesaplandı (catapult dalında ranger'daki gibi hazır değildi).

**Ateş anı jesti — kova parlama + toz (`b4bafa0`):** komutan çağrısından sonra, `recoilN>0.05` iken:
kısa sarı-beyaz radial flash + 4 gri (`#b0a89a`) toz bulutu (recoil sönerken dışa yayılıp solar).
İlk **"fırlayan gülle"** denemesi (kovadan çıkan gri top) **kaldırıldı** — yerine bu jest. Gerçek
`bullets` güllesi + AOE patlama AYRI, dokunulmadı (bu salt binanın üstünde görsel jest).

---

## 3. SON COMMIT'LER

```
(git log --oneline ile en günceli doğrula — bu docs commit'i HEAD olur)
HEAD = b4bafa0  (main + claude/gifted-planck-JSihd SENKRON, ikisi de origin'e push'lu)

--- Bu oturum: bina PNG sprite geçişi (moat/barricade/alarm/catapult) ---
b4bafa0 Visual: catapult ates ani kova parlama + toz jesti (dusen gulle kaldirildi)
607588a Visual: catapult komutan zirh detay - katmanli altin, omuzluk, madalyon, lamel
5598674 Feat: catapult komutan karinca - mor govde + altin zirh + tuy (P3)
e0c3a85 Feat: catapult nisan acisi (aimAngle) + geri-tepme (recoilT) - P2
d79497a Visual: catapult (Mancinik) PNG sprite - moat kalibi (P1 draw)
1c1c94c Visual: alarm (Sifa Tasi) shimmering efekti - nabiz glow + kivilcim
1cdda65 Visual: alarm (Sifa Tasi) PNG sprite - dikey kaya + jade kristal
b8edf30 Visual: barricade (Barikat) PNG sprite - moat kalibi, spike'li, konum ayarli
0ce8994 Visual: moat (Hendek) PNG sprite - ranger kalibi, olcek+oturma ayarli
5d6b0b9 Visual: moat (Hendek) PNG sprite - ranger kalibi + guvenli fallback
```
> **Branch durumu:** `main` ve `claude/gifted-planck-JSihd` **tamamen senkron** (ikisi de
> `b4bafa0`, origin'e push'lu). Bina PNG sprite işleri (moat→catapult, `5d6b0b9` → `b4bafa0`)
> main'e fast-forward merge edildi. Önceki tüm işler (Faz 4, Faz 5-öncesi ön işler) da main'de.
> En güncel durum için `git log --oneline -10` + `git status -sb`.

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

3. ~~**Faz 4**~~ — **✓ TAM KAPANDI** — tüm parçalar bitti, **main'e merge edildi**:
   - ~~**Ana menü cila**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 4 — Ana Menü Cila P1": skor kartı +
     dikey denge + dekor karınca arka katman) ve lejant dürüstlük güncellemesi.
   - ~~**Power-up sistemi**~~ — **✓ TAMAMLANDI** (YENİ, bkz. Bölüm 2 "Faz 4 — Power-up Sistemi":
     ⚡ hız + 🥚 yumurta, tap-to-collect, programatik çizim, sndPowerup arpeji).
   - ~~**Ses genişletme**~~ — **✓ TAMAMLANDI** (bkz. Bölüm 2 "Faz 4 —
     Ses Genişletme + Game Over Polish": ses P1 sndShoot/sndWave/sndBossSpawn + yem sesi kıs/throttle
     + ses P2 sndQueenHurt/sndGameOver + game over başlık fix + dinamik teşvik mesajı).
   - ~~**HUD cila**~~ — **✓ TAMAMLANDI (SON PARÇA)** (bkz. Bölüm 2 "Faz 4 — HUD Cila": P1 merkezi
     tema sabiti + P2a pill yenileme/r.* isim + P2b alt panel afford + P3a radyal + P3b upgrade/yuva
     menü). Afford netliği (yetmeyen fiyat tam opak kırmızı) + drop shadow derinliği + ince altın
     çizgi + queen HP < %30 kırmızı uyarı.
   - **localStorage skor** — DÜŞÜLDÜ (zaten tam implement + test edildi).
   - **➡️ Faz 4 artık tamamen bitti.** Tüm Faz 4 commitleri (`1fc0471` → `eeaad44`) main'e
     merge edildi.

4. **Kerem talebi — ✓ TAMAMLANDI (A + B, bu oturum):** bkz. Bölüm 2 "Faz 4 — Boss Reward +
   Şifa Alan Etkisi". A: dalga-ölçekli boss yem ödülü + "+N 🍖" float (yerine geçer).
   B: Şifa Taşı çevredeki binaları onarır (`repairRange=120`, yuvadan bağımsız, 3-cap paylaşımı,
   `maxHP*0.15*eff` healer-seviyesine bağlı, "+N 🔧" float). A/B karar soruları kapandı.
   Eski keşif notları aşağıda referans olarak korundu:

   **A) Boss reward kademeli + yüzen yazı.** Boss food ödülü dalgaya göre kademeli artsın
   (dalga5=250, 10=500, 15=750) + boss ölünce üstünde "+N yem" yüzen yazı.
   - **Doğrulanan durum:** `killEnemy` [index.html:2919]'da **isBoss dalı YOK**. Tüm düşmanlar:
     `score += e.reward` + `food += round(e.maxHp*0.04)`. Boss `reward = cfg.reward*5` (=125 skor,
     sabit). Boss food şu an **dolaylı** dalga-ölçekli (`maxHp*0.04`): dalga5 ≈75, dalga10 ≈204 —
     istenenden düşük + görünmez.
   - **"Boss level" diye alan YOK** — boss her 5. dalgada (`isBossWave = w%5===0`). "lvl" = `wave`
     global. Formül: `Math.round(wave/5)*250`.
   - **floatTexts sistemi VAR** ([index.html:2092] çizim, outline+yükselme hazır) → yeni sistem
     gerekmez, tek `floatTexts.push({wx,wy,life:1,text,color})` yeter.
   - **Ekleme noktası:** `killEnemy` içine `if (e.isBoss)` dalı (`wave` + `floatTexts` scope'ta erişilir).
   - **KARAR (Kerem):** tiered bonus mevcut `maxHp*0.04`'ün **YERİNE mi ÜSTÜNE mi**? (yerine = tam
     250/500/750, temiz; üstüne = biraz cömert).

   **B) Şifa binası alan etkisi (bina-bina onarım).** Şifa Taşı çevresindeki **binaları** da
   iyileştirsin (şu an sadece kraliçeyi iyileştiriyor).
   - **Doğrulanan durum:** alarm dalı [index.html:1509-1527] sadece `hudQueenHP`'yi iyileştirir
     (15sn, `healAmount×effectMul`, MAX clamp). İki kısıt: healer **yuvaya** `healRange`(130) içinde
     olmalı (yuva-yakınlığı, healer-yarıçapı DEĞİL) + `alarmCanHeal` en fazla 3 healer.
   - **Bina HP sistemi:** `this.hp`/`this.maxHP` var; `applyLevel` maxHP=base×LVL_MUL, hp=maxHP.
     Düşmanlar azaltıyor. **Mevcut tek "onarım" = upgrade** (applyLevel hp'yi doldurur); başka rejen YOK.
   - **Ekleme noktası:** alarm dalına, kraliçe-şifadan sonra, aynı `healTimer` tikinde `buildings`
     üzerinde yarıçap taraması (`o!==this && o.hp<o.maxHP` → `o.hp=min(maxHP, hp+heal)`). Additive,
     sadece `building.hp` okur/yazar → **düşük risk** (düşman/karınca/pathfinding'e dokunmaz).
   - **KARAR (Kerem):** (1) healer-etrafı yarıçap YENİ kavram (yeni config `buildHealRange` ya da
     `healRange` yeniden yorumu); (2) yuva kısıtından bağımsız mı; (3) bina-şifaya da 3-cap mı;
     (4) miktar/interval kraliçeyle aynı mı ayrı mı.

5. **Ses/Pause kontrolleri — ✓ TAMAMLANDI (bu oturum):** bkz. Bölüm 2 "Faz 4 — Mute + Pause/Resume +
   drawScene refactor". Mute (menü+HUD, 🔊/🔇) + Pause/Resume (oyun-içi, ⏸/▶, PAUSED state, donmuş
   dünya + DURAKLADI) eklendi; çizim `drawScene()`'e refactor edildi.
   - **Dil sorusu ÇÖZÜLDÜ:** oyunda **çoklu dil YOK** (sabit Türkçe, sadece HTML `lang="tr"`;
     i18n/STRINGS/lang objesi yok — metinler doğrudan `fillText`'e gömülü). Tuşlar **ikon-only**
     yapıldı → **i18n ihtiyacı yok**. İleride dil katmanı eklenirse gömülü metinler tek tek çıkar.

6. **3 Safari render bug'ı + PAUSED "Ana Menüye Dön" — ✓ TAMAMLANDI (bu oturum, main'e merge):**
   bkz. Bölüm 2 "3 Safari Render Bug'ı" (menü emoji görünürlük + başlık glow fallback + HUD pill
   min-genişlik kilidi) ve "PAUSED ekranı Ana Menüye Dön" (iki aşamalı onay, çıkışta skor kaydedilmez).

> **SIRADAKİ ÖNCELİK — FAZ 5 BAŞLANGICI: KARINCA HP / ÖLÜM MEKANİĞİ.**
> Faz 4 tam kapandı (HUD cila son parçaydı). Sıradaki büyük iş: **karınca HP + ölüm mekaniği** —
> şu an karıncalarda hp YOK (`Ant`'ta hp alanı yok, hiçbir yer `ant.active=false` yapmıyor;
> bilinçle ertelenmişti). Bu **büyük, izole bir workstream**: ant AI'ya dokunur, balans gerektirir
> (düşman dps vs karınca hp, doğum hızı), ve birçok ertelenen işin temelidir.
>
> **Karınca HP kurulunca açılan zincir (tek temel, çoklu küçük ekleme):**
> - **Enemyant soldier avı** — temel kurulunca neredeyse **tek satır** eklenebilir.
> - Akrep alan saldırısına "karıncaya da vur" (aynı yarıçap döngüsüne `ants` taraması — tek satır).
> - Dungbeetle topunun karınca hedeflemesi.
> - 🛡 Kalkan power-up'ı (şu an sadece görsel; koruyacak hp olmadığı için decrement loop EKLENMEDİ —
>   bkz. Bölüm 2 "🛡 Kalkan bilinçle ATLANDI" notu) → hp gelince implement + lejanta geri eklenir.
>
> **Not:** Faz 4 commitleri main'e ÇOKTAN merge edildi. Faz 5 büyük ve izole bir workstream
> olduğu için **temiz (fresh) bir oturumla** başlanması önerilir.

### Ertelenen / Notlar
- **Karınca-bina collision — ON HORIZON.** Toplayıcı/asker karıncalar binaların İÇİNDEN
  geçiyor, yanından dolanmalı. Ant AI'ya (DOKUNMA) dokunduğu için **ayrı izole iş** — önce
  keşif, sonra dikkatli prompt.
- **Kalan bina PNG geçişleri — ON HORIZON.** 5/7 bina PNG'ye geçti (bkz. Bölüm 2 "Bina PNG
  Sprite Geçişi"). Kalan ikisi:
  - **barracks (🗡️ Kışla) → ÇADIR** (büyükçe, başlangıç sade — seviye görsel gelişimine uygun).
  - **tower (🍄 Zehir Kulesi) → PNG + canlı çizim + DUMAN SALINIMI** efekti (nabız/pulse DEĞİL,
    yükselen duman hayali — alarm shimmering'in `performance.now()` faz deseniyle yapılabilir).
  Kalıp hazır (`loadSprite` + draw dalı + ölçek tavanı + fallback). Düşmanlara da uygulanabilir.
- **AUTO-TILING / DUVAR BİRLEŞME SİSTEMİ — ERTELENDİ (tam bütçeli izole faz).** moat + barricade
  yan yana gelince **Clash of Clans mantığıyla** birleşen / yönü değişen duvar. Komşu-kontrolü
  (neighbor bitmask) + çoklu bağlantı sprite seti (uç/köşe/T/düz parçalar) gerekir. DOKUNMA'daki
  yerleştirme/grid sistemine yakın → dikkatli, tam bütçeli izole faz.
- **SEVİYE = GÖRSEL GELİŞİM SİSTEMİ — ERTELENDİ (büyük workstream, glow'a dokunur).** Seviye
  atladıkça bina **BÜYÜMESİN, GÖRSELİ zenginleşsin** (sv1 sade → sv3 spike'lı/çatallı/zengin).
  Mevcut glow/halka + `tier.scale` büyütme mekaniği KALDIRILIP gerçek görsel gelişime geçilir.
  TÜM binalar için seviye-başına ayrı sprite/çizim setleri. **Not:** catapult komutan zırhı ve
  alarm kristali bu sisteme uygun **sv1 sade** başlatıldı → ileride sv2/sv3 varyantları eklenecek.
  Glow sistemi (DOKUNMA'ya yakın) değişeceği için tam bütçeli izole faz.
- **GELİŞMİŞ MANCINIK KOL ANİMASYONU (Seçenek C) — ERTELENDİ.** `catapult.png`'yi **2 parçaya böl**
  (taban + ayrı kol sprite), kolu pivotla gerçek gerilme/savrulma animasyonu. Şu an **Seçenek A**
  uygulandı (komutan karınca + `recoilT` + kova jesti); C ayrı izole faz (PNG dilimleme + pivot).
- **Yuva hasar görsel göstergesi — ERTELENDİ (kendi izole fazına).** Höyükte HP'ye bağlı hasar
  görseli (eşikler **100/75/50/25**) hedefleniyor. Bu oturumda iki yöntem denendi ve **beğenilmedi
  / kaldırıldı**: (1) çizgi-tabanlı çatlaklar (çift katman gölge+hat, dallanma, yumuşak eğri —
  `3ca4edb`/`ffcd2a6`/`30fa174`, revert `126bf3d`); (2) multiply blend radyal yıpranma lekeleri
  (`5e428eb`, revert `697eb21`). İkisi de **izometrik höyük yüzeyine oturmadı**. İleride KENDİ izole
  fazında, sıfırdan daha iyi bir yöntemle yapılacak. **Veri hazır:** yuva canı = kraliçe canı,
  oran = `hudQueenHP / CONFIG.QUEEN.MAX_HP`; `drawNestEntrance` scope'unda globaller doğrudan
  erişilebilir, HP azalma mantığına (satır 2298/2939/3045) dokunmaya gerek yok (salt görsel).
- **Atış kulesi (ranger) 360° dönüp en yakın düşmana bakması — ERTELENDİ.** Kule namlusunun
  hedefe dönmesi **sprite overhaul workstream'ine** ait; o zaman yapılacak.
- **SFX master bus YOK — bilinçli.** Tüm SFX `beep()` ile doğrudan `actx.destination`'a bağlanır
  (müzik ayrı `musicGain` bus'ında). Master gain / filtre / ducking **eklenmedi** — **tutorial/intro
  (Kraliçe anlatım) workstream'ine ertelendi** (anlatım sırasında SFX'i kısmak için ducking orada gerekecek).
- **Zafer / WIN durumu YOK.** Oyun **sonsuz dalga**; sadece kraliçe HP ≤ 0 olunca biter. State seti:
  **`MENU` | `PLAYING` | `PAUSED` | `GAMEOVER`** (WIN/VICTORY yok). Game over başlığı bu yüzden
  "🐜 Yuva Düştü!" (süre değil). İleride zafer koşulu istenirse yeni state + dalga tavanı gerekir.
- **Yem sesi throttle pattern (referans):** `_lastFood` / `_lastDeposit` modül-seviyesi guard +
  `performance.now()`; `if (now - _last < 600) return; _last = now;` → toplu olayda tek tık.
  `sndQueenHurt` aynı deseni 500ms ile kullanır (`_lastHurt`). Yeni sık-çalan SFX eklerken bu deseni taklit et.
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
