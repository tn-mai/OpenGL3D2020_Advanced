[OpenGL 3D Advanced 2020 第02回]

# 当たるか、当たらぬか、<br>それが問題だ

To collide, or not to collide: that is the question:

## 習得目標

* 内積による射影
* 2つの線分の<ruby>最近接点<rt>さいきんせつてん</rt></ruby>の求め方
* <ruby>分離軸判定<rt>ぶんりじくはんてい</rt></ruby>のやり方

## 1. カプセル同士の衝突判定

### 1.1 球の衝突判定のおさらい

これまで、球と球、球とカプセル、球と有向境界ボックス(OBB)という、3つの衝突判定を作成しました。これらは以下の方法で判定できるのでした。

形状 | 判定方法
---|---
球-球 | 2点間の距離 < 2つの球の半径の合計
球-カプセル | 点と線分の最短距離 < 球とカプセルの半径の合計
球-OBB | 点とOBBの最短距離 < 球の半径

いずれの方法も、基本となるのは以下の2つです。

* 2つの図形間の最短距離を計算
* 2点間の距離と半径を比較

2つの球の距離は、球の中心座標の差です。2点間の距離は、点Aから点Bへ向かうベクトルの内積によって求められます。

球とカプセルの距離は、カプセルの中心をとおる線分上にある、球の中心への最近接点から求められます。

そして、球とOBBの距離は、OBBの3つの軸について、球の中心への最近接点を計算することで求められます。

衝突判定が可能な図形の組み合わせを、次の表にまとめました。

|              | 球 | カプセル | OBB
|:------------:|:--:|:--------:|:---:
|**球**       | ○ | ○       | ○
|**カプセル** | ○ | ☓       | ☓
|**OBB**      | ○ | ☓       | ☓(今回は作成しない)

本テキストでは、この表で`☓`のついている組み合わせについて衝突判定を作成していきます。

### 1.2 衝突結果を表す構造体を作る

これまで、衝突結果は「衝突の有無」と「衝突位置」という2つのパラメータで表されていました。しかし、この情報は単純すぎて、より複雑な衝突結果を表すことができません。

そこで、衝突結果を表す構造体を追加します。構造体名は`Result`(リザルト)とします。`Collision.h`を開き、`CreateOBB`関数の宣言の下に、次のプログラムを追加してください。

```diff
 Shape CreateOBB(const glm::vec3& center, const glm::vec3& axisX,
   const glm::vec3& axisY, const glm::vec3& axisZ, const glm::vec3& e);
+
+/**
+* 衝突結果を表す構造体.
+*/
+struct Result
+{
+  bool isHit = false; ///< 衝突の有無.
+  glm::vec3 pa; ///< 形状A上の衝突点.
+  glm::vec3 na; ///< 形状A上の衝突平面の法線.
+  glm::vec3 pb; ///< 形状B上の衝突点.
+  glm::vec3 nb; ///< 形状B上の衝突平面の法線.
+};

 bool TestSphereSphere(const Sphere&, const Sphere&);
 bool TestSphereCapsule(const Sphere& s, const Capsule& c, glm::vec3* p);
```

今回作成する衝突判定では、この`Result`構造体によって衝突結果を返すことにします。

<div style="page-break-after: always"></div>

## 2. カプセル同士の衝突判定

この章で扱う衝突判定

|              | 球 | カプセル | OBB
|:------------:|:--:|:--------:|:---:
|**球**       | ○ | ○       | ○
|**カプセル** | ○ | ✔       | ☓
|**OBB**      | ○ | ☓       | ☓

### 2.1 2つの直線の最近接点

「カプセルとカプセル」の衝突判定から作成しましょう。「球とカプセルの衝突判定」で見たように、カプセルの中心には「線分」が通っています。衝突判定を行う場合、この線分の上で相手の図形に最も近い点、つまり最近接点を見つけなくてはなりません。

ただし、今回は相手の図形もカプセルです。つまり、実際には「2つの線分`A`、`B`を最短距離で結ぶベクトル`V`について、その両端の座標」を見つける必要があるわけです。

ところで、線分は「無限の長さを持つ直線の長さを制限したもの」です。そこで、まずは線分ではなく直線同士の最近接点を求めてみましょう。

直線`A`、`B`があり、それぞれ以下の式で表されるとします。

`A(s) = Pa + s * Da`<br>
`B(t) = Pb + t * Db`

ここで`Pa`、`Pb`は直線A, B上の任意の座標、`Da`、`Db`は直線A, Bの方向ベクトル、`s`、`t`は任意の実数です。2つの直線を結ぶベクトル`V(s, t)`があるとすると、問題はこのベクトルの長さが最短になる`s`と`t`を求めることに置き換えられます。

`V(s, t) = A(s) - B(t)`

「ベクトル`V`の長さが最短になるとき」というのは、`V`が`A`に対しても`B`に対しても垂直であるときです。そして、「垂直である」は「内積が0である」と言いかえることができます。つまり、以下の式が成り立ちます。

`Da・V(s, t) = 0`<br>
`Db・V(s, t) = 0`

ここで`V(s, t)`を置き換えると以下が得られます。

`Da・(A(s) - B(t)) =`<br>&ensp;`Da・((Pa + s * Da) - (Pb + t * Db)) = Da・((Pa - Pb) + s * Da - t * Db) = 0`<br>
`Db・(A(s) - B(t)) =`<br>&ensp;`Db・((Pa + s * Da) - (Pb + t * Db)) = Db・((Pa - Pb) + s * Da - t * Db) = 0`

これを、さらに分配法則と結合法則を使って変形すると、以下を得ます。

`(Da・Da)s - (Da・Db)t = -Da・(Pa - Pb)`<br>
`(Da・Db)s - (Db・Db)t = -Db・(Pa - Pb)`

ちょっと見づらくなってきましたね。共通部分を分かりやすくするため、以下の式を導入します。

`a = Da・Da`<br>
`b = Da・Db`<br>
`c = Da・(Pa - Pb)`<br>
`e = Db・Db`<br>
`f = Db・(Pa - Pb)`

この式を導入して書き直したものが以下になります。

`a*s - b*t = -c`<br>
`b*s - e*t = -f`

上式を`s`、`t`について解くと、次の式が得られます。

`s = (b*f - c*e) / (a*e - b*b)`
`t = (a*f - b*c) / (a*e - b*b)`

これで、式に直線のパラメータを代入すれば、`s`と`t`が得られるようになりました。

### 2.2 2つの線分の最近接点

線分`A`、`B`があるとします。これらは以下の式で表すことができます。

`A(s) = Pa + s * Da, 0 ≦ s ≦ 1`<br>
`B(t) = Pb + t * Db, 0 ≦ t ≦ 1`

ここで`Pa`、`Pb`は線分A, B上の一方の端点の座標、`Da`、`Db`は他方の端点への方向ベクトル、`s`、`t`は`0～1`の間の実数です。

直線の場合と同様に、2つの線分を結ぶベクトル`V(s, t)`の長さが最短になる`s`と`t`を求めることになります。

とある点`Sb(t) = Pb + t * Db`が線分`B`上に与えられたとします。このとき、線分`A`上の最近接点の係数`s`は、内積を利用して以下の式で表すことができます。

`s = (Sb(t) - Pa)・Da / (Da・Da) = (Pb + t * Db - Pa)・Da / Da・Da`

この式は、まず内積を利用して線分`(Pa, Sb(t))`を線分Aに射影します(`(Sb(t)-Pa)・Da`の部分)。そして射影して得られた値を線分の長さ(`Da・Da`の部分)で割ります。これによって、線分の始点が`s = 0`、終点が`s = 1`となります。

同じように、とある点`Sa(s) = Pa + s * Da`が線分`A`上に与えられたとすると、線分`B`上の最近接点の係数`t`は以下の式で表すことができます。

`t = (Sa(s) - Pb)・Db / (Db・Db) = (Pa + s * Da - Pb)・Db / Db・Db`

この2つの`s`、`t`を求める式と、直線の最近接点を求める式を使うことで、線分の最近接点を求めることができます。

1. 線分A, Bを直線とみなし、直線の最近接点を求める式を使って仮の`s`を計算する。線分内に収めるため、仮の`s`を`0～1`の範囲に制限する。
2. 仮の`s`を使って線分`B`上の最近接点の係数`t`を計算する。`t`が`0～1`の範囲内にある場合は計算完了。
3. `t`が`0`より小さい、または`1`より大きい場合、`t`を`0～1`の範囲に制限する。そして、制限した`t`を使って線分`A`上の最近接点の係数`s`を計算しなおす。

それでは、2つの線分の最近接点を求める関数を書いてみましょう。関数名は`ClosestPoint`(クローゼスト・ポイント)とします。多少長い関数なので、少しずつ作っていきましょう。

まずは式の共通部分を変数に代入します。`Collision.cpp`を開き、`TestSphereOBB`関数の定義の下に、次のプログラムを追加してください。

```diff
   const glm::vec3 distance = *p - s.center;
   return dot(distance, distance) <= s.r * s.r;
 }
+
+/**
+* 2つの線分の最近接点を調べる.
+*
+* @param s0 線分A.
+* @param s1 線分B.
+* @param c0 線分A上の最近接点.
+* @param c1 線分B上の最近接点.
+*
+* @return c0, c1間のの長さの2乗.
+*/
+float ClosestPoint(const Segment& s0, const Segment& s1, glm::vec3& c0, glm::vec3& c1)
+{
+  // 線分の始点と方向ベクトルを変数に代入.
+  const glm::vec3 Pa = s0.a;
+  const glm::vec3 Da = s0.b - s0.a;
+  const glm::vec3 Pb = s1.a;
+  const glm::vec3 Db = s1.b - s1.a;
+
+  // 式の共通部分を計算して変数に代入.
+  const float a = glm::dot(Da, Da);
+  const float b = glm::dot(Da, Db);
+  const float c = glm::dot(Da, Pa - Pb);
+  const float e = glm::dot(Db, Db);
+  const float f = glm::dot(Db, Pa - Pb);
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

変数名は先ほど説明したのと同じ名前にしていますので、確認は容易でしょう。

次に`s`を計算します。最初は「直線の最近接点の式」を使って仮の`s`を求めます。式の共通分を変数に代入するプログラムの下に、次のプログラムを追加してください。

```diff
   const float c = glm::dot(Da, Pa - Pb);
   const float e = glm::dot(Db, Db);
   const float f = glm::dot(Db, Pa - Pb);
+
+  // 直線の最近接点計算に使う除数.
+  const float denom = a * e - b * b;
+
+  // 線分が平行でないなら(denomが0以外)、直線の最近接点を求める式で仮のsを計算.
+  // 平行な場合は線分A上の適当な位置(とりあえず0)を選択.
+  float s = 0;
+  if (denom != 0.0f) {
+    s = (b * f - c * e) / denom;
+    s = glm::clamp(s, 0.0f, 1.0f);
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

このプログラムには、先ほどの説明にはなかった部分があります。それは、式の除数`denom`(デノム)が`0`の場合の処理です。

`denom`が`0`の場合、2つの線分は平行です。これは「仮の`s`が一意に決まらない(`0～1`のどの値でもよい)」ことを示します。`0～1`の範囲であればなんでもいいので、とりあえず`0`を設定しています。

続いて、仮の`s`を使って、線分B上の最近接点の係数`t`を求めます。仮の`s`を計算するプログラムの下に、次のプログラムを追加してください。

```diff
     s = (b * f - c * e) / denom;
     s = glm::clamp(s, 0.0f, 1.0f);
   }
+
+  // 線分B上の最近接点を求める式でtを計算.
+  float t = (b * s + f) / e;
+
+  // tが0より小さい、または1より大きい場合、tを0～1に制限し、
+  // 線分A上の最近接点を求める式でsを再計算. tが0～1の場合は計算完了.
+  if (t < 0) {
+    t = 0;
+    // s = (tb - c) / aよりt = 0なのでs = -c / aとなる
+    s = glm::clamp(-c / a, 0.0f, 1.0f);
+  } else if (t > 1) {
+    t = 1;
+    // s = (tb - c) / aよりt = 1なのでs = (b - c) / aとなる
+    s = glm::clamp((b - c) / a, 0.0f, 1.0f);
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

`t`が`0`より小さかった場合、`t`に`0`を代入します。そして、`t`を使って`s`を再計算します。同様に`t`が`1`より大きかった場合は、`t`に`1`を代入し、`t`を使って`s`を再計算します。

どちらの場合も同じ計算式を使いますが、`t`が`0`または`1`だと分かっているため、式の一部は省略できます。

最後に、求めた`s`と`t`から実際の座標を計算します。`s`を再計算するプログラムの下に、次のプログラムを追加してください。

```diff
     // s = (tb - c) / aよりt = 1なのでs = (b - c) / aとなる
     s = glm::clamp((b - c) / a, 0.0f, 1.0f);
   }
+
+  // 計算したs, tから最近接点の座標を計算.
+  c0 = Pa + Da * s;
+  c1 = Pb + Db * t;
+
+  // 最近接点の距離の2乗を返す.
+  return glm::dot(c0 - c1, c0 - c1);
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

これで、2つの線分の最近接点を計算するプログラムは完成です。

### 2.3 カプセル同士の衝突判定

2.2節で作成した`ClosestPoint`関数を使って、カプセル同士の衝突を判定する関数を書きましょう。関数名は`TestCapsuleCapsule`(テスト・カプセル・カプセル)とします。`ClosestPoint`関数の定義の下に、次のプログラムを追加してください。

```diff
   // 最近接点の距離の2乗を返す.
   return glm::dot(c0 - c1, c0 - c1);
 }
+
+/**
+* カプセルとカプセルが衝突しているか調べる.
+*
+* @param c0 カプセル0.
+* @param c1 カプセル1.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestCapsuleCapsule(const Capsule& c0, const Capsule& c1)
+{
+  // カプセル内部の線分同士の距離を計算し、その距離が2つのカプセルの半径の和以下なら衝突している.
+  glm::vec3 p0, p1;
+  const float d = ClosestPoint(c0.seg, c1.seg, p0, p1);
+  const float r = c0.r + c1.r;
+  if (d > r * r) {
+    return {};
+  }
+
+  Result result;
+  result.isHit = true;
+  result.na = glm::normalize(p1 - p0);
+  result.nb = -result.na;
+  result.pa = p0 + result.na * c0.r;
+  result.pb = p1 + result.nb * c1.r;
+  return result;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

まず`ClosestPoint`関数によって、線分の最近接点と距離を計算します。この距離と2つのカプセルの半径の和を比較し、距離が半径の和以下の場合は衝突しています。

距離の比較では、ルートの計算を避けるために両辺とも2乗した値で比較している点に注意してください。長さの2乗による比較は、一般的な最適化テクニックのひとつです。

カプセル0側の衝突点の法線は、最近接点による線分の方向ベクトルと同じです。上記のプログラムでは、`glm::normalize(p1 - p0)`として計算しています。カプセルB側の法線は、カプセルAの法線を逆向きにしたものです。

また、最近接点は実際の衝突点ではないことに注意してください。最近接点はカプセル内部の線分上にあります。実際の衝突点は、最近接点を法線方向に半径の長さだけ移動した位置になります。

これでカプセルとカプセルの衝突判定は完成です。

<div style="page-break-after: always"></div>

## 3. カプセルとOBBの衝突判定

この章で扱う衝突判定

|              | 球 | カプセル | OBB
|:------------:|:--:|:--------:|:---:
|**球**       | ○ | ○       | ○
|**カプセル** | ○ | ○       | ✔
|**OBB**      | ○ | ☓       | ☓

### 3.1 軸平行境界ボックス(AABB)

カプセルとOBBの衝突判定も、基本的にはカプセルの線分に対してOBBの衝突判定を行うことになります。OBBは球やカプセルと比べて複雑な形状なので、衝突判定は相応に複雑なものになります。しかし、OBBの代わりに「軸平行境界ボックス(AABB)」を使うことで、多少ですが判定を簡単にすることが可能です。

「AABB(エー・エー・ビー・ビー)」は「Axis Aligned Bounding Box(アクシス・アラインド・バウンディング・ボックス)」の略称で、OBBの各軸がワールド座標系のX, Y, Z軸と等しくなったものです。軸がワールド座標系と一致しているため、いくつかの計算を省略することができるのです。

そして、カプセルの座標系をOBBのローカル座標系へと変換することで、「カプセルとOBBの判定」を「カプセルとAABBの判定」で置き換えることができます。

それではAABBを定義しましょう。`Collision.h`を開き、`Capsule`構造体の定義の下に、次のプログラムを追加してください。

```diff
   Segment seg; ///< 円柱部の中心の線分.
   float r = 0; ///< カプセルの半径.
 };
+
+/**
+* 軸平行境界ボックス.
+*/
+struct AABB {
+  glm::vec3 min; ///< 各軸の最小値.
+  glm::vec3 max; ///< 各軸の最大値.
+};

 /**
 * 有向境界ボックス.
```

AABBは、2つの頂点`min`と`max`によって定義されます。

### 3.2 光線とAABBの交差

次に「光線」とAABBの交差判定を作成します。「光線」は、座標`P`から方向ベクトル`D`に向かう直線です。AABBは各軸に平行な2つの平面が3つ集まったものなので、それぞれの平面に対して交差判定を行います。

ひとつの軸には2つの平面があるため、交差点は2つ見つかります。光線の始点に近いほう入射点、遠いほうを出射点とすると、軸は3つあるので3つの入射点と出射点が存在するわけです。このとき、いずれかの出射点がいずれかの入射点よりも光線の始点に近い場合、交差はありません。

光線と平面の交差は、光線の式`R(t) = P + t * d`を平面の式`X・n = D`に代入し、`t`について解くことによって得られます。これは以下の式になります。

`t = (D - P・n) / (d・n)`

AABBの場合、有効な法線`n`の成分はx, y, zのいずれかひとつだけです。そのため、内積を取り除くことができます。始点を`P=(Px, Py, Pz)`、方向ベクトルを`d=(dx, dy, dz)`とすると、内積を取り除いた各軸の式は以下のようになります。

`t = (D - Px) / dx`<br>
`t = (D - Py) / dy`<br>
`t = (D - Pz) / dz`

また、光線が軸に対して平行な場合も考えられます。その場合、光線の始点がAABBを構成する2つの平面の内側にあるかどうかだけをチェックします。内側になければ交差はありません。

今回はいくつかのC++標準ライブラリの関数を使いたいので、まずヘッダファイルをインクルードします。`Collision.cpp`を開き、次のプログラムを追加してください。

```diff
 * @file Collision.cpp
 */
 #include "Collision.h"
+#include <algorithm>

 namespace Collision {
```

それでは交差判定関数を書いていきます。関数名は`IntersectRayAABB`(インターセクト・レイ・エー・エー・ビー・ビー)とします。`Collision.cpp`を開き、`TestCapsuleCapsule`関数の定義の下に、次のプログラムを追加してください。

```diff
   result.pb = p1 + result.nb * c1.r;
   return result;
 }
+
+/**
+* 光線と軸平行境界ボックスが交差しているか調べる.
+*
+* @param p    光線の始点.
+* @param d    光線の方向ベクトル.
+* @param aabb 軸平行境界ボックス.
+* @param tmin 交差距離を格納する変数.
+* @param q    交差点を格納する変数.
+*
+* @retval true  交差している.
+* @retval false 交差していない.
+*/
+bool IntersectRayAABB(const glm::vec3& p, const glm::vec3& d, const AABB& aabb,
+  float& tmin, glm::vec3& q)
+{
+  tmin = 0; // 最も遠い入射点までの距離.
+  float tmax = FLT_MAX; // 最も近い出射点までの距離.
+
+  for (int i = 0; i < 3; ++i) {
+    // 光線と軸が平行かどうか.
+    if (std::abs(d[i]) < FLT_EPSILON) {
+      // 平行な場合、光線の始点が軸の平面の外側にあったら交差なし.
+      if (p[i] < aabb.min[i] || p[i] > aabb.max[i]) {
+        return false;
+      }
+    } else {
+      // 始点から入射点および出射点までの距離を計算.
+      const float ood = 1.0f / d[i]; // 除算の回数を減らす.
+      float t1 = (aabb.min[i] - p[i]) * ood;
+      float t2 = (aabb.max[i] - p[i]) * ood;
+
+      // 近いほうを入射点、遠いほうを出射点とする.
+      if (t1 > t2) {
+        std::swap(t1, t2);
+      }
+
+      // 始点から最も遠い入射点までの距離、最も近い出射点までの距離を更新.
+      tmin = std::max(tmin, t1);
+      tmax = std::min(tmax, t2);
+
+      // 入射点が出射点より遠くなったら交差なし.
+      if (tmin > tmax) {
+        return false;
+      }
+    }
+  }
+
+  // 光線はAABBと交差している. 交差点qを計算する.
+  q = p + d * tmin;
+  return true;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

これで光線とAABBの交差判定ができるようになりました。

### 3.3 判定を補助するCorner(コーナー)関数

カプセルとAABBの衝突判定を行うとき、AABBの頂点座標が必要となることがあります。しかし、AABB構造体は明示的な頂点座標を持っていません。そこで、頂点を取得するための関数を作成します。

`IntersectRayAABB`関数の定義の下に、次のプログラムを追加してください。

```diff
   q = p + d * tmin;
   return true;
 }
+
+/**
+* AABBの頂点を取得する.
+*
+* @param aabb AABB.
+* @param n    頂点を選択するビットフラグ.
+*
+* @return nに対応する頂点座標.
+*/
+glm::vec3 Corner(const AABB& aabb, int n)
+{
+  glm::vec3 p = aabb.min;
+  if (n & 1) { p.x = aabb.max.x; }
+  if (n & 2) { p.y = aabb.max.y; }
+  if (n & 4) { p.z = aabb.max.z; }
+  return p;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

これで頂点を取得できるようになりました。

### 3.4 カプセルとAABBの衝突判定

それでは、カプセルとAABBの衝突判定関数を書きましょう。関数名は`TestCapsuleAABB`(テスト・カプセル・エー・エー・ビー・ビー)とします。この関数はかなり長いものになりますから、少しずつ書いていくことにします。`Corner`関数の定義の下に、次のプログラムを追加してください。

```diff
  if (n & 4) { p.z = aabb.max.z; }
  return p;
}
+
+/**
+* カプセルとAABBが衝突しているか調べる.
+*
+* @param c    カプセル.
+* @param aabb 軸平行境界ボックス.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestCapsuleAABB(const Capsule& c, const AABB& aabb)
+{
+  // カプセルの半径だけ拡大したAABBを作成.
+  AABB e = aabb;
+  e.min -= c.r;
+  e.max += c.r;
+
+  // 光線の方向ベクトルを計算. 0 <= t < 1で線分の判定をできるようにするため、正規化はしない.
+  const glm::vec3 d(c.seg.b - c.seg.a);
+
+  // 拡大したAABBと光線の交差判定.
+  float t;
+  glm::vec3 p;
+  if (!IntersectRayAABB(c.seg.a, d, e, t, p) || t > 1) {
+    return {};
+  }
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

まず、AABBをカプセルの半径だけ拡大し、拡大したAABBに対して交差判定を行います。これで交差しなかった場合は衝突していません。これによって、交差の可能性がないほとんどの物体を除外できます。光線とAABBの交差判定比較的高速なので、早期の衝突判定にはうってつけです。

続いて、交差が元のAABBのどの領域に対して起こったかを記録します。変数`m`の値に応じて「内部」、「面領域」、「辺領域」、「頂点領域」の4種類の領域に分けられ、領域ごとに異なる方法で衝突判定を行っていきます。

```diff
   if (!IntersectRayAABB(c.seg.a, d, e, t, p) || t > 1) {
     return {};
   }
+
+  // 交差の起こった領域を変数u(負方向), v(正方向)に記録.
+  int u = 0, v = 0;
+  if (p.x < aabb.min.x) { u |= 1; }
+  else if (p.x > aabb.max.x) { v |= 1; }
+
+  if (p.y < aabb.min.y) { u |= 2; }
+  else if (p.y > aabb.max.y) { v |= 2; }
+
+  if (p.z < aabb.min.z) { u |= 4; }
+  else if (p.z > aabb.max.z) { v |= 4; }
+
+  const int m = u + v;
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

それでは領域ごとの衝突判定を行いましょう。まず「内部」領域を扱います。「内部」の場合、それ以上判定するまでもなく衝突は起きています。残る問題は、衝突点と衝突面の法線を求めることです。

今回は、交点`p`に最も近い面に対して衝突が起きたことにします。`p`を最も近い面に射影した点が衝突点になります。衝突面の法線には、最も近い面の法線をそのまま使います。最も近い面の判定は、地道に6つの面について`if`文で判定していきます。それでは、交差領域を記録するプログラムの下に、次のプログラムを追加してください。

```diff
   else if (p.z > aabb.max.z) { v |= 4; }

   const int m = u + v;
+
+  // m = 0ならば、交点pはAABBの内側にある.
+  // pを最も近い面に射影し、その点を衝突点とする.
+  if (!m) {
+    Result result;
+    result.isHit = true;
+
+    result.pb = glm::vec3(aabb.min.x, p.y, p.z);
+    result.nb = glm::vec3(-1, 0, 0);
+    float d = p.x - aabb.min.x; // 最も近い面までの距離.
+    if (aabb.max.x - p.x < d) {
+      d = aabb.max.x - p.x;
+      result.pb = glm::vec3(aabb.max.x, p.y, p.z);
+      result.nb = glm::vec3(1, 0, 0);
+    }
+    if (p.y - aabb.min.y < d) {
+      d = p.y - aabb.min.y;
+      result.pb = glm::vec3(p.x, aabb.min.y, p.z);
+      result.nb = glm::vec3(0,-1, 0);
+    }
+    if (aabb.max.y - p.y < d) {
+      d = aabb.max.y - p.y;
+      result.pb = glm::vec3(p.x, aabb.max.y, p.z);
+      result.nb = glm::vec3(0, 1, 0);
+    }
+    if (p.z - aabb.min.z < d) {
+      d = p.z - aabb.min.z;
+      result.pb = glm::vec3(p.x, p.y, aabb.min.z);
+      result.nb = glm::vec3(0, 0,-1);
+    }
+    if (aabb.max.z - p.z < d) {
+      d = aabb.max.z - p.z;
+      result.pb = glm::vec3(p.x, p.y, aabb.max.z);
+      result.nb = glm::vec3(0, 0, 1);
+    }
+    result.na = -result.nb;
+    result.pa = p + result.na * c.r;
+    return result;
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

カプセル側の法線は、最も近い面の法線を逆にしたものです。カプセルや球は、常に衝突面に対して垂直に衝突するからです。

次は、交点`p`が「面領域」にある場合を扱います。変数`m`のいずれか1ビットだけが立っている場合、交点`p`は面領域に存在します。面領域で衝突している場合も、それ以上の衝突判定は必要ありません。「内部」と同様に、衝突点と衝突面の法線を求めるだけです。

AABBのどの面に衝突したのかは、変数`u`と`v`を調べれば分かります。面領域の場合、`u`と`v`のうちビットが立っている面が衝突面です。ここでも、交点`p`は拡大したAABB上にあることに注意してください。実際の衝突点を得るには、拡大した距離を戻さなくてはなりません。

それでは、「内側」を処理するプログラムの下に、次のプログラムを追加してください。

```diff
     result.pa = p + result.na * c.r;
     return result;
   }
+
+  // pは面領域にある.
+  if ((m & (m - 1)) == 0) {
+    Result result;
+    result.isHit = true;
+
+    // この時点でpは拡張されたAABB上にある.
+    // 法線の逆方向に移動した点を実際の衝突点とする.
+    if (u & 1) {
+      result.nb = glm::vec3(-1, 0, 0);
+      result.pb = p + result.nb * (p.x - aabb.min.x);
+    } else if (u & 2) {
+      result.nb = glm::vec3(0,-1, 0);
+      result.pb = p + result.nb * (p.y - aabb.min.y);
+    } else if (u & 4) {
+      result.nb = glm::vec3(0, 0,-1);
+      result.pb = p + result.nb * (p.z - aabb.min.z);
+    } else if (v & 1) {
+      result.nb = glm::vec3(1, 0, 0);
+      result.pb = p - result.nb * (p.x - aabb.max.x);
+    } else if (v & 2) {
+      result.nb = glm::vec3(0, 1, 0);
+      result.pb = p - result.nb * (p.y - aabb.max.y);
+    } else if (v & 4) {
+      result.nb = glm::vec3(0, 0, 1);
+      result.pb = p - result.nb * (p.z - aabb.max.z);
+    }
+    result.na = -result.nb;
+    result.pa = p + result.na * c.r;
+    return result;
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

3つめは「頂点領域」です。変数`m`が`7`のとき、つまりすべての面ビットが立っている場合、交点`p`は頂点領域にあります。頂点領域の場合は、衝突が確定していません。頂点につながる3つの辺について最近接点を求め、本当に衝突しているかを判定していきます。

面領域を処理するプログラムの下に、次のプログラムを追加してください。

```diff
     result.pa = p + result.na * c.r;
     return result;
   }
+
+  // pは頂点領域にある.
+  // 頂点に接する3辺のうち、最も接近した最近接点を持つ辺と最初に衝突したとみなす.
+  if (m == 7) {
+    const glm::vec3 bv = Corner(aabb, v);
+    glm::vec3 c0, c1, c2, c3;
+    float d = ClosestPoint(c.seg, Segment{ bv, Corner(aabb, v ^ 1) }, c0, c1);
+    float d0 = ClosestPoint(c.seg, Segment{ bv, Corner(aabb, v ^ 2) }, c2, c3);
+    if (d0 < d) {
+      d = d0;
+      c0 = c2;
+      c1 = c3;
+    }
+    d0 = ClosestPoint(c.seg, Segment{ bv, Corner(aabb, v ^ 4) }, c2, c3);
+    if (d0 < d) {
+      d = d0;
+      c0 = c2;
+      c1 = c3;
+    }
+    if (d > c.r * c.r) {
+      return {};
+    }
+    Result result;
+    result.isHit = true;
+    result.na = glm::normalize(c1 - c0);
+    result.pa = c0 + result.na * c.r;
+    result.nb = -result.na;
+    result.pb = c1;
+    return result;
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

残るは「辺領域」です。変数`m`がこれまでのどのパターンにも当てはまらなかった場合、交点`p`は辺領域にあります。

辺領域も衝突が確定していないため、頂点領域と同様に最近接点を求める必要があります。ただし、対象となるのは1つの辺だけです。頂点領域を処理するプログラムの下に、次のプログラムを追加してください。

```diff
     result.pb = c1;
     return result;
   }
+
+  // pは辺領域にある.
+  {
+    glm::vec3 c0, c1;
+    const Segment edge = { Corner(aabb, u ^ 7), Corner(aabb, v) };
+    const float d = ClosestPoint(c.seg, edge, c0, c1);
+    if (d > c.r * c.r) {
+      return {};
+    }
+    Result result;
+    result.isHit = true;
+    result.na = glm::normalize(c1 - c0);
+    result.pa = c0 + result.na * c.r;
+    result.nb = -result.na;
+    result.pb = c1;
+    return result;
+  }
 }

 /**
 * シェイプ同士が衝突しているか調べる.
```

これでカプセルとAABBの衝突判定は完成です。

### 3.5 カプセルとOBBの衝突判定

3.4節で作成した`TestCapsuleAABB`関数を使って、カプセルとOBBの衝突判定を作成します。カプセルの座標系をOBBの座標系に変換するには、カプセルの座標からOBBの座標を引いた後、OBBの回転行列を掛けます。

衝突が起きていたら、衝突点と法線に逆回転行列を掛けて回転を元に戻します。衝突点についてはさらにOBBの座標を足すことで、完全にワールド座標系に戻すことができます。

`TestCapsuleAABB`関数の定義の下に、次のプログラムを追加してください。

```diff
     result.nb = -result.na;
     return result;
   }
 }
+
+/**
+* カプセルとOBBが衝突しているか調べる.
+*
+* @param c   カプセル.
+* @param obb 有向境界ボックス.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestCapsuleOBB(const Capsule& c, const OrientedBoundingBox& obb)
+{
+  // 線分をOBBのローカル座標系に変換.
+  Capsule cc = c;
+  cc.seg.a -= obb.center;
+  cc.seg.b -= obb.center;
+  const glm::mat3 matOBB(
+    glm::transpose(glm::mat3(obb.axis[0], obb.axis[1], obb.axis[2])));
+  cc.seg.a = matOBB * cc.seg.a;
+  cc.seg.b = matOBB * cc.seg.b;
+
+  // 衝突判定.
+  Result result = TestCapsuleAABB(cc, AABB{ -obb.e, obb.e });
+  if (result.isHit) {
+    // 衝突結果をワールド座標系に変換.
+    const glm::mat3 matInvOBB(glm::inverse(matOBB));
+    result.pa = matInvOBB * result.pa + obb.center;
+    result.pb = matInvOBB * result.pb + obb.center;
+    result.na = matInvOBB * result.na;
+    result.nb = matInvOBB * result.nb;
+  }
+  return result;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

これでカプセルとOBBの衝突判定は完成です。

<div style="page-break-after: always"></div>

## 4. Result型を返す衝突判定関数

### 4.1 球と何らかの形状の衝突判定

この章では`Result`型を返す衝突判定関数を作成していきます。以前作成した`TestShapeShape`関数は、全て`if`文で分岐していました。しかしこれは見づらいため、今回は右辺の形状ごとに3つ関数に分けて書くことにします。

まずは「球と何らかの形状の衝突判定」を作成しましょう。実際の衝突判定には既存の関数を流用し、その結果から`Result`型を作成して返す、という形にします。関数名は`TestSphereShape`(テスト・スフィア・シェイプ)とします。それでは、`TestCapsuleOBB`関数の定義の下に、次のプログラムを追加してください。

```diff
     result.nb = matInvOBB * result.nb;
   }
   return result;
 }
+
+/**
+* 球と何らかの形状の衝突判定.
+*
+* @param a  判定対象のシェイプその１.
+* @param b  判定対象のシェイプその２.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestSphereShape(const Sphere& a, const Shape& b)
+{
+  Result result;
+  glm::vec3 p;
+  switch (b.type) {
+  case Shape::Type::sphere:
+    if (TestSphereSphere(a, b.s)) {
+      result.isHit = true;
+      result.na = glm::normalize(b.s.center - a.center);
+      result.nb = -result.na;
+      result.pa = a.center + result.na * a.r;
+      result.pb = b.s.center + result.nb * b.s.r;
+    }
+    break;
+  case Shape::Type::capsule:
+    if (TestSphereCapsule(a, b.c, &p)) {
+      result.isHit = true;
+      result.na = glm::normalize(p - a.center);
+      result.nb = -result.na;
+      result.pa = a.center + result.na * a.r;
+      result.pb = p + result.nb * b.c.r;
+    }
+    break;
+  case Shape::Type::obb:
+    if (TestSphereOBB(a, b.obb, &p)) {
+      result.isHit = true;
+      result.na = glm::normalize(p - a.center);
+      result.nb = -result.na;
+      result.pa = a.center + result.na * a.r;
+      result.pb = p;
+    }
+    break;
+  }
+  return result;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

### 4.2 カプセルと何らかの形状の衝突判定

次に、カプセルと何らかの形状の衝突判定を作成します。カプセルと球の衝突判定については既存の`TestSphereCapsule`関数を流用し、カプセルとカプセル、カプセルとOBBについては今回作成した関数を使います。関数名は`TestCapsuleShape`(テスト・カプセル・シェイプ)とします。

`TestSphereShape`関数の定義の下に、次のプログラムを追加してください。

```diff
     break;
   }
   return result;
 }
+
+/**
+* カプセルと何らかの形状の衝突判定.
+*
+* @param a  判定対象のシェイプその１.
+* @param b  判定対象のシェイプその２.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestCapsuleShape(const Capsule& a, const Shape& b)
+{
+  glm::vec3 p;
+  switch (b.type) {
+  case Shape::Type::sphere:
+    if (TestSphereCapsule(b.s, a, &p)) {
+      Result result;
+      result.isHit = true;
+      result.na = glm::normalize(b.s.center - p);
+      result.nb = -result.na;
+      result.pa = p + result.na * a.r;
+      result.pb = b.s.center + result.nb * b.s.r;
+      return result;
+    }
+    break;
+
+  case Shape::Type::capsule:
+    return TestCapsuleCapsule(a, b.c);
+
+  case Shape::Type::obb:
+    return TestCapsuleOBB(a, b.obb);
+  }
+  return {};
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

### 4.3 OBBと何らかの形状の衝突判定

続いて、OBBと何らかの形状の衝突判定を作成します。これも実際の判定には、既存の関数と今回作成した衝突判定関数を使います。しかし、OBB同士の衝突判定は作成していないので、衝突判定は常に失敗します。関数名は`TestOBBShape`(テスト・オー・ビー・ビー・シェイプ)とします。

`TestCapsuleShape`関数の定義の下に、次のプログラムを追加してください。

```diff
     return TestCapsuleOBB(a, b.obb);
   }
   return {};
 }
+
+/**
+* OBBと何らかの形状の衝突判定.
+*
+* @param a  判定対象のシェイプその１.
+* @param b  判定対象のシェイプその２.
+*
+* @return 衝突結果を格納するResult型の値.
+*/
+Result TestOBBShape(const OrientedBoundingBox& a, const Shape& b)
+{
+  Result result;
+  glm::vec3 p;
+  switch (b.type) {
+  case Shape::Type::sphere:
+    if (TestSphereOBB(b.s, a, &p)) {
+      result.isHit = true;
+      result.na = glm::normalize(b.s.center - p);
+      result.nb = -result.na;
+      result.pa = p;
+      result.pb = b.s.center + result.nb * b.s.r;
+    }
+    break;
+
+  case Shape::Type::capsule:
+    result = TestCapsuleOBB(b.c, a);
+    std::swap(result.na, result.nb);
+    std::swap(result.pa, result.pb);
+    break;
+
+  case Shape::Type::obb:
+    // 未実装.
+    break;
+  }
+  return result;
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

### 4.4 シェイプ同士の衝突判定

最後に、3つの関数をまとめてシェイプ同士の衝突判定関数を作成します。関数名は`TestShapeShape`(テスト・シェイプ・シェイプ)とします。`TestOBBShape`関数の定義の下に、次のプログラムを追加してください。

```diff
     // 未実装.
     break;
   }
   return result;
 }
+
+/**
+* シェイプ同士が衝突しているか調べる.
+*
+* @param a  判定対象のシェイプその１.
+* @param b  判定対象のシェイプその２.
+*
+* @return 衝突結果を表すResult型の値.
+*/
+Result TestShapeShape(const Shape& a, const Shape& b)
+{
+  switch (a.type) {
+  case Shape::Type::sphere:
+    return TestSphereShape(a.s, b);
+
+  case Shape::Type::capsule:
+    return TestCapsuleShape(a.c, b);
+
+  case Shape::Type::obb:
+    return TestOBBShape(a.obb, b);
+  }
+  return Result{};
+}

 /**
 * シェイプ同士が衝突しているか調べる.
```

### 4.5 TestShapeShape関数の宣言を追加する

作成した`TestShapeShape`関数を他のプログラムから呼び出せるように、関数宣言を追加しましょう。`Collision.h`を開き、次のプログラムを追加してください。

```diff
 bool TestSphereSphere(const Sphere&, const Sphere&);
 bool TestSphereCapsule(const Sphere& s, const Capsule& c, glm::vec3* p);
 bool TestSphereOBB(const Sphere& s, const OrientedBoundingBox& obb, glm::vec3* p);
 bool TestShapeShape(const Shape&, const Shape&, glm::vec3* pa, glm::vec3* pb);
+Result TestShapeShape(const Shape&, const Shape&);

 glm::vec3 ClosestPointSegment(const Segment& seg, const glm::vec3& p);

 } // namespace Collision
```

これで`Result`型を返す衝突判定関数の作成は完了です。

<div style="page-break-after: always"></div>

## 5. Result型による衝突の解決

### 5.1 Result型を受け取るActor::OnHit関数を追加する

`Result`型を使った衝突判定を行うために、`Actor`クラスに`Result`型を受け取る`OnHit`メンバ関数を追加しましょう。`Actor.h`を開き、`Actor`クラスに次のプログラムを追加してください。

```diff
   virtual void Draw(Mesh::DrawType = Mesh::DrawType::color);
   virtual void OnHit(const ActorPtr&, const glm::vec3&) {}
+  virtual void OnHit(const ActorPtr& other, const Collision::Result& result) {
+    OnHit(other, result.pb);
+  }

 public:
   std::string name; ///< アクターの名前.
```

新しく追加した`OnHit`メンバ関数は、デフォルトでは古いほうの`OnHit`関数を呼び出すだけです。

>**【補足】**<br>
>古いほうの`OnHit`関数のオーバーライド関数をすべて新しい`OnHit`関数で置き換えることができたら、古いほうの`OnHit`メンバ関数は削除し、新しい`OnHit`関数については、デフォルトでは何もしないように書き換えるとよいでしょう。

### 5.2 DetectCollision関数をResult型に対応させる

次に、3つある`DetectCollision`関数を、`Result`型を使うように書き換えていきます。`Actor.cpp`を開き、アクター同士の`DetectCollision`関数を、次のように書きかえてください。

```diff
 void DetectCollision(const ActorPtr& a, const ActorPtr& b, CollisionHandlerType handler)
 {
   if (a->health <= 0 || b->health <= 0) {
     return;
   }
-  glm::vec3 pa, pb;
-  if (Collision::TestShapeShape(a->colWorld, b->colWorld, &pa, &pb)) {
+  Collision::Result r = Collision::TestShapeShape(a->colWorld, b->colWorld);
+  if (r.isHit) {
     if (handler) {
-      handler(a, b, pb);
+      handler(a, b, r.pa);
     } else {
-      a->OnHit(b, pb);
-      b->OnHit(a, pa);
+      a->OnHit(b, r);
+      std::swap(r.pa, r.pb);
+      std::swap(r.na, r.nb);
+      b->OnHit(a, r);
     }
   }
 }
```

続いて、アクターとアクターリストの`DetectCollision`関数を、次のように書きかえてください。

```diff
   for (const ActorPtr& actorB : b) {
     if (actorB->health <= 0) {
       continue;
     }
-    glm::vec3 pa, pb;
-    if (Collision::TestShapeShape(a->colWorld, actorB->colWorld, &pa, &pb)) {
+    Collision::Result r = Collision::TestShapeShape(a->colWorld, actorB->colWorld);
+    if (r.isHit) {
       if (handler) {
-        handler(a, actorB, pb);
+        handler(a, actorB, r.pa);
       } else {
-        a->OnHit(actorB, pb);
-        actorB->OnHit(a, pa);
+        a->OnHit(actorB, r);
+        std::swap(r.pa, r.pb);
+        std::swap(r.na, r.nb);
+        actorB->OnHit(a, r);
       }
       if (a->health <= 0) {
         break;
       }
     }
   }
```

最後に、アクターリスト同士の`DetectCollision`関数を、次のように書きかえてください。

```diff
     for (const ActorPtr& actorB : b) {
       if (actorB->health <= 0) {
         continue;
       }
-      glm::vec3 pa, pb;
-      if (Collision::TestShapeShape(actorA->colWorld, actorB->colWorld, &pa, &pb)) {
+      Collision::Result r =
+        Collision::TestShapeShape(actorA->colWorld, actorB->colWorld);
+      if (r.isHit) {
         if (handler) {
-          handler(actorA, actorB, pb);
+          handler(actorA, actorB, r.pa);
         } else {
-          actorA->OnHit(actorB, pb);
-          actorB->OnHit(actorA, pa);
+          actorA->OnHit(actorB, r);
+          std::swap(r.pa, r.pb);
+          std::swap(r.na, r.nb);
+          actorB->OnHit(actorA, r);
         }
         if (actorA->health <= 0) {
           break;
         }
       }
     }
```

これで`DetectCollision`関数の修正は完了です。

### 5.3 Result型を受け取るPlayerActor::OnHit関数を追加する

`Result`型の動作テストのために、`PlayerActor`の`OnHit`メンバ関数を`Result`型に対応させましょう。`PlayerActor.h`を開き、次のプログラムを追加してください。

```diff
   virtual void Update(float) override;
   virtual void OnHit(const ActorPtr&, const glm::vec3&) override;
+  virtual void OnHit(const ActorPtr&, const Collision::Result&) override;
   void Jump();
   void ProcessInput();
   void SetBoardingActor(ActorPtr);
```

次に`PlayerActor.cpp`を開き、既存の`OnHit`関数の定義の下に、次のプログラムを追加してください。

```diff
   SetBoardingActor(b);
 }
+
+/**
+* 衝突ハンドラ.
+*
+* @param b      衝突相手のアクター.
+* @param result 衝突結果.
+*/
+void PlayerActor::OnHit(const ActorPtr& b, const Collision::Result& result)
+{
+  // 貫通しない位置まで衝突面の法線方向に移動させる.
+  const float d = glm::dot(result.nb, result.pb - result.pa);
+  const glm::vec3 v = result.nb * (d + 0.01f);
+  colWorld.c.seg.a += v;
+  colWorld.c.seg.b += v;
+  position += v;
+  if (!isInAir && !boardingActor) {
+    const float newY = heightMap->Height(position);
+    colWorld.c.seg.a.y += newY - position.y;
+    colWorld.c.seg.b.y += newY - position.y;
+    position.y = newY;
+  }
+  // 衝突面の法線が真上から30度の範囲にあれば乗ることができる(角度は要調整).
+  const float theta = glm::dot(result.nb, glm::vec3(0, 1, 0));
+  if (theta >= cos(glm::radians(30.0f))) {
+    SetBoardingActor(b);
+  }
+}

 /**
 * ジャンプさせる.
```

### 5.4 プレイヤーの衝突判定形状をカプセルに変更する

実際に衝突するかどうか、プレイヤーの衝突判定の形状をカプセルに変えて試してみるのが簡単です。`PlayerActor`コンストラクタを次のように書きかえてください。

```diff
 {
-  colLocal = Collision::CreateSphere(glm::vec3(0, 0.7f, 0), 0.7f);
+  colLocal = Collision::CreateCapsule(glm::vec3(0, 0.5f, 0), glm::vec3(0, 1, 0), 0.5f);
   GetMesh()->Play("Idle");
   state = State::idle;
 }
```

衝突判定形状は落下判定でもつかっています。こちらも変更しなくてはなりません。`PlayerActor::Update`メンバ関数を次のように書きかえてください。

```diff
     // 乗っている物体から離れたら空中判定にする.
     if (boardingActor) {
-      Collision::Shape col = colWorld;
-      col.s.r += 0.1f; // 衝突判定を少し大きくする.
+      // 落下判定用の形状は、プレイヤーの衝突判定形状に合わせて調整すること.
+      const Collision::Shape col = Collision::CreateCapsule(
+        position + glm::vec3(0, 0.4f, 0), position + glm::vec3(0, 1, 0), 0.25f);
-      glm::vec3 pa, pb;
-      if (!Collision::TestShapeShape(col, boardingActor->colWorld, &pa, &pb)) {
+      const Collision::Result result =
+        Collision::TestShapeShape(col, boardingActor->colWorld);
+      if (!result.isHit) {
         boardingActor.reset();
+      } else {
+        // 衝突面の法線が真上から30度の範囲になければ落下.
+        const float theta = glm::dot(result.nb, glm::vec3(0, 1, 0));
+        if (theta < glm::cos(glm::radians(30.0f))) {
+          boardingActor.reset();
+        }
       }
     }
```

プログラムが書けたらビルドして実行してください。木や敵、壁などを貫通しなければ成功です。
