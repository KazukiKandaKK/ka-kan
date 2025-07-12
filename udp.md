---
title: "RustでUDPを簡単に実装する"
emoji: "🚠"
type: "tech"
topics:
  - "rust"
  - "初心者向け"
  - "ネットワーク"
  - "udp"
published: true
published_at: "2025-07-06 09:10"
---

## はじめに
rustでUDPを実装してみます。

## マシンスペック
MacBook Air M2 arm64

## 事前知識
### UDPとは
UDPとはOSI参照モデルのトランスポート層のプロトコルです。
![](https://storage.googleapis.com/zenn-user-upload/d156ab2fa4ea-20250705.png)
トランスポート層では、TCPとUDPの代表的なプロトコルがあります。
TCPは信頼性のある通信、UDPは同報通信すなわちブローだキャストなど、一方的に相手に送りつける通信と捉えていただけると良いと思います。
TCPがコネクション型の通信、UDPはコネクションレス型の通信になります。TCPとUDPに甲乙はつけ難いのですが、用途によって使い分けます。
TCPでは、オンデマンドの動画配信などに使用され、UDPはIP電話やテレビの配信などがあります。リアルタイム性を要求するか否かの認識として問題ないでしょう。
### UDPのパケットの構成
UDPは下記の様なパケットの構成をとります。
![](https://storage.googleapis.com/zenn-user-upload/6ee6ae395e49-20250705.png)

## 準備
### Dockerの起動
```docker
docker run --rm -it --name rust-udp -v "$PWD":/app rust:1.78 bash
# 以降はコンテナ内で作業
apt-get update
apt-get install -y vim

cd /app
cargo new --bin udp_demo
cd udp_demo
```
### ファイルの準備
#### サーバ側
```bash
mkdir -p src/bin

cat <<'EOF' > src/bin/server.rs
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    // ① 5555/UDP
    let sock = UdpSocket::bind("127.0.0.1:5555")?;
    println!("listening on udp/5555 …");

    let mut buf = [0u8; 1500];  // 1500 = Ethernet MTU
    loop {
        // ② ブロッキング受信(戻り値=(受信バイト数, 送信元アドレス))
        let (n, addr) = sock.recv_from(&mut buf)?;
        println!(
            "main from {} : {}",
            addr,
            String::from_utf8_lossy(&buf[..n])
        );
    }
}
EOF
```
#### クライアント側
```bash
cat <<'EOF' > src/bin/client.rs
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    // ①送信専用ソケットを取得
    let sock = UdpSocket::bind("0.0.0.0:0")?;

    // ② 宛先 (127.0.0.1:5555)を固定してconnect
    sock.connect("127.0.0.1:5555")?;

    // ③ 送信
    let msg = b"Hello, UDP!";
    sock.send(msg)?;
    println!("sent '{}' ({} bytes)", String::from_utf8_lossy(msg), msg.len());

    Ok(())
}
EOF
```

#### Cargo.tomlの編集
```bash
vim Cargo.toml
# 下記を追加する。
[[bin]]
name = "server"
path = "src/bin/server.rs"

[[bin]]
name = "client"
path = "src/bin/client.rs"
```

## 実験
2つのターミナルを準備しますので、別のタブで下記のコマンドを打ってDockerに新たに接続してください。以降、サーバ側、クライアント側と呼びます。
```bash
docker ps # ここでコンテナIDが出てくるので控えておく。
docker exec -it {CONTAINER_ID} bash
```
### サーバ側
```bash
cargo run --release --bin server
   Compiling udp_demo v0.1.0 (/app/udp_demo)
    Finished `release` profile [optimized] target(s) in 0.18s
     Running `target/release/server`
listening on udp/5555 …
```

### クライアント側
```bash
cargo run --release --bin client
    Finished `release` profile [optimized] target(s) in 0.01s
     Running `target/release/client`
sent 'Hello, UDP!' (11 bytes)
```

## まとめ
今回はrustでUDPを実装してみました。学習の補助になれば幸いです。