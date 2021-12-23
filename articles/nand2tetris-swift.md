---
title: "「コンピュータシステムの理論と実装」をSwiftで実装してみた"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ios,swift]
published: false
---

（導入）

* 先人の事例
* なぜやろうと思ったのか
* どこまでの範囲を含むか

実装難易度表記

* ★☆☆☆☆ : 前提知識でほぼ完全に実装できる
* ★★☆☆☆ : 本の内容を追えば実装できる
* ★★★☆☆ : 本の内容を追いかつ少し考えれば実装できる
* ★★★★☆ : 本の内容を追いかつ熟考すれば実装できる
* ★★★★★ : 本の内容を追いかつ考えても実装が怪しく、深い理解が必要

## 1章　ブール論理

★★★☆☆
マルチプレクサのところで少し悩みましたが、地道に検討すればできました。
論理演算に慣れるまでがつらかったです。

全てのデジタル機器は論理ゲートという構成要素の持ちます。
ANDゲートやORゲートいったものです。そしてそれらのゲートはすべてNANDゲートの組み合わせで実現できます。
この章ではNADNゲートを作成しそれをベースに複数の論理ゲートを作成します。

例えば、NANDゲートは以下のように定義できます。

```swift
public static func nand(a: Bit, b: Bit) -> Bit {
    .init(!(a.value && b.value))
}
```

これを利用して他のあらゆるゲートを作成します。

## 2章　ブール算術

★★☆☆☆

基本的な論理ゲートを利用し、計算をできるようなゲートを作成します。
そして、論理演算と算術演算を処理するALU（算術論理演算器）というものを作成します。
ALUは複数の関数をもっており、そのうちどれを実行するかを制御ビットを用いて決めます。

```swift
struct ALU: ALUProtocol  {
    public static func alu(x: Bit16, y: Bit16, zx: Bit, nx: Bit, zy: Bit, ny: Bit, f: Bit, no: Bit) -> ALUOutput {
        
        let x = negate(zero(x, control: zx), control: nx)
        let y = negate(zero(y, control: zy), control: ny)
        let tmp = selectFunction(a: x, b: y, control: f)
        let out = negate(tmp, control: no)
        
        return .init(out: out, zr: out.isZero, ng: out.isNegative)
    }
    
    static func zero(_ input: Bit16, control: Bit) -> Bit16 {
        MultiGate.mux16(a: input, b: Bit16.allZero, sel: control)
    }
    
    static func negate(_ input: Bit16, control: Bit) -> Bit16 {
        MultiGate.mux16(a: input, b: MultiGate.not16(a: input), sel: control)
    }
    
    static func selectFunction(a: Bit16, b: Bit16, control: Bit) -> Bit16 {
        MultiGate.mux16(a: MultiGate.and16(a: a, b: b), b: Adder16.add(a: a, b: b), sel: control)
    }
}
```



## 3章　順序回路

★★★☆☆

状態を保持できるように = t と t-1 との関係性が生まれるように順序回路を作成します。
その中でも本書ではD型フリップフロップ（DFF）というタイプものを作っていきます。

また、それをもとにデータの書き込み/読み込みを可能にするためにレジスタを作成します。
そしてCPUにおいて、「次に実行するプログラムのアドレス」を示すものとしてプログラムカウンタも作成しますが、それもレジスタと同じインターフェースで作成できます。

```swift
/// 1つ前の値を入力を出力するゲート
struct DFF: DFFProtocol {
    var `in`: Bit
    
    init(_ initialValue: Bit) {
        `in` = initialValue
    }
    
    mutating func dff(_ input: Bit) -> Bit {
        defer { `in` = input }
        return `in`
    }
}
```

```swift
///　値の読み書きが可能な回路
struct Register: RegisterProtocol {
    var dffGate: DFF = DFF(.init(false))
    var load: Bit = .init(false)
    var preOut: Bit = .init(false)

    mutating func output(in: Bit, load: Bit) -> Bit {
        let out = Gate.mux(a: preOut, b: dffGate.dff(`in`), sel: self.load)
       
        self.load = load
        preOut = out
        
        return out
    }
}
```

```swift
struct ProgramCounter {
    private var register16: Register16 = Register16()
    
    mutating func output(`in`: Bit16, inc: Bit, load: Bit, reset: Bit) -> Bit16 {
        let preValue: Bit16 = register16.output(in: `in`, load: .init(false))
        
        let val = MultiGate.mux8Way16(a: preValue,
                                      b: Adder16.inc(a: preValue),
                                      c: `in`,
                                      d: `in`,
                                      e: Bit16.allZero,
                                      f: Bit16.allZero,
                                      g: Bit16.allZero,
                                      h: Bit16.allZero,
                                      sel: .init((reset,
                                                  load,
                                                  inc)))
        
        return register16.output(in: val, load: .init(true))
    }
}
```



## 4章　機械語

★★★☆☆

機械語（1010001100011001のようなバイナリ）の仕様と、それとアセンブリ言語（アセンブリ）の関係についての章です。

機械語：CPU、レジスタを用いてメモリを操作するように設計されている
アセンブリ言語：機械語の命令を「ADD, R3, R1, R9」のような記号を用いて表現できる

6章がアセンブラについての章なので、その内容を合わせることでSwiftでアセンブラ（アセンブリ言語から機械語であるバイナリへと変換するもの）を作成しました。

## 5章　コンピュータアーキテクチャ

★★★★☆

1~3章で作成した回路を利用してCPU、コンピュータを作ります。

CPUの役割はプログラムの命令の実行と次の命令を取得です。
CPUは命令メモリ、データメモリに接続されておりその名前の通り、命令メモリからは命令を取得しデータメモリに対しては値の読み書きを行います。

CPU処理の大まかな流れは次のようになっています。

（図）

コンピュータはHack 機械語と呼ばれる本書の後半で作成する言語を実行するために設計された、ノイマン型コンピュータです。ノイマン型コンピュータはCPUを用いて、メモリデバイスを操作し、入力デバイスからデータを受け取り出力デバイスへデータを送信するものです。またこれはプログラム内蔵方式と呼ばれるものであり、コンピュータのメモリには計算データと共に命令のデータも含まれます。従って読み込むプログラムを変えることでさまざまな命令を実行できます。

## 6章　アセンブラ

★★★☆☆
仕様通りに実装していけば良いのですが、テストで失敗することが多くまたその修正に時間がかかりました。

本章ではアセンブリ言語をアセンブラが機械語にどのように変換しているかが示されています。

主な役割はシンボル解決とパース処理です。
シンボルは変数名を表すためや、特定の位置を示すために利用します。

```
LOAD R3,weight
```

例えばこの場合、`weight`がシンボルであり、その値があるメモリアドレスを示します。

```
LOOP:
   if i=101 goto END
   .
   .
   .
   goto LOOP
END:
   goto END
```

この場合、`LOOP`や`END`が特定の位置を示すシンボルです。

この`goto`の移動先のようにシンボルはそれが定義前の状態でも、利用される場合があります。
従ってアセンブリプログラムを順にパースしていくのではこの仕様を満たすことができません。
そのため、1度アセンブリプログラム全体を見てシンボルを抽出し、その後プログラムの最初から順にパースするということを行います。

```swift
struct Assembler {
    .
    .
    .
    func parse(text: String) -> [String] {
        let lines: [String] = text.components(separatedBy: "\n")
      
        // シンボルを探してシンボルテーブルを作成する
        setUpLabelSymbol(lines: lines)
        
        // パース
        var codes: [String] = []
        for line in lines {
            let code = parse(line: line)
            if !code.isEmpty {
                codes.append(code)
            }
        }
        
        return codes
    }
}
```

