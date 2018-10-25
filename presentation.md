# refinedで安全なコードを書く

石川 裕樹 (Yuki Ishikawa)

Twitter: [@rider_yi](https://twitter.com/rider_yi) / GitHub: [rider-yi](https://github.com/rider-yi)

---

## 自己紹介

### 石川 裕樹 (Yuki Ishikawa)

- プログラマ歴11年, Scala歴4年

- 株式会社ファンコミュニケーションズ 5年目

- ネット広告の配信システムをScalaで開発してる

- refined, ScalaCacheとかのメンテナンス

- バイク大好き🏍

---

## 今日紹介するライブラリ

### [fthomas/refined](https://github.com/fthomas/refined)

Simple refinement types for Scala

<span class="star">★779</span>

---

## refined無しの世界では...


```scala
final case class Config(user: String, bitcoinAddress: String)
```

```scala
Config("yuki", "18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS") // OK
``` 

<!-- .element: class="fragment" -->

```scala
Config("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS", "yuki") // バグ発生
```

<!-- .element: class="fragment" -->


### Stringの何が問題か

- Stringという型は意味を持たない

- Stringはなんでも持つことができてしまう


### 型エイリアスで解決できる？

```scala
type User = String
type BitcoinAddress = String

final case class Config(user: User, bitcoinAddress: BitcoinAddress)

val user: User = "yuki"
val addr: BitcoinAddress = "18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS"

Config(user, addr) // OK

Config(addr, user) // バグ発生
```

Stringに意味は追加できたが、バグは防げない


### 値クラスで解決できる？

```scala
final case class User(value: String) extends AnyVal
final case class BitcoinAddress(value: String) extends AnyVal
final case class Config(user: User, bitcoinAddress: BitcoinAddress)

Config(
  BitcoinAddress("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS"),
  User("yuki")
) // これはコンパイルされない

Config(
  User("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS"),
  BitcoinAddress("yuki")
) // これは防ぐことができない
```


### スマートコンストラクタでバリデーションを追加

```scala
final case class BitcoinAddress private (value: String) extends AnyVal

object BitcoinAddress {
  def fromString(value: String): Option[BitcoinAddress] =
    if (value.matches("[13][a-km-zA-HJ-NP-Z1-9]{25,34}"))
      Some(BitcoinAddress(value))
    else None
}
```

```scala
@ BitcoinAddress.fromString("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS")
res9: Option[BitcoinAddress] = Some(
  BitcoinAddress("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS")
)

@ BitcoinAddress.fromString("")
res10: Option[BitcoinAddress] = None
```

<!-- .element: class="fragment" -->


```scala
@ new BitcoinAddress("") // newはコンパイルが通らない
cmd12.sc:1: constructor BitcoinAddress in class BitcoinAddress cannot be accessed in object cmd12
```

```scala
@ BitcoinAddress("") // しかし、コンパニオンのapplyは普通に使えてしまう
res6: BitcoinAddress = BitcoinAddress("")

@ {
  BitcoinAddress.fromString("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS")
    .get
    .copy(value = "") // copyでも不正な値を入れることができる
  }
res18: BitcoinAddress = BitcoinAddress("")
```

<!-- .element: class="fragment" -->

`apply`, `copy`という穴があり、バリデーションを通さずインスタンスが生成できてしまう

<!-- .element: class="fragment" -->


### ファクトリメソッド以外の方法でインスタンスを生成できないようにしたい...


### sealed abstract case class

```scala
sealed abstract case class BitcoinAddress(value: String)

object BitcoinAddress {
  def fromString(value: String): Option[BitcoinAddress] =
    if (value.matches("[13][a-km-zA-HJ-NP-Z1-9]{25,34}"))
      Some(new BitcoinAddress(value) {})
    else None
}
```

https://gist.github.com/tpolecat/a5cb0dc9adeacc93f846835ed21c92d2


```scala
@ val x = BitcoinAddress.fromString("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS")
x: Option[BitcoinAddress] = Some(
  BitcoinAddress("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS")
)

@ x.get.copy(value = "") // copy不可能
cmd19.sc:1: value copy is not a member of ammonite.$sess.cmd13.BitcoinAddress

@ BitcoinAddress("") // コンパニオンのapply不可能
cmd20.sc:1: ammonite.$sess.cmd13.BitcoinAddress.type does not take parameters

@ new BitcoinAddress("") // newも不可能
cmd20.sc:1: class BitcoinAddress is abstract; cannot be instantiated
```

- <!-- .element: class="fragment" --> `fromString`以外の方法でインスタンスを生成できないケースクラスが完成
- <!-- .element: class="fragment" --> BitcoinAddressオブジェクトの値は正規表現にマッチすることが保証されている


### sealed abstract case classの不便なところ

```scala
// .getは使いたくない😢
BitcoinAddress.fromString("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS").get
```

- リテラルもOptionでラップされてしまう

- リテラルも実行時にバリデーションが走る

- 大量のボイラープレート
  - 全てのStringに対してクラスを用意するのはつらい
  - ライブラリのコーデックを手作業で書く必要がある
    - circeのAutomatic Derivationは使えない

---

### こんなものが欲しくなる

- 安全で気軽に使える

- リテラルでも便利に使える

---

## 篩(ふるい)型

<span class="gray">Refinement types</span>


### 篩型 = <span class="base-type">型</span> + <span class="predicate">述語</span>

<span class="gray">refinement type = base type + predicate</span>

- <!-- .element: class="fragment" --> <span class="base-type">Int</span> + <span class="predicate">(i => i > 0)</span>

- <!-- .element: class="fragment" --> <span class="base-type">String</span> + <span class="predicate">(s => s.nonEmpty)</span>

述語で取りうる値を制限することができる

<!-- .element: class="fragment" -->

---

## refined

Scalaでシンプルな篩型を実装したライブラリ


SBT

```scala
libraryDependencies += "eu.timepit" %% "refined" % "0.9.2"
```

Ammonite

```scala
import $ivy.`eu.timepit::refined:0.9.2`
```


```scala
import eu.timepit.refined.api.Refined
import eu.timepit.refined.auto._
import eu.timepit.refined.predicates.numeric.Positive

type Price = Int Refined Positive
```

```scala
@ 500: Price // コンパイル通る
res21: Price = 500

@ 0: Price // コンパイル通らない
cmd22.sc:1: Predicate failed: (0 > 0).
val res22 = 0: Price
            ^
Compilation Failed
```

<!-- .element: class="fragment" -->

- <!-- .element: class="fragment" --> リテラルは直接refinedの型に変換可能
- <!-- .element: class="fragment" --> コンパイル時にマクロによってバリデーションが走る


### refinedでConfigを実装

```scala
import eu.timepit.refined.W // Wはshapeless.Witnessのショートカット
import eu.timepit.refined.api.Refined
import eu.timepit.refined.auto._
import eu.timepit.refined.predicates.string.MatchesRegex
import eu.timepit.refined.types.string.NonEmptyString

type User = NonEmptyString // String Refined NonEmptyと同じ

type BitcoinAddress =
  String Refined MatchesRegex[W.`"[13][a-km-zA-HJ-NP-Z1-9]{25,34}"`.T]

final case class Config(user: User, bitcoinAddress: BitcoinAddress)
```

<!-- .element: class="fragment" -->

```scala
@ Config("yuki", "18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS") // OK
res9: Config = Config(yuki, 18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS)
```

<!-- .element: class="fragment" -->

```scala
@ Config("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS", "yuki") // コンパイルエラー
cmd10.sc:1: Predicate failed: "yuki".matches("[13][a-km-zA-HJ-NP-Z1-9]{25,34}").
val res10 = Config("18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS", "yuki")
                                                        ^
Compilation Failed
```

<!-- .element: class="fragment" -->


### リテラルを明示的に変換する

```scala
import eu.timepit.refined.refineMV
import eu.timepit.refined.api.{Refined, RefType}
import eu.timepit.refined.predicates.numeric.Positive
```

<!-- .element: class="fragment" -->

```scala
@ refineMV[Positive](500)
res7: eu.timepit.refined.api.Refined[Int, Positive] = 500
```

<!-- .element: class="fragment" -->

or

<!-- .element: class="fragment" -->

```scala
type Price = Int Refined Positive

@ RefType.applyRefM[Price](500)
res14: Price = 500
```

<!-- .element: class="fragment" -->


### 任意の値をrefinedの型に変換する

```scala
import eu.timepit.refined.refineV
import eu.timepit.refined.api.{Refined, RefType}
import eu.timepit.refined.predicates.numeric.Positive

val x = 500 // xの中身はコンパイル時に判らないとします
```

<!-- .element: class="fragment" -->

```scala
@ refineV[Positive](x)
res17: Either[String, Refined[Int, Positive]] = Right(500)
```

<!-- .element: class="fragment" -->

or

<!-- .element: class="fragment" -->

```scala
type Price = Int Refined Positive

@ RefType.applyRef[Price](x)
res18: Either[String, Price] = Right(500)
```

<!-- .element: class="fragment" -->

---

## 他ライブラリとのインテグレーション

refinedは他ライブラリとのインテグレーションが充実しており、エンコーダ・デコーダを手作業で書くことはあまりない

<!-- .element: class="fragment" -->

circe, doobie, finch, Play Framework, PureConfig, scopt, etc.

<!-- .element: class="fragment" -->


### 例: PureConfig

```scala
libraryDependencies += "eu.timepit" %% "refined-pureconfig" % "0.9.2"
```

```yaml
# application.conf
user: "yuki"
bitcoin-address: "18cpgjq4VrTRC1HUASS2xgatVyPqg6CTS"
```

```scala
import eu.timepit.refined.pureconfig._

val config = pureconfig.loadConfig[Config] // ロード時にバリデーションが走る
```

---

## Refined.unsafeApply

- バリデーション無しでrefinedの型に変換できる禁断の手

- 基本的に使うべきではない

- どうしても必要な場合のみ

---

## refined or sealed abstract case class

- ただの値か、振る舞いを持つドメインオブジェクトか

---

## まとめ

- 2種類の方法でコードの安全性を高めることができる
 - sealed abstract case class
 - 篩型 (Refinement types)

- refiendはScalaで篩型を使うためのライブラリ

- ただの値ではないならsealed abstract case class

---

## 参考資料

- [Refinement Types in Scala with refined](http://fthomas.github.io/talks/2016-05-04-refined/#1)
  
- [Refinement Types In Practice](https://kwark.github.io/refined-in-practice-bescala/#1)

---

## ありがとうございました

### なにか質問はありますか？
