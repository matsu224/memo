# Shader 備忘録

> 共通概念 → HLSL / GLSL の記法 → Unity / URP の実装、の順に整理する。
> 数学や描画の原理と、言語・Graphics API・Engine 固有の仕様を区別する。
> Unity / URP では主に **ShaderLab + HLSL** を使用する。

## 読み方

| 範囲 | 内容 |
|---|---|
| 第1〜8章 | 共通知識。数学や可視化の記法は HLSL / GLSL を併記 |
| 第9〜11章 | HLSL・GLSL の基本記法と対応表 |
| 第12〜14章 | Unity 固有の ShaderLab・Render Pipeline・URP |
| 第15〜17章 | デバッグ、用語集、参考資料 |

* GLSL の基本例は OpenGL 向け GLSL 3.30 Core。Unity 固有の説明は参考資料の Unity 6.3 を基準とする。
* 言語が同じでも、対象 API・バージョン・Engine が違うと座標規約や使える機能は異なる。

---

## 1. Shader の基本

### Shader（シェーダー）とは

* **Shader**
  * GPU 上で実行されるプログラム。
  * 頂点の位置や、最終的に画面へ描画する色などを計算する。
  * ゲームではモデル、地形、エフェクト、UI、ポストプロセスなど、様々な描画に使用される。

### CPU と GPU

* **CPU**
  * ゲームロジック、オブジェクト管理、AI、描画命令など幅広い処理を担当する。
* **GPU**
  * 大量の頂点やピクセルに対して、同じ種類の計算を並列に実行することを得意とする。
* Shader は主に GPU 上で実行される。

### Material と Shader の違い

* **Shader**
  * 「どのように描画するか」を記述したプログラム。
* **Material**
  * 使用する Shader と、その Shader に渡す色・Texture・数値などの設定値をまとめたもの。
* 同じ Shader を使用していても、Material の値を変えることで異なる見た目にできる。

---

## 2. グラフィックスパイプライン

### 大まかな描画の流れ

1. Mesh から頂点データを受け取る。
2. Vertex Shader が頂点ごとに処理を行い、頂点位置を Homogeneous Clip Space へ変換する。
3. 頂点から三角形などの Primitive が組み立てられる。
4. Rasterizer Stage で視錐台に対する Clipping、Perspective Divide、Viewport への変換、Face Culling などが行われる。
5. Primitive が Fragment に変換され、Vertex Shader から渡した値が補間される。
6. Fragment Shader が各 Fragment の色などを計算する。
7. Depth / Stencil Test や Blending などを経て Render Target へ書き込まれる。

* 実際の GPU 内部では最適化によって処理順が前後したり統合されたりするため、上記は「概念上の流れ」として理解する。
* Tessellation Shader や Geometry Shader など、用途によって途中に追加されるステージもある。

### Vertex Shader

* **Vertex Shader（頂点シェーダー）**
  * 頂点ごとに実行される Shader。
  * 主に頂点の位置を描画可能な座標へ変換する。
  * UV、Normal、Color などを Fragment Shader 側へ渡す役割も持つ。

### Primitive Assembly

* **Primitive**
  * 点、線、三角形など、Rasterizer が処理する図形単位。
* 一般的な3D Meshでは、複数の頂点を組み合わせて Triangle を構成する。

### Clipping

* **Clipping**
  * Camera の視錐台から完全に外れている部分を描画対象から除外したり、境界をまたぐ Primitive を切り詰めたりする処理。
* Clip Space では、`w` を利用した範囲判定を Perspective Divide より前に行える。
* Vertex Shader の位置出力を、自分で先に `w` で割ってしまわないことが重要。

### Backface Culling

* **Backface Culling**
  * Camera から見て裏向きの Triangle を描画対象から除外する処理。
  * 閉じた3Dモデルでは裏面を描く必要がないことが多いため、描画コストの削減に利用される。
* 表裏の判定には頂点の並び順を使う。Normal の向きで判定するわけではない。
* Culling、Clipping、Perspective Divide などは Rasterizer Stage に含まれる固定機能であり、GPU/API内部の細かな実行順を固定的に暗記する必要はない。

### Rasterization

* **Rasterization（ラスタライズ）**
  * Triangle などの Primitive を、画面上の Fragment に変換する処理。
* Vertex Shader から渡した値は、Triangle 内部で補間される。

### Perspective-correct Interpolation

* UV や Color などの Varying は、通常 **Perspective-correct（透視補正付き）** で補間される。
* 画面上で単純な線形補間をすると、奥行きのある面では Texture が不自然に歪む。
* GPU は Homogeneous Coordinate の `w` を利用して、この歪みを補正する。
* 補間せず一定の値を渡す方法や、画面上で線形補間する方法もある。言語ごとの指定方法は第9〜11章を参照する。

### Fragment Shader

* **Fragment Shader（フラグメントシェーダー）**
  * Rasterization によって生成された Fragment ごとに実行される Shader。
  * Texture、色、光などを使って最終的な出力色などを計算する。
* DirectX / HLSL では **Pixel Shader** と呼ばれる。

### Fragment と Pixel の違い

* **Fragment**
  * 「画面上の Pixel へ書き込まれる候補」。
* **Pixel**
  * Depth / Stencil Test や Blending などを経て、最終的に Render Target に保存される画素。
* 初学者の段階では Fragment Shader を「各 Pixel 候補の色を計算する処理」と考えてよい。


### 頂点入力とステージ間の受け渡し

```text
Mesh の頂点データ
    ↓
Vertex Shader
    ↓ 位置出力と UV・Normal など
Rasterization / 補間
    ↓
Fragment Shader
```

* 頂点データの入力と、Shader 間で値を受け渡す仕組みは共通する。具体的な宣言は HLSL の Semantic や GLSL の `in` / `out` で表す。

---

## 3. Shader で扱う基本データ

### Position

* 頂点の位置。
* Vertex Shader で最も基本となる入力。

### UV

* Texture 上のどこを参照するかを表す2次元座標。
* 通常の正規化 UV では、Texture 全体を表す基本範囲は各軸 `0.0 ～ 1.0`。
* UV は画面の Pixel 座標とは別の座標。画像の上下方向や原点の扱いは API・画像の読み込み方・描画経路に依存し、HLSL / GLSL という言語だけでは決まらない。
* UV を持つ Mesh では、各頂点の UV が三角形内部で補間される。
* UV 自体は `0.0 ～ 1.0` の範囲外の値も取れる。範囲外の読み取り方は Wrap Mode によって決まり、`Repeat` の場合は Texture の繰り返しに利用できる。

### Normal

* 表面がどちらを向いているかを表す方向ベクトル。
* Normal を持つ Mesh では、頂点ごとに Vertex Normal が保持される。滑らかな陰影を表現するため、Triangle 面そのものの法線とは異なる向きに設定されることもある。
* Fragment Shader では、頂点間で補間された Normal を利用することも多い。補間後は長さが1とは限らないため、ライティング計算前に必要に応じて正規化する。
* 主にライティング計算で使用する。
* Position と Normal は、同じ方法で座標変換できるとは限らない。
  * Position は Model Matrix で変換する。
  * Normal は、非一様スケール（X/Y/Z が異なる倍率）を含む場合、面に垂直な性質を保つため **Model Matrix の逆行列の転置（Inverse Transpose）** に相当する変換が必要。


### Color

* 一般的に以下の4成分で表す。
  * `R` : Red
  * `G` : Green
  * `B` : Blue
  * `A` : Alpha
* 通常の色や Alpha は `0.0 ～ 1.0` が基本だが、型がこの範囲に制限するわけではない。HDR の RGB では `1.0` を超える値も使う。

### Texture

* 色・Normal・深度・マスクなどを格納するデータ。
* UV を使用して Texture の特定位置の色を読み取る。

### Sampler

* Texture をどのように読み取るかを定義するもの。
* Filter や Wrap など、Texture Sampling の方法に関係する。

---

## 4. 座標空間

### Object Space

* **Object Space（Local Space）**
  * 各 Model 自身を基準とする座標。
* Mesh に保存されている頂点位置は基本的に Object Space。

### World Space

* **World Space**
  * Scene 全体を基準とする座標。
* Model Matrix によって、各 Model の位置を Scene 全体の座標へ変換する。

### View Space

* **View Space**
  * Camera を基準とする座標空間。

### Homogeneous Clip Space

* **Homogeneous Clip Space**
  * View Space の位置へ Projection Matrix を適用した後の4次元座標 `(x, y, z, w)`。
  * Rasterizer へ渡す Vertex Shader の位置出力は、この段階の値。
* Perspective Projection では「遠いものほど小さく見える」ために、最終的に `x / w`、`y / w` のような除算が必要になる。
* Matrix の乗算だけでは Perspective Divide そのものを表現できないため、Projection Matrix は後で割るための情報を `w` に保持する。

```text
Clip Space : (x, y, z, w)
                 ↓ Perspective Divide
NDC        : (x/w, y/w, z/w)
```

### なぜ Clipping は Perspective Divide より前なのか

* Clip Space では、例えば `x` が `-w ～ +w` の範囲にあるかを利用して視錐台の内外を判定できる。
* Camera の背後などでは `w <= 0` になる場合があり、先に `w` で割ると座標が大きく反転するなど不都合が起こり得る。
* そのため Vertex Shader の位置出力は Clip Space のまま渡し、自分で先に `w` で割らない。

### NDC

* **NDC (Normalized Device Coordinates)**
  * Clip Space の座標に Perspective Divide を行った後の座標。
* `x` / `y` は正規化された画面範囲を表す。
* `z` の範囲や向きは Graphics API / Platform によって異なるため、固定値として思い込まない。

### Screen / Window Space

* NDC を Viewport の大きさへ変換すると、画面上の Pixel 座標に対応する位置になる。
* Fragment Shader で利用できる組み込みの画面位置は、Vertex Shader が出力した Clip Space 値そのものではない。
* 画面原点や深度範囲は API・設定に依存する。座標変換の原理と言語の記法を分けて考える。

### 座標変換の基本

```text
Object Space
    ↓ Model
World Space
    ↓ View
View Space
    ↓ Projection
Homogeneous Clip Space
    ↓ Perspective Divide
NDC
    ↓ Viewport Transform
Screen / Window Space
```

* Shader を読む際は「現在の値がどの座標空間なのか」を意識することが重要。


### Reversed-Z の考え方

* 深度を Near 側 `1`、Far 側 `0` として扱う方式。浮動小数点 Depth Buffer の精度を活用できる。
* 採用するかどうかは Engine / API の設定による。Unity での判定方法は第14章を参照する。

---

## 5. 数学・関数（HLSL / GLSL 共通）

概念と計算は共通する。以下は通常の有限な浮動小数点値を扱う場合の対応で、NaN などの特殊値や全オーバーロードまで同一とは限らない。

### Vector と成分の取り出し

| 用途 | HLSL | GLSL |
|---|---|---|
| 1つの数値 | `float` | `float` |
| UV などの2成分 | `float2` | `vec2` |
| Position / Normal などの3成分 | `float3` | `vec3` |
| 同次座標 / RGBA などの4成分 | `float4` | `vec4` |

* ベクトルは複数の数値をまとめたもの。型自体に「色専用」などの制約はない。
* 両言語とも `.xy` / `.rgb` などの swizzle で成分を取り出せる。`.yx` のように並べ替えたり、`.xxx` のように同じ成分を繰り返したりできる。

### 数学関数の対応表

| 概念 | HLSL | GLSL |
|---|---|---|
| 内積 | `dot(a, b)` | `dot(a, b)` |
| 正規化 | `normalize(v)` | `normalize(v)` |
| 線形補間 | `lerp(a, b, t)` | `mix(a, b, t)` |
| 範囲制限 | `clamp(x, low, high)` | `clamp(x, low, high)` |
| 0〜1 に制限 | `saturate(x)` | `clamp(x, 0.0, 1.0)` |
| 切り下げ | `floor(x)` | `floor(x)` |
| 小数部分 | `frac(x)` | `fract(x)` |
| 閾値による切り替え | `step(edge, x)` | `step(edge, x)` |
| 滑らかな閾値 | `smoothstep(low, high, x)` | `smoothstep(low, high, x)` |
| 三角関数 | `sin(x)` / `cos(x)` | `sin(x)` / `cos(x)` |

### 内積

* 各成分同士の積を足す。3成分なら `a.x*b.x + a.y*b.y + a.z*b.z`。
* 両方が単位ベクトルなら、同じ向きで `1`、直交で `0`、反対向きで `-1`。正規化されていない場合は長さにも影響される。
* Lighting では、同じ座標空間の Normal と光への方向の内積などに使用する。

### 正規化

* ベクトルをその長さで割り、長さを1にする。方向だけを扱いたい場合に使う。
* ゼロベクトルは正規化できない。結果に依存せず、事前に分岐して代わりの値を使うなどの対処が必要。

### 線形補間

* 計算は `a + t * (b - a)`。`t = 0` で `a`、`t = 1` で `b`。
* `t` は自動で `0 ～ 1` に制限されず、範囲外では外挿になる。必要なら先に `t` を範囲制限する。
* 対応表の GLSL `mix` は、割合 `t` に浮動小数点値を渡す版。真偽値で値を選択する別の版もある。

### 範囲制限

* 下限未満を下限へ、上限超過を上限へそろえる。下限は上限以下にする。
* `0 ～ 1` に収める操作は、HLSL の `saturate`、GLSL の `clamp` で表現できる。

### 小数部分

* `x - floor(x)` を返す。`floor(x)` は `x` 以下の最大の整数。
* 結果は数学的には `0` 以上 `1` 未満。例: `2.3 → 0.3`、`5.0 → 0.0`、`-1.2 → 0.8`（浮動小数点では丸め誤差がある）。
* 負数では「小数点より右を取り出す」動作とは異なる。UV の繰り返しパターンなどに利用できる。

### 閾値による切り替え

* `step(edge, x)` は `x < edge` なら `0`、それ以外なら `1`。閾値と等しい場合は `1`。
* マスクや境界を作る際に使う。

### 滑らかな閾値

* `low < high` として使う。下限以下で `0`、上限以上で `1`、その間を滑らかな Hermite 曲線で補間する。
* 両言語での考え方は次のとおり（疑似コード）。

```text
t = (x - low) / (high - low) を 0〜1 に制限
結果 = t * t * (3 - 2 * t)
```

* 両端で傾きが0になるため、単純な線形変化より境界が滑らかになる。
* GLSL では `low >= high` の結果は未定義。共通の書き方として、上下限を等しくしたり逆転させたりしない。

### 三角関数

* `sin` / `cos` の角度はラジアン。出力範囲は `-1 ～ 1`、周期は `2π`。
* 波、点滅、揺れなどに利用できる。`sin(x) * 0.5 + 0.5` で `0 ～ 1` に変換できる。

### Matrix と座標変換

* 位置などを行列で変換する考え方は共通。列ベクトルとして扱う例では、`Projection × View × Model × Position` の順で表せる。
* 同じ数学的な行列・列ベクトルを使う場合、HLSL の行列積は `mul(M, v)`、GLSL は `M * v`。
* HLSL の行列同士の `*` は成分ごとの積で、GLSL の行列同士の `*` は行列積。記号だけをそのまま移植しない。
* メモリ上の行列の並び方と、数式上の乗算順は別の問題。CPU 側から渡す形式もそろえる。

---

## 6. Texture・UV・Sampling

### Texture Sampling の考え方

```text
UV
 ↓
Texture上の位置を指定
 ↓
その位置の色を取得
 ↓
Fragment Shaderから出力
```

### UV の可視化

```text
R = UV.x, G = UV.y, B = 0, A = 1
```

| 言語 | 色を作る式（出力への代入は各言語章を参照） |
|---|---|
| HLSL | `float4(uv.x, uv.y, 0.0, 1.0)` |
| GLSL | `vec4(uv.x, uv.y, 0.0, 1.0)` |

* UV.x を Red。
* UV.y を Green。
* UV の値をそのまま色として見ることで、UV の配置を視覚的に確認できる。

### Tiling

* UV に掛ける倍率を変更する。
* Wrap Mode が `Repeat` の場合、値を大きくすると Texture が細かく繰り返される。
* `Clamp` の場合は繰り返されず、`0 ～ 1` の範囲外で Texture の端の値が引き延ばされる。

### Offset

* UV 全体をずらす。
* Texture のスクロール表現などにも利用できる。

### Filter とミップマップ

* Sampling は必ずしも1つの Texel（Texture の画素）を読む処理ではない。Filter によって周辺の値が補間される。
* ミップマップは段階的に縮小した Texture 群。画面上で小さく見える場合などに使い、ちらつきを抑える。
* Fragment Shader では UV の画面上の変化量を使い、ミップレベルを自動選択する Sampling が一般的。頂点段階などでは、ミップレベルを明示する読み取りを使う。
* Texture / Sampler の具体的な宣言と呼び出しは第9・10・14章に分ける。

---

## 7. 描画状態・深度・透明表現

描画状態は HLSL / GLSL の共通概念だが、多くは Shader の外側の Graphics API / Engine で設定する。ShaderLab の指定方法は第12章を参照する。

### Face Culling

* 頂点の並び順によって表裏を判定し、指定した側の Triangle を除外する。Normal の向きとは別。
* 裏面を除外する、表面を除外する、両面を描く、といった選択がある。両面描画は Fragment の処理量が増える可能性がある。

### Depth / Stencil Test

* Depth Test は Fragment の深度と Depth Buffer の値を比較し、通過させるかを決める。
* 「小さいほうが手前」とは限らない。投影・Reversed-Z・比較方法・クリア値をそろえる必要がある。
* Stencil Test は Stencil Buffer の整数値と参照値を比較し、マスクなどとして描画範囲を制御する。

### Depth Write

* Depth Buffer を更新するかを指定する。深度を比較する処理と、書き込む処理は別。
* Opaque は通常書き込みあり、一般的な半透明では書き込みなしにすることが多い。

### Blending

* 新しい色と描画先にある色を合成する。通常の非乗算済み Alpha の RGB 合成は `新しいRGB * Alpha + 既存RGB * (1 - Alpha)`。
* Alpha を `0.5` として出力するだけでは、自動的に半透明にはならない。Blending の有効化と合成係数の設定が必要。
* 深度書き込みや描画順も見た目に影響する。通常の半透明合成は順序に依存するので、奥から手前へ描くなどの制御が必要。

### Alpha Test / Cutout

* Alpha が閾値未満なら Fragment を破棄する。草や柵などを「描く / 描かない」に分ける表現。
* 通常の Alpha Blend のような連続した半透明ではなく、基本的には二値的な表示になる。
* HLSL では `clip`、GLSL では条件分岐と `discard` で実装できる。具体例は各言語章を参照する。

### 【発展】Early Depth / Early-Z

* Fragment Shader より先に深度などを判定し、隠れた部分の Shader 実行を省略できる場合がある。
* Shader が深度を書き換えたりメモリ書き込みの副作用を持ったりすると、最適化が制約される。
* 深度出力の変化方向を宣言する Conservative Depth など、最適化を維持する仕組みもある。
* Fragment の破棄や Blending を使うだけで必ず無効になるとは限らない。実際の挙動は GPU / API / 描画状態によるため、必要なら Profiler で確認する。

---

## 8. Shading の基本：Unlit / Lit

### Unlit とは

* **Unlit Shader**
  * 基本的にライトの影響を受けずに描画する Shader。
* 自分で指定した色や Texture をライティング計算なしで出力するため、Shader の基礎を学びやすい。
* Unlit でも後続の露出調整やトーンマッピングなどによって、最終表示は変化し得る。

### Lit Shader との違い

* **Unlit**
  * ライティング計算を基本的に行わない。
* **Lit**
  * Light、Normal、Material特性などを使用して明るさや反射を計算する。

### Unlit が学習に向いている理由

* Vertex Shader と Fragment Shader の基本構造に集中できる。
* Lighting の計算を考えずに、UV・Texture・Colorなどを確認できる。

---

## 9. HLSL

### HLSL とは

* **HLSL (High-Level Shading Language)**
  * Microsoft が開発した C 言語風のシェーダー言語。
  * 主に DirectX / Direct3D の Shader を記述するために使用される。
* Unity 専用の言語ではない。
* Unity でも Shader プログラムを記述するために使用される。

### 基本的な型

```hlsl
float value;
float2 uv;
float3 position;
float4 color;
```

* `float`
  * 1つの浮動小数点数。
* `float2`
  * 2成分ベクトル。
* `float3`
  * 3成分ベクトル。
* `float4`
  * 4成分ベクトル。

### swizzle

```hlsl
float4 value = float4(1.0, 2.0, 3.0, 4.0);

float first = value.x;       // 1.0
float2 pair = value.xy;      // (1.0, 2.0)
float3 position = value.xyz; // (1.0, 2.0, 3.0)
float3 rgb = value.rgb;      // value.xyz と同じ
float4 rgba = value.rgba;    // value.xyzw と同じ
```

* Vector の一部の成分を取り出す記法。
* `x, y, z, w` と `r, g, b, a` は用途に応じて使い分ける。

### 関数

```hlsl
float4 Example(float2 uv)
{
    return float4(uv.x, uv.y, 0.0, 1.0);
}
```

* C / C++ と似た形式で関数を記述できる。

### Semantic

```hlsl
// Mesh から Vertex Shader への入力。
struct VertexInput
{
    float4 positionOS : POSITION;
    float2 uv : TEXCOORD0;
};

// Vertex Shader からの出力。
struct VertexOutput
{
    float4 positionHCS : SV_POSITION;
    float2 uv : TEXCOORD0;
};
```

* **Semantic（セマンティクス）**
  * その変数がグラフィックスパイプライン上で何を意味するかを指定する。
* 代表例:
  * `POSITION`
  * `NORMAL`
  * `TEXCOORD0`
  * `SV_POSITION`
  * `SV_Target`

### `SV_POSITION` は Shader Stage によって意味が変わる

* Vertex Shader の出力としての `SV_POSITION`
  * Rasterizer が処理するための Homogeneous Clip Space の位置を渡す。
* Fragment / Pixel Shader の入力としての `SV_POSITION`
  * Vertex Shader が返した Clip Space `(x, y, z, w)` をそのまま受け取るわけではない。
  * Direct3D 10+ の HLSL では、`xy` は Render Target 上の Screen Space 座標（Pixel Center の `0.5` Offset を含む）。
  * `z` は Perspective Divide 後に Viewport の深度範囲へ変換された値。深度範囲が `0 ～ 1` の通常の Direct3D 設定では `z / w` に相当する。
  * `w` は元の Homogeneous Coordinate の `w` の逆数 (`1 / w`)。
* Unity は複数の Graphics API へ Shader を変換するため、Platform 差は意識する必要があるが、少なくとも「Fragment 入力の `SV_POSITION` = 元の Clip Space」と考えないことが重要。

### Interpolation Modifier

* Vertex Shader から Fragment Shader へ渡す値は、通常は Perspective Correction 付きで補間される。
* HLSL では必要に応じて補間方法を指定できる。
  * `nointerpolation` : 値を補間しない。
  * `noperspective` : Perspective Correction を行わない。
  * `centroid` : MSAA 時などに Covered Area 内の位置を使う。
  * `sample` : Sample 単位で補間・Shader 実行を行う。

### Texture / Sampler（Direct3D の HLSL）

```hlsl
Texture2D baseMap;
SamplerState baseSampler;

float4 SampleColor(float2 uv)
{
    return baseMap.Sample(baseSampler, uv);
}
```

* Texture と Sampling の設定を別に宣言し、アプリケーション側で対応するリソースを結び付ける。
* 上の例は Fragment / Pixel Shader から呼び出す想定。明示的なミップレベルで読む場合は `baseMap.SampleLevel(baseSampler, uv, lod)` を使う。
* Unity / URP の `TEXTURE2D` / `SAMPLER` / `SAMPLE_TEXTURE2D` は第14章のマクロ。

### Fragment 出力と破棄

```hlsl
float4 frag(VertexOutput IN) : SV_Target
{
    return float4(IN.uv, 0.0, 1.0);
}
```

* `SV_Target` は描画先への色出力を示す。普通のローカル変数には Semantic は不要。
* Cutout の例は `clip(alpha - cutoff);`。引数が負なら破棄し、等しい場合は残す。ベクトル引数ならいずれかの成分が負のとき破棄する。

### 【発展】Direct3D の深度出力

* 深度を書き換えず、UAV 書き込みなどの副作用がない場合、Hardware は Early Depth Test を最適化として行える。
* 通常の `SV_Depth` 出力は、Shader 実行前に最終深度が分からないため、一般的な Early-Z 最適化を妨げる。
* `SV_DepthGreaterEqual` / `SV_DepthLessEqual` は変化方向を制限し、最適化を維持できる場合がある。
* 破棄や Blending を含めた共通の注意点は第7章を参照する。

---

## 10. GLSL

### GLSL とは

* GLSL（OpenGL Shading Language）は C 言語風の Shader 言語。ここでは **OpenGL 向け GLSL 3.30 Core** を基準に基本例を書く。
* GLSL ES や Vulkan 向け GLSL では、バージョン指定・精度指定・リソースの結び付け方などが異なる。この例をそのまま共通には使わない。
* 数学関数の対応は第5章を参照する。

### 基本データ型と swizzle

```glsl
float value = 1.0;
vec2 uv = vec2(0.0, 1.0);
vec3 position = vec3(0.0, 0.0, 0.0);
vec4 color = vec4(1.0, 0.5, 0.0, 1.0);
vec3 rgb = color.rgb;
```

* 行列は `mat3` / `mat4` など。`half` はここで使う標準の数値型ではない。

### Vertex Shader と座標変換

```glsl
#version 330 core
layout(location = 0) in vec3 positionOS;
layout(location = 1) in vec2 uv;
uniform mat4 modelViewProjection;
out vec2 uvToFragment;

void main()
{
    gl_Position = modelViewProjection * vec4(positionOS, 1.0);
    uvToFragment = uv;
}
```

* `main` が実行開始点。`gl_Position` に同次 Clip Space の位置を代入する。
* 行列はアプリケーション側から渡す。頂点の入力 location と頂点バッファの設定も一致させる。

### Fragment Shader と Texture / Sampler

```glsl
#version 330 core
in vec2 uvToFragment;
uniform sampler2D baseMap;
layout(location = 0) out vec4 outColor;

void main()
{
    outColor = texture(baseMap, uvToFragment);
}
```

* 上の Vertex Shader と組み合わせる例。`uvToFragment` は補間された入力。
* `sampler2D` は2D Texture の Sampling に使う型。OpenGL 側で Texture Unit を設定し、Texture と Sampling 設定を結び付ける。
* ミップレベルを明示する場合は `textureLod(baseMap, uv, lod)`。UV を可視化するなら `outColor = vec4(uvToFragment, 0.0, 1.0);`。
* Shader だけでは描画は完結しない。アプリケーション側のコンパイル・リンク、頂点データ、行列、Texture の設定が必要。

### in / out と layout(location)

* 頂点段階の `in` は頂点入力、頂点段階の `out` とフラグメント段階の `in` はステージ間の受け渡し。
* 上の GLSL 3.30 例では、ステージ間の名前と型を一致させてリンクする。
* 頂点入力の `location = 0` は頂点属性の番号。Fragment 出力の `location = 0` は色出力先の番号。別の用途の番号であり、互いを接続するものではない。

### uniform

* アプリケーション側から渡す行列やパラメータなどを宣言する。Shader 内から書き換えない。
* 頂点ごとの入力や、頂点間で補間される値とは異なる。

### 補間と Fragment の位置

* 通常の浮動小数点の受け渡しは `smooth`（透視補正付き、既定値）。`flat` は補間なし、`noperspective` は画面上の線形補間。
* `centroid` は MSAA 時などに覆われた領域内の補間位置を使う。整数の Fragment 入力には `flat` が必要。
* `gl_FragCoord` は描画先の位置であり、`gl_Position` がそのまま届く変数ではない。
* OpenGL の既定では画面位置の原点は左下、Pixel Center は半整数。原点などを変更できる指定もあるため、常に固定とは考えない。

### Fragment の破棄

```glsl
// Fragment Shader の関数内で使用する。
if (alpha < cutoff)
{
    discard;
}
```

* HLSL のスカラー版 `clip(alpha - cutoff)` に対応する考え方。GLSL の `discard` 自体は条件を受け取らない。
* 深度を自分で出力する場合は `gl_FragDepth` を使う。Early-Z への影響は第7章を参照する。

---

## 11. HLSL と GLSL の比較

数学関数は第5章、言語別の具体例は第9・10章を参照する。GLSL 欄は OpenGL 向けを基本とする。

| 概念 | HLSL | GLSL |
|---|---|---|
| 2 / 3 / 4成分 | `float2` / `float3` / `float4` | `vec2` / `vec3` / `vec4` |
| 4×4行列 | `float4x4` | `mat4` |
| 行列×列ベクトル | `mul(M, v)` | `M * v` |
| Vertex 入力 | `POSITION` などの Semantic と入力レイアウト | `in` と頂点属性の location |
| ステージ間の受け渡し | `TEXCOORD0` などの Semantic | `out` / `in`（名前・型などを対応させる） |
| Vertex 出力位置 | `SV_POSITION` | `gl_Position` |
| Fragment 入力位置 | `SV_POSITION` | `gl_FragCoord` |
| Fragment 色出力 | `SV_Target` | `out vec4` と出力 location |
| Fragment 深度出力 | `SV_Depth` | `gl_FragDepth` |
| 補間なし | `nointerpolation` | `flat` |
| 透視補正なし | `noperspective` | `noperspective` |
| Texture Sampling | `Texture2D` + `SamplerState`、`.Sample(...)` | `sampler2D`、`texture(...)` |
| 外部の数値パラメータ | `cbuffer` など | `uniform` / uniform block |
| 条件付き Fragment 破棄 | `clip(x)` | `if (x < 0.0) discard;`（スカラーの場合） |

* `SV_` で始まる Semantic と GLSL の組み込み変数は特別な役割を持つ。通常の変数名を似せるだけでは代用できない。
* `TEXCOORD0` はステージ間では UV 専用ではなく、他の値も渡せる。
* 表は役割の対応であり、API の座標規約やメモリ配置まで同一という意味ではない。
* 描画状態・リソースの設定は API / Engine の仕事。Shader 言語を変えたことだけで決まるわけではない。

---

## 12. Unity の ShaderLab

### ShaderLab とは

* **ShaderLab**
  * Unity 独自の Shader 定義用言語。
* HLSL そのものとは別物。
* Unity の `.shader` ファイルでは、ShaderLab の構造の中に HLSL の Shader プログラムを記述する。

### 基本構造

```shaderlab
Shader "Example/MyShader"
{
    Properties
    {
    }

    SubShader
    {
        Pass
        {
            HLSLPROGRAM

            // HLSL

            ENDHLSL
        }
    }
}
```

* 上の例は構造のみ。描画には頂点・フラグメント関数と、その指定などが必要。

### Shader

```shaderlab
Shader "Example/MyShader"
```

* Shader の登録名と選択メニューの階層を定義する。`.shader` ファイル名とは別。
* Material の Shader 選択画面などで使用される。

### Properties

```shaderlab
Properties
{
    _BaseColor("Base Color", Color) = (1, 1, 1, 1)
}
```

* Material Inspector から変更可能な値を定義する。
* Color、Float、Texture などを公開できる。

### SubShader

* 実際の描画処理をまとめる領域。
* Render Pipeline やハードウェアなどに応じて複数用意できる。

### Pass

* 1回分の描画処理を定義する。
* Vertex Shader / Fragment Shader や、描画状態の設定などを記述する。

### HLSLPROGRAM / ENDHLSL

```shaderlab
HLSLPROGRAM

// HLSLコード

ENDHLSL
```

* ShaderLab の中で HLSL を記述する範囲を示す。

### pragma

```hlsl
#pragma vertex vert
#pragma fragment frag
```

* Shader Compiler へ情報を渡す Directive。
* `#pragma vertex`
  * Vertex Shader として使用する関数を指定する。
* `#pragma fragment`
  * Fragment Shader として使用する関数を指定する。

### 必要に応じて使用する pragma

#### `#pragma target`

```hlsl
#pragma target 4.5
```

* Shader が必要とする Shader Model / GPU 機能レベルを指定する。
* **`4.5` が常に必要という意味ではない。**
* Unity では指定しなければ既定の Target が使用されるため、使用する機能や対象 Platform に応じて指定する。

#### `#pragma multi_compile` / `#pragma shader_feature`

```hlsl
#pragma multi_compile _ FEATURE_A
#pragma shader_feature _ FEATURE_B
```

* Keyword の組み合わせごとに **Shader Variant** を生成する。
* `multi_compile`
  * 指定した組み合わせの Variant を基本的にすべて用意する。
  * Runtime で Keyword を切り替える可能性がある機能などに使用される。
* `shader_feature`
  * Material 等で実際に使用されていない Variant を Build から除外できるため、Material 固有の Feature に向く。
* Keyword を増やすほど Variant の組み合わせ数が急増するため、Compile Time や Build Size に影響する。

#### `#pragma multi_compile_instancing`

```hlsl
#pragma multi_compile_instancing
```

* GPU Instancing 用の Shader Variant を生成する。
* **すべての Shader に必須ではない。**
* GPU Instancing を自作 Shader で利用する場合に、Instance ID 用 Macro などと合わせて使用する。

### 描画状態の指定

共通概念は第7章を参照する。以下は ShaderLab 固有の指定。`Cull` や `ZWrite` の複数行の例は選択肢であり、同時にすべて指定する例ではない。

#### Cull

```shaderlab
Cull Back
Cull Front
Cull Off
```

* Triangle のどちら側を描画しないかを指定する。
* `Cull Back`
  * 裏面を描画しない。一般的な3D Meshで標準的。
* `Cull Front`
  * 表面を描画しない。
* `Cull Off`
  * 両面を描画する。
* 両面描画は便利だが、描画する Fragment が増える可能性がある。

#### ZTest

```shaderlab
ZTest LEqual
```

* 現在の Fragment の Depth と、Depth Buffer に保存されている値を比較して描画可否を決める。
* `LEqual`、`Less`、`Greater`、`Always` などを指定できる。
* ShaderLab の指定は Unity が Platform 向けに処理するため、Reversed-Z 環境の生の Depth 値と単純に同一視しない。

#### ZWrite

```shaderlab
ZWrite On
ZWrite Off
```

* 描画した Fragment の Depth を Depth Buffer に書き込むかを指定する。
* Opaque では通常 `On`。
* 半透明描画では通常 `Off` がよく使用されるが、目的によって異なる。

#### Blend

```shaderlab
Blend SrcAlpha OneMinusSrcAlpha
```

* Shader が出力した色と、Render Target に既に存在する色をどのように合成するかを指定する。
* 一般的な Alpha Blend では `SrcAlpha` と `OneMinusSrcAlpha` を使用する。

#### Alpha を下げるだけでは透明にならない

```hlsl
return half4(1.0, 0.0, 0.0, 0.5);
```

* Fragment Shader が Alpha `0.5` を返しただけで、自動的に半透明になるわけではない。
* 半透明にするには `Blend`、Render Queue / Tags、`ZWrite` などの Render State も適切に設定する必要がある。

#### Render Queue と分類

* SubShader の `Tags { "Queue" = "Transparent" "RenderType" = "Transparent" }` は、半透明の描画順分類と種類を指定する。
* `RenderType` を変えるだけでは Blending は有効にならない。上記 Tags と `Blend` / `ZWrite` を目的に合わせて設定する。

---

## 13. Unity の Render Pipeline

### Render Pipeline とは

* **Render Pipeline**
  * Unity が「何を・どの順番で・どのように描画するか」を管理する仕組み。
* 使用する Render Pipeline によって利用できる Shader の機能やライブラリが異なる。

### Built-in Render Pipeline

* Unity で古くから使用されている標準の Render Pipeline。
* URP / HDRP とは Shader の書き方や利用するライブラリが異なる部分がある。

### URP

* **URP (Universal Render Pipeline)**
  * Unity の Scriptable Render Pipeline の1つ。
  * 幅広いプラットフォームを対象とした Render Pipeline。

### HDRP

* **HDRP (High Definition Render Pipeline)**
  * 高品質なグラフィックスを目的とした Render Pipeline。

### Render Pipeline と Shader の互換性

* Built-in 用の Shader が、そのまま URP で使用できるとは限らない。
* URP の Shader を書く場合は、URP 用の Tags、ライブラリ、関数などを使用する。

---

## 14. URP の Shader

### URP と HLSL の関係

* URP はシェーダー言語そのものではない。
* URP のカスタム Shader では主に HLSL を使用する。
* URP が提供する HLSL の関数・マクロ・定義を `#include` して利用する。

### Core.hlsl

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
```

* URP の Shader で基本となる HLSL ライブラリ。
* 座標変換などで使用する関数・マクロが含まれている。

### URP / SRP Core の `real` 型

* SRP Core の Shader Library では `real` / `real2` / `real3` / `real4` という Alias が使用される。
* 一般的には Mobile Platform では `half` 系、Desktop Platform では `float` 系として定義される。
* HLSL 標準の独立した基本型というより、Unity の SRP Shader Library 側で用意される「Platform に応じた精度選択用の Alias」と考える。

### URP の Tags

```shaderlab
Tags
{
    "RenderType" = "Opaque"
    "RenderPipeline" = "UniversalPipeline"
}
```

* `RenderPipeline`
  * どの Render Pipeline 用の SubShader なのかを示す。
* `RenderType`
  * Opaque / Transparent など描画上の分類に利用される。

### Attributes

```hlsl
struct Attributes
{
    float4 positionOS : POSITION;
    float2 uv : TEXCOORD0;
};
```

* Mesh から Vertex Shader へ入力されるデータをまとめた構造体。
* `Attributes` という名前自体は決まりではなく、Unity の例でよく使用される命名。

### Varyings

```hlsl
struct Varyings
{
    float4 positionHCS : SV_POSITION;
    float2 uv : TEXCOORD0;
};
```

* Vertex Shader から Fragment Shader へ渡す値をまとめた構造体。
* 頂点間の値はラスタライズ時に補間される。
* `Varyings` という名前自体も慣例的な命名。

### positionOS

```hlsl
float4 positionOS : POSITION;
```

* `OS`
  * Object Space を表す命名。
* Mesh から受け取った頂点位置。

### positionHCS

```hlsl
float4 positionHCS : SV_POSITION;
```

* `HCS`
  * Homogeneous Clip Space を表す Unity / SRP の命名。
* Vertex Shader の出力時点では Homogeneous Clip Space の位置。
* 同じ `Varyings` を Fragment Shader の入力として受け取った時点では、Rasterizer を通過した後の `SV_POSITION` なので、変数名が `positionHCS` でも「元の Clip Space 値そのもの」ではない。

### Object Space → Clip Space

```hlsl
OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
```

* URP の関数を使用して Object Space の頂点位置を Clip Space へ変換する。

### Material Property と CBUFFER

```shaderlab
Properties
{
    _BaseColor("Base Color", Color) = (1, 1, 1, 1)
}
```

```hlsl
CBUFFER_START(UnityPerMaterial)
    half4 _BaseColor;
CBUFFER_END
```

* `Properties`
  * Material Inspector に公開する値。
* HLSL 側でも対応する変数を宣言する必要がある。
* SRP Batcher と互換性を持たせる Custom Shader では、Material ごとの数値 Property を **`UnityPerMaterial` という単一の CBUFFER** にまとめる。
* オブジェクトの変換行列など、描画オブジェクトごとの Built-in Engine Property は `UnityPerDraw` 側で管理される。カメラやフレーム単位の値まで、すべてここに入るわけではない。
* Texture / Sampler Object は定数バッファへ入れず、`TEXTURE2D` / `SAMPLER` などで CBUFFER の外に宣言する。
* `_BaseMap_ST` のような Tiling / Offset 用 `float4` は数値データなので `UnityPerMaterial` の中に置く。
* 複数 Pass を持つ Shader では、Pass ごとに Material Property の CBUFFER 構成を食い違わせない設計にするのが安全。
* Shader Inspector で SRP Batcher compatibility を確認できる。

### Texture

```hlsl
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);
```

* `TEXTURE2D`
  * Texture を宣言する URP / SRP のマクロ。
* `SAMPLER`
  * Sampler を宣言するマクロ。

### Texture Sampling

```hlsl
half4 color =
    SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
```

* Fragment Shader 内で、UV 位置に対応する Texture の値を取得する。
* 頂点段階などでミップレベルを明示して読む場合は `SAMPLE_TEXTURE2D_LOD` を使う。

### Texture の Tiling / Offset

```hlsl
float4 _BaseMap_ST;
```

```hlsl
OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
```

* `_ST`
  * Texture の Tiling / Offset 情報を格納するUnity側の命名規則。
* `TRANSFORM_TEX`
  * UV へ Tiling / Offset を適用するマクロ。
* `_BaseMap_ST.xy` が Tiling、`.zw` が Offset。計算は `IN.uv * _BaseMap_ST.xy + _BaseMap_ST.zw` に相当する。

### Normal の座標変換

* URP の `TransformObjectToWorldNormal()` は第3章で説明した非一様スケールの影響を考慮して Object Space の Normal を World Space へ変換する。
  * `UNITY_ASSUME_UNIFORM_SCALING` が有効な場合は、Uniform Scale を前提により単純な方向ベクトル変換を使用する。

### half

```hlsl
half4 color;
```

* `half`
  * Unity の HLSL では中精度の浮動小数点型として扱われる。
  * 対応 Platform では一般に16bit相当だが、Platform によっては `float` 相当として扱われる。
* 色、短い方向ベクトルなど、高精度を必要としない値に向いている。
* `half` と書けば必ず高速になるわけではなく、実際の精度・性能差は Hardware / Graphics API に依存する。

### 【重要】Unity と Reversed-Z

* Unity は Graphics API / Platform による Depth の違いをある程度吸収しているが、Depth Texture や Clip Space の `z` を自分で扱う場合は差異に注意する。
* DirectX 11 / 12 や Metal など、Unity が対応する多くの主要 Platform では **Reversed-Z** が使用される。
  * Near Plane 側の Depth Buffer 値 : `1.0`
  * Far Plane 側の Depth Buffer 値 : `0.0`
* ただし **Unity の全 Platform が Reversed-Z という意味ではない**。
* Shader では `UNITY_REVERSED_Z` を使って Platform 差を判定できる。
* C# 側では `SystemInfo.usesReversedZBuffer` で確認できる。
* URP / Unity が用意する Depth 変換関数や Macro が使える場合は、自前で `1 - depth` のような処理を決め打ちするより、それらを優先する。

---

## 15. デバッグとよくある問題

### 【共通】コンパイル・入出力の確認

* コンパイラのエラーを確認し、対象言語・バージョン・対応機能を確認する。
* 頂点バッファと入力宣言、ステージ間の型や受け渡しが一致しているかを確認する。
* GLSL では個別 Shader のコンパイルに加え、Program のリンク結果も確認する。

### 【Unity】Shader がピンク色になる

* Shader のコンパイルエラー。
* Render Pipeline と Shader が対応していない。
* 必要な include や記述が不足している。
* Console の Shader Error を確認する。

### 【Unity / URP】Material の値が反映されない

* `Properties` と HLSL 側の変数名が一致しているか確認する。
* HLSL 側への変数宣言があるか確認する。
* URP の場合は CBUFFER の記述を確認する。

### 【共通】Texture が表示されない

* UV が Vertex Shader から Fragment Shader まで渡されているか。
* Texture / Sampler が宣言されているか。
* Texture Sampling の引数が正しいか。
* アプリケーション / Engine 側で意図した Texture と Sampler が結び付けられているか。

### 【共通】Shader の処理を切り分ける

1. Fragment Shader から固定色を返す。
2. UV を色として表示する。
3. Texture をそのまま表示する。
4. 自分で追加した計算を1つずつ戻す。

* 複雑な計算を一度に確認せず、最も単純な状態から原因を切り分ける。

---

## 16. 用語集

* **Shader**
  * GPU上で実行される描画・計算プログラム。
* **HLSL**
  * Microsoft が開発した高水準シェーダー言語。
* **GLSL**
  * OpenGL 系で使用されるシェーダー言語。
* **ShaderLab**
  * Unity 独自の Shader 定義用言語。
* **URP**
  * Unity の Universal Render Pipeline。
* **Vertex**
  * 頂点。
* **Vertex Shader**
  * 頂点単位で実行される Shader。
* **Fragment**
  * 最終的な Pixel になる候補。
* **Fragment Shader**
  * Fragment 単位で色などを計算する Shader。
* **UV**
  * Texture 上の位置を表す2次元座標。
* **Normal**
  * 表面の向きを表すVector。Vertex Normal は Triangle 面そのものの法線とは異なる場合がある。
* **Texture**
  * Shader から参照する色・Normal・深度などのデータ。
* **Sampler**
  * Texture をどのように読み取るかを定義するもの。
* **Material**
  * Shader と、その Shader に渡すパラメータをまとめたもの。
* **Semantic**
  * HLSL で変数の用途をグラフィックスパイプラインへ伝える情報。
* **Render Pipeline**
  * 描画処理の流れを管理する仕組み。

---

## 17. 参考資料

### Unity 6.3 LTS

* Writing custom shaders in URP
  * https://docs.unity3d.com/6000.3/Documentation/Manual/urp/writing-custom-shaders-urp.html
* Examples of writing a custom shader in URP
  * https://docs.unity3d.com/6000.3/Documentation/Manual/urp/writing-shaders-urp-landing.html
* Write a basic unlit shader in URP
  * https://docs.unity3d.com/6000.3/Documentation/Manual/urp/writing-shaders-urp-basic-unlit-structure.html
* ShaderLab / HLSL / Shader compilation 関連
  * Unity Manual の ShaderLab、HLSL pragma、Shader Variant、Platform-specific rendering differences を参照する。

### Unity Graphics Repository

* SRP Core `SpaceTransforms.hlsl`
  * `TransformObjectToHClip`
  * `TransformObjectToWorldNormal`
* SRP Core [`Common.hlsl`](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl)
  * `real` precision alias の定義を確認する。
* Repository の `master` は更新されるため、使用中の URP / SRP Core Package と一致するバージョンの実装を参照する。

### Microsoft Learn / HLSL

* [Microsoft Learn — HLSL Semantics](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics)
  * `POSITION`, `SV_POSITION`, `SV_Target`, `SV_Depth` など。
* HLSL Interpolation Modifiers
  * `nointerpolation`, `noperspective`, `centroid`, `sample`。
* HLSL Intrinsic Functions
  * `clip`, `frac`, `smoothstep` など。
* Direct3D Rasterizer Stage
  * Clipping、Perspective Divide、Viewport、Rasterization の流れ。

### GLSL

* [Khronos — GLSL 4.60 仕様](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.html) — 言語仕様。第10章の基本例は GLSL 3.30 Core の範囲を使用する。
* [Khronos — GLSL 3.30 仕様](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.3.30.pdf) — 基本例の対象バージョン。
