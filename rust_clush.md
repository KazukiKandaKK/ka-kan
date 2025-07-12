---
title: "Rustに入門したので意図的にクラッシュできるかを試したい"
emoji: "⛑️"
type: "tech"
topics:
  - "rust"
  - "コンパイラ"
  - "c"
published: true
published_at: "2025-07-02 06:00"
---

## はじめに
最近ブラウザの書籍でRustに入門しているので、C言語のように意図的にクラッシュができるかどうかを確認します。
結果として、Rustのコンパイラは堅牢であると判断できました。

## マシンスペック
MacBook Air M2 arm64

## 準備
### Dockerの起動・インストール
```bash
docker run -it --rm -v $(pwd):/mnt rust:latest bash
apt update && apt install -y gcc make vim build-essential
```

## 検証項目
下記の観点で順次検証を進めます。

| 項目                     | C言語                                   | Rust                              |
|--------------------------|-------------------------------------|-----------------------------------|
| 解放済みのメモリへのアクセス     |      |  |
| Nullポインタ参照                 |       |     |
| 不正な参照         |                  | |
| バッファオーバーラン     |                |            |


## 検証
### 解放済みのメモリへのアクセス
解放済みのメモリにアクセスすると、ゴミ値が出てきたりNullを参照してしまうことになります。
#### C言語
mallocでメモリを確保して、操作後にメモリを解放して続けて操作したいと思います。
```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    int* p = malloc(sizeof(int));
    *p = 10;
    free(p);
    *p = 20; // ここで未定義動作（Segfaultするかも）
    return 0;
}
```
```bash
gcc -o test1 test1.c
./test1
```
問題なくコンパイル・実行できてしまいました。おそらく、参照していたメモリに何かが入ったかもしれません。

#### Rust
同様に変数を2つ用意して、1つを解放して標準出力してみます。
```rust
fn main() {
    let v = vec![1, 2, 3];
    let v2 = v;
    println!("{:?}", v); // ❌ error[E0382]: borrow of moved value: `v`
}
```
```bash
rustc test1.rs 
warning: unused variable: `v2`
 --> test1.rs:3:9
  |
3 |     let v2 = v;
  |         ^^ help: if this is intentional, prefix it with an underscore: `_v2`
  |
  = note: `#[warn(unused_variables)]` on by default

error[E0382]: borrow of moved value: `v`
 --> test1.rs:4:22
  |
2 |     let v = vec![1, 2, 3];
  |         - move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
3 |     let v2 = v;
  |              - value moved here
4 |     println!("{:?}", v); // ❌ error[E0382]: borrow of moved value: `v`
  |                      ^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let v2 = v.clone();
  |               ++++++++

error: aborting due to 1 previous error; 1 warning emitted

For more information about this error, try `rustc --explain E0382`.
```
`let v2 = v;`で`v`の実体が`v2`に移行したことにより、`v`が解放されており、`println!("{:?}", v);`でコンパイルエラーとなったと判断できます。

### Nullポインタ参照
次はNullポインタの参照です。
#### C言語
int型のポインタを準備して、操作してみます。
```c
#include <stdio.h>

int main() {
    int* p = NULL;
    *p = 10; // Segmentation fault
    return 0;
}
```
```bash
gcc -o test2 test2.c
```
コンパイルが通ったので実行してみます。
```bash
./test2
Segmentation fault
```
Segmentation faultになりました。

#### Rust
`Option<i32>`をNoneで定義して、`unwrap()`でpanicを起こせるかを試します。
```rust
fn main() {
    let maybe: Option<i32> = None;
    let val = maybe.unwrap(); // panic: called `Option::unwrap()` on a `None` value
}
```
```bash
rustc test2.rs 
warning: unused variable: `val`
 --> test2.rs:3:9
  |
3 |     let val = maybe.unwrap(); // panic: called `Option::unwrap()` on a `None` value
  |         ^^^ help: if this is intentional, prefix it with an underscore: `_val`
  |
  = note: `#[warn(unused_variables)]` on by default

warning: 1 warning emitted
```
warningだけなので、実行してみます。
```bash
thread 'main' panicked at test2.rs:3:21:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
panicが出てくれました。しかし、warningを解消すると防げるように設計されています。

## 不正な参照
今回は、いずれもスコープ外に変数を渡すようにしてみます。
### C言語（ダングリングポインタ）
関数の中にchar型の配列を定義して、戻り値に先頭ポインタを返します。
一見、構文上は問題ない様に見えますが、関数終了時に変数がメモリ上から消え、ゴミやNullを参照することになります。
```c
char* get_ptr() {
    char buf[10] = "hello";
    return buf; // 警告は出るが、コンパイルは通る
}
```
```bash
gcc -o test3 test3.c 
test3.c: In function 'get_ptr':
test3.c:3:12: warning: function returns address of local variable [-Wreturn-local-addr]
    3 |     return buf;
      |            ^~~
/usr/bin/ld: /usr/lib/gcc/aarch64-linux-gnu/12/../../../aarch64-linux-gnu/Scrt1.o: in function `_start':
(.text+0x1c): undefined reference to `main'
/usr/bin/ld: (.text+0x20): undefined reference to `main'
collect2: error: ld returned 1 exit status
```
今回はコンパイラが制御してくれました。

### Rust（借用チェック）
C言語と同様に、戻り値に参照で文字列を渡そうとします。
```rust
fn get_ref() -> &String {
    let s = String::from("hello");
    &s
}
```
```bash
rustc test3.rs 
error[E0106]: missing lifetime specifier
 --> test3.rs:1:17
  |
1 | fn get_ref() -> &String {
  |                 ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
  |
1 | fn get_ref() -> &'static String {
  |                  +++++++
help: instead, you are more likely to want to return an owned value
  |
1 - fn get_ref() -> &String {
1 + fn get_ref() -> String {
  |

error[E0601]: `main` function not found in crate `test3`
 --> test3.rs:4:2
  |
4 | }
  |  ^ consider adding a `main` function to `test3.rs`

error: aborting due to 2 previous errors

Some errors have detailed explanations: E0106, E0601.
For more information about an error, try `rustc --explain E0106`.
```
`error[E0106]: missing lifetime specifier`が出ており、コンパイル時にスコープ外の変数アクセスをErrorで検出しています。

## バッファオーバーラン
配列を定義して、定義した範囲外のアクセスをする際の挙動を確認します。
### C言語
```c
#include <stdio.h>

int main() {
    int arr[3] = {1, 2, 3};
    printf("%d\n", arr[10]); // 配列の範囲外アクセス
    return 0;
}
```
```bash
./test4
-441186476
```
コンパイル・実行ができ、何かのゴミ値が出力されました。
### Rust
```rust
fn main() {
    let arr = [1, 2, 3];
    println!("{}", arr[10]); // panic! index out of bounds
}
```
```bash
rustc test4rs.rs 
error: this operation will panic at runtime
 --> test4rs.rs:3:20
  |
3 |     println!("{}", arr[10]); // panic! index out of bounds
  |                    ^^^^^^^ index out of bounds: the length is 3 but the index is 10
  |
  = note: `#[deny(unconditional_panic)]` on by default

error: aborting due to 1 previous error
```
panicになりました、領域外のアクセスを検知している様です。
for文でループを回す際にも安全にコードを記述できます。

## まとめ
| 項目                     | C言語                                   | Rust                              |
|--------------------------|-------------------------------------|-----------------------------------|
| 解放済みのメモリへのアクセス     | コンパイル可能。実行時にクラッシュまたは未定義動作     | コンパイル時に検知 |
| Nullポインタ参照                 | コンパイル可能。実行時にSegmentation fault     | Warningで警告    |
| 不正な参照         | コンパイル時に検知                 | コンパイル時に検知|
| バッファオーバーラン     | コンパイル可能。実行時に検知               | コンパイル時に検知           |


RustがC言語のように意図的にクラッシュができるかどうかを確認しました。
結果として、Rustのコンパイラは堅牢であると判断できました。
米国では、メモリ安全性の高い言語の推奨がされ始めていますが、Rustも選択肢の一つとして良いと感じました。