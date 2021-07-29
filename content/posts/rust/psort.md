---
title: "[Rust] rayonを使ったマルチスレッド版クイックソート・マージソート"
date: 2021-07-29T18:35:26+09:00
draft: false
tags: ["Rust", "rayon", "sort", "algorithm"]
---

ソートにもさまざまな種類がある。
その中でも、分割統治法に分類されるアルゴリズムを用いるソートはマルチスレッド処理と相性が良さそうに見える。
同じような話題を扱った記事はネットにごまんとあるが、
読むだけではつまらないので実際に試してみた。

## 検証対象

- マージソート
- クイックソート

分割統治法を使い、なおかつ実装が簡単なソートとしてこの2つを選んだ。

## 実装
並行プログラミングということで、もちろんRust言語で実装していく。
手軽にマルチスレッド化ができる[`rayon`](https://github.com/rayon-rs/rayon)を使う。


### 下準備
```rust
/// 並行性
pub enum Concurrency {
    /// シングルスレッド
    SingleThread,
    /// マルチスレッド
    MultiThread,
}

/// ソートのトレイト
pub trait Sort<T: Ord> {
    const CONCURRENCY: Concurrency;

    fn sort(slice: &mut [T]);
}
```

最終的にはソートごとに型を宣言することをイメージして、
まずは単なるソートを意味するトレイトを定義してみる。
シングルスレッドとマルチスレッドの比較がしたいので、
その型の表すソートがシングルスレッドでの処理なのか、
マルチスレッドでの処理なのかを示す関連定数 `CONCURRENCY` をつけておいた。

```rust
/// 分割統治法（= Divide and Conquer algorithm）を用いているソートのトレイト
pub trait DCSort<T: Ord> {
    const CONCURRENCY: Concurrency;

    fn divide(slice: &mut [T]) -> (&mut [T], &mut [T]);

    fn unite(left: &mut [T], right: &mut [T]);
}
```

続いて、分割統治法を用いるソートを表すトレイトを書く。
もちろん`divide()`が分割時，`unite()`が結合時の処理を表す。
関連定数 `CONCURRENCY`は、後で先述の `Sort::CONCURRENCY` に結びつける。

分割統治法を用いるソートの大まかな流れはこうだ。
1. 配列を2つに分割する[^divide]
1. 分割後の配列をそれぞれソートする
1. 配列を結合する

それぞれのステップに、これまでに宣言した関数を当てはめるとこうなる。
1. 配列を2つに分割する[^divide] <- `DCSort<T>::divide()`
1. 分割後の配列をそれぞれソートする <- `Sort<T>::sort()`
1. 配列を結合する <- `DCSort<T>::unite()`

この流れは次のコードで実装した。

中盤の`match`文には、分割後の2つの配列をシングルスレッドで1つずつ逐次処理する`S::sort(left); S::sort(right);`と、マルチスレッドで処理する`rayon::join(|| S::sort(left), || S::sort(right));`の2つの分岐がある。
どちらの処理を実行するかは、関連定数`CONCURRENCY`の値によって決まる。
ちなみに、今回の実装において、シングルスレッドとマルチスレッドで処理が違うのはここだけだ。

```rust
impl<T: Ord + Send, S: DCSort<T>> Sort<T> for S {
    const CONCURRENCY: Concurrency = S::CONCURRENCY;

    fn sort(slice: &mut [T]) {
        let len = slice.len();
        if len <= 1 { return; }

        let (left, right) = S::divide(slice);

        match S::CONCURRENCY {
            Concurrency::SingleThread => {
                S::sort(left);
                S::sort(right);
            }
            Concurrency::MultiThread => {
                rayon::join(|| S::sort(left), || S::sort(right));
            }
        }

        S::unite(left, right);
    }
}
```

これで、`DCSort<T>`トレイトを実装している型に`Sort<T>`トレイトを実装する仕組みができた。
あとはソートごと&並行性ごとに作った型に`DCSort<T>`を実装してやれば、`Sort<T>::sort()`が使えるようになる。

## ソートの実装
2種類のソートと2種類の並行性で4パターン。
つまり4つの型を実装することになる。

||マージソート|クイックソート|
|:--|:--|:--|
|シングルスレッド|`STMergeSort`|`STQuickSort`|
|マルチスレッド|`MTMergeSort`|`MTQuickSort`|

とはいえ、並行性は`DCSort<T>::CONCURRENCY`の値を設定するだけだし、
並行性にかかわらず同じソートを使うなら`DCSort<T>::devide()`も`DCSort<T>::unite()`も同じ内容である。
なので、実装はさほど難しくない。

### マージソート
`devide()`は単純で、スライスを半分に割るだけだ。

Rust言語では1つの値に対する可変参照は1つまでと決められている。
だから、`let (left, right) = (&mut slice[..mid], &mut slice[mid..])`というようなコードは書けない。
1つの可変スライスをカチ割って2つの可変スライスにするには、`split_at_mut()`を使う。

```rust
fn divide<T: Ord>(slice: &mut [T]) -> (&mut [T], &mut [T]) {
    let mid = slice.len() / 2;
    slice.split_at_mut(mid)
}
```

マージソートの本体は`unite()`の方にある。
なお、マージソートの基本的な仕組みについては割愛する。

```rust
fn unite<T: Ord + Clone>(left: &mut [T], right: &mut [T]) {
    let mut buf = Vec::with_capacity(left.len() + right.len());

    let (mut left_iter, mut right_iter)
        = (left.iter().peekable(), right.iter().peekable());

    while buf.len() < (left_iter.len() + right_iter.len()) {
        let item = match (left_iter.peek(), right_iter.peek()) {
            (Some(l), Some(r)) => {
                if l > r {
                    left_iter.next().unwrap()
                } else {
                    right_iter.next().unwrap()
                }
            }
            (Some(_), None) => {
                left_iter.next().unwrap()
            }
            (None, Some(_)) => {
                right_iter.next().unwrap()
            }
            (None, None) => break,
        };

        // .clone()を使わずに済む方法を知りたい
        buf.push(item.clone());
    }

    for (idx, item) in buf.into_iter().enumerate() {
        if idx < left.len() {
            left[idx] = item;
        } else {
            right[idx - left.len()] = item;
        };
    }
}
```

この実装では、`sort()`を実行するたびにいちいちバッファ`buf`を用意する実装になっている。
もし高速なマージソートを実装する必要があるなら、ソートの一番初めにソートの対象と同じ長さのバッファを作り、そのバッファの可変スライスを`buf`として使うようにすると、メモリ確保の回数が減らせると思う。

### クイックソート
マージソートとは打って変わって、クイックソートの本体は`devide()`そのものだ。
なお、クイックソートの基本的な仕組みについては割愛する。

```rust
fn divide<T: Ord + Clone>(slice: &mut [T]) -> (&mut [T], &mut [T]) {
    let pivot = slice.last().unwrap().clone();

    let (mut i, mut j) = (0usize, Some(slice.len() - 1));

    loop {
        while slice[i] < pivot {
            i += 1;
        }

        loop {
            j = match j {
                Some(j) if slice[j] >= pivot => j.checked_sub(1),
                _ => break,
            };
        }

        if let Some(j_) = j {
            if i < j_ {
                slice.swap(i, j_);

                i += 1;
                j = j_.checked_sub(1);
            } else {
                return slice.split_at_mut(i);
            }
        } else {
            // pivotがsliceの中で最小値のとき

            // 最小値（pivot）を一番左に持っていく
            slice.swap(0, slice.len() - 1);
            return slice.split_at_mut(1);
        }
    }
}
```

ちなみに`unite()`は何もしない。

## ベンチマークの実装
nigntly限定の`cargo bench`コマンドを使えば、ベンチマークがとれる。

ソートする配列は`rand`クレートで生成する。

```rust
fn vec(n: usize) -> Vec<i32> {
    let mut rng = rand::thread_rng();
    (0..n).into_iter().map(|_| rng.gen()).collect()
}
```

配列の長さは、今回は1Mとした。

```rust
const LEN: usize = 1024 * 1024;
```

比較対象として、`Vec::sort()`および`Vec::sort_unstable()`を用意した。

```rust
#[bench]
fn vec_sort(b: &mut Bencher) {
    b.iter(|| {
        let mut vec = vec(LEN);
        vec.sort();
    })
}

#[bench]
fn vec_sort_unstable(b: &mut Bencher) {
    b.iter(|| {
        let mut vec = vec(LEN);
        vec.sort_unstable();
    })
}
```

シングルスレッドのマージソートのベンチマーク用関数の実装はこんな感じ。

```rust
#[bench]
fn merge_st(b: &mut Bencher) {
    b.iter(|| {
        let mut vec = vec(LEN);
        STMergeSort::sort(&mut vec[..]);
    })
}
```

他の条件についても同様に実装した。

## 計測
### 環境
|OS|プロセッサ|メモリ|
|:--|:--|:--|
|macOS Big Sur 11.4|Intel(R) Core(TM) i5-1038NG7 CPU @ 2.00GHz|16GB|

### 結果
`cargo bench`による計測の結果はこうなった。
（ソート名）_mtはマルチスレッド版、（ソート名）_stはシングルスレッド版を表す。

```
test bench::merge_mt          ... bench:  47,156,677 ns/iter (+/- 2,500,981)
test bench::merge_st          ... bench: 147,144,491 ns/iter (+/- 22,425,095)
test bench::quick_mt          ... bench:  31,325,250 ns/iter (+/- 6,634,943)
test bench::quick_st          ... bench: 103,741,271 ns/iter (+/- 16,463,993)
test bench::vec_sort          ... bench:  63,478,509 ns/iter (+/- 13,374,482)
test bench::vec_sort_unstable ... bench:  35,561,017 ns/iter (+/- 8,099,090)
```

マージソートもクイックソートも、マルチスレッド版のほうがまあまあ速い。
`rayon`を使ったこれらのソートのマルチスレッド化には、それなりの効果があるのだろう。

スレッドの生成やタスクの割り振りなど、
マルチスレッド処理にはそれなりにオーバーヘッドがある。
しかし、それでも効果が出ているということは、
逆に言えばそれだけ（今回実装した）ソートの処理がオーバーヘッドよりも重いということでもありそう。

`Vec::sort_unstable()`がマルチスレッド版クイックソートといい勝負してるのがすごい。

## まとめ
`rayon::join()`を使って、クイックソートおよびマージソートを高速化できた。あと、`Vec::sort_unstable()`は速い。

Rustチームと`rayon`のコントリビュータの皆様に感謝。

[^divide]: 2つ以上のものもあったりするんだろうか。