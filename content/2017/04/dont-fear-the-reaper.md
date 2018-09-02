---
date: 2017-04-13
aliases:
- /2017/04/13/dont-fear-the-reaper.html
title: "死神を恐れないで - GCについて知る【翻訳】"
tags:
- dlang
- tech
- translation
- dlang_gc_series
- d_blog
excerpt: ‪"Dには、こんにち使用されている多くの言語と同じように、すぐに使えるガベージコレクタがあります。‬‪GCのことを心配せずにかけて、それを最大限活用できるタイプのソフトウェアが多くあります。‬しかしGCは不利な点を持ち、ガベージコレクションが望ましくないシナリオがたしかにあります。‬"
---

この記事は

[Don’t Fear the Reaper – The D Blog](http://dlang.org/blog/2017/03/20/dont-fear-the-reaper/)

の翻訳である。
翻訳したものを公開する
[許可をもらえた](http://dlang.org/blog/2017/03/20/dont-fear-the-reaper/#comment-1355)
ので、ここに公開する。
誤字や誤訳などを見つけたら今すぐ
[Pull request](https://github.com/{{ site.github.repository_nwo }}/edit/{{ site.github.source.branch }}/{{ page.path }})だ!

---

Dには、こんにち使用されている多くの言語と同じように、すぐに使えるガベージコレクタがあります。
GCのことを心配せずにかけて、それを最大限活用できるタイプのソフトウェアが多くあります。
しかしGCは不利な点を持ち、ガベージコレクションが望ましくないシナリオがたしかにあります。
そのようなシチュエーションで、言語はそれを一時的に無効化したり、完全に回避する方法を提供します。

ガベージコレクションのポジティブな影響を最大化し、ネガティブな面を最小化するために、
DでGCがどのように動作するかの下地を持つことが必要です。
スタート地点として最適なのは
[dlang.orgのガベージコレクションのページ](http://dlang.org/spec/garbage.html)
でしょう、DのGCの原理を概説し、それを使うためのヒントを提供します。
この投稿はそのページで提供されている情報を拡大するシリーズの最初にするつもりです。

今回は、非常に基本的なこと、GCアロケーションを引き起こす言語機能にフォーカスしていきます。
将来の投稿で必要な時にGCを無効化する方法や、非決定論的性質に対処するのに役立つイディオムを紹介します
(GCで管理されたオブジェクトのデストラクタのリソースの管理など)。

まず、Dのガベージコレクタについて理解することは、それがアロケーション中の、
アロケートに利用できるメモリがないときのみ動作するということです。
それは後ろで居座ったりせず、ちらほらヒープをスキャンしゴミ集めをしたりしません。
この知識はGC管理されたメモリを効率的に使うコードを書くことにおいての基本です。
下の例を見てみましょう:

```d
void main() {
    int[] ints;
    foreach(i; 0..100) {
        ints ~= i;
    }
}
```

これは`int`の動的配列を宣言し、
[foreachレンジループ](https://dlang.org/spec/statement.html#foreach-range-statement)
内で0から99までの数字を追加するためにDの追加演算子を使っています。
素人目には追加演算子が配列に追加する値のスペースをアロケートするためにGCヒープを使用していることはわかりません。

DRuntimeの配列の実装は馬鹿ではありません。
この例の中では、それぞれの値でいちいち100回のアロケーションは行われません。
さらなるメモリが必要なとき、実装は要求されたのより多くのスペースをアロケートします。
この特殊なケースで、実際にどれくらいのアロケーションが行われたかをDの動的配列とスライスの
[`capacity`](https://dlang.org/phobos/object.html#.capacity)プロパティを使って特定できます。
これはアロケーションが必要になる前に配列が持つことができる要素の合計値を返します。

```d
void main() {
    import std.stdio : writefln;
    int[] ints;
    size_t before, after;
    foreach(i; 0..100) {
        before = ints.capacity;
        ints ~= i;
        after = ints.capacity;
        if(before != after) {
            writefln("Before: %s After: %s",
                before, after);
        }
    }
}
```

**DMD 2.073.2**でコンパイルしてこれを実行した時、メッセージは6回プリントされ、
これは合計6回のGCアロケーションがループの中であったことを意味します。
つまりGCがゴミを収集する機会が6回あったということです。
この小さな例では、それはほぼ確実に起こりません。
もしこのループがもっと大きなプログラムの一部だった場合、全体にGCのアロケーションがあれば、確実に起こるでしょう。

付け加えると、これは`事前`と`事後`の値を調べることの参考になります。
これを行うと0、3、7、15、31、63、127というシーケンスが見られます。
最終的に、`ints`には100の値が入り、次のアロケーションの前に27以上の追加できるスペースを持ち、
シーケンスの値から推定するに、255になるでしょう。
これはDRuntimeの実装の詳細ですが、リリースで微調整や変更され得ます。
配列やスライスがGCによってどのように管理されるかのさらなる詳細は、Steven Schveighofferの
このトピックに関する[素晴らしいアーティクル](https://dlang.org/d-array-article.html)を見てください。

それで、6回のアロケーションは、単純で小さなループにおいてもGCがそれを予測不能な長さ休止させる6回の機会です。
一般的に、それはループがコードのホットパートかと、GCヒープから合計どれくらいのメモリが
アロケートされているかに依存する問題になりえます。
しかし、それは必ずしもコードのその部分でGCを無効化する理由にはなりません。

CやC++のような、独創的なストックGCのついていない言語と同じように、
出来る限り前方でアロケートすることによって全体のパフォーマンスをより良くしたり、
内側のループでのアロケーションを最小化することを多くのプログラマは学びます。
それは本当の諸悪の根源ではなく、**ベストプラクティス**と呼ばれる傾向のある時期尚早な最適化のタイプの1つです。
DのGCがメモリがアロケートされるときにのみ走ることを考えると、
パフォーマンスへの潜在的影響を軽減するシンプルな方法として同じ戦略が適用できます。
こちらは例を書き換えたひとつです:

```d
void main() {
    int[] ints = new int[](100);
    foreach(i; 0..100) {
        ints[i] = i;
    }
}
```

今や6つのアロケーションは1つになりました。
GCが実行される唯一の機会は内側のループの前です。
これは実際にループに入る前に少なくとも100の値のスペースをアロケートし、それらすべてを0で初期化します。
`new`のあと配列の長さは100になりますが、ほとんど確実に追加のキャパシティがあります。

配列の`new`する代わりの方法があります:`reserve`関数です:

```d
void main() {
    int[] ints;
    ints.reserve(100);
    foreach(i; 0..100) {
        ints ~= i;
    }
}
```

これは少なくとも100の値のメモリをアロケートしますが、返した時点で配列はまだ空(`length`プロパティは0を返す)
で、デフォルトの初期化はされません。
ループが100個の値のみを追加することを考えると、アロケートが行われないことが保証されます。

`new`と`reserve`に加えて、明示的なアロケーションのために`GC.malloc`を呼ぶことができます。

```d
import core.memory;
void* intsPtr = GC.malloc(int.sizeof * 100);
auto ints = (cast(int*)intsPtr)[0 .. 100];
```

[配列リテラル](https://dlang.org/spec/arrays.html#dynamic-arrays)
はたいていアロケートをします。

```d
auto ints = [0, 1, 2];
```

これは配列リテラル`enum`が使われた時にもいえます。

```d
enum intsLiteral = [0, 1, 2];
auto ints1 = intsLiteral;
auto ints2 = intsLiteral;
```

`enum`はコンパイル時にのみ存在し、メモリアドレスを持ちません。
その名前はその値の同義語です。
それをどこで使っても、その値をその場にコピペするようなものです。
`ints1`と`ints2`の両方がそのように宣言されたのとちょうど同じようにアロケーションを引き起こします:

```d
auto ints1 = [0, 1, 2];
auto ints2 = [0, 1, 2];
```

ターゲットが
[静的配列](http://dlang.org/spec/arrays.html#static-arrays)
の場合配列リテラルはアロケートをしません。
また、文字列リテラル(Dで文字列は内部では配列です)はルールの例外です。

```d
int[3] noAlloc1 = [0, 1, 2];
auto noAlloc2 = "No Allocation!";
```

連結演算子は常にアロケートをします:

```d
auto a1 = [0, 1, 2];
auto a2 = [3, 4, 5];
auto a3 = a1 ~ a2;
```

Dの[連想配列](https://dlang.org/spec/hash-map.html)は独自のアロケーション戦略をもちますが、
あなたはアイテムが追加されたり潜在的に削除されたりした時にアロケートされることを期待します。
連想配列は2つのプロパティ、`key`と`value`を公開し、これらは配列をアロケートし、
それぞれのアイテムのコピーで埋めるものです。
イテレーション中にもとの連想配列を変更することを望む場合、
またはそのアイテムがソートされている必要があるとき、
または連想配列と独立して操作されるとき、これらのプロパティはまさにちょうど必要なものです。
そうでなければ、これらはGCに過度の負荷をかける余計なアロケーションです。

GCが走るとき、スキャンが必要なメモリの合計の量はそのガベージコレクションがどれだけかかるかを決めます。
小さいことは、良いことです。
不必要なアロケーションを避けることは誰にも害を及ぼさず、別の良い軽減戦略です。
そのようなことをするために
[Dの連想配列は3つのプロパティを提供します](http://dlang.org/spec/hash-map.html#properties):
`byKey`、`byValue`、`byKeyValue`です。
これらはそれぞれlazyにイテレートされるforwardレンジを返します。
これらは実際に連想配列のアイテムを参照し、それがイテレート中に変更されないためアロケートを行いません。
レンジの詳細については、Ali Çehreliの
[Programming in D](http://ddili.org/ders/d.en/index.html)の
[Ranges](http://ddili.org/ders/d.en/ranges.html)と
[More Ranges](http://ddili.org/ders/d.en/ranges_more.html)のチャプターを見てください。

ローカルスタックフレームのポインタを持ち歩く必要があるデリゲートまたは関数リテラルである
[クロージャ](https://dlang.org/spec/function.html#closures)は、アロケートをすることがあります。
[Garbage Collectionのページ](http://dlang.org/spec/garbage.html)
の最後にリストアップされているアロケートをする言語機能は
[アサーション](http://dlang.org/spec/expression.html#AssertExpression)です。
アサーションはDのクラスベースヒエラルキーの一部である`AssertError`を投げる必要があるため、
それが失敗した時アロケートします(今後の投稿でクラスがGCとどのようにやり取りをするか見ていきます)。

ところで、Dの標準ライブラリの[Phobos](https://dlang.org/phobos/index.html)というのがあります。
かつて、PhobosのほとんどはGCアロケーションをほぼ気にせず実装されており、
それがGCアロケーションが望ましくないシチュエーションでのPhobosの使用を難しくしていました。
しかし、GCの使用についてより保守的にする大きな取り組みが開始されました。
いくつかの関数はlazyなレンジで動作するようになり、
他のものは呼び出し側の提供するバッファをとるように書きなおされ、
さらにいくつかは内部的に不必要なアロケーションを避けるよう再実装されました。
結果として標準ライブラリはよりGCフリーのコードに対して素直になりました
(が、おそらくまだライブラリの隅にまだ改装されていないものがあります
 — [PRを受け付けています](https://github.com/dlang/phobos))。

GCの使用の基礎を見てきたので、このシリーズの次の投稿ではGCをオフにし特定のセクションがGCフリーである
ことを確かめる、言語とコンパイラが提供するツールについて紹介します。

この記事について協力してくれたGuillaume PiolatとSteven Schveighofferに感謝します。