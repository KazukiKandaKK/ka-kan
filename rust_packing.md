---
title: "Rustに入門したので構造体のアライメントとパッキングの状態を明らかにする"
emoji: "🔘"
type: "tech"
topics:
  - "rust"
  - "c"
  - "構造体"
published: true
published_at: "2025-07-05 06:00"
---

## はじめに
前回、Rustに入門した記事を書きました。
https://zenn.dev/ka_kan/articles/2664adeb06a89b

Rustが手に馴染んでいないので、構造体の実情について深掘りしてみます。

## マシンスペック
MacBook Air M2 arm64

## 事前知識
### 構造体
過去の記事にまとめていますので下記をご覧ください。
https://zenn.dev/ka_kan/articles/60801aed05f5ec

## 準備
### Dockerのセットアップ・接続
```bash
mkdir rust-align && cd rust-align
cat << 'EOF' > Dockerfile
FROM rust:latest
RUN apt update && apt install -y vim
WORKDIR /workspace
EOF
docker build -t rust-align .
docker run --rm -it -v "$(pwd)":/workspace rust-align
```

## 実験
下記を実験してみます。
| フラグ名           | 付いている `repr` 属性                | 想定される用途                                                    | ビルド例                                        |
| -------------- | ------------------------------ | ---------------------------------------------------------- | ------------------------------------------- |
| `repr_rust`    | *(属性なし)*<br>＝ **`repr(Rust)`** | Rust 内部で最適化された安全デフォルト。フィールド順は保持されるが将来的に並べ替え最適化の可能性あり。      | `rustc --cfg repr_rust   layout.rs -o rust` |
| `repr_c`       | `#[repr(C)]`                   | **C ABI 完全互換**。宣言順と C のパディング規則を厳守。FFI 先で安心。                | `rustc --cfg repr_c      layout.rs -o c`    |
| `repr_packed1` | `#[repr(C, packed)]`           | **1 byte アライン**で隙間ゼロ。I/O やネットワークヘッダ用。ただしフィールド参照は *unsafe*。 | `rustc --cfg repr_packed1 layout.rs -o p1`  |
| `repr_packed2` | `#[repr(C, packed(2))]`        | **2 byte アライン**（例：BLE GATT など 16 bit 境界前提のプロトコル）           | `rustc --cfg repr_packed2 layout.rs -o p2`  |
| `repr_align8`  | `#[repr(C, align(8))]`         | **8 byte 境界**を強制。SIMD / DMA バーストやハード制約がある場合。               | `rustc --cfg repr_align8  layout.rs -o a8`  |

コードは下記になります。

```rust
// layout.rs  ─ cfg フラグで 5 パターン切替
#[cfg(repr_rust)]        #[derive(Debug)] struct L { a: u8, b: u32, c: u16 }
#[cfg(repr_c)]           #[repr(C)]       struct L { a: u8, b: u32, c: u16 }
#[cfg(repr_packed1)]     #[repr(C, packed)]       struct L { a: u8, b: u32, c: u16 }
#[cfg(repr_packed2)]     #[repr(C, packed(2))]    struct L { a: u8, b: u32, c: u16 }
#[cfg(repr_align8)]      #[repr(C, align(8))]     struct L { a: u8, b: u32, c: u16 }

fn main() {
    println!("size  = {}", std::mem::size_of::<L>());
    println!("align = {}", std::mem::align_of::<L>());
}
```

## 結果
```bash
# デフォルト repr(Rust) – 自動最適 (padding 最小)
rustc --cfg repr_rust layout.rs   -o rust  && ./rust
size  = 8
align = 4

# C 互換 (宣言順・C ABI)
rustc --cfg repr_c    layout.rs   -o c     && ./c
size  = 12
align = 4

# 完全 packing (1 バイト単位)
rustc --cfg repr_packed1 layout.rs -o p1    && ./p1
size  = 7
align = 1

# 指定幅 packing (2 バイト境界)
rustc --cfg repr_packed2 layout.rs -o p2    && ./p2
size  = 8
align = 2

# 強制アラインメント 8
rustc --cfg repr_align8  layout.rs -o a8    && ./a8
size  = 16
align = 8
```
まとめると次の様になります。
| モード                  | `size` | `align` |
| -------------------- | ------------ | ------------- |
| `repr(Rust)`         | 8B      | 4B           |
| `repr(C)`            | 12B     | 4B           |
| `repr(C, packed)`    | 7B     | 1B           |
| `repr(C, packed(2))` | 8B      | 2B           |
| `repr(C, align(8))`  | 16B     | 8B           |

## まとめ
Rustの構造体はモードによってメモリに優しい実装ができることがわかりました。
| モード | `size` | `align` | 理由                                                                   |
| -------------------- | ---------------- | ----------------- | -------------------------------------------------------------------- |
| `repr(Rust)`         | 8B            | 4B                 | Rustは自動で1byteパディングを入れ、8,8byte で揃える                            |
| `repr(C)`            | 12B           | 4B                 | 宣言順のまま → `u8(1)`+*padding(3)* + `u32(4)` + `u16(2)` + *padding(2)* |
| `packed(1)`          | 7B            | 1B                 | 全隙間除去．ただし `b` は mis-aligned                                          |
| `packed(2)`          | 8B            | 2B                 | 2byte境界で詰め，末尾は 8 になる                                               |
| `align(8)`           | 16*           | 8B                 | 末尾に8byteアラインを満たすパディング                                             |