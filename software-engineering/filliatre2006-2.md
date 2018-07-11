<!--
$size: 4:3
$theme: basic
page_number: true
*page_number: false
prerender: true
-->

<style>
.author {
  position: absolute;
  right: 0px;
  top: 380px;
  text-align: right;
}
</style>

# Type-Safe Modular Hash-Consing

J.-C. Filliatre and S. Conchon:

The 2006 ACM SIGPLAN Workshop on ML (ML 2006)

Portland, Oregon, 16th September 2006.

<div class="author">2018年 6月 14日<br>@tyabu12</div>

---

# 2 節. hash-consing ライブラリ

>This section presents an **OCAML** hash-consing library.

この章では **OCAML** の hash-consing ライブラリを示す.

>We first detail its design and interface, then its practical use and finally its implementation.

まず設計とインターフェイス, 次に実用的用途, 最後に実装の順で詳しく述べる.

---

## 2.1 節. 設計とインターフェイス (Design and interface)

>The main idea is to tag the hash-consed values with unique integers.

本旨は hash-consing された値に, 一意の整数をタグ付けすることである.

>First, it introduces a type distinction between the values that are hash-consed and those that are not.

これは1つ目に, タグ付けは hash-consing されている値とされていない値の違いが区別できるようになる.

>Second, these tags can be used to build efficient data structures over hash-consed values, such as hash tables or balanced trees.

2つ目に, タグ付けは hash-consing された値について効率的なデータ構造 (ハッシュテーブルや平衡木) を作るために使うことができる.

---

タグ付けのために, 以下のレコード型を導入する:

```ocaml
type α hash_consed = private {
  node : α;
  tag : int;
  hkey : int;
}
```

>The field `node` contains the value itself and the field `tag` the unique integer.

`node` フィールドは自身の値を持ち, `tag` フィールドは一意の整数を持つ.

>The `private` modifier prevents the user from building values of type hash consed manually, yet allowing field access and pattern matching over this type.

`private` 修飾はユーザーに対して, 手動で hash-consing した値を生成することを防止するが, フィールドへのアクセスやパターンマッチングは許可する.

---

>The field `hkey` stores the hash key of the hash-consed term; it is justified in the implementation section below.

`hkey` フィールドは hash-consing された term のハッシュキーを保持する; これは後の実装の節 (2.3 節) で正当化されている.

---

>The library consists in a functor Make taking a type equipped with equality and hashing functions as argument and returning a data structure for hash-consing tables.

ライブラリは, 等価性とハッシュ関数を備えた型を引数に取り, hash-consing テーブルのためのデータ構造を返す, `Make` ファンクタから構成される.

>The input signature is the following:

入力シグネチャ (モジュールのインタフェース) は以下である:

```ocaml
module type HashedType = sig type t
  val equal: t * t -> bool
  val hash: t -> int
end
```

---


>with the implicit assumption that the hashing function is consistent with the equality function i.e. hash x = hash y as soon as equal (x, y) holds.

ハッシュ関数は, `equal (x, y)` が成り立つならば直ちに等式関数, すなわち `hash x = hash y` と一致する, という暗黙の仮定を伴う.

---


>Then the functor looks like:

次にファンクタはこのようになる:

```ocaml
module Make(H : HashedType) : sig type t
  val create : int -> t
  val hashcons : t -> H.t -> H.t hash_consed
end
```

>The datatype t for hash-consing tables is abstract.

hash-consing テーブルためのデータ型 `t` は抽象的なものである.

---


### 補足: (OCaml における) ファンクタ

- 別のモジュールで, パラメータ化されるモジュールのこと

- 関数が別の値 (引数) によってパラメータ化された値であるのと同じようなもの

例: OCaml 標準ライブラリの `Set` ファンクタを用い, `int` の集合を扱うモジュール `Int_set` を定義

```ocaml
module Int_set = Set.Make (struct
                             type t = int
                             let compare = compare
                           end);;
```

参考: [https://ocaml.org/learn/tutorials/modules.ja.html](https://ocaml.org/learn/tutorials/modules.ja.html)

<!-- 補足おわり. -->

---

>`create n` builds a new hash-consing table with initial size `n` .

`create n` は初期サイズ `n` の新しい hash-consing テーブルを生成する.

>As for traditional hash tables, this size is somewhat arbitrary (the table will be resized when necessary).

従来のハッシュテーブルでは, このサイズはやや任意である (必要な際はテーブルはリサイズされる) .

---

> Then, if $h$ is a hash-consing table, `hashcons` $h$ $t$ builds the hash-consed value for a given term $t$.

そして, $h$ を hash-consing テーブルとすると, `hashcons` $h$ $t$ は与えられた term $t$ について hash-consing した値を生成する.

---

>In practice, there is also a function `iter` to iterate over all the terms contained in a hash-consing table and a function `stats` to get statistics on a table (number of entries, biggest bucket length, etc.).

実際には, hash-consing テーブルに含まれるすべての term を反復するための `iter` 関数と, テーブルの統計 (入力数, 最大バッファ長など) を取得する `stats` 関数もある.

---

>There is also a non-functorial version of the library, based on structural equality and the generic *OCAML* hash function, but this is clearly less useful (these two functions do not exploit the sharing of sub-terms).

構造的等価性と一般的な **OCAML** ハッシュ関数に基づいた, ファンクタでないバージョンのライブラリもあるが, 明らかに有用性に欠ける (これらの2つの関数は sub-terms の共有が利用できない) .

→ ファンクタありでは抽象的なデータ型 `t` を使うため, sub-terms を共有できる (？)

---

>Due to the private type hash consed, maximal sharing is now easily enforced by type checking (as long as we use a single hash-consing table, of course).

`private` 型の hash-consing により, maximal sharing は型検査で容易に実施できる (もちろん単一の hash-consing テーブルを使う限りで) .

*maximal sharing*: 二つの値が構造的等価であると, 直ちにそれらの値が共有されると言う性質

---

>As a consequence, the following holds:

結果として, 以下が成り立つ:

$$x=y \Leftrightarrow x==y \Leftrightarrow x.tag = y.tag$$

- (=) : 構造的等価性. データが格納されたアドレスを比較した時の等しさ
- (==) : 物理的等価性. データの値を比較したときの等しさ

---

## 2.2 節. 使用方法 (Usage)

>Before entering the implementation details of the hash-consing library, let us demonstrate its use on our running example.

hash-consing ライブラリの実装の詳細に入る前に, 例を使ったデモをする.

---

>First, we need to adapt our datatype for λ-terms to interleave the hash consed nodes with our original definition:

まず最初に, データ型に λ-terms を適用する必要があり, hash-consing された nodes にオリジナルの定義に割り込む.

```ocaml
(*
type α hash_consed = private {
  node : α;
  tag : int;
  hkey : int;
}
*)

type term = term_node hash_consed
and term_node =
  | Var of int
  | Lam of term
  | App of term * term
```

---

>It is important to notice that this modification is not intrusive at all (from the syntactic point of view) and applies to mutually recursive types as well.

この変更は, (構文的な観点から) 全く煩わしいものではなく, 同様に共通して再帰型になることに注目することが重要である。

---

>Indeed, the definition of a set of types such as

実際, 型の集合の定義は

`type` $t_1 =$ <$\mathrm{definition_1}$>
.
.
.
`and` $t_n =$ <$\mathrm{definition_n}$>

---

>simply needs to be turned into

単純に書き換えると

`type` $t_1 = t_1$\_`node hash_consed and` $t_1$\_`node` $=$ <$\mathrm{definition_1}$>
.
.
.
`and` $t_n = t_n$\_`node hash_consed and` $t_n$\_`node` $=$ <$\mathrm{definition_n}$>

>and we notice that the type definitions <$\mathrm{definition_i}$> are left unchanged.

型定義 <$\mathrm{definition_i}$> は変化しないままなことがわかる.

---

>Then we can equip the type term node with suitable equality and hashing function in order to apply the hash-consing functor.

つぎに hash-consing ファンクタを適用するために, 適切な等価性とハッシュ関数をもつ type term node を用意する.

>As we did in Section 1, equality may safely use physical equality on sub-terms and thus runs in O(1).

1節でしたように, 等価性は確実に sub-terms の物理的等価性を使っよいため, O(1) で実行できる.

>The hashing function can also be improved to run in constant time by making use of the hash keys for the sub-terms (the `hkey` field) or equivalently their tags (the `tag` field).

ハッシュ関数は, sub-terms (`hkey` フィールド) やタグ (`tag` フィールド) のハッシュキーを利用することで, 定数時間の実行に改良できる.

---

>Altogether, we get the following module definition

結局, 以下のモジュール定義が得られる

```ocaml
module Term_node = struct
  type t = term_node
  let equal t1 t2 = match t1, t2 with
    | Var i, Var j -> i == j
    | Lam u, Lam v -> u == v
    | App (u1,v1), App (u2,v2) -> u1 == u2 && v1 == v2
    | _ -> false
  let hash = function
    | Var i -> i
    | Lam t -> abs(19 * t.hkey + 1)
    | App (u,v) -> abs (19 * (19 * u.hkey + v.hkey) + 2)
end
```

---

>We get an implementation of hash-consing tables for this type via a simple functor application:

単純なファンクタの適用により, この型のための hash-consing テーブルの実装が得られる:

```ocaml
module Hterm = Hashcons.Make(Term_node)
```

---

>Then we can define a global hash-consing table and smart constructors for Var, Lam and App performing hash-consing, as follows:

つぎに, グローバルの hash-consing テーブルと, `Var`, `Lam`, `App` のための hash-consing を行う *スマートコンストラクタ* を定義でき, 以下となる:

```ocaml
let ht = Hterm.create 251
let var n = Hterm.hashcons ht (Var n)
let lam u = Hterm.hashcons ht (Lam u)
let app (u,v) = Hterm.hashcons ht (App (u,v))
```

---

>These “constructors” have the expected types, namely

これらの **"コンストラクタ"** は期待される型をもち, すなわち

```ocaml
val var : int -> term
val lam : term -> term
val app : term * term -> term
```

>and thus, from this point, the user can simply ignore the hash- consing mechanics that is performed underneath when building terms.

これにより, 以降ユーザーは terms を生成する際に水面下で行われる hash-consing 機構を完全に無視することができる.

---

>We can now exploit the unique tags to build efficient data structures over terms.

ここで terms 上の効率的なデータ構造を作るために, 一意のタグを利用できる.

>For instance, these tags define a total ordering over terms and we can use it to build balanced binary trees containing terms.

例えば, タグは terms 上で全順序を定義しており, これを利用することで terms を含んだ平衡二分木を作ることができる.

- 全順序: すべての2つの要素 (ここでは term) が比較可能
- 平衡二分木: 木の高さを自動的にできるだけ小さく維持しようとする二分木

---

>Using functors from OCAML’s standard library, we get
implementations of sets of terms and dictionaries indexed by terms as follows:

**OCAML** の標準ライブラリのファンクタを使うことで, terms の集合と terms によりインデックスされた辞書の実装を得られ, 以下に示す$^3$:


```ocaml
module OrderedTerm = struct
  type t = term
  let compare x y =
    Pervasives.compare x.tag y.tag
end
module TermSet = Set.Make(OrderedTerm)
module TermMap = Map.Make(OrderedTerm)
```

>$^3$ Pervasives.compare is to be preferred to a subtraction due to possible integer overflows.

$^3$ Pervasives.compare は整数のオーバーフローの可能性がある引き算よりも良い.

---

>The modules Set and Map have fairly good performances.

`Set` と `Map` モジュールはかなりパフォーマンスが高い.

>But since elements are actually (tagged with) integers, we can build even more efficient data structures such as Patricia trees [12].

しかし, 要素は実際には整数であるため, 基数木のようなさらにいっそう効率的なデータ構造が作れる [12] .

>Our library comes with Patricia trees-based sets and dictionaries specialized for hash-consed values.

我々のライブラリは, hash-consing した値のための, 基数木に基づいた集合と辞書を備えている.

---

>Similarly, we could build efficient hash tables indexed by terms, using the field hkey (or even the tag) as an immediate hash value.

同様に, 即時ハッシュ値として `hkey` (または `tag` でも) フィールドを使うことで, terms によりインデックスされた効率的なハッシュテーブルを作ることができる.

---

## 2.3 節. 実装 (Implementation)

>Up to now, we have solved issues 1 and 2 that were listed Section 1 (a distinct type for hash-consed values and efficient data structures based on physical equality).

今までで, 1節で一覧にした課題1と2 (hash-consing された値の区別と, 物理的等価性に基づく効率的なデータ構造) は解決した.

>We now address issues 3 and 4.

次に課題3と4に取り組む.

- 課題3. 生きていない term を解放する GC が作れない
- 課題4. (筆者らの) ハッシュコンシング関数が, ハッシュテーブルに値が存在しない場合, 変数に紐付いたハッシュ値を2度計算してしまう

---

>Regarding time and space inefficiency related to the use of hash tables from **OCAML**’s standard library, the first improvement is to build a custom hash table where buckets are simply *lists* of values, and not mapping of values to themselves, saving one pointer for each value.

**OCAML** の標準ライブラリのハッシュテーブル使用に関する時間と空間の非効率性については, 1つ目の改良としては, 単純な値のリストなバッファで, 値をリストに紐付けず, 各値に対するポインタを持つような, カスタムハッシュを生成することである.

---

>Then the next improvement is to write a single lookup-or-insert function that computes the hash key only once

次の改良は, ハッシュキーを一度だけ計算する, 単独の lookup-or-insert 関数を書くことである.

>Finally, the last improvement is to record the hash key (in the field hkey which is exposed to the user) to avoid recomputing it when resizing the table (if required).

最後の改良は, (ハッシュ) テーブルのリサイズが必要な際に, (ユーザーから見える `hkey` フィールドにある) ハッシュキーを記録し, 再計算を避けることである.

---

>Regarding the ability for the garbage collector to reclaim the hash-consed values that are not referenced anymore (from anywhere else than the hash-consing table), the solution is to use weak pointers.

(hash-consing テーブルのどこからも) もう参照されなくなった hash-consing された値を回収するためのガーベージコレクタの能力に関しての解決方法は, weak pointer を使うことである.

---

>A weak pointer is precisely a reference that does not prevent the garbage collector from erasing the value it points to.

weak pointer は, 厳密にはガーベージコレクタが参照先の値を消すことを防げないという参照である.

>Said otherwise, a memory cell may be collected as soon as it is referenced only by weak pointers.

言い換えると, メモリセルは weak pointer のみによって参照されていると, すぐに解放されるだろう.

---

>The **OCAML** standard library provides arrays of weak pointers [1] where the access operation may return either Some x when there is some available element x and None otherwise (which means that the element has been reclaimed by the GC at some point in the past).

**OCAML** 標準ライブラリは weak pointer の配列を提供しており, アクセス演算は $x$ が存在している要素なら `Some` $x$ , それ以外なら `None` (これはつまり, すでにどこかの時点で要素はGCによって解放されていることを意味する) を返す.

---

>Combining the ideas of a custom hash table and the use of weak pointers, we get a weak hash table$^4$ , that is a hash table where the buckets are arrays of weak pointers.

カスタムハッシュテーブルと weak pointer の使用の組み合わせにより, 我々は weak ハッシュテーブル$^4$ , つまりバッファが weak pointer の配列であるハッシュテーブル, を得る.

>$^4$ The **OCAML** standard library also provides weak hash tables and we borrowed most code from this implementation.

$^4$ **OCAML** 標準ライブラリは weak ハッシュテーブルも提供しており, 我々はそのほとんどの実装を拝借した.

---

>The beginning of our hash-consing functor is thus as follows:

我々の hash-consing ファンクタの始めは結果として以下になる:

```ocaml
module Make(H : HashedType) = struct
  type t = {
    table : H.t hash_consed Weak.t array;
    ...
}
```

>We omit the other fields that contain irrelevant data related to the resizing heuristics.

ヒューリスティックなリサイズに関する重要でないデータを含んでいる, その他のフィールドを省略している.

>The insertion and resizing code is standard and we give the main ideas only.

挿入とリサイズのコードは一般的なものであり, 主旨のみを述べる.

---

>The sole difference with respect to **OCAML** weak hash tables is that we do no need to (re)compute the hash key when inserting or resizing since it is contained in the data itself.

**OCAML** の weak ハッシュテーブルとの唯一の違いは, データ自身にハッシュキーが格納されているため, 挿入やリサイズ時にハッシュキーを (再) 計算する必要がないという点である. 

```ocaml
let rec resize t =
  increase the size of the main array
  and redistribute the elements in the new buckets

and add t d =
  let index = d.hkey mod (Array.length t.table) in

  lookup for an empty slot in the bucket t.data.(index)

  if found then insert d
  else increase the bucket and insert d

  if the limit is reached then resize t
```

---

>Then the main `hashcons` operation consists in a single lookup-or-insert function. The pseudo-code is as follows:

メインの `hashcons` 演算は単独の lookup-or-insert 関数からなる. 擬似コードは以下である:

```ocaml
let hashcons t d =
  let hkey = H.hash d in
  let index = hkey mod (Array.length t.table) in

  lookup in the bucket t.data.(index)

  for a value v such that H.equal v.node d

  if found then return v
  else
    let n = { hkey = hkey; tag = newtag (); node = d } in
    add t n;
    n
```

>where `newtag` is a function returning distinct integers.

ただし `newtag` は, 違う整数を返す関数である.

---

>It may seem rather inefficient to first scan the bucket for an equal value already hash-consed and then, in case of a failure, to scan it again for an empty slot to insert it.

すでに hash-consing された等価な値がないかバッファをまず調べ, 見つからなかった際, 値の挿入 (する空の場所) のために再度バッファを調べるのは, やや非効率にみえる.

---

>We could indeed remember any empty slot encountered while scanning for an equal value and then use it for insertion, if any.

等価な値を探す間に, 遭遇した空の場所がもしあるならその場所を記憶し, 挿入時に利用できるだろう.

>But in practice this is not worth maintaining this information.

しかし実際には, (空の場所の) 情報を維持する価値はない.

>Indeed, the buckets are quite small (we start with 3 elements buckets and add only 3 new slots each time we increase them).

バッファは非常に小さい (我々は 3 要素のバッファから始め, 増やす際も 3 要素のみずつ増やしている) .
