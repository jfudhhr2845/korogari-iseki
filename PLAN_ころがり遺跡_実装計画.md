# ころがり遺跡（仮）実装計画書 v1

`HANDOFF_ころがり遺跡.md` を実装可能な粒度まで落とした詳細設計。実装担当（Sonnet想定）はこの文書と HANDOFF を並べて読み、**本書の決定事項に従う**こと。矛盾があれば HANDOFF が正（ただし §0.2 の確定済み解釈を除く）。

---

## 0. 前提と確定事項

### 0.1 参照資産の実際の所在（HANDOFF記載のパスは古い）
| 用途 | 実ファイル |
|---|---|
| **開始フロー一式の最小完成形**：センサー許可→キャリブレーション→プレイ、指モードフォールバック | 本フォルダ `sotto_tower.html` |
| センサー許可・キャリブレーション・⚙デバッグパネル・beep/tone音・指モードUIガードの雛形 | `C:\Users\NY\Desktop\ゲームクリエイトフォルダ\yume_utsutsu\yume_utsutsu.html` |
| 連結性フラッドフィル検証（`generate()` L226-244: 生成→passable集計→DFS→`seen.size!==pass.length`なら`null`返して再生成） | `C:\Users\NY\Desktop\ゲームクリエイトフォルダ\ゲームクリエイト2\mori_no_toride\mori_no_toride.html` |

`mori_dash.html` は無いが、上記2本でセンサー系は完全にカバーできる。流用ポイント：

sotto_tower.html（本フォルダ、シンプルで読みやすいのでまずこちらを読む）:
- `requestSensor()`（L142-156）: iOS `DeviceMotionEvent.requestPermission()` → devicemotion購読 → 700ms以内にイベントが来たかで成否判定
- `calibrate()`（L159-172）: 3秒カウントダウン中に `accelerationIncludingGravity` をサンプリングして平均を基準姿勢（baseGX）に
- 開始ボタンのセンサー失敗フォールバック（L278-289）: 失敗時にボタン文言を一時変更して指モードへ誘導
- 傾き算出（L129-141）: `(g.x-baseGX)/9.8` を校正済み傾きとして使用、`TILT_MAX` でキャップ
- メインループのdtクランプ（L272: `dt=Math.min(...,0.05)`）: タブ復帰時の物理吹っ飛び防止。**本作でも必須**
- キャリブレーション/タイトル/結果のオーバーレイHTML・CSS構造

yume_utsutsu.html:
- `tone()` / `PENTA`配列（L205-215）: WebAudio合成（beepはsottoにもある。toneとペンタトニックはyumeから）
- ⚙パネルのHTML/CSS構造（`#dbgBtn` `#dbg`、L54-58, 97-104）
- 指モードの `onUI()` ガード（L567: UI要素上のタッチはゲーム入力にしない）
- 3軸キャリブレーション（L549-563: baseGX/baseGZ の2軸を平均。本作はこれを3軸に拡張）

### 0.2 本書で確定した解釈・決定（HANDOFFの曖昧点）
1. **塔サイズはタイルグリッド数**（壁タイル込み）。奇数なのが根拠。論理迷路セルは `(N-1)/2`：
   - こじんまり 15×15タイル = 7×7セル × 2階
   - 標準 21×21タイル = 10×10セル × 3階
   - 巨大 31×31タイル = 15×15セル × 4階
2. **階段は双方向**（上り下り自由）。詰み防止・石碑帰還と相性が良い。
3. **鍵扉は「最上階の出口」を守る関所に1枚**、鍵は密閉区画（落ちるが正解ルート）に置く。これで全サイズで「登る→穴を探す→落ちる→鍵→一方通行扉→登り直す→出口」のループが成立する（2階建てでも矛盾しない）。
4. **出口と各階段はdead-endセルに配置**し、扉はその唯一の開口に置く（ブレイド化で迂回路ができて扉が無意味化するのを防ぐ）。
5. ファイル名は `korogari_iseki.html`（本フォルダ直下）。タイトル文言は実装開始時にユーザーと確定（仮表示は「ころがり遺跡」）。

---

## 1. 成果物と検証規約

- 成果物: `korogari_iseki.html` 単一ファイル。Three.js r0.160.0 を importmap でCDN読込（yume_utsutsu L146 と同一の記述）。
- 編集のたびに構文チェック（ブラウザ実行は不可、実機確認はユーザー担当）:

```powershell
$html = Get-Content -Raw -Encoding UTF8 korogari_iseki.html
$m = [regex]::Match($html, '(?s)<script type="module">(.*?)</script>')
Set-Content -NoNewline -Encoding UTF8 _check.mjs $m.Groups[1].Value
node --check _check.mjs
```

- UIフォント: Zen Maru Gothic（Google Fonts、yume_utsutsuと同じlinkタグ）。
- コメント・UI文言は日本語。コード識別子はASCIIのみ。全角スペース・不可視文字の混入禁止。
- 調整値は必ず `TUNE`（プレイフィール）と `GEN`（生成パラメータ）に集約。マジックナンバー禁止。

---

## 2. ファイル内セクション構成（この順で書く）

```
<head>  meta viewport / fonts / <style>
<body>  #app(canvas親) / HUD群 / オーバーレイ群 / #dbg / importmap / <script type="module">
JS:
  S1  定数: TUNE, GEN, SIZES, TILE定義, 色定数
  S2  ユーティリティ: mulberry32 RNG, el(), clamp, lerp
  S3  オーディオ: beep/tone/wind/thud/chime（AudioContext遅延生成）
  S4  迷路生成: genFloor(), braid(), carveRooms(), genTower(), validateTower()
  S5  Three初期化: シーン/カメラ/レンダラ/ライト
  S6  タワー描画: buildFloorMeshes()（InstancedMesh）, 小物(松明/宝箱/石碑/扉/玉/格子)
  S7  ボール: 生成、物理step、衝突、タイル反応
  S8  カメラ追従・フロア遷移演出（落下/階段/クリア）
  S9  入力: センサー(requestSensor/calibrate/onMotion), 指モード(仮想スティック)
  S10 ナッジ: 停止検知、脈動キック、羅針盤BFS
  S11 ミニマップ: visited配列、Canvas2D描画、タブ/拡大
  S12 HUD・画面フロー・⚙パネル・メニュー
  S13 ゲーム進行: startGame/finishGame/タイマー/localStorage
  S14 メインループ: loop() → update(dt) → render
```

---

## 3. データモデル

### 3.1 タイルグリッド
- 1タイル = ワールド1.0単位。グリッドは `N×N`（Nは SIZES から）。
- 奇数座標 `(x,y)`（x,y共に奇数）= 迷路セル。偶数を含む座標 = 壁候補。
- フロアごとに `Uint8Array(N*N)` の `tiles`。値：

```js
const T={ WALL:0, FLOOR:1, HOLE:2, STAIR_UP:3, STAIR_DOWN:4, EXIT:5,
          ICE:6, SAND:7, CRACK:8, GRATE:9, DOOR_LOCK:10, DOOR_ONEWAY:11 };
```

- **通行可能** = WALL, DOOR_LOCK(未開錠), DOOR_ONEWAY(外側から) 以外。ICE/SAND/CRACK/GRATE/STAIR/HOLE/EXITはすべて床として転がれる。
- 扉は開くと `FLOOR` に書き換える（状態を持たない。開錠は不可逆）。CRACKも崩壊時に `HOLE` に書き換える。

### 3.2 フロアオブジェクト
```js
floor = {
  tiles: Uint8Array,          // 上記
  items: [],                  // {kind:'orb'|'key'|'chest'|'stele'|'torch', x,y, taken:false, mesh, ...}
                              // chestは {content:'orbs'|'key'|'compass'}
  onewayDir: Map,             // "x,y" -> {dx,dy} 一方通行扉の通過許可方向（内→外）
  leakHoles: Set,             // "x,y" 光漏れ演出をつける穴（=正解ルート穴）
  visited: Uint8Array,        // ミニマップ用フォグ
  distField: Int16Array|null, // 羅針盤用（§10で構築）
  group: THREE.Group,         // このフロアの描画ルート
};
tower = { seed, size, floors:[floor...], startXY, compartment:{f,rect} };
```

### 3.3 ゲーム状態
```js
state = {
  phase:'menu'|'calib'|'play'|'pause'|'fall'|'stairs'|'clear',
  f:0,                        // 現在フロア(0=1階)
  ball:{x,z,vx,vz, r:0.32, lightR},  // ワールド座標（タイル座標=floor(x),floor(z)）
  orbs:0, hasKey:false, hasCompass:false,
  checkpoint:{f,x,y}|null,    // 石碑
  timeMs:0,                   // playフェーズ中のみ加算
  cellPrev:{x,y},             // CRACK崩壊トリガ用（中心セルの前回値）
  stairArmed:true,            // 階段の再発動防止（§7.5）
  nudge:{stillT:0, active:false, kickIdx:0, kickT:0},
};
```

---

## 4. 生成アルゴリズム

### 4.1 シードRNG
mulberry32。`rng=mulberry32(seed)` を生成処理全体で共有（描画の揺らぎ等は `Math.random` 可、生成系は必ずrng）。シードは新規時 `Math.floor(Math.random()*1e6)`、結果画面に表示、「同じ迷路でもう一回」で再利用。

### 4.2 単層生成 `genFloor(rng, N)`
1. 全タイル WALL で初期化。
2. **穴掘り法（recursive backtracker）**: 論理セル `(cx,cy)`（タイル座標 `(2cx+1, 2cy+1)`）をスタックで深さ優先。未訪問隣接セルへ進む際、間の壁タイルとセルタイルを FLOOR に。
3. **ブレイド化 `braid()`**: 全dead-endセル（開口1のセル）を列挙し、`GEN.braidRate`(0.12) の確率で「反対側もセルである壁タイル」を1枚除去してループ化。**ただし §4.3 で階段/出口に予約されたdead-endは除外**。
4. **部屋 `carveRooms()`**: `GEN.roomsPerFloor` 個、論理2×2〜3×3セル分の矩形（タイル5×5〜7×7）をランダム位置に選び、内部の壁タイルを全部 FLOOR に。既存の部屋と重ならないこと。部屋の矩形リストを返す（宝箱・松明・密閉区画の置き場に使う）。

### 4.3 多層生成 `genTower(seed, size)`
順序が重要。以下を1試行とし、`validateTower()` 不合格なら seed+=1 で再試行（**最大40回**。全滅時はエラーメッセージ表示——実際にはまず起きない）。

1. 各フロアを `genFloor` で独立生成（braid前に階段用dead-endを確保するため、braidは手順3の後に実行してもよい。実装しやすい方で良いが、**階段開口のループ化除外**だけは必ず守る）。
2. **スタート**: 1階のランダムなセル。周囲2セルに穴・ギミックを置かない（後続手順の除外リストに入れる）。
3. **階段**: フロア f (0..F-2) ごとに dead-end セルを1つ選び `STAIR_UP`。フロア f+1 の同座標セルを `STAIR_DOWN` にする（同座標は必ずセルなので床であることが保証される。ただし後続手順で上書きしないよう予約）。フロア間で階段座標が縦に連続しないよう、直前フロアの階段から距離 N/3 以上離す。
4. **出口**: 最上階の dead-end セル（階段から遠い位置）を `EXIT`。その唯一の開口タイルを `DOOR_LOCK` にする。
5. **密閉区画（落ちるが正解）**: 対象フロア＝`GEN.compartmentFloors(size)`（標準・こじんまり=[0]、巨大=[0,1]）。各対象フロア f について：
   1. 手順4.2-4の部屋から、スタート・階段・出口を含まない部屋を1つ選ぶ（なければ新規に3×3セル部屋を彫る）。
   2. 部屋矩形の**境界上の開口タイルを全て WALL に戻して密閉**。
   3. 境界の壁タイルのうち「外側が FLOOR」の1枚を `DOOR_ONEWAY` にし、`onewayDir` に内→外ベクトルを登録。
   4. 区画内のセルを1つ選び、フロア f+1 の同座標を `HOLE` に（既に階段等なら別セル）。`leakHoles` に登録（光漏れ演出）。
   5. 報酬: 最初の区画には **鍵**（床置きitem）。2つ目（巨大のみ）には宝箱（content='compass'）＋灯り玉×3。
6. **ランダム穴**: フロア f≥1 に `GEN.holesPerFloor`(2) 個。奇数×奇数セルのみ（→落下先が必ず床）。除外: 階段・出口・スタート直上・区画の真上以外の区画内部（=区画の天井に余計な入口を開けない。区画矩形上のセルは手順5-4の1個だけ）。
7. **格子床 GRATE**: フロア f≥1 に `GEN.gratesPerFloor`(5) 個、セル位置。下階が部屋・宝箱付近になる位置を優先（「下に何かある」演出のため。実装は「下階の同座標から半径2にitemがある候補を優先、なければランダム」程度でよい）。
8. **氷床・砂だまり**: フロアごとに `GEN.icePatches`(1〜2)・`GEN.sandPatches`(1〜2) 領域。部屋内部または廊下の連続タイルを2×2〜3×3で塗る（WALL以外のFLOORのみ置換。階段・穴・扉の隣接1タイルには置かない）。
9. **ひび割れ床 CRACK**: フロア f≥1 に `GEN.cracksPerFloor`(2) 個、奇数×奇数セルのみ（=穴になっても落下先が床）。検証は「CRACKを全部HOLEとみなした最悪ケース」で行う（§4.4）ため、この時点では位置制約は穴と同じでよい。
10. **アイテム**: 灯り玉 `GEN.orbsPerFloor`(6) をFLOORセルに散布（行き止まり・部屋を優先）。宝箱 `GEN.chestsPerTower`(2＋区画分) を部屋に（content: 'orbs'=灯り玉×5 / 'compass'。compassは塔に1個だけ、鍵入り宝箱は使わない=鍵は床置きで見える方が親切）。石碑1基/フロア（階段の近くのFLOORセル）。松明 `GEN.torchesPerFloor`(8) を部屋の壁際・分岐点の壁タイルに面したFLOORセル脇へ。
11. `validateTower()` → 合格なら tower を返す。

### 4.4 検証 `validateTower(tower)` — 省略禁止
mori_no_toride の「不合格ならnull→再生成」パターンの多層版。ノード= `(f, x, y)` の通行可能タイル、`id = (f*N+y)*N+x`。

**エッジ構築**（有向）:
- フロア内4近傍: 双方向。ただし `DOOR_ONEWAY` は onewayDir の向きのみ、`DOOR_LOCK` は「開錠後グラフ」でのみ双方向。
- `HOLE`（＋最悪ケースでは`CRACK`もHOLE扱い）: 下向き `(f,x,y)→(f-1,x,y)` のみ。**穴タイルからの横移動エッジは張らない**（乗ったら落ちるため。横からの流入エッジは張る）。
- `STAIR_UP f ↔ STAIR_DOWN f+1`: 双方向。

**チェック**（BFS。全てCRACK=HOLEの最悪ケースグラフで実施）:
1. 【鍵到達】開錠前グラフでスタートから前方BFS → 鍵ノードに到達できること。
2. 【鍵前の詰み無し】開錠前グラフで鍵ノードから**逆辺BFS** → 手順1の前方到達集合が全て含まれること（どこに落ちても必ず鍵まで戻れる）。
3. 【クリア到達】開錠後グラフでスタートから前方BFS → EXITに到達できること。
4. 【鍵後の詰み無し】開錠後グラフでEXITから逆辺BFS → 手順3の前方到達集合が全て含まれること（**どの穴に落ちても出口に到達可能**の一般化）。

4条件すべて合格で採用。逆辺BFSは隣接リストを転置して普通のBFSでよい（ノード数は最大 31*31*4≒3.8k、コスト無視できる）。

---

## 5. ボール物理（自作・物理エンジン禁止）

### 5.1 入力→加速度
```
tilt = {x, y}  // -1..1 正規化（センサー: §9.1、指: §9.2）
mag = length(tilt)
if (mag < TUNE.deadzone) tilt = 0
else tilt *= (mag - TUNE.deadzone) / (1 - TUNE.deadzone) / mag  // デッドゾーン外を滑らかに再マップ
tilt = clampLength(tilt, TUNE.maxTilt)
accel = tilt * TUNE.accel * (砂だまり ? 0.5 : 1)
```
- ワールド対応: カメラは -Z を見る固定向き。`vx += tilt.x*a*dt`、`vz += tilt.y*a*dt`（画面上=奥=-Z なので tilt.y の符号は実機で確定。⚙パネルに X/Y反転・XY入替チェックを必ず付ける）。

### 5.2 摩擦
指数減衰 `v *= exp(-damp*dt)`。`damp` は中心セルのタイルで切替: 通常 `TUNE.damp`(3.5)、氷 `TUNE.dampIce`(0.25)、砂 `TUNE.dampSand`(9.0)。

### 5.3 壁衝突（グリッドAABB＋スライド）
- 1フレームの移動量 `|v|*dt` が `0.2` を超える場合はサブステップ分割（`n=ceil(|v|*dt/0.2)`）。トンネリング防止。
- 各サブステップ: 位置を進めた後、ボール中心の周囲3×3タイルのうち**非通行タイル**（WALL・未開錠DOOR_LOCK・外側からのDOOR_ONEWAY）それぞれをAABB(1×1)として、円との最近接点で押し出し。押し出した軸の速度成分を0に（法線方向のみ殺す→接線成分が残り自然にスライド）。反発係数は0（`TUNE.bounce`=0で定義だけしておく）。
- DOOR_ONEWAYの「内側から」判定: ボール中心タイルが区画内側（onewayDirの逆側）にあれば通行可能扱い→接触したら開扉（`FLOOR`化＋音＋軋み演出）。外側からは完全に壁。
- DOOR_LOCK: `hasKey` かつ接触で開扉（鍵消費、南京錠が落ちる演出＋音）。鍵なしなら壁。

### 5.4 タイル反応（中心セル判定）
毎フレーム `cell = (floor(x), floor(z))`：
- `HOLE`: 中心が穴タイルに入り、かつ穴中心との距離 < `TUNE.holeCapture`(0.35) → 落下シーケンス（§8.2）。
- `STAIR_UP/DOWN`: `stairArmed` なら遷移開始（§8.3）。遷移後 `stairArmed=false`、中心セルが階段外に出たら再アーム。
- `CRACK`: 中心セルが「前フレームCRACK → 今別セル」になった瞬間、そのCRACKタイルを `HOLE` に書換え（破片パーティクル＋崩落音＋ミニマップ更新＋メッシュ差替え）。§4.4の最悪ケース検証済みなので到達可能性は壊れない。
- `item` との取得判定: 距離 < 0.5。orb=光半径+`TUNE.orbLightGain`＋チャイム、key=`hasKey=true`、chest=開封演出→中身付与、stele=チェックポイント登録（青く点灯＋音）。
- `EXIT`: クリアシーケンス（§8.4）。

---

## 6. 描画

### 6.1 シーン基本
- `scene.background = 0x000000`、`scene.fog = new THREE.Fog(0x000000, near, far)`。`near = ball.lightR*0.7`, `far = ball.lightR*2.4`（灯り玉取得で更新）。
- 環境光: `AmbientLight(0x223344, 0.06)`（完全な黒つぶれ回避、ほぼ見えないレベル）。
- renderer: `setPixelRatio(min(devicePixelRatio,2))`、シャドウ無効（ポイントライトは遮蔽されず床下も照らすが、床板が視線を遮るので見た目は問題ない。むしろ格子床から下が見える仕様に合致）。

### 6.2 フロアメッシュ `buildFloorMeshes(floor)` 
フロアごとに `THREE.Group` にまとめ、**表示は「現在フロア＋直下フロア」のみ visible**（直下は格子床・穴からの覗き用）。
- **壁**: `InstancedMesh(BoxGeometry(1, 1.2, 1), stoneMat, 壁数)`。`instanceColor` で明度±10%の揺らぎ。座標 `(x+0.5, 0.6, y+0.5)`。
- **床**: `InstancedMesh(BoxGeometry(1, 0.1, 1), floorMat, 床数)`。y=-0.05。HOLE・GRATEタイルはインスタンスを置かない。ICE=水色系・SAND=黄土色・CRACK=ひび色は `instanceColor` で塗り分け（CRACKはひび模様のCanvasTextureを別マテリアルの小Meshで重ねてもよい。簡素優先）。
- **GRATE**: 床の代わりに細い格子バー（Box 0.08幅×5本を2方向、または1枚のPlaneに格子CanvasTexture+alphaTest）。下階が隙間から見える。
- **フロアの高さ**: フロアfの床面 y = `f * GEN.floorHeight`(3.0)。直下フロアは y が -3 の位置に丸ごと描画される（同一Group内は相対0で、Group.position.yで段差を付ける）。
- **穴**: 床インスタンス無し＋縁に薄い枠Mesh。`leakHoles` は穴の中に上向きの淡い光（emissiveな半透明Cone＋微パーティクル。PointLightは使わずフェイクで軽く）。
- **小物**: 松明=壁面の小さな炎スプライト（emissive平面＋ゆらぎ）、宝箱・石碑・扉・鍵・灯り玉は数個のBox/Sphereで組む（mori_no_toride の小物の組み方が参考になる）。灯り玉は emissive球＋浮遊アニメ。

### 6.3 光
- **ボール**: emissive球（暖色 0xffd9a0）＋ `PointLight(0xffd9a0, 2.2, distance=lightR, decay=1.6)`。lightR 初期 `TUNE.lightR0`(4.5)、orbごとに `+0.35`、上限 `TUNE.lightRMax`(9)。
- **松明ライトプール**: PointLightを増やすとシェーダコストが跳ねるので、実PointLightは `TUNE.torchPool`(4) 個だけ生成し、0.5秒ごとにボールに近い松明へ付け替える。遠い松明は炎スプライトのみ（フォグで見えないので破綻しない）。
- 出口: 天窓からの光の柱（半透明Cylinder、fog:false）。

---

## 7.（§5に統合済み・欠番）

## 8. カメラとフロア遷移

### 8.1 追従カメラ（向き固定・回転禁止）
- `camera.position = ballPos + (0, TUNE.camH(7), TUNE.camD(5))`、`lookAt(ballPos)`（約54°見下ろし）。毎フレーム `pos.lerp(target, 1-exp(-TUNE.camLerp(6)*dt))`。ヨー回転は一切しない（鉄則）。

### 8.2 落下演出（phase='fall'）
1. 発動: 入力無効化、ボールを穴中心へ吸着。
2. ボールとカメラを一緒に `-GEN.floorHeight` までイージング落下（0.6s、最後に軽いバウンド）。土煙パーティクル＋ドスン音。
3. `state.f--`、フロアvisible切替、phase='play'。アイテム保持・ノーペナルティ。タイマーは流れ続ける。

### 8.3 階段演出（phase='stairs'）
上昇: 画面を一瞬フェード（0.4s）しつつボールy を +floorHeight、`state.f++`、`stairArmed=false`。下りも同様に逆方向。

### 8.4 クリア（phase='clear'）
出口タイル到達 → 入力無効 → 光の柱が強くなりボールが浮かんで吸い込まれる（1.5s）→ ホワイトアウト → 結果画面。

### 8.5 石碑帰還
メニューから「石碑に戻る」→ フェード → checkpoint の (f,x,y) へ配置（v=0）。未登録ならスタート地点。ノーペナルティ。

---

## 9. 入力

### 9.1 センサーモード
- yume_utsutsu の `requestSensor()`/`calibrate()` を流用。キャリブレーションで `baseGX, baseGY, baseGZ` を保存（3軸とも取る）。
- 傾き算出: `tiltRawX=(g.x-baseGX)/9.8`、`tiltRawY=(g.z-baseGZ)/9.8`（yume実績の軸。実機で違和感があれば⚙の「XY入替」「X反転」「Y反転」で即補正できるようにしておく——**フェーズ1実機確認の最重要項目**）。
- スムージング: `tiltSm = lerp(tiltSm, tiltRaw, 1-exp(-TUNE.tiltSmooth(12)*dt))`。
- ⚙パネルに生値 (g.x g.y g.z / tiltX tiltY) を常時表示＋「再キャリブレーション」ボタン（プレイ中可、3秒カウント中はpause扱い）。

### 9.2 指モード（仮想スティック）
- `pointerdown`（UI外）: その座標をスティック中心とし、ベース円＋ノブをDOM表示。
- `pointermove`: `tilt = clampLength((cur-center)/TUNE.stickRadius(70px), 1)`。
- `pointerup`: tilt=0、スティック非表示。デッドゾーン等の下流処理はセンサーと完全共通。
- yume の `onUI()` ガード（`#dbg,#dbgBtn,.overlay` 上のタッチ無視）を流用。

### 9.3 ⚙デバッグパネル（必須・プレイ中変更可）
スライダー: 感度(accel)、デッドゾーン、摩擦(damp)、光半径(lightR0)、ナッジ距離、カメラ高さ。チェック: X反転/Y反転/XY入替。ボタン: 再キャリブレーション。表示: センサー生値、現在速度、現在セル、FPS。変更値は localStorage `iseki_tune` に保存し次回起動時に復元。

---

## 10. 脈動ナッジ（独自機構）

- **発動条件**: `|v| < 0.05` かつ 傾きがデッドゾーン内 が `TUNE.nudgeWait`(1.0s) 連続。
- **挙動**: 3回のキック。`TUNE.nudgeInterval`(0.35s) ごとに `v += dir * TUNE.nudgeDist * damp`（指数摩擦下で移動距離が約 nudgeDist(0.3) になる初速）。3回終了後は再度「停止1秒」を満たすまで再発動しない。プレイヤー入力がデッドゾーンを超えたら即中断。
- **安全弁**: キック方向の `TUNE.nudgeDist*2` 以内にHOLE/CRACKタイル中心があるキックは移動を抑止（音と明滅だけ出す）。穴に押し込まない。
- **演出**: キックと同期して風音（§13）＋ボール光の明滅（intensity を 2.2→3.2→2.2、0.15s）。「これは信号」と分かるリズム感を最優先。
- **方向**:
  - 通常: 現在フロアの `STAIR_UP`（最上階では `EXIT`）への**直線方向**（壁無視の弱ヒント）。
  - **羅針盤入手後（永続）**: 多層有向グラフをEXITから逆辺BFSした距離場 `distField` を使い、現在タイルの4近傍（通行可能エッジのみ）で距離最小の方向。距離場は「羅針盤入手時・扉開錠時・CRACK崩壊時」に再計算（数千ノードなので一瞬）。現在タイルの最短次手が「穴に落ちる」の場合は穴方向を指す（安全弁より優先＝羅針盤時は穴へ誘導してよい。ただしその穴が正解ルートのときのみ距離場が自然にそうなる）。

---

## 11. ミニマップ

- フロアごと `visited: Uint8Array(N*N)`。毎フレーム、ボール周囲半径1.5タイルを visited=1。
- 画面隅の `<canvas>`(120×120px) に現在フロアの visited 部分だけ描画: 床=暗灰、壁=明るめ線、穴=黒丸、階段=▲▼、石碑=◆、自機=光点。1セル= `120/N` px。更新は0.2秒ごとで十分。
- タップで拡大オーバーレイ（画面中央、フロアタブ 1F/2F/... 切替可。訪問済みフロアのみタブ活性）。プレイは止めない（ゲーム内タイマーも流れる＝覗くのもコスト、の設計）。

---

## 12. HUD・画面フロー

### 12.1 DOM構成（yume_utsutsuのCSSパターン踏襲）
```
#hudTop: 経過タイム / 🔮灯り玉数 / 現在フロア(2F/4F)
#minimap(canvas) #minimapBig(overlay)
#dbgBtn #dbg
#menuBtn → #menu overlay: つづける / 石碑に戻る / リタイア（タイトルへ）
#start overlay: タイトル・ルール・塔サイズ選択(3ボタン)・「センサーで遊ぶ」「指で遊ぶ」
#calib overlay: 3秒カウントダウン
#result overlay: タイム / 灯り玉 / シード / ベスト表示 / 「同じ迷路でもう一回」「新しい塔へ」
#toast: 中央の一時テキスト（「鍵を手に入れた！」等。yumeのannounce流用）
```
- メニュー表示中は phase='pause'、タイマー停止（§13.1）。
- センサーモード選択→ requestSensor 失敗時は「センサーが取れないので指モードで開始します」トースト→指モードにフォールバック。

### 12.2 HUDイベント文言（トースト）
鍵入手/扉開錠/一方通行扉が開いた/石碑に記録した/羅針盤入手（「ナッジが正しい道を指すようになった」）/ひび割れ崩落 など。演出だけで伝わるものは文言を短く。

---

## 13. オーディオ（WebAudio合成のみ）

- `beep()`/`tone()`/`PENTA` を yume からコピー。
- **風音（ナッジ）**: 1.2s のホワイトノイズBufferSource → BiquadFilter(lowpass) を 300Hz→1400Hz→300Hz にスイープ、gain 0→0.15→0。キックごとに短く3回。
- 灯り玉: PENTAチャイム（連続取得で音階上昇、コンボ窓2s）。
- 落下: 低音サイン90Hz(0.3s)＋ノイズバースト（ドスン）。着地時に土煙。
- 階段: 上昇アルペジオ。扉: 石の軋み（矩形波低音＋短ノイズ）。崩落: ノイズ+ピッチ落ちる矩形波。クリア: PENTAでファンファーレ。
- 全SEは `state.phase==='play'` 系のみ。AudioContextは最初のユーザー操作で生成（iOS対策、yume方式）。

---

## 14. スコア・保存

- タイマー: `performance.now()` 差分を play/fall/stairs 中のみ `timeMs` に加算。pause・キャリブレーション中・オーバーレイ中は加算しない。
- localStorage: `iseki_best_{small|std|big}` = ms。`iseki_tune` = ⚙調整値。書込みは try-catch。
- 結果画面: タイム(mm:ss.d)、灯り玉 n/総数、シード、ベスト更新なら強調。

---

## 15. TUNE / GEN 初期値表（すべて⚙かコード先頭で一元管理）

```js
const TUNE={
  accel:14, deadzone:0.06, maxTilt:0.45, damp:3.5, dampIce:0.25, dampSand:9.0,
  ballR:0.32, bounce:0, holeCapture:0.35, tiltSmooth:12,
  lightR0:4.5, lightRMax:9, orbLightGain:0.35, torchPool:4,
  camH:7, camD:5, camLerp:6,
  nudgeWait:1.0, nudgeInterval:0.35, nudgeDist:0.3,
  stickRadius:70,
  invX:1, invY:1, swapXY:false,
};
const GEN={
  floorHeight:3.0, braidRate:0.12, maxRetry:40,
  roomsPerFloor:3, holesPerFloor:2, gratesPerFloor:5,
  icePatches:2, sandPatches:1, cracksPerFloor:2,
  orbsPerFloor:6, torchesPerFloor:8, chestsPerTower:2,
};
const SIZES={
  small:{label:'こじんまり', N:15, floors:2, compartmentFloors:[0]},
  std:  {label:'標準',       N:21, floors:3, compartmentFloors:[0]},
  big:  {label:'巨大',       N:31, floors:4, compartmentFloors:[0,1]},
};
```
数値は初期仮置き。**フェーズ1の実機確認で accel/deadzone/damp/camH を確定させるのが最優先**（HANDOFF §11）。

---

## 16. 実装手順（マイルストーン式）

各Mの終わりに必ず `node --check`。「📱」= ユーザー実機確認ポイント（GitHub Pages等HTTPSに置いて確認してもらう）。

### フェーズ1（転がしの手応え確定が目的）
- **M0 骨組み**: HTML/CSS/importmap/TUNE/GEN/画面オーバーレイの殻/メインループ。空のシーンが回る。
- **M1 単層迷路の生成と描画**: genFloor＋braid＋carveRooms、InstancedMesh壁・床、**一時的に全体照明ON**（暗闇はM4から）。固定シードで再現確認。カメラ追従。
- **M2 ボール物理＋指モード**: 仮想スティック→加速度→摩擦→AABB衝突スライド＋サブステップ。デスクトップのマウスドラッグでも動くこと（pointerイベントなので自然に対応）。
- **M3 センサー＋キャリブレーション＋⚙**: requestSensor/calibrate流用、デッドゾーン、軸反転/入替、生値表示、再キャリブレーション、tune保存。📱**実機①: ドリフトせず止まれるか・操作の気持ちよさ**。ここでTUNEを確定させるまで先へ進まない。
- **M4 暗闇と灯り**: 背景黒＋フォグ＋ボールPointLight＋emissive、灯り玉散布と取得で光半径拡大、松明プール。📱**実機②: 暗さ・見える範囲の快適さ**。
- **M5 2層化＋落下＋ミニマップ**: 2フロア生成（まだ検証は単純連結のみ）、穴・落下演出・階段遷移・stairArmed、フォグミニマップ。📱**実機③: 落下の気持ちよさ**。

### フェーズ2（ゲームとして完成）
- **M6 多層生成＋検証**: genTower全手順（dead-end階段・出口関所・密閉区画・一方通行扉・鍵）＋validateTower（4条件BFS）＋40回リトライ。**console.logに検証結果（試行回数）を出す**。全サイズ×シード100本のスモークテストを手元スクリプトで回す（生成関数は純粋JSなのでnodeで実行可能なように、生成S4セクションはThree非依存で書くこと）。
- **M7 ギミック**: 氷・砂・ひび割れ崩落・鍵扉・一方通行扉の物理/演出、宝箱・石碑・帰還メニュー。
- **M8 ナッジ＋羅針盤**: 停止検知→脈動キック→風音＋明滅、安全弁、距離場BFSと再計算フック。
- **M9 進行系**: タイマー（pause考慮）、結果画面、ベスト保存、シード再挑戦、タイトル→サイズ選択→モード選択フロー完成。
- **M10 仕上げ**: 格子床と下階覗き、光漏れ穴、クリア演出（光の柱）、全SE、トースト文言、パフォーマンス確認（巨大サイズでinstance数・ライト数を確認）。📱**実機④: 通しプレイ**。

### M6のスモークテスト補足
生成＋検証を `node` 単体で回せるよう、S4は `THREE` を一切参照しない純ロジックにする（メッシュ構築S6と分離）。テストは使い捨てスクリプトで良い（成果物に含めない）:
```js
// scratch/gen_test.mjs 例: korogari_iseki.html からS4を目視コピペ or 抽出して
for (const size of ['small','std','big'])
  for (let seed=0; seed<100; seed++) genTower(seed,size); // 例外・null無しを確認
```

---

## 17. 既知の罠 → 本書での対処対応表

| HANDOFF §12 | 対処箇所 |
|---|---|
| ドリフト対策が生命線 | §9.1 キャリブレーション＋§5.1 デッドゾーン再マップ＋⚙再キャリブレーション（M3で実機確定） |
| 中心セル判定・AABBスライド・サブステップ | §5.3, §5.4 |
| 多層グラフ検証の省略禁止 | §4.4（4条件・逆辺BFS）＋M6スモークテスト100シード |
| ひび割れが到達可能性を壊す | §4.4 最悪ケース（CRACK=HOLE）で事前検証 |
| ミニマップは軽く | §11 Uint8Array＋0.2s間隔Canvas2D |
| タイマーのpause考慮 | §14 |
| 非ASCII混入・構文チェック | §1 のPowerShell抽出→`node --check` を毎編集後 |
| 階段の即再発動（本書で追加発見） | §5.4 / §8.3 stairArmed |
| PointLight増殖による負荷（同上） | §6.3 松明ライトプール |
| 2階建て塔での鍵デッドロック（同上） | §0.2-3 鍵扉は出口関所に固定 |

---

## 18. 未決事項（実装開始時にユーザーへ確認）

1. **タイトル確定**: ころがり遺跡／いせきのとう／ひかりの玉と遺跡の塔（HANDOFF記載の候補）。
2. **塔サイズの解釈**（§0.2-1）: 本書は「15/21/31=タイル数（迷路セルは7/10/15）」で設計した。もし「迷路セル数」の意図なら GEN/SIZES の N を `2*セル+1` に変えるだけで対応可能（設計は不変）。
3. 巨大サイズの密閉区画2つ目の中身（本書は羅針盤宝箱＋灯り玉としたが、羅針盤を通常部屋の宝箱にする案もある）。
