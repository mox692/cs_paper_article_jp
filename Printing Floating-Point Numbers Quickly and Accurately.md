# [日本語訳]浮動小数点数の高速・高精度な表示方法 -Printing Floating-Point Numbers Quickly and Accurately-
> *Original Article*: https://legacy.cs.indiana.edu/~dyb/pubs/FP-Printing-PLDI96.pdf
> 
> This article is translated by [@mox692](https://twitter.com/mox692). Please contact me if any problem.

* 数式が多い章は、多分原文を読んだ方が正確だし理解が速そうです.

## Abstruct
要旨本論文では，浮動小数点数を自由形式と固定形式の両方で表示するための高速かつ正確なアルゴリズムを紹介する．自由形式モードでは、このアルゴリズムは、最短で正しく丸められた出力文字列を生成し、読み込んだときに、丸めるときにどのようにタイを分割しても、同じ数に変換される。固定形式モードでは、アルゴリズムは、重要でない末尾の桁を示す特別な#マークを使用して、正しく丸められた出力文字列を生成します。どちらのモードでも、アルゴリズムは、浮動小数点数を効率的にスケーリングする高速推定器を使用します。

## 概要
> [要旨]
> ·2進表現を10進表現に直す効率的なアルゴリズムを
> ·自由形式 -> 最短で(余分な桁を表示する事なく)10進表現を得ることが目標
> ·固定形式 -> 指定された桁しゅうで小数を表示することが目的
本論文では、入力ベース（通常は2の累乗）から出力ベース（通常は10）への浮動小数点数の変換という出力問題を解決する、効率的な浮動小数点数表示アルゴリズムを紹介します。このアルゴリズムは、自由形式と固定形式の2種類の出力をサポートしています。自由形式の出力では、正確な浮動小数点入力ルーチンで読み込んだときに同じ内部浮動小数点数に変換される、最短で正しく丸められた出力文字列を生成することが目的です[1]。
例えば、3/10は0.2999999ではなく0.3として表示されます。入力がニアレストモードであると仮定すると、このアルゴリズムは、例えばIEEE不偏丸めなどのあらゆるタイブレイク戦略に対応します。丸めなど、あらゆる同点解消法に対応します。
固定フォーマット出力では、重要なポイントを超える「ゴミ桁」なしに、与えられた桁数で正しく丸められた出力を生成することが目標です。例えば、1/3の浮動小数点表現は、最初の7桁だけが重要であるにもかかわらず、0.3333333148と表示されるかもしれません。このアルゴリズムでは、特別な#マークを使って重要でない末尾の桁を示し、1/3が0.3333333##と表示されるようにします。これらのマークは、数桁の精度しかないような非正規化された数値を表示するときや、大きな桁数で表示するときに便利です。

彼らのアルゴリズムは、自由形式と固定形式の両方の出力をサポートしますが、入力のタイブレーキング戦略を適切に処理せず、有意なゼロと重要でない末尾のゼロを区別しません。さらに、実用上許容できないほど遅い。変換アルゴリズムで重要なステップは、浮動小数点数を出力基底の適切なべき乗でスケーリングすることです。Steele と White の反復アルゴリズムでは、O(|log x|) の高精度整数演算を必要とするため、非常に大きな振幅と非常に小さな振幅を持つ浮動小数点数に対して性能が低くなってしまいます。我々は、常に正しいべき乗の1以内の推定値を生成する効率的な推定器を開発し、我々のアルゴリズムは、わずか数回の高精度整数演算ですべての浮動小数点数をスケーリングします。セクション 2 では、厳密な有理数演算の観点から、基本的な浮動小数点印刷アルゴリズムを開発する。セクション3では、高精度整数演算と我々の効率的なスケーリングファクトリステメータを用いたアルゴリズムの実装を説明する。セクション4では、このアルゴリズムを拡張し、固定形式の出力を扱えるようにし、#マークを導入した。セクション5では、我々の結果を要約し、関連する仕事について議論する。

## 基本的なアルゴリズムの概要
基本アルゴリズムの説明では、まず、IEEE の倍精度浮動小数点仕様を例に、浮動小数点数の表現方法について説明する[3]。次に、浮動小数点数の表現の主要な特徴である浮動小数点数間のギャップに基づく出力アルゴリズムを開発する。最後に、このアルゴリズムが、入力時に元の浮動小数点数を復元できる最短の、正しく丸められた出力文字列を生成することを証明する。

## 2.1 浮動小数点数の表現
浮動小数点数の設計における重要な目標は、実数をある桁数の精度で近似することができる表現を提供することです。浮動小数点数は、最初の数桁の有効数字に対応する仮数と、小数点の位置に対応する指数によって数学的にモデル化されます。例えば、vがb（通常は2）進数の浮動小数点数であるとします。仮数 f と指数 e は v = f × b<sup>e</sup> かつ |f| < b<sub>p</sub> となるような base-b 整数で、p は仮数の固定サイズ (base-bdigits) です。さらに、b<sup>p-1</sup> ≤ |f|、すなわち、仮数がゼロでない桁で始まる場合、v は正規化(normalize)されている、と呼ばれます。正規化されていない、ゼロでない浮動小数点数は、仮数を左にシフトし、それに応じて指数を減らすことによって正規化することができます。しかし、指数は固定サイズを持っているので、いくつかの数は正規化することはできません。そのような数は、非正規化浮動小数点数と呼ばれています。入力ベースが2である場合、正規化、非ゼロ数の仮数は常に1で始まる。その結果、この最初のビットはしばしば表現から省かれ、隠れビットと呼ばれます。IEEE仕様[3]は、-0.0、正の無限大（+inf）、負の無限大（-inf）、および「数字ではない」（NaN）についての表現も提供しています。 IEEE倍精度浮動小数点数vは、1ビットの符号、11ビットの符号なしバイアス指数（be）、および隠しビット付きの52ビット符号なし仮数（m）の3つのフィールドからなる64ビットデータとして表現されます。1≦be≦2046のとき、vは符号(252+m)×2be-1075の正規浮動小数点数であり、be=0のとき、vは符号m×2-1074で、+0と-0を含む正規化浮動小数点数である。be = 2047 and m = 0, vは符号によって+infまたは-infとなる。このような表現では、浮動小数点数の間に不均等な隙間が生じます。浮動小数点数はゼロ付近で最も密度が高く、実数線に沿って外側に向かうにつれて密度が低くなります。浮動小数点数vが与えられたとき、v+で示される後継の浮動小数点数とv-で示される前任者を定義しておくと便利です。v-+v2とv+v+2間の実数はすべてvに丸まります。ここではf > 0の場合を考えるが、f < 0の場合も全く同様である。すべてのvについて、v+は(f + 1) ×beである。f + 1がもはや固定サイズの仮数に収まらない場合、すなわちf + 1 = bpの場合、v+はbp-1 × be+1である。eを最大指数とすると、v+は+infとなる。ほとんどのvについて、v-は(f - 1) ×beとなる。残りのvでは、その差はより小さくなる。f=bp-1でeが最小指数より大きい場合、v-は(bp-1)×be-1である。

## 2.2 アルゴリズム
ここでは、入力時に元の浮動小数点数を復元できる、最短で正しく丸められた出力文字列を生成するために、浮動小数点数間のギャップを利用するアルゴリズムを開発します。仮数と指数で正の浮動小数点数 v が与えられると、アルゴリズムは v- と v+ を使用して、入力時に v に丸められる正確な値の範囲を決定します。入力丸めアルゴリズムは、同値を解消するために異なる戦略を用いるので（例えば、切り上げまたは偶数に丸める）、我々は当初、丸め範囲の両端が入力時にvに丸めることを保証できないと仮定する。第3節では、特定の入力丸めアルゴリズムに関する知識に基づいて、この制約を緩和する方法を示す。このアルゴリズムは、正確な有理数演算を用いて計算を行うため、精度の損失はない。
桁を生成するために、このアルゴリズムは、0.d1d2 ...（d1、d2、...はBase-B桁）の形になるように、数をスケーリングする。最初の桁は、スケーリングされた数値に出力ベースであるBを乗じ、整数部を取ることで計算されます。余りは、同じ方法を用いて残りの桁を計算するために使用されます。各桁が生成された後、アルゴリズムは、結果の数または最後の桁を増加させた結果の数のいずれかがvの丸め範囲内であるかどうかをテストします。同数の場合は、入力時にどちらの可能性もvに丸められるので、どのような方法で決定してもよい。各桁で出力数をテストすることにより、このアルゴリズムは、入力時にvに正しく丸められる最短の出力文字列を生成する。さらに、キャリーを伝搬させることなく、左から右へ数字を生成する。以下は、このアルゴリズムのより正式な説明である。x以下の最大の整数をbxc、x以上の最小の整数をdxe、x-bxcを{x}と表現する。乗算は常に×印で表す。これは、位取り表記で桁を示すために並置を用いるからである。

## 2.3 正確性
ここで、このアルゴリズムが正しいことを証明する。まず、このアルゴリズムが有効なBase-B桁を生成すること、その最初の桁は0でないこと、そして最後の桁をインクリメントする場合にはキャリーを伝播させる必要がないことを示すことから始める。終了条件(2)は、dnがインクリメントされてもキャリーが発生しないことを保証している。kの最小値と終了条件(2)から、最初の桁は0でないことが必要である。(完全な証明は付録Aの定理1を参照)次に、出力条件(1)が成り立つこと、すなわち、入力時にvに正しく丸められる数でアルゴリズムが常に終了することを示し、情報保存の目的を満足させる。すなわち、ステップiで生成される数はv以下のqi ×Bk-iである(Appendix AのLemma 2参照)。この不変量により、終了条件がより簡潔になる(Appendix AのLemma 2のcollaryを参照)。

<img width="283" alt="スクリーンショット 2022-05-12 11 08 34" src="https://user-images.githubusercontent.com/55653825/167977910-d7e31c2e-7473-4b92-ac17-fba55b39e11e.png">

0 ≤ qn < 1 であり、Bk-n は n が増加すると任意にゼロに近づくので、最終的に終了条件 (1) が成立し、アルゴリズムは常に終了する。さらに、不変量と上記の終了条件は、このアルゴリズムが低と高の間の数で終了することを保証している。(付録Aの定理3参照)入力時に出力がvに丸められることを示したので、次に、出力の最後の桁が正しく丸められること、すなわち、出力条件(2)が成立することを示そう。アルゴリズムは0.d1 ... dn × Bkと0.d1 ... dn-1[dn+1]×Bk の近い方を選ぶので、最後の桁は正しく丸められる(付録Aの定理4参照)。最後に、より短い出力列は入力時にvに丸めないことを示そう。同様に、(n - 1)桁の数(末尾の0は許される)も入力時にvに丸められることはない。このような数V0が存在するとする。この不変量を用いて、0.d1 ... dn-1 × Bkと0.d1 ... dn-2[dn-1+1] × Bkがvに最も近い2つの（n - 1）桁の数であることを容易に示すことができる。

一般性を損なわない程度に、V0がその一つであると仮定する。V0が1番目であれば、終了条件(1)はステップn - 1で成立していたことになり、矛盾する。もしV0が2番目なら、終了条件(2)はステップn - 1で成立していたことになり、矛盾する。従って、入力時にvより短い出力文字列は存在しない。(付録Aの定理5参照)。

## 3 実装
0 ≤ qn < 1 であり、Bk-n は n が増加すると任意にゼロに近づくので、最終的に終了条件 (1) が成立し、アルゴリズムは常に終了する。 さらに、不変量と上記の終了条件は、このアルゴリズムが低と高の間の数で終了することを保証している。 (付録Aの3参照)出力が入力時にvに丸められることを示したので、次に、出力の最後の桁が正しく丸められること、すなわち、出力条件(2)が成立することを示す。 アルゴリズムは0.d1 ... dn × Bkと0.d1 ... dn-1[dn+1]×Bk の近い方を選ぶので、最後の桁は正しく丸められる(付録Aの定理4参照)。最後に、より短い出力列は入力時にvに丸めないことを示そう。

## 3.1 整数アルゴリズム
高精度な有理数演算を排除するために 分母を明示的に導入し、高精度の整数演算ができるようにした。 アルゴリズムが高精度整数演算を使用できるようにする。また また、前節で示したより簡潔な終了条件も利用する。 を利用する。

<img width="304" alt="スクリーンショット 2022-05-12 11 11 41" src="https://user-images.githubusercontent.com/55653825/167978183-4c7d5dac-f03c-4a80-8a6c-a36c50e58a51.png">

<img width="302" alt="スクリーンショット 2022-05-12 11 12 15" src="https://user-images.githubusercontent.com/55653825/167978233-bbcaa787-f8e0-4918-bc01-671c8ec86fb6.png">

入力ルーチンのタイブレークアルゴリズムが既知の場合、Vmはlowかhigh、またはその両方に等しいことが許されるかもしれない。入力時にlowがvに丸められるなら、終了条件(1)はrn ≤ m-n となる。入力時にhighがvに切り下がる場合、終了条件（2）はrn + m+n ≤ snとなり、kはr+m+s < Bkとなる最小の整数（すなわち 例えば、1023は2つのIEEE浮動小数点数のちょうど中間に位置し、小さい方の仮数が偶数であるため、入力時には小さい方に丸められます。不偏丸めにより、このアルゴリズムは9.999999999e22ではなく1e23と表示されます。これは[5]で示されたものと同様の反復アルゴリズム(スケール)を使用してkを求めるもので、入力ルーチンがIEEE不偏丸めを使用すると仮定している。dnの決定が同点の場合、常にdn + 1を選択して丸める。関数flonum→digitsは、第1要素をk、第2要素を桁のリストとするペアを返します。

## 3.2 効果的なスケーリング
Steele および White の反復アルゴリズムでは、 k、r0、s0、m+0、および m-0 を計算するために O(| log v|) 高精度整数演算が必要です。明らか な代替案は、浮動小数点対数関数を使用して dlogB ve で k を近似し、その後、 s または r、m+、および m- に乗じる適切な B 乗を計 算する効率的なアルゴリズムを使用して、このよう なアルゴリズムを実行することです。浮動小数点対数は真の対数よりわずかに小さかったり大きかったりするため、結果の天井が k または k - 1 になるように、浮動小数点対数から小さな定数（考えられる最大の誤差よりわずかに大きくなるように選択）が差し引かれま す。このコードでは、テーブルを使用して、0 ≤ k ≤ 325 の場合の 10k の値を調べますが、これはすべての IEEE 倍精度浮動小数点数を処理するのに十分です。また、logB vの計算を高速化するために、1log Bの値を調べるテーブルを2≦B≦36で使用します。浮動小数点対数関数のコストがかなり高い場合、対数の精度が低い近似値を計算する方が効率的な場合があります。ほとんどすべての浮動小数点表現では、入力ベース、bは、2（または2の累乗）であるため、我々は対数推定量についての議論では、b = 2と仮定します。また、出力基底が入力基底と同じであれば、変換アルゴリズムを使用する理由がないため、B > 2と仮定します。v = f × 2e ですから、log2v = log2f + e です。v =x × 2s、1 ≤ x < 2 となるように整数 s と浮動小数点数 x を計算すると、 log2v = log2 x + s となり、0 ≤ log2 x < 1 となります。とすると、s = e+len(f)-1 となります。logB v =log2 vlog2 Bを推定するために、slog2 Bを使用します。この推定値はlogB vをオーバーシュートすることはなく、1log2 3 < 0.631よりもアンダーシュートすることがあります。繰り返しますが、浮動小数点演算はslog2 Bの正確な値を計算しないので、推定値が決してオーバーシュートしないという特性を維持するために、小さな定数を差し引きます。推定値がIEEE倍精度浮動小数点演算で計算されると仮定すると、1log2 Bは10-14以下の誤差で表現できることになります。sは-1074から1023の間なので、s×1log2 Bの浮動小数点演算の結果は10-10以下の誤差となる。推定値がkをオーバーシュートすることはなく、誤差は1より小さいので、lslog2 Bmはkまたはk - 1となります。浮動小数点対数の推定値がほとんど常にkであったのに対し、我々のより単純な推定値は頻繁にk - 1になります。推定値が1ずれると余計なオーバーヘッドが発生しますが、このオーバーヘッドはなくすことができます。推定値がk - 1のとき、fixupはsにBを掛けてからgenerateを呼び出して桁を生成します。generateへの入力時、r、m+、m-はBで乗算される。これらの乗算をgenerateの呼び出し部位に戻すことにより、推定値がk - 1を返すときにfixupで乗算を排除することができる。その結果、推定値が1つずれてもペナルティはない。図3は、この推定器と修正された桁生成ループのScheme実装である。表2は、Steele andWhiteの反復スケーリングアルゴリズム[5]と浮動小数点対数スケーリングアルゴリズムのCPU時間と、我々の単純な推定およびスケーリングアルゴリズムの相対的な時間である。時間はDigitalUNIX V3.2Cが動作するDEC AXP 8420上のChez Schemeで計測されたものである。入力は250,680個の正の正規化されたIEEE倍精度浮動小数点数で、出力の基底は10です。このセットは、Schryer が浮動小数点ユニッ トのテスト用に開発した形式 [4]に従って生成されまし た。予想どおり、反復スケーリングアルゴリズムは、 推計ベースのアルゴリズムよりも 2 桁近くも遅いことがわか りました。
<img width="527" alt="スクリーンショット 2022-05-12 11 17 09" src="https://user-images.githubusercontent.com/55653825/167978746-a51bf60b-def4-4c48-a69e-d3fab41afd78.png">

## 4.0 固定形式
ここまでは、自由形式の出力の問題を扱ってきた。次に、固定フォーマット出力を生成するために、基本的なアルゴリズムをどのように修正するかを説明する。出力変換アルゴリズムの重要な特性は、v+とv-を計算することによって決定されるvの丸め範囲を使用することである。固定フォーマット出力では、この範囲は要求された精度を示すように条件付きで変更されます。もし浮動小数点数が与えられた桁位置までプリントされるのに十分な精度を 持っているならば，出力が与えられた位置で停止するように丸め範囲が拡張されま す。浮動小数点数が十分な精度を持っていない場合、丸め範囲は拡大されず、出力は最後の有効桁を過ぎたところに#マークを含みます。固定フォーマットモードで印刷する桁数を指定する方法は2つあります。絶対桁位置とは、出力を停止させる基数点からの距離のことで、Base-B桁で指定します。相対桁位置とは、出力するBase-B桁の数です。出力Vが正しく丸められるためには、v -Bj2 ≦ V ≦ v +Bj2でなければなりません。つまり、v-+v2 と v -Bj2 の小さい方を low とし、v+v+2 と v +Bj2 の大きい方を high として、low と high を計算した後、スケーリング係数 k を先ほどと同様に決定します。端点highがラウンド範囲にある場合(すなわち, high = v +Bj2の場合), kはhigh < Bkとなる最小の整数であり, すなわち, k = 1+blogB highc. それ以外の場合、kはhigh≦Bkとなる最小の整数、すなわち、k＝dlogB higheである。終了条件(1)は、端点lowが丸め範囲にある場合の等式を含むように拡張される。同様に、終了条件(2)は終了点thighが丸め範囲にある場合の等式を含むように拡張される。前述と同様に、0.d1 ... dn-1[dn+1] × Bkが0.d1 ... dn×Bkよりvに近い場合（または同点の場合）、桁dnがインクリメントされる。j = k-nの場合、アルゴリズムは目的の桁の位置で停止するので、単に結果を返すだけです。終了条件を定義したため、アルゴリズムはあまり多くの桁を生成することができません。したがって、j 6= k -nの場合、j < k -nであり、アルゴリズムは残りの桁を生成しなければなりません。残念ながら、アルゴリズムはここからjの位置まで単純に#マークを表示することはできません。終了条件(1)は最初の桁を生成した後に成立しますが、残りの桁の位置は重要であるため、#ではなく0でなければなりません。その結果、アルゴリズムは、それらが有意である限りゼロを生成し、その後、＃マークを生成する必要があります。桁は、それが、それ以降のすべての桁が入力時の数値の値を変更することなく任意のベースBの数字で置き換えることができるときに重要ではありません。言い換えれば、直前の桁を増加させても数値がvの丸め範囲から外れない場合、その桁は重要ではありません。低 = v -Bj2 および高 = v +Bj2 の場合、残りの桁位置はすべて重要であり、アルゴリズムはそれらをゼロで埋め、戻ります。そうでなければ、出力の精度は浮動小数点表現によって制限されます。アルゴリズムは、直前の桁をインクリメントするとhigh以下の数値になるまで0を生成し、その時点で残りの桁の位置を#マークで埋めます。例えば、100をIEEE倍精度で桁位置-20にプリントする場合、アルゴリズムは 100.000000000000000###とプリントします。ここで、相対位置を指定したとします。対応する絶対位置jを計算するために、アルゴリズムはまず最初の桁の絶対 位置を計算します。残念ながら、最初の桁の位置k-1はvの丸め範囲の上限に依存し、さらにjに依存する可能性がある。このサイクルは、jに依存しないkの初期推定値を使用し、必要に応じてそれを改良することによって解決される。初期推定値kˆはllogBv+v+2mであり、これは第3.2節で述べた技術を用いて効率的に計算することが可能である。v+Bkˆ-i2< Bkˆ ならば、初期推定値は正しいので k = kˆ となり、そうでなければ初期推定値は1つずれているので、 k = kˆ + 1 となる。このとき、アルゴリズムは、桁の絶対位置k - iが与えられたかのように進行する。固定フォーマット印刷で使用される有理演算は、前記のように共通分母を導入することにより、高精度の整数演算に変換することが可能である。定型印刷に用いる有理演算は、従来通り共通項を導入することで高精度整数演算に変換することができるが、考慮すべきケースが数倍あるため、コードが長くなるため本稿では割愛した。


## 5.0 結論
我々は、浮動小数点数を入力ベースから出力ベースに変換するための効率的なアルゴリズムを開発した。自由形式の出力に対しては、入力時に元の浮動小数点数に丸められ、必要に応じて入力のタイブレーク丸めアルゴリズムを考慮した、最短で正しく丸められた数を証明的に生成する。固定フォーマット出力では、重要でない末尾の桁の代わりに#マークが付いた正しく丸められた数を生成します。これらの#マークは、要求された桁数が内部精度を超える可能性がある場合に有用である。我々のアルゴリズムは、スケーリングファクタを計算するために高速なエスティ メータを採用している。アルゴリズムを若干変更することにより，推定値が1ずれるというペナルティをなくし，推定値を非常に安価にすることができました．私たちの自由形式アルゴリズムの基底10出力への実装を，いくつかの異なるシステム上で単純な固定形式アルゴリズムの実装と比較しました．このテストでは、250,680個の正の正規化IEEE倍精度浮動小数点数[4]を使用しました。固定書式アルゴリズムは、IEEE倍精度数を区別するために保証された最小桁数である17桁の有効数字で印刷した。また、I/O性能を考慮し、全てのケースで/dev/nullに出力した。平均必要桁数は15.2桁であり、自由書式アルゴリズムが固定書式アルゴリズムに対して特に有利な点はない。表3は、我々のフリーフォーマットアルゴリズムが、ストレートな固定フォーマットアルゴリズムよりも平均で66%多くCPU時間を要することを示しています。また、各システムの標準的な固定書式アルゴリズムとの比較の基準として、Cライブラリのprintf関数とストレートな固定書式アルゴリズムとを比較し、printfによって丸め誤まってしまった浮動小数点数の個数を示しています。printfがかなり高速なシステムについては、我々の実装をチューニングすることで、同等の結果を得られると思われます。(特に，現在の実装では64ビット演算を使用しており，効率的な64ビットサポートのないシステムではパフォーマンスが低下します)．我々のアルゴリズムはSteeleとWhiteの変換アルゴリズム[5]に基づいている．これは主にスケーリングファクタを計算するための高速な推定量を使用するためである。このアルゴリズムは、末尾のゼロを有意なものと重要でないものとを区別せず、また入力の丸めモードも考慮しません。David Gayは、我々と同様の推定器を独自に開発した[2]。この推定器は1次のテイラー級数を使って対数10 vを推定するもので、彼の推定器より精度は劣るものの、5回の浮動小数点演算ではなく2回の演算を必要とし、コストも低く抑えられています。さらに，我々のスケーリングアルゴリズムは推定値が1ずれた場合でも追加のオーバーヘッドを発生させないため，精度の低下は重要ではなく，スケーリングはすべてのケースでより効率的です．また，Gayは固定フォーマット出力に対してより効率的な桁生成技術をいつ採用できるかを判断する優れた一連のヒューリスティクスを開発しました．特に、要求される桁数が少ない場合には、ほとんどの場合において浮動小数点演算が十分に正確であることを示した。この論文で述べた固定フォーマット印刷アルゴリズムは、これらのヒューリスティックが失敗した場合に役立つ。この論文で述べたアルゴリズムの実装は、著者らから入手可能である。実際、ANSI/IEEE Scheme標準の要求である正確で最小限の長さの数値出力と、Chez Schemeで可能な限り効率的にそれを実現したいという願いが、ここで報告された作業の動機となっています。
