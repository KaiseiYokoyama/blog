---
title: "60秒でわかるF#の文法"
date: 2019-10-15T19:47:25+09:00
draft: false
tags: ["f_sharp", "dotnet"]
---

まず始めに, タイトル詐欺であることをここにお断りしておく. この記事はおそらく60秒では読めないだろう. 元ネタの[F# syntax in 60 seconds](https://fsharpforfunandprofit.com/posts/fsharp-in-60-seconds/)に倣ったまでである. 英語を臆せずに読めるのなら, あなたにとってこの記事の価値はほぼゼロに違いないということも付け足しておく. 元ネタの記事を読んでほしい. 

@wraiknyの誘いでF#言語に興味を持った。OCamlみたいな言語? らしい。関数型言語は一年前にちょっと授業で習ったのだが, 記憶力が悪いのでもう何も覚えていない。やばい。

それにしても, F#に関する日本語のドキュメントはRust以上に(というかRustはまあまあ充実してきた)少ない。おそらくこれまで国内では、臆せず英語のドキュメントを読めるつよつよマンの独壇場だったのだろう。

とはいえ日本語の文書も欲しいので、自分で少し書くことにする。[F# syntax in 60 seconds](https://fsharpforfunandprofit.com/posts/fsharp-in-60-seconds/)の解釈をやっていこう。これまで手続き型言語ばかりを書いてきた同志のために, うろ覚えの知識も僅かながら添えておく。

# F# syntax in 60 seconds
## "変数"
```
// ======== "Variables" (but not really) ==========
// The "let" keyword defines an (immutable) value
let myInt = 5
let myFloat = 3.14
let myString = "hello"	//note that no types needed
```

"変数"(語弊のある言い方らしい)は`let`キーワードで宣言する. デフォルトで非可変なのはRustと一緒. 型推論も効く. 

## [リスト](https://docs.microsoft.com/ja-jp/dotnet/fsharp/language-reference/lists)
```
// ======== Lists ============
let twoToFive = [2;3;4;5]        // Square brackets create a list with
                                 // semicolon delimiters.
let oneToFive = 1 :: twoToFive   // :: creates list with new 1st element
// The result is [1;2;3;4;5]
let zeroToFive = [0;1] @ twoToFive   // @ concats two lists
```

> F#リストは、シングルリンクリストとして実装されます。これは、リストの先頭にのみアクセスする操作が o (1) であり、要素アクセスが o (n) であることを意味します。

先頭以外にも頻繁にアクセスする場合, [配列](https://docs.microsoft.com/ja-jp/dotnet/fsharp/language-reference/arrays)を使用するほうが良いらしい。

## 関数
```
// ======== Functions ========
// The "let" keyword also defines a named function.
let square x = x * x          // Note that no parens are used.
square 3                      // Now run the function. Again, no parens.

let add x y = x + y           // don't use add (x,y)! It means something
                              // completely different.
add 2 3                       // Now run the function.
```

関数の宣言キーワードも`let`を使用する。
```
let 関数名 引数... = 式
```
という形なのか?

関数を使用するときは
```
関数名 引数...
```
この書き方はOCamlでも見た気がする. 関数型の特徴かもしれない. 

```
// to define a multiline function, just use indents. No semicolons needed.
let evens list =
   let isEven x = x%2 = 0     // Define "isEven" as an inner ("nested") function
   List.filter isEven list    // List.filter is a library function
                              // with two parameters: a boolean function
                              // and a list to work on

evens oneToFive               // Now run the function
```

複数行の関数はインデントを利用してまとめる. `List.filter`関数は名前からして`Tを取ってboolを返す関数 Tからなるリスト`の変数を取るっぽい. 

## ()による優先順位の指定
```
// You can use parens to clarify precedence. In this example,
// do "map" first, with two args, then do "sum" on the result.
// Without the parens, "List.map" would be passed as an arg to List.sum
let sumOfSquaresTo100 =
   List.sum ( List.map square [1..100] )
```

## パイプライン演算子
```
// You can pipe the output of one operation to the next using "|>"
// Here is the same sumOfSquares function written using pipes
let sumOfSquaresTo100piped =
   [1..100] |> List.map square |> List.sum  // "square" was defined earlier
```

`|>`を使えば, 左の値を右の関数に代入することができるらしい. 

`List.map square`という式の値は, リストを取って(おそらく)それぞれの要素を二乗したリストを返す**関数**である.  
手続き型言語を常用してきた人には見慣れないものだが, 関数型言語にはカリー化という現象がある. 関数の仕様に際して変数を一部だけ指定しておいて, のこりの変数を取って結果を返す関数を生成する. 

## ラムダ関数 / 無名関数
```
// you can define lambdas (anonymous functions) using the "fun" keyword
let sumOfSquaresTo100withFun =
   [1..100] |> List.map (fun x->x*x) |> List.sum
```

`fun`キーワードを用いて, 無名関数を宣言できる. 

## 関数の返り値
```
// In F# returns are implicit -- no "return" needed. A function always
// returns the value of the last expression used.
```

`return`キーワードはF#には必要ないとのこと. 関数は常に, 最後の式が返す値を返す. 

## パターンマッチング
```
// ======== Pattern Matching ========
// Match..with.. is a supercharged case/switch statement.
let simplePatternMatch =
   let x = "a"
   match x with
    | "a" -> printfn "x is a"
    | "b" -> printfn "x is b"
    | _ -> printfn "x is something else"   // underscore matches anything
```

`switch`のやべーやつ, それが`match`である. F#の`match`はSwiftの`switch`に近く, 使用感はRustの`match`に似ているようだ. 

## Option型
```
// Some(..) and None are roughly analogous to Nullable wrappers
let validValue = Some(99)
let invalidValue = None
```

F#でNull安全を実現したいのなら, `Option`型を使う. 

## unpack
```
// In this example, match..with matches the "Some" and the "None",
// and also unpacks the value in the "Some" at the same time.
let optionPatternMatch input =
   match input with
    | Some i -> printfn "input is an int=%d" i
    | None -> printfn "input is missing"

optionPatternMatch validValue
optionPatternMatch invalidValue
```

パターンマッチを使用して値をunpackする. Rustの`Option<T>.unwrap()`みたいなメソッド(そもそも, メソッドという概念)はあるのだろうか. 

## 複合型
```
// Tuple types are pairs, triples, etc. Tuples use commas.
let twoTuple = 1,2
let threeTuple = "a",2,true
```

タプル(単純な形の値の集合)は`,`を用いて宣言する.

## [レコード型](https://docs.microsoft.com/ja-jp/dotnet/fsharp/language-reference/records)
```
// Record types have named fields. Semicolons are separators.
type Person = {First:string; Last:string}
let person1 = {First="john"; Last="Doe"}
```

デフォルトは参照型. 詳しくは次のURLを参照すると良い. https://midoliy.com/content/fsharp/text/type-detail/0_record.html

## [判別共用体](https://docs.microsoft.com/ja-jp/dotnet/fsharp/language-reference/discriminated-unions)
```
// Types can be combined recursively in complex ways.
// E.g. here is a union type that contains a list of the same type:
type Employee = 
  | Worker of Person
  | Manager of Employee list
let jdoe = {First="John";Last="Doe"}
let worker = Worker jdoe
```

複数の異なるオプションのうち、いずれかの値を格納できる型. 代数的データ型の一種?

## 出力
```
// The printf/printfn functions are similar to the
// Console.Write/WriteLine functions in C#.
printfn "Printing an int %i, a float %f, a bool %b" 1 2.0 true
printfn "A string %s, and something generic %A" "hello" [1;2;3;4]
```

```
// all complex types have pretty printing built in
printfn "twoTuple=%A,\nPerson=%A,\nTemp=%A,\nEmployee=%A" 
         twoTuple person1 temp worker
```

複合型はいい感じの出力フォーマットを持っている. 

```
// There are also sprintf/sprintfn functions for formatting data
// into a string, similar to String.Format.
```

出力と同様の文字列が欲しい場合は`sprintf`, `sprintfn`を使う. 