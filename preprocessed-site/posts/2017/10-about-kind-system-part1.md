---
title: About kind system of Haskell (Part 1)
headingBackgroundImage: ../../img/post-bg.jpg
headingDivClass: post-heading
subHeading: 種の仕組みとそれに付随する言語拡張について
author: mizunashi-mana
date: August 23, 2017
...
---

Haskellには種(kind)という仕組みがあります。大雑把に言ってしまえば、「型の型」を実現する仕組みです。この仕組みについて、あまり情報が出回っていないようなので、解説記事を残しておこうと思います。

この記事は、[Ladder of Functional Programming](http://lambdaconf.us/downloads/documents/lambdaconf_slfp.pdf) ([日本語訳](http://qiita.com/lotz/items/0d68c8440d1f362d0c32))の**FIRE LUBLINE(ADVANCED BEGINNER)**を対象に、種の仕組みとそれに付随するGHC言語拡張やパッケージを紹介するものです。

なお、特に断らない限り、対象としてGHC8系を設定しています。`stack`を使ってる方は`resolver`をLTS Haskell 8以降に設定しておくことを推奨します。

## 基本的な種の仕組み

### 種に慣れる

私たちは良きHaskellerなので、トップレベルの関数には以下のように型注釈をつけます:

```haskell
increment :: Int -> Int
increment n = n + 1
```

この`increment`という関数は、`Int`型の値を受け取って、1を加算した`Int`型の値を返します。なので、型システムによってそのような型として検証されます:

```haskell
increment (1 :: Int)            -- ok => (2 :: Int)
increment ("str" :: String)     -- error!
increment (1 :: Double)         -- error!
increment (1 :: Int) (2 :: Int) -- error!
```

種も大体同じようなものですが、種は型の形式が正しいかを検証する仕組みです。例えば、以下のデータ型を見てください:

```haskell
data Id a = Id a
```

このデータ型宣言は、

* **型**の名前空間上に、`Id`という名前の**型**コンストラクタ
* **値**の名前空間上に、`Id`という名前の**値**コンストラクタ

を作ります[^notice-type-data-c-name]。値コンストラクタ`Id`は`a -> Id a`という型をしています[^notice-type-constructor]。つまり、値コンストラクタ`Id`は、`Id a`という型の値を作れる唯一のコンストラクタになります。そして、値コンストラクタは、何らかの値を受け取らなければ`Id a`を構成できないことが、型システムによって保証できます。

[^notice-type-data-c-name]: 名前を一緒にする風習がややこしいですが、Haskellはそういう文化があるので慣れるしかないですね
[^notice-type-constructor]: この型注釈上の`Id`は型コンストラクタであることに注意してください！

さて、型コンストラクタ`Id`の方はどうでしょうか？ データ型宣言からは、型コンストラクタ`Id`はそのままでは型になれず、何らかの型を受け取る必要があるように見えます。ですが、それは誰が保証してくれるのでしょうか？ さらには、値コンストラクタは受け取った型`a`によって、その型が決まります。例えばもし、`a`に`Maybe`などの型コンストラクタを入れてしまった場合、値コンストラクタ`Id`の型は`Maybe -> Id Maybe`という一見おかしな型になってしまいます。このように`a`に`Maybe`を渡すことは実際にはできません。一体どういうメカニズムで、このような一見おかしなものが弾かれるのでしょうか？ もう、みなさんお気付きだと思いますが、これを保証する仕組みが種なのです。

値コンストラクタ`Id`が型`a -> Id a`という型を持つように、型コンストラクタ`Id`は種`* -> *`を持ちます。この種がどういう意味を持つのかを見る前に、まずは種を分析するためのツールを用いて、型の種を見てみましょう。そのツールとは、GHCiの`kind`コマンドです。では、使ってみます:

```haskell
>>> data Id a = Id a
>>> -- 値コンストラクタIdの型を分析
>>> :type Id
Id :: a -> Id a
>>> -- 型コンストラクタIdの種を分析
>>> :kind Id
Id :: * -> *
```

ここで、`type`コマンドは**値**の名前空間を、`kind`は**型**の名前空間を取っていることに注意してください。`:kind 1`というように`kind`コマンドに値を分析させることはできませんし、`:type Int`というように`type`コマンドに型を分析させることはできません。では、この`kind`コマンドで、他の幾つかの型の種もみてみます:

```haskell
>>> :kind Int
Int :: *
>>> :kind Maybe
Maybe :: * -> *
>>> :kind Either
Either :: * -> * -> *
>>> :kind Either Int
Either Int :: * -> *
```

なんとなく、種の意味が分かってきましたか？ 基本的には、`*`が型を、`* -> *`は型をとって型を返す型コンストラクタを、`* -> * -> *`は型を二つとって型を返す型コンストラクタを表しているようです。型コンストラクタには部分適用もできるようです。ただ、単純に全てが`* -> * -> ...`という形の種になるわけではありません。次のようなデータ型を見てみてください:

```haskell
>>> data AppInt m = AppInt (m Int)
>>> :type AppInt
AppInt :: m Int -> AppInt m
>>> :kind AppInt
AppInt :: (* -> *) -> *
```

種に`()`が付きました。この型コンストラクタ`AppInt`は、単純に型をとるようなものではなく、型を一つとる型コンストラクタによって、型が作られます。実際に、値は以下のように作れます:

```haskell
>>> :type AppInt $ Just 1
AppInt $ Just 1 :: AppInt Maybe
>>> :type AppInt [1, 2]
AppInt [1, 2] :: AppInt []
>>> :type AppInt $ Right 1
AppInt $ Right 1 :: AppInt (Either a)
>>> :type AppInt $ Left True
AppInt $ Left "str" :: AppInt (Either Bool)
```

ちょっと不思議な型ですね。型コンストラクタ`AppInt`は`* -> *`にマッチする型コンストラクタしか受け取れません。試してみましょう:

```haskell
>>> :kind AppInt Int

<interactive>:1:3: error:
    • Expected kind ‘* -> *’, but ‘Int’ has kind ‘*’
    • In the first argument of ‘AppInt’, namely ‘Int’
      In the type ‘AppInt Int’
>>> :kind AppInt Either

<interactive>:1:3: error:
    • Expecting one more argument to ‘Either’
      Expected kind ‘* -> *’, but ‘Either’ has kind ‘* -> * -> *’
    • In the first argument of ‘AppInt’, namely ‘Either’
      In the type ‘AppInt Either’
```

エラー文がそのままですね。

* `AppInt Int`の方は「`* -> *`を期待していたが、受け取った型`Int`の種は`*`ですよ」と言っています。
* `AppInt Either`の方は「`* -> *`を期待していたが、受け取った型コンストラクタ`Either`の種は`* -> * -> *`ですよ」と言っています。

このようにして、型注釈によって受け取る値を制限できるように、種によって受け取る型を制限できるわけです。何となく、種がどういうものかは分かっていただけたでしょうか？ では、種がどのような意味を持っているのかを、ちゃんと見ていきましょう。

### 種の意味と種推論

Haskellには、標準で二種類の種があります。それは

* `*`という種
* `k1`、`k2`を何かしらの種とした時、`k1 -> k2`という形をした種

の二つです。今まで見てきたように、

* `*`は、データ型
* `k1 -> k2`は、`k1`の種を持つ型を受け取り`k2`の種を持つ型を返すような型コンストラクタ

をそれぞれ表します。値コンストラクタから作った`AppInt [1, 2]`が一つの値であったように、型コンストラクタから作った`AppInt Maybe`なども一つのデータ型です。

`k1 -> k2`は右結合で解釈されます。なので、`* -> * -> *`は、実際には`* -> (* -> *)`と同じです。なので、右に括弧が付く場合は省略が可能ですが、左に付く場合は省略ができません。つまり、`(* -> *) -> *`と`* -> * -> *`は別物になります。

さて、ここで一つ重要な型コンストラクタを紹介しておきましょう。それは関数型コンストラクタ`(->)`です。型コンストラクタが`()`で囲まれて、新しい表記方法が出てきたように思えますが、どうか落ち着いてください。通常の関数において(値の世界において)、私たちは中置演算子を`()`で囲むことで、通常の関数として扱うことができました。型注釈上でも同じようなことができます。

```haskell
>>> :type id :: a -> a
id :: a -> a :: a -> a
>>> :type id :: (->) a a
id :: (->) a a :: a -> a
```

上の二つの型は、表記は違えど同じ型を表しています。実は私たちは、中置がデフォルトの型コンストラクタを自分で作ることもできます。それには`TypeOperators`拡張を使わなければいけませんが。ちょっと作ってみましょう:

```haskell
>>> :set -XTypeOperators
>>> data a + b = Coproduct (Either a b)
>>> -- 型コンストラクタ
>>> :kind (+)
(+) :: * -> * -> *
>>> -- 値コンストラクタ
>>> :type Coproduct 
Coproduct :: Either a b -> a + b
>>> :type Coproduct $ Right True
Coproduct $ Right True :: a + Bool
>>> :type Coproduct $ Left True
Coproduct $ Left True :: Bool + a
>>> :kind (+) Int
(+) Int :: * -> *
>>> :kind (+) Int Bool
(+) Int Bool :: *
>>> :kind Int + Bool
Int + Bool :: *
```

残念ながらセクションは使えませんが、その他は大体中置演算子と同じで、部分適用などもできます。関数型コンストラクタ`(->)`も、`(+)`と似たようなものです。種を見てみましょう:

```haskell
>>> :kind (->)
(->) :: * -> * -> *
>>> :kind (->) Int
(->) Int :: * -> *
>>> :kind (->) Int Bool
(->) Int Bool :: *
>>> :kind Int -> Bool
Int -> Bool :: *
```
**追記**: GHC 8.2.1では、`:kind (->)`の表示結果が、`TYPE q -> TYPE r -> *`というものに変更されたようです。この表記に関しては、[続編][part2-link]の方で解説します。今は、`* -> * -> *`と大体同等のものであると思ってもらって構わないので、以降では`(->) :: * -> * -> *`であるとして話を進めていきます。宜しくお願いします。

`(->)`は二つの型を取り、データ型を返します。そのデータ型とは関数型です。例えば、`Int -> Maybe Bool`(関数表記では`(->) Int (Maybe Bool)`)は`Int`型の値を受け取り`Maybe Bool`型の値を返す関数の型を表しているのでしたね。関数型コンストラクタは二つの引数の種を`*`に制限しています。なので、`Maybe -> Int`といったような型注釈は書けません。これは、型コンストラクタが値を持たないことに反しません。

さて、次のようなデータ宣言を考えてみましょう:

```haskell
data Id a = Id a
data AppInt m = AppInt (m Int)
```

一番最初に見たデータ宣言です。

* 型コンストラクタ`Id`の種は`* -> *`、値コンストラクタ`Id`の型は`a -> Id a`
* 型コンストラクタ`AppInt`の種は`(* -> *) -> *`、値コンストラクタ`AppInt`の型は`m Int -> AppInt m`

になるのでした。このデータ宣言には、特に種に関する情報を書いているわけではありません。`Id`と`AppInt`の種は、どうやって定まったのでしょうか？ 実は、種に関する推論によって、これらの種は決定されるのです。

Haskellでは、型注釈なしの関数は、型推論されて型が決まります。種でも同じように、推論が行われます。`(->)`の種は`* -> * -> *`であったことを思い出してください。値コンストラクタは関数ですから、そのパラメータは`*`という種を持つことになります[^notice-data-constructor-kind]。

* `Id`の方を考えてみると、値コンストラクタから`a`型は`*`という種であることが分かります。
* `AppInt`の方も同じく`Int`が`*`という種であることと`m Int`が`*`であることから、`m`は`* -> *`と推論されます。

[^notice-data-constructor-kind]: データコンストラクタが値から構築されることを考えれば、当たり前ですね

このようにして、`Id`と`AppInt`の種は自動的に決まったわけです。では、推論に頼らず種を指定することはできるのでしょうか？ 残念ながら、Haskellの標準システムでは、推論に頼らずデータ型コンストラクタの種を指定することはできません。次のようなデータ宣言を考えてみましょう:

```haskell
data App f a = App (f a)
data TaggedData t = TaggedData
```

* `App`型コンストラクタのパラメータ`f`と`a`は、それぞれ何かしらの種`k`に対して`k -> *`、`k`という形をしていればいいはずですが、実際には`* -> *`、`*`という型になります。
* `TaggedData`型コンストラクタのパラメータ`t`も、どのような種であってもいいはずですが、`*`となります。

このように、標準のHaskellでは、デフォルトで`*`が設定されており、確定しないような種は`*`として扱われます。なので、型コンストラクタでタグ付けしたデータ型を作るといったことはできません。

### 型と種の評価順序

上では、種の意味と種推論について話しました。種推論は、種を推論してくれるわけですが、正しく私たちが思ったことを推論してくれるわけではなく、表現できない型コンストラクタもありました。さて、その他にも推論が失敗するようなケースもあります。以下をみてください:

```haskell
data Ill m a = Ill (m a) m
```

型コンストラクタ`Ill`のパラメータ`m`と`a`の種はどうなるでしょうか？ 実は、このような場合につじつまが合う種はありません。もしこのデータ宣言が成り立つなら、値コンストラクタ`Ill`の型は`Ill :: m a -> m -> Ill m a`になりますが、この場合`m`が型コンストラクタなことは明白なので型コンストラクタに紐づく値が存在することになりますし、`(->)`の種にも合いません。もう一つ、推論が失敗する面白いケースがあります。以下のデータ宣言を考えてみましょう:

```haskell
data Inf a b = Inf (a b) (b a)
```

ここで、`a`の種を`k0 -> *`とおくと、`b`の種は`k0`になるわけですが、`b`も`a`を受け取っているのでやはり`k1 -> *`というような形をしているはずです。このように、両方に辻褄が合うような種を探していくと、永遠に同じ操作の繰り返しになり終わりません。このような場合も種の推論は失敗し、コンパイルエラーになります。

また、型コンストラクタに型を渡す場合も、種がちゃんと合うかを確認し、種が合わない場合コンパイルエラーになるのでした。このように、コンパイルする際は、種の推論や種の検証を行い、辻褄が合うかを保障し、種を確定させる必要があります。

さて、Haskellではもう一つコンパイル時に行われる重要な評価があります。それは、型に関する評価です。Haskellのプログラム中の型を推論し、ちゃんと型の辻褄が合っているかも評価しなければなりません。これらの二つの評価はGHCでは別々に行われます。これは当たり前のように思えるかもしれませんが、種に関して考えるときは常に意識しなければなりません。次のプログラムをみてください:

```haskell
-- TestKind.hs

module TestKind where

f :: Maybe -> Int
f _ = 0

g :: Int -> Bool
g '0' = True
g _   = False
```

これをコンパイルすると以下のエラーが出されます:

```bash
$ stack ghc -- -Wall TestKind.hs
[1 of 1] Compiling TestKind         ( TestKind.hs, TestKind.o )

TestKind.hs:5:6: error:
    • Expecting one more argument to ‘Maybe’
      Expected a type, but ‘Maybe’ has kind ‘* -> *’
    • In the type signature:
        f :: Maybe -> Int
```

ここでは、「`Maybe`は`* -> *`という種を持っているが、`(->)`が期待している種は`*`だ」と言っています。ですが、上のプログラムにはもう一つおかしな点があります。それは関数`g`の型注釈です。関数`g`の受け取る値は`Int`型のはずですが、実際には`Char`型の値を受け取っています。ただし、関数`g`の型注釈の種に関しては何の問題もありません。

GHCでは、種と型の検査は別々に行われるという話をしました。実は、さらにこの二つの間には評価順序があります。まず種の検査を行ってから、型の検査が行われるようになっているのです。種の検査に失敗すれば型の検査は行われません。これらは、`:type`コマンドや`:kind`コマンドにも影響するので注意が必要です。`:kind`コマンドは種の評価を行いますが、型の評価は行いません。あまり、`:kind`コマンド上で種の検査が通って型の検査が通らないといった場面には遭遇しないかもしれないですが、これは心に留めておくと良いでしょう。

### この章のまとめ

この章では、基本的な種の仕組みを紹介しました。種というのは、標準では二つ存在するのでした。それは、以下のものです:

* `*`: データ型を表す種
* `k1 -> k2`: `k1`の種を持つ型を受け取り、`k2`の種を持つ型を返す、型コンストラクタを表す種

また、データ宣言において種は推論され、確定しない場合はデフォルトで`*`を用いるのでした。また、種と型の評価はそれぞれ別々に行われ、種の評価の後に型の評価が行われることも学びました。

以降では、Haskell標準の種の仕組みを拡張する、幾つかの重要なGHC拡張について話していきましょう。

## 種に付随したGHC拡張

### 種注釈

Haskell標準では、データ宣言において、型コンストラクタの種は種推論によって決定するのでした。このため、表現できない型コンストラクタがあることも話しました。これは、不便な場合があります。`* -> *`の種を持つ型コンストラクタをタグとした、データ型を表現することができないのはもちろんですが、そもそも複雑なデータ型の場合に注釈としての種が欲しかったり、推論に任せずそもそも型の種を明示的に宣言したい場合があるのです。これは、Haskellにおいてトップレベルの関数の型注釈を行うことが、良い風習とされているのと同じですね。例えば、以下のデータ宣言を考えてみてください:

```haskell
data Complex a b c = Complex (a (Maybe (b c)))
```

このような場合に、パッとそれぞれのパラメータの種を考えることは出来るでしょうか？ 出来る人もいるかもしれませんが、混乱してしまう人もいるでしょう。もし、次のような注釈があればどうでしょうか？

```haskell
data Complex (a :: * -> *) (b :: * -> *) (c :: *) = Complex (a (Maybe (b c)))
```

これならば、値コンストラクタの型について深く考えなくても、それぞれのパラメータがどういう種を持つ型なのかはすぐに分かるようになりますし、どういう意図で書いたのかが明白です。何の注釈もない場合、`b`には`Maybe`を渡せばいいのか、それとも具体的な型を渡せばいいのか少し考える必要がありますが、注釈がある場合には種の読み方が分かっていればすぐ分かります。残念ながら、Haskellの標準でこのような注釈を書くことはできません。そこで、`KindSignatures`拡張の出番になります。

`KindSignatures`はその名の通り、種の注釈を可能にするGHC拡張です。この拡張により、データ宣言や型シノニムなどでも種注釈が書けるようになります。以下のプログラムをみてください:

```haskell
{-# LANGUAGE KindSignatures #-}

data App f a = App (f a)
type FlipApp a (f :: * -> *) = App f a
```

このプログラムでは、`App`の方の種は見た目からすぐ分かります。しかし、`FlipApp`の方はどうでしょうか？ 上記の例では、すぐそばに`App`のデータ宣言があるから分かりますが、`App`と`FlipApp`の宣言が別々の場所にあることを想像してみてください。Haskellでは、型コンストラクタ(型関数)には`f`や`m`、`t`をメタ変数として使う文化があるので、それから推測することは可能ですが、明確に知りたい場合には実装を見にいく必要が出てくる場合もあるでしょう。しかし、きちんと種注釈が書いてあれば、混乱を避けることができます。これが一つの種注釈の魅力です。

また、種注釈を明示することで、推論に頼らず種の制約を書きたい場合もあります。よくあるケースは`GADTs`拡張を併用する場合です。`GADTs`拡張については、今回は詳しく扱いませんので、`GADTs`を知らない人は以下は読み飛ばしてください。

`GADTs`との併用では、次のような種注釈を書く場合があります:

```haskell
data GadtsSample :: * -> * where
  GadtsSample :: a -> GadtsSample a
```

`GADTs`のスタイルは、値コンストラクタの型を明示的に書くため、型コンストラクタのパラメータ名を明記する必要がありません。型コンストラクタはその種が分かればいいですし、値コンストラクタはその型が分かれば問題ないからです。通常のデータ宣言では、型コンストラクタと値コンストラクタの型がごっちゃになっているため、このように種の注釈と型の注釈を完全に分離することは困難です。もちろん`GADTs`において、パラメータ名に種注釈をつけていく書き方も許容されています。上の表記は、次の表記と同一です:

```haskell
data GadtsSample (a :: *) where
  GadtsSample :: a -> GadtsSample a
```

ただし、`GADTs`では型コンストラクタのパラメータ名は特に意味を持たないことに注意してください。値コンストラクタの型注釈は、特に型コンストラクタのパラメータ名に名前を合わせる必要はありません:

```haskell
data GadtsSample (a :: *) where
  GadtsSample :: b -> GadtsSample b -- aを使わなくてもいい！
```

このため、一番最初に提示したような、型コンストラクタにはその種注釈を、値コンストラクタにはその型注釈をそれぞれ書くというスタイルを好む人も多くいます。これも、一つの`KindSignatures`拡張の魅力と言えるでしょう。なにより重要なことは、`GADTs`では値コンストラクタの型を明示しないといけないため、意図しない型コンストラクタへの適用を、誤って型注釈に書いてしまう可能性が、通常のデータ宣言より高くなります。種注釈をつけることで、型コンストラクタの意図している種を明示することにより、意図していなかった型コンストラクタの使用法が、種推論によってすりぬけてしまうことを防ぐことができます。

### 種多相

さて、種注釈を行えるようにする`KindSignatures`拡張の他に、もう一つ重要な拡張があります。それが、種多相を行えるようにする拡張です。「基本的な種の仕組み」の章で紹介した、以下のデータ宣言を思い出してください:

```haskell
data App f a = App (f a)
```

標準では、型コンストラクタ`App`は、`(* -> *) -> * -> *`という種になるのでした。しかしながら、`f :: k -> *`、`a :: k`という形をしていれば、どんな種でもいいはずだという話は覚えていますか？ `f a :: *`になればいいのですから、わざわざ`*`に強めてしまう必要はありません。そこで、デフォルトの`*`まで具体化をせずに、抽象的に「何かしらの`k`の種において、`f :: k -> *`、`a :: k`という形をしていれば良い」という情報を残すようにするのが、`PolyKinds`拡張、種多相の基本的な考え方です。`PolyKinds`拡張を有効にする前とした後での`App`型コンストラクタの種を見てみましょう:

```haskell
>>> data App f a = App (f a)
>>> :kind App
App :: (* -> *) -> * -> *
>>> -- PolyKinds拡張の有効化
>>> :set -XPolyKinds
>>> data App f a = App (f a)
>>> :kind App
App :: (k -> *) -> k -> *
```

`PolyKinds`拡張を有効にした後では、デフォルトで具体化が必要ない部分は、`k`という形のまま残っているのが見て取れます！ 私たちHaskellerは、多相関数で、具体化された型ではなく任意の型についてマッチするような関数を書くことに慣れています。多相関数の場合、具体化されていない型を型変数と呼ぶのでした。種の場合は種変数といったところでしょう。種多相は、GHCの標準パッケージ`base`において、様々なところで用いられています。有名なものとしては、`Data.Proxy`にある`Proxy`データ型がそうです。その種を見てみましょう:

```haskell
>>> import Data.Proxy
>>> :kind Proxy
Proxy :: k -> *
>>> :type Proxy
Proxy :: forall k (t :: k). Proxy t
```

少し`Proxy`の値コンストラクタの型注釈が分かりにくいですが、`Proxy`値コンストラクタは特に引数を取らず`Proxy t`という値になります。このように実体(値)を持たない型パラメータを幽霊型と言ったりします。`Proxy`型コンストラクタは、どんな種でも良いので何かしらの幽霊型`t :: k`をとり、`Proxy t`というデータ型に成ります。例えば、型コンストラクタを幽霊型として付属させることも可能です:

```haskell
>>> :type Proxy :: Proxy Maybe
Proxy :: Proxy Maybe :: Proxy Maybe
```

不思議なデータ型ですね。種多相がなくても、`Proxy`データ型のような幽霊型をパラメータに持つ型コンストラクタを作ることはできます。しかし、種によってそれぞれ型コンストラクタを用意しなければなりません。今回の例のように、種多相を使えば、一つのデータ宣言によって様々な種の型に対応できるようになるのが、魅力的です。また、`PolyKinds`拡張は、一緒に`KindSignatures`拡張も有効にします。これらを組み合わせることで、明示的に多相化された種の注釈を書くことも可能です。それは、以下のようになります:

```haskell
>>> data BiTagged (tag1 :: k) (tag2 :: k) = BiTaggedData
>>> :kind BiTagged
BiTagged :: k -> k -> *
>>> :type BiTaggedData
BiTaggedData :: forall k (tag2 :: k) (tag1 :: k). BiTagged tag1 tag2 
```

このように種変数を使った種注釈も可能です。これを活用すれば、より強力な型コンストラクタを作ることも可能になるでしょう。

### この章のまとめ

この章では、種に付随する、二つの重要なGHC拡張を紹介しました。

`KindSignatures`拡張は、種注釈を行えるようにする拡張でした。種注釈によってこれまで表現できなかった型コンストラクタが作れるようになるのはもちろんのこと、分かりやすさや種推論による混乱を避けるための明示的な注釈として、この拡張はとても便利でした。

もう一つの`PolyKinds`拡張は、種多相を可能にしてくれる拡張でした。標準では、全ての種は具体化され、曖昧なところは全て標準の種`*`によって具体化されます。しかし、この拡張によりデフォルトの動作を、抽象化されたまま型変数として残す動作に切り替えることができるようになります。これによって、それぞれの種に対しての具体的な型コンストラクタを用意する必要も無くなります。また、種注釈を多相的に行うことも可能になるのでした。

## まとめ

今回は、種の基本概念と、種に関連するGHC拡張を紹介しました。

続編[^information-part2]では、`*`の他の幾つかの種と、種とは別の型の分類についての紹介などを踏まえた、幾つかの種に関連する話題について、話したいと思います。

**追記**: [続編][part2-link]を書きました。続きが気になる方は、読んでみてください。

[^information-part2]: 多分9月中に出す。きっとね！ 続きが気になる人は、期待しないで待っててください。

## 参考文献

* [Haskell2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/haskell.html): Haskell2010の仕様書です。主に標準の仕組みを紹介する際に参照しました。
    - [4.1.1 Kinds](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-640004.1.1): 種の定義が書いてあります。
    - [4.6 Kind inference](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-970004.6): 種推論について書かれています。
* [GHC 8.0.2 Users Guide](https://downloads.haskell.org/~ghc/8.0.2/docs/html/users_guide/): 主な種に関する参考資料としてとGHC拡張についての資料として参考にしました。
    - [10.11 Kind polymorphism and Type-in-Type](https://downloads.haskell.org/~ghc/8.0.2/docs/html/users_guide/glasgow_exts.html#kind-polymorphism-and-type-in-type): GHCにおいての種推論などの、種に関することが総括してあります。
    - [10.15.4 Explicitly-kinded quantification](https://downloads.haskell.org/~ghc/8.0.2/docs/html/users_guide/glasgow_exts.html#explicitly-kinded-quantification): `KindSignatures`拡張の概要が書かれています。
* [What I Wish I Knew When Learning Haskell - Promotion](http://dev.stephendiehl.com/hask/#promotion): 簡単にですが幾つか種に関する話題がまとまっています。あんまり参考にしていませんが、リンクとして置いておきます。
* [GHC Wiki - Commentary/Compiler/Kinds](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/Kinds): この記事のストーリーを決める際に参照しました。続編では、このページを元にしたもう少し踏み込んだトピックも扱う予定です。

[part2-link]: 13-about-kind-system-part2.html