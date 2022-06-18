# [日本語訳]正規表現マッチング: VMによるアプローチ -Regular Expression Matching: the Virtual Machine Approach- (By Russ Cox)
> *Original Article*: https://swtch.com/~rsc/regexp/regexp2.html
> 
> This article is translated by [@mox692](https://twitter.com/mox692). Please contact me if any problem.

* 記事中のソースコードは[こちら](https://swtch.com/~rsc/regexp/)から参照できます.


## Introduction
最も広く使われているバイトコードインタプリタまたは仮想マシンを挙げてください。サンのJVM？AdobeのFlash? .NETとMono?Perl？Python？PHP？これらはすべて確かに人気があるが、それらをすべて合わせたよりも広く使われているものがある。そのバイトコードインタプリタとは、ヘンリー・スペンサーの正規表現ライブラリとその多くの子孫たちである。

それは awk や egrep (そして今ではほとんどの grep) で使われている最悪の場合の線形時間 NFA および DFA ベースの戦略と、ed, sed, Perl, PCRE, Python など他のほとんどの場所で使われている最悪の場合の指数時間バックトラックの戦略です。

この記事では、.NET と Mono が CLI バイトコードにコンパイルされたプログラムを実行する仮想マシンを実装する異なる方法であるように、テキストマッチングバイトコードにコンパイルされた正規表現を実行する仮想マシンを実装する2つの異なる方法として2つの戦略を紹介します。

正規表現マッチングを特殊なマシンの実行と見なすことで、新しいマシン命令を追加（実装）するだけで、新しい機能を追加することができるようになります。特に、正規表現のサブマッチ命令を追加することで、aabbbbに対して(a+)(b+)をマッチングした後、括弧付きの(a+)(しばしば \1 または $1 と呼ばれる)がaaにマッチしたこと、(b+)がbbbbにマッチしたことをプログラムが発見できるようにすることができるのです。サブマッチはバックトラックVMでも非バックトラックVMでも実装可能である。(これを行うコードは1985年までさかのぼりますが、この記事が最初に書かれた説明だと思います)。

## 正規表現VM
まず始めに、正規表現仮想マシン(Java VMを想像してください)を定義します。VMは1つ以上のスレッドを実行し、それぞれが正規表現プログラム（正規表現命令のリスト）を実行します。各スレッドは実行中、プログラムカウンタ（PC）と文字列ポインタ（SP）という2つのレジスタを保持します。

正規表現VMにおける命令は下記のようになります:

* *char c* : SPが指す文字がcでない場合、このスレッドを停止します。そうでなければ、SPを次の文字に進め、PCを次の命令に進める。
* *match* : matchした場合、このスレッドを停止する。
* *jmp* :  x にある命令にジャンプする（その後PC がxを指すように設定する）。
* *split x, y* : スプリット実行：x と y の両方で継続する。一方のスレッドはPC xで継続し、もう一方のスレッドはPC yで継続する（両位置への同時ジャンプのようなもの）。

VMは、PCがプログラムの先頭を、SPが入力文字列の先頭を指している1つのスレッドが実行されている状態からスタートします。スレッドを実行するには、VMはスレッドのPCが指す命令を実行し、その命令を実行するとスレッドのPCは次に実行する命令を指すように変化する。ある命令（failed charまたはmatch）がスレッドを停止させるまで繰り返す。正規表現は、いずれかのスレッドがマッチを見つけた場合、文字列にマッチします。
正規表現をバイトコードにコンパイルすることは、正規表現の形式に応じて再帰的に進行します。正規表現には、aのような1文字、e1e2という連結、e1|e2という交互、e?（0または1）、e*（0または複数）、e+（1または複数）の繰り返しの4種類があることを前回の記事で思い出してください。連結は、2つの部分式のコンパイル形式を連結する。交替式は、どちらかの選択が成功するように分割を使用します。ゼロまたは1の繰り返し e? は、空文字列を含む交互表現のようにコンパイルするために分割を使用します。ゼロ以上の繰り返し e* と1回以上の繰り返し e+ は、e にマッチするか繰り返しを抜けるかを選択するためにスプリットを使用します。  
実際のバイトコードは次のようになります

<img width="381" alt="スクリーンショット 2022-05-29 19 34 36" src="https://user-images.githubusercontent.com/55653825/170863677-48d88728-00d3-48b4-8548-ed984fd4b251.png">  

正規表現全体がコンパイルされると、生成されたコードは最後のマッチング命令で終了する。

例として、正規表現 a+b+ は次のようにコンパイルされます。

```
0   char a
1   split 0, 2
2   char b
3   split 2, 4
4   match

```

aabという入力に対して上記の正規表現VMを実行する場合、このようにプログラムを実行することができます。  

<img width="630" alt="スクリーンショット 2022-05-29 19 36 41" src="https://user-images.githubusercontent.com/55653825/170863753-6dcf298f-b371-497f-8114-7967f9394cfa.png">

この例では、現在のスレッドが終了するまで新しいスレッドの実行を待ち、作成された順番にスレッドを実行します（古いものから）。これはVMの仕様では要求されていません。スレッドをスケジュールするのは実装次第です。他の実装では、スレッドを別の順番で実行したり、スレッドの実行をインターリーブしたりすることもできます。

## CによるVMインターフェース
この記事の残りの部分では、VMの一連の実装をCのソースコードを使って説明する。正規表現プログラムはInst構造体の配列として表現され、C言語では次のように定義される。

```c
enum {    /* Inst.opcode */
    Char,
    Match,
    Jmp,
    Split
};

struct Inst {
    int opcode;
    int c;
    Inst *x;
    Inst *y;
};
```
このバイトコードは、第1回の記事のNFAグラフの表現とほぼ同じである。バイトコードをNFAグラフの機械命令へのエンコードと見ることもできるし、NFAグラフをバイトコードの制御フローグラフと見ることもできる。それぞれの見方によって、考えやすくなることが異なる。この記事では、どちらを先に見たとしても、機械語命令という見方を取り上げます。
各VMの実装は、プログラムと入力文字列を引数に取り、プログラムが入力文字列にマッチしたかどうかを示す整数を返す関数となる（マッチしない場合は0、マッチした場合は0以外）。

```c
int implementation(Inst *prog, char *input);
```

## 再起的なバックトラッキング実装
最も単純なVMの実装は、スレッドを直接モデル化しません。その代わり、新しいスレッドを探索する必要があるとき、prog と入力関数パラメータが最初のスレッドの pc と sp の初期値として倍増するという事実を利用して、再帰的に自分自身を呼び出す。

```c
int
recursive(Inst *pc, char *sp)
{
    switch(pc->opcode){
    case Char:
        if(*sp != pc->c)
            return 0;
        return recursive(pc+1, sp+1);
    case Match:
        return 1;
    case Jmp:
        return recursive(pc->x, sp);
    case Split:
        if(recursive(pc->x, sp))
            return 1;
        return recursive(pc->y, sp);
    }
    assert(0);
    return -1;  /* not reached */
}
```

上のバージョンは非常に再帰的で、Lisp、ML、Erlangのような再帰を多用する言語に 慣れているプログラマには快適なはずです。ほとんどのCコンパイラは、"return recursive(...);" という文（いわゆる末尾呼び出し）を関数の先頭へのgotoに書き換えるので、上記のコンパイルは以下のようなものになります。

```c
int
recursiveloop(Inst *pc, char *sp)
{
    for(;;){
        switch(pc->opcode){
        case Char:
            if(*sp != pc->c)
                return 0;
            pc++;
            sp++;
            continue;
        case Match:
            return 1;
        case Jmp:
            pc = pc->x;
            continue;
        case Split:
            if(recursiveloop(pc->x, sp))
                return 1;
            pc = pc->y;
            continue;
        }
        assert(0);
        return -1;  /* not reached */
    }
}
```

なお、このバージョンでは、ケースSplitで、pc->yを試す前にpc->xを試すために、まだ1つの（非尾部の）再帰が必要であることに注意してください。

この実装は、Henry Spencerのオリジナルのライブラリや、Java、Perl、PCRE、Pythonのバックトラック実装、オリジナルのed、sed、grepのエッセンスである。この実装は、バックトラックが少ないときは非常に高速に動作するが、前回見たように多くの可能性を試さなければならないときはかなり遅くなる。

(a*)*のような正規表現はコンパイルされたプログラムに無限ループを引き起こす可能性がありますが、このVMの実装ではそのようなループを検出することができません。この問題を解決するのは簡単ですが(詳細は記事の最後を参照)、バックトラックは我々の焦点ではないので、この問題は単に無視することにします。

## 再起しないバックトラッキングの実装
再帰的バックトラック実装は、1つのスレッドを死ぬまで実行し、その後、スレッドを生成した順序の逆順（新しいものが先）に実行します。実行待ちのスレッドは明示的にエンコードされず、コードが再帰するたびにCのコールスタックに保存されるpcとspの値で暗黙的に処理されます。実行待ちのスレッドが多すぎると、Cコールスタックがオーバーフローし、性能の問題よりもデバッグが困難なエラーが発生する可能性があります。この問題は、各文字の後に新しいスレッドを作成する（上の例ではa+がそうだった）.*のような繰り返しの際に最もよく起こります。これは、スタックサイズに制限があり、スタックオーバーフローをチェックするハードウェアがないことが多い、マルチスレッドプログラムにおける実際の懸念事項です。

C のスタックをオーバーフローさせないようにするには、代わりに明示的なスレッドスタックを維持する必要があります。手始めに、Thread 構造体と単純なコンストラクタ関数 thread を定義します。

```c
struct Thread {
    Inst *pc;
    char *sp;
};

Thread thread(Inst *pc, char *sp);
```

そして、VMの実装は、その準備完了リストからスレッドを取り出して、完了まで実行することを繰り返す。1つのスレッドが一致を見つけたら、早期に停止することができる。残りのスレッドを実行する必要はない。すべてのスレッドがマッチを見つけられずに終了した場合、マッチは存在しない。実行待ちのスレッド数には単純な制限があり、制限に達した場合はエラーが報告される。

```c
int
backtrackingvm(Inst *prog, char *input)
{
    enum { MAXTHREAD = 1000 };
    Thread ready[MAXTHREAD];
    int nready;
    Inst *pc;
    char *sp;

    /* queue initial thread */
    ready[0] = thread(prog, input);
    nready = 1;
    
    /* run threads in stack order */
    while(nready > 0){
        --nready;  /* pop state for next thread to run */
        pc = ready[nready].pc;
        sp = ready[nready].sp;
        for(;;){
            switch(pc->opcode){
            case Char:
                if(*sp != pc->c)
                    goto Dead;
                pc++;
                sp++;
                continue;
            case Match:
                return 1;
            case Jmp:
                pc = pc->x;
                continue;
            case Split:
                if(nready >= MAXTHREAD){
                    fprintf(stderr, "regexp overflow");
                    return -1;
                }
                /* queue new thread */
                ready[nready++] = thread(pc->y, sp);
                pc = pc->x;  /* continue current thread */
                continue;
            }
        }
    Dead:;
    }
    return 0;
}
```

この実装は、recursiveやrecursiveloopと同じ動作をする。Cのスタックを使わないだけである。2つのスプリットケースを比較してみましょう。

```c
/* recursiveloop */
case Split:
    if(recursiveloop(pc->x, sp))
        return 1;
    pc = pc->y;
    continue;

	
/* backtrackingvm */
case Split:
    if(nready >= MAXTHREAD){
        fprintf(stderr, "regexp overflow");
        return -1;
    }
    /* queue new thread */
    ready[nready++] = thread(pc->y, sp);
    pc = pc->x;  /* continue current thread */
    continue;
```

バックトラックはまだ存在するが、backtrackingvm のコードは recursiveloop が暗黙的に行っていたことを明示的に行わなければならない：再帰の後に使われる PC と SP を保存して、現在のスレッドが失敗したときに試せるようにする。明示的に行うことで、オーバーフローチェックを追加することができる。
