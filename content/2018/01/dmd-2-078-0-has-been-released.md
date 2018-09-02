---
date: 2018-01-19
aliases:
- /2018/01/19/dmd-2-078-0-has-been-released.html
title: "DMD 2.078.0 Has Been Released【翻訳】"
tags:
- dlang
- tech
- translation
- d_blog
excerpt: "DMDのメジャーリリース、2.078.0が新年にパッケージされ、提供されていました。 完全なチェンジログはdlang.orgで見ることができ、あなたのプラットフォーム向けのコンパイラがメインダウンロードページか2.078.0 リリースディレクトリからダウンロードできます。"
---

この記事は
[DMD 2.078.0 Has Been Released – The D Blog](https://dlang.org/blog/2018/01/04/dmd-2-078-0-has-been-released/)
を自分用に翻訳したものを
[許可を得て](http://dlang.org/blog/2017/06/16/life-in-the-fast-lane/#comment-1631)
公開するものである。

ソース中にコメントの形で原文を残している。
誤字や誤訳など気になったら
[Pull requestを投げつけて](https://github.com/{{ site.github.repository_nwo }}/edit/{{ site.github.source.branch }}/{{ page.path }})
くれると喜ぶ。

---

<!-- Another major release of DMD, this time 2.078.0, has been packaged and delivered in time for the new year. See the full changelog at [dlang.org](https://dlang.org/changelog/2.078.0.html) and download the compiler for your platform either from the [main download](https://dlang.org/download.html) page or [the 2.078.0 release directory](http://downloads.dlang.org/releases/2.x/2.078.0/). -->

DMDのメジャーリリース、2.078.0が新年にパッケージされ、提供されていました。
完全なチェンジログは[dlang.org](https://dlang.org/changelog/2.078.0.html)で見ることができ、あなたのプラットフォーム向けのコンパイラが[メインダウンロード](https://dlang.org/download.html)ページか[2.078.0 リリースディレクトリ](http://downloads.dlang.org/releases/2.x/2.078.0/)からダウンロードできます。

<!-- This release brings a number of quality-of-life improvements, fixing some minor annoyances and inconsistencies, three of which are targeted at smoothing out the experience of programming in D without DRuntime. -->

このリリースでは複数のQoLの改善、軽微な問題や矛盾の修正が提供され、そのうち3つはDRuntimeを使わずにDでプログラミングをする際の問題を解決するためのものです。

<!-- ### C runtime construction and destruction -->

### Cランタイムのコンストラクションとデストラクション

<!-- D has included [static constructors and destructors](https://dlang.org/spec/class.html#StaticConstructor), both as aggregate type members and at module level, for most of its existence. The former are called in lexical order as DRuntime goes through its initialization routine, and the latter are called in reverse lexical order as the runtime shuts down. But when programming in an environment without DRuntime, such as when using [the `-betterC` compiler switch](https://dlang.org/spec/betterc.html), or using a stubbed-out runtime, static construction and destruction are not available. -->

Dには[静的コンストラクタ/デストラクタ](https://dlang.org/spec/class.html#StaticConstructor)があり、大抵aggregate typeのメンバとしてや、モジュールレベルに存在します。
コンストラクタはDRuntimeの初期化ルーチンの間にレキシカルオーダーで呼ばれ、デストラクタはランタイムが終了する時に逆順に呼ばれます。
しかし[`-betterC`コンパイラスイッチ](https://dlang.org/spec/betterc.html)を使うなどしてDRuntimeのない環境でプログラミングをする時や、ランタイムが途中でなくなった時には、静的コンストラクションとデストラクションができなくなります。

<!-- DMD 2.078.0 brings static module construction and destruction to those environments in the form of [two new pragmas](https://dlang.org/changelog/2.078.0.html#crt-constructor), `pragma(crt_constructor)` and `pragma(crt_destructor)` respectively. The former causes any function to which it’s applied to be executed before the C `main`, and the latter after the C `main`, as in this example: -->

DMD 2.078.0はそれらの環境に静的モジュールコンストラクションとデストラクションを、それぞれ`pragma(crt_constructor)`、`pragma(crt_destructor)`という[新しい2つのプラグマ](https://dlang.org/changelog/2.078.0.html#crt-constructor)の形で導入します。
前者は関数をCの`main`よりも前に、後者はCの`main`よりも後に実行させます。たとえばこのように:

<!-- **crun1.d** -->

**crun1.d**

<!-- ```d
// Compile with:    dmd crun1.d
// Alternatively:   dmd -betterC crun1.d

import core.stdc.stdio;

// Each of the following functions should have
// C linkage (cdecl).
extern(C):

pragma(crt_constructor)
void init()
{
    puts("init");
}

pragma(crt_destructor)
void fini()
{
    puts("fini");
}

void main()
{
    puts("C main");
}
``` -->

```d
// コンパイル方法:    dmd crun1.d
// または:   dmd -betterC crun1.d

import core.stdc.stdio;

// 以下すべての関数はC リンケージ(cdecl)
// になります。
extern(C):

pragma(crt_constructor)
void init()
{
    puts("init");
}

pragma(crt_destructor)
void fini()
{
    puts("fini");
}

void main()
{
    puts("C main");
}
```

<!-- The compiler requires that any function annotated with the new pragmas be declared with the `extern(C)` [linkage attribute](https://dlang.org/spec/attribute.html#linkage). In this example, though it isn’t required, `main` is also declared as `extern(C)`. The colon syntax on line 8 applies the attribute to every function that follows, up to the end of the module or until a new linkage attribute appears. -->

新しいプラグマのついた関数は`extern(C)`[リンケージ属性](https://dlang.org/spec/attribute.html#linkage)で宣言される必要があります。
この例では`main`も、必須ではありませんが`extern(C)`として宣言されています。
8行目のコロンによってそれ以降、モジュールの終わりか新しいリンケージ属性が適用されるまでのすべての関数に属性が適用されます。

<!-- In a normal D program, the C `main` is the entry point for DRuntime and is generated by the compiler. When the C runtime calls the C `main`, the D runtime does its initialization, which includes starting up the GC, executing static constructors, gathering command-line arguments into a string array, and calling the application’s `main` function, a.k.a. D `main`. -->

通常のDのプログラムでは、Cの`main`はDRuntimeのエントリポイントでありコンパイラによって生成されます。
CのランタイムがCの`main`を呼ぶと、DのランタイムはGCの開始、静的コンストラクタの実行、コマンドライン引数の文字列配列化、アプリケーションの`main`関数、つまりDの`main`の呼び出しなどの初期化を行います。

<!-- When a D module’s `main` is annotated with `extern(C)`, it essentially replaces DRuntime’s implementation, as the compiler will never generate a C `main` function for the runtime in that case. If `-betterC` is not supplied on the command line, or an alternative implementation is not provided, DRuntime itself is still available and can be manually initialized and terminated. -->

Dのモジュールの`main`に`extern(C)`がついている時はコンパイラがCの`main`関数を生成しないため、DRuntimeの実装が根本的に置き換えられます。
`-betterC`がコマンドラインに与えられていない、または置き換え実装が提供されていない場合、DRuntimeは利用可能で手動で初期化/終了できます。

<!-- The example above is intended to clearly show that the `crt_constructor` pragma will cause `init` to execute before the C `main` and the `crt_destructor` causes `fini` to run after. This introduces new options for scenarios where DRuntime is unavailable. However, take away the `extern(C)` from `main` and the same execution order will print to the command line: -->

上の例は明らかに`crt_constructor`プラグマが`init`の実行をCの`main`よりも前に発生させ、`crt_destructor`が`fini`を後に実行させることを示しています。
これによってDRuntimeが使えない時の新たな選択肢が導入されます。
しかし、`main`から`extern(C)`を取り除き、同じコマンドラインを実行すると:

<!-- **crun2.d** -->

**crun2.d**

<!-- ```d
// Compile with:    dmd crun2.d

import core.stdc.stdio;

pragma(crt_constructor)
extern(C) void init()
{
    puts("init");
}

pragma(crt_destructor)
extern(C) void fini()
{
    puts("fini");
}

void main()
{
    import std.stdio : writeln;
    writeln("D main");
}
``` -->

```d
// コンパイル方法:    dmd crun2.d

import core.stdc.stdio;

pragma(crt_constructor)
extern(C) void init()
{
    puts("init");
}

pragma(crt_destructor)
extern(C) void fini()
{
    puts("fini");
}

void main()
{
    import std.stdio : writeln;
    writeln("D main");
}
```

<!-- The difference is that the C `main` now belongs to DRuntime and our main is the D `main`. The execution order is: `init`, C `main`, D `main`, `fini`. This means `init` is effectively called before DRuntime is initialized and `fini` after it terminates. Because this example uses the DRuntime function `writeln`, it can’t be compiled with `-betterC`. -->

Cの`main`はDRuntimeに属し、mainはDの`main`であるという点が異なります。
実行順は`init`、Cの`main`、Dの`main`、`fini`となります。
つまり事実上`init`はDRuntimeが初期化される前に、`fini`は終了した後に呼ばれます。
この例はDRuntimeの関数`writeln`を使うため、`-betterC`ではコンパイルできません。

<!-- You may discover that `writeln` works if you import it at the top of the module and substitute it for `puts` in the example. However, always remember that even though DRuntime may be available, it’s not in a valid state when a `crt_constructor` and a `crt_destructor` are executed. -->

`writeln`をモジュールの1番上でインポートし、`puts`と置き換えると動作することに気づいたかもしれません。
しかしDRuntimeが使えたとしても、`crt_constructor`や`crt_destructor`が実行されるときは有効ではないことを忘れないでください。

<!-- ### RAII for `-betterC` -->

### `-betterC`のRAII

<!-- One of the limitations in `-betterC` mode has been the absence of RAII. In normal D code, `struct` destructors are executed when an instance goes out of scope. This is handled by DRuntime, and since the runtime isn’t available in `-betterC` mode, neither are `struct` destructors. With DMD 2.078.0, the _are_ in the preceding sentence [becomes _were_](https://dlang.org/changelog/2.078.0.html#raii). -->

`-betterC`モードの制限のひとつにRAIIの欠如がありました。
通常のDのコードでは、`struct`のデストラクタはインスタンスがスコープから出た時に実行されます。
これはDRuntimeによって行われ、そして`-betterC`モードではそのランタイムが使えないため、`struct`のデストラクタもまた使えません。
DMD 2.078.0で、それは[過去の話になります](https://dlang.org/changelog/2.078.0.html#raii)。

<!-- **destruct.d** -->

**destruct.d**

<!-- ```d
// Compile with:    dmd -betterC destruct.d

import core.stdc.stdio : puts;

struct DestroyMe
{
    ~this()
    {
        puts("Destruction complete.");
    }
}

extern(C) void main()
{
    DestroyMe d;
}
``` -->

```d
// コンパイル方法:    dmd -betterC destruct.d

import core.stdc.stdio : puts;

struct DestroyMe
{
    ~this()
    {
        puts("Destruction complete.");
    }
}

extern(C) void main()
{
    DestroyMe d;
}
```

<!-- Interestingly, this is implemented in terms of `try..finally`, so a side-effect is that `-betterC` mode now supports `try` and `finally` blocks: -->

面白いことに、これは`try..finally`の文脈で実装されており、そのため副作用として`-betterC`モードが`try`、`finally` ブロックをサポートするようになりました:

<!-- **cleanup1.d** -->

**cleanup1.d**

<!-- ```d
// Compile with:    dmd -betterC cleanup1.d

import core.stdc.stdlib,
       core.stdc.stdio;

extern(C) void main()
{
    int* ints;
    try
    {
        // acquire resources here
        ints = cast(int*)malloc(int.sizeof * 10);
        puts("Allocated!");
    }
    finally
    {
        // release resources here
        free(ints);
        puts("Freed!");
    }
}
``` -->

```d
// コンパイル方法:    dmd -betterC cleanup1.d

import core.stdc.stdlib,
       core.stdc.stdio;

extern(C) void main()
{
    int* ints;
    try
    {
        // ここでリソースを取得
        ints = cast(int*)malloc(int.sizeof * 10);
        puts("Allocated!");
    }
    finally
    {
        // ここでリソースを開放
        free(ints);
        puts("Freed!");
    }
}
```

<!-- Since D’s [`scope(exit)` feature](https://dlang.org/spec/statement.html#ScopeGuardStatement) is also implemented in terms of `try..finally`, this is now possible in `-betterC` mode also: -->

Dの[`scope(exit)`機能](https://dlang.org/spec/statement.html#ScopeGuardStatement)も`try..finally`の文脈で実装されているため、これも`-betterC`モードで使うことができます:

<!-- **cleanup2.d** -->

**cleanup2.d**

<!-- ```d
// Compile with: dmd -betterC cleanup2.d

import core.stdc.stdlib,
       core.stdc.stdio;

extern(C) void main()
{
    auto ints1 = cast(int*)malloc(int.sizeof * 10);
    scope(exit)
    {
        puts("Freeing ints1!");
        free(ints1);
    }

    auto ints2 = cast(int*)malloc(int.sizeof * 10);
    scope(exit)
    {
        puts("Freeing ints2!");
        free(ints2);
    }
}
``` -->

```d
// コンパイル方法: dmd -betterC cleanup2.d

import core.stdc.stdlib,
       core.stdc.stdio;

extern(C) void main()
{
    auto ints1 = cast(int*)malloc(int.sizeof * 10);
    scope(exit)
    {
        puts("Freeing ints1!");
        free(ints1);
    }

    auto ints2 = cast(int*)malloc(int.sizeof * 10);
    scope(exit)
    {
        puts("Freeing ints2!");
        free(ints2);
    }
}
```

<!-- Note that exceptions are not implemented for `-betterC` mode, so there’s no `catch`, `scope(success)`, or `scope(failure)`. -->

例外が`-betterC`モードで実装されていないため、`catch`、`scope(success)`、`scope(failure)`はありません。

<!-- ### Optional `ModuleInfo` -->

### 選択的`ModuleInfo`

<!-- One of the seemingly obscure features dependent upon DRuntime is [the `ModuleInfo` type](https://dlang.org/phobos/object.html#.ModuleInfo). It’s a type that works quietly behind the scenes as one of the enabling mechanisms of reflection and most D programmers will likely never hear of it. That is, unless they start trying to stub out their own minimal runtime. That’s when linker errors start cropping up complaining about the missing `ModuleInfo` type, since the compiler will have generated an instance of it for each module in the program. -->

DRuntimeに依存している一見あいまいな機能に[`ModuleInfo`型](https://dlang.org/phobos/object.html#.ModuleInfo)があります。
これはリフレクションを実現するために背後で働く型で、ほとんどのDプログラマは聞いたこともありません。
ランタイムをなくそうとする人を除いて。
コンパイラがプログラムのモジュールごとにそのインスタンスを生成するため、`ModuleInfo`型の不足についてリンカエラーがいきなり発生することがあります。

<!-- DMD 2.078.0 [changes things up](https://dlang.org/changelog/2.078.0.html#optional_ModuleInfo). The compiler is aware of the presence of the runtime implementation at compile time, so it can see whether or not the current implementation provides a `ModuleInfo` declaration. If it does, instances will be generated as appropriate. If it doesn’t, then the instances won’t be generated. This makes it just that much easier to stub out your own runtime, which is something you’d want to do if you were, say, [writing a kernel in D](https://dlang.org/blog/2016/06/24/project-highlight-the-powernex-kernel/). -->

DMD 2.078.0は[それを変更します](https://dlang.org/changelog/2.078.0.html#optional_ModuleInfo)。
コンパイラはコンパイル時のランタイム実装の存在を認めることで、現在の実装が`ModuleInfo`の宣言を提供しているかどうかを確認できます。
宣言されている場合、インスタンスが適切に生成されます。
宣言されていない場合、インスタンスは生成されません。
これにより、たとえば[Dでカーネルを書く](https://dlang.org/blog/2016/06/24/project-highlight-the-powernex-kernel/)などする際にランタイムをなくすことが簡単になります。

<!-- ### Other notable changes -->

### その他の変更

<!-- New users of DMD on Windows will now have an easier time getting a 64-bit environment set up. It’s still necessary [to install the Microsoft build tools](https://dlang.org/blog/2017/10/25/dmd-windows-and-c/), but now DMD will [detect the installation](https://dlang.org/changelog/2.078.0.html#vs-auto-detection) of either the Microsoft Build Tools package or Visual Studio at runtime when either `-m64` or `-m32mscoff` is specified on the command line. Previously, configuration was handled automatically only by the installer; manual installs had to be configured manually. -->

WindowsでのDMDの新しいユーザは64ビットの環境をセットアップするのが簡単になりました。
まだ[Microsoft build toolsのインストール](https://dlang.org/blog/2017/10/25/dmd-windows-and-c/)は必要ですが、DMDは`-m64`または`-m32mscoff`をコマンドラインで指定して実行された際に、Microsoft Build ToolsパッケージかVisual Studioがインストールされているか[自動検出するようになりました](https://dlang.org/changelog/2.078.0.html#vs-auto-detection)。
以前は自動設定はインストーラのみが行なっており、手動インストールの際は手動で設定する必要がありました。

<!-- DRuntime has been enhanced to allow more fine-grained control over unit tests. Of particular note is the `--DRT-testmode` flag which can be passed to any D executable. With the argument `"run-main"`, the current default, any unit tests present will be run and then `main` will execute if they all pass; with `"test-or-main"`, the planned default beginning with DMD 2.080.0, any unit tests present will run and the program will exit with a summary of the results, otherwise `main` will execute; with `"test-only"`, `main` will not be executed, but test results will still be summarized if present. -->

ユニットテストに対してより粒度の細かいコントロールができるようにDRuntimeが強化されました。
注目すべき点はDの実行ファイルに渡せる`--DRT-testmode`フラグです。
現在のデフォルト引数`"run-main"`では、存在するユニットテストが全て実行され、それが通った後に`main`が実行されます。
DMD 2.080.0からデフォルトになる予定の`"test-or-main"`が渡された場合、ユニットテストが存在すればそれが実行されて、プログラムはその結果のサマリーを出して終了します。
ユニットテストが存在しないなら`main`が実行されます。
`"test-only"`の場合`main`は実行されず、テストが存在すればその結果が要約されます。

<!-- ### Onward into 2018 -->

### 2018年に向けて

<!-- This is the first DMD release of 2018. We can’t wait to see what the next 12 months bring for the D programming language community. From everyone at the D Language Foundation, we hope you have a very Happy New Year! -->

これは2018年最初のDMDのリリースです。
今後12ヶ月間Dプログラミング言語コミュニティで何が起こるか非常に楽しみです。
D言語財団の皆様、明けましておめでとうございます！