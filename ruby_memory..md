---
title: "Rubyでクラスを作成してメソッドに処理加えると、どのくらいのメモリを消費するのか観測する"
emoji: "💠"
type: "tech"
topics:
  - "ruby"
  - "memory"
  - "メモリ管理"
published: true
published_at: "2025-06-26 06:00"
---

## サマリ
Rubyのクラスについてメモリの使用方法を簡単に分析してみました。

## マシンスペック
MacBook Air M2 arm64

## 目的
rubyでクラスやメソッドを定義した際に、どのくらいのメモリを消費するのかを可視化して、処理1つに対するメモリの気持ちに寄り添えるようになります。

## 事前知識
### ヒープ領域・メモリ割り当て
プログラムを実行する際に動的にメモリを確保や解放ができる領域です。
C言語をベースにメモリの割り当てを解説すると、画像のメモリの0番地から予約されている領域となり、その次にMISPの機械語のコードが配置されるテキスト領域があります。静的データは定数や静的変数が入ります。ヒープ領域は先述の通り、プログラムを実行する際に動的にメモリを確保や解放ができる領域です。スタック領域は関数のローカル変数や引数、戻り値が割り当てられ、関数の開始時にメモリ上にスタックされ、関数の終了時にメモリが解放されます。
![](https://storage.googleapis.com/zenn-user-upload/16af15391dd4-20250625.png)

## 準備
### 計測用コード
```ruby
require "json"
require "objspace"
require "net/http"
require "uri"

# メモリ集計
def snapshot(label, mod)
  meth_memsize = mod.instance_methods(false)
                    .sum { |m| ObjectSpace.memsize_of(mod.instance_method(m)) }
  iseq_memsize = mod.instance_methods(false)
                    .sum { |m| ObjectSpace.memsize_of(RubyVM::InstructionSequence.of(mod.instance_method(m))) }
  iseq_bytes   = mod.instance_methods(false)
                    .sum { |m| RubyVM::InstructionSequence.of(mod.instance_method(m)).to_binary.bytesize }

  puts "== #{label} =="
  puts "  Class memsize   : #{ObjectSpace.memsize_of(mod)} B"
  puts "  Method memsize  : #{meth_memsize} B"
  puts "  ISeq  memsize   : #{iseq_memsize} B"
  puts "  ISeq  bytes     : #{iseq_bytes} B"
  puts
end

# STEP 0: 空クラス
class Demo; end
snapshot("STEP 0 (class only)", Demo)

# STEP 1: 1のみのメソッド
class Demo
  def tiny; 1; end
end
snapshot("STEP 1 (+ tiny)", Demo)

# STEP 2: 通常 FizzBuzz
class Demo
  def fizzbuzz(n = 100)
    (1..n).map { |i|
      if i % 15 == 0 then 'FizzBuzz'
      elsif i %  3 == 0 then 'Fizz'
      elsif i %  5 == 0 then 'Buzz'
      else i end
    }
  end
end
snapshot("STEP 2 (+ fizzbuzz)", Demo)

# STEP 3: 再帰版 FizzBuzz
class Demo
  def fizzbuzz_recursive(n, i = 1, acc = [])
    return acc if i > n
    acc << (i % 15 == 0 ? 'FizzBuzz' :
            i % 3  == 0 ? 'Fizz'     :
            i % 5  == 0 ? 'Buzz'     : i)
    fizzbuzz_recursive(n, i + 1, acc)
  end
end
snapshot("STEP 3 (+ fizzbuzz_recursive)", Demo)

# STEP 4: server.rb へ HTTP リクエストを送るメソッド
class Demo
  def fetch_fizzbuzz_remote(n = 100, host = "127.0.0.1", port = 4567)
    uri  = URI("http://#{host}:#{port}/fizzbuzz?n=#{n}")
    JSON.parse(Net::HTTP.get(uri))
  end
end
# 実際に100件取ってみる
puts "Remote result (first 10) => #{Demo.new.fetch_fizzbuzz_remote(100).first(30)}"
snapshot("STEP 4 (+ fetch_fizzbuzz_remote)", Demo)

# STEP 5: メソッドを2つにする
class Demo
  def tiny; 1; end
  def tiny2; 1; end
end
snapshot("STEP 5 (+ tiny2)", Demo)

# STEP 6: メソッドを3つにする
class Demo
  def tiny; 1; end
  def tiny2; 1; end
  def tiny3; 1; end
end
snapshot("STEP 6 (+ tiny3)", Demo)

# STEP 7: メソッドを10個にする
class Demo
  def tiny; 1; end
  def tiny2; 1; end
  def tiny3; 1; end
  def tiny4; 1; end
  def tiny5; 1; end
  def tiny6; 1; end
  def tiny7; 1; end
  def tiny8; 1; end
  def tiny9; 1; end
  def tiny10; 1; end
end
snapshot("STEP 7 (+ tiny10)", Demo)
```
### FizzBuzz用サーバ
```ruby
#!/usr/bin/env ruby
# stdlib だけで動く FizzBuzz API サーバ
require 'webrick'   # HTTP サーバ
require 'json'      # JSON 生成

class FizzBuzzServlet < WEBrick::HTTPServlet::AbstractServlet
  def do_GET(req, res)
    n = (req.query['n'] || 100).to_i
    res['Content-Type'] = 'application/json'
    res.status = 200
    res.body   = JSON.dump(fizzbuzz(n))
  end

  private

  def fizzbuzz(n)
    (1..n).map do |i|
      if    i % 15 == 0 then 'FizzBuzz'
      elsif i %  3 == 0 then 'Fizz'
      elsif i %  5 == 0 then 'Buzz'
      else i end
    end
  end
end

server = WEBrick::HTTPServer.new(Port: 4567, BindAddress: '0.0.0.0')
server.mount '/fizzbuzz', FizzBuzzServlet
trap('INT') { server.shutdown }
server.start
```
## 比較結果
### 参考読み方
誤りがあればご容赦ください。
```bash
== STEP N ==
  Class memsize   : クラスの占有するメモリ領域
  Method memsize  : メソッドの占有する領域
  ISeq  memsize   : Rubyのローカル変数などが入る
  ISeq  bytes     : C言語が占有する領域
```

```bash
== STEP 0 (class only) ==
  Class memsize   : 192 B
  Method memsize  : 0 B
  ISeq  memsize   : 0 B
  ISeq  bytes     : 0 B

== STEP 1 (+ tiny) ==
  Class memsize   : 256 B
  Method memsize  : 80 B
  ISeq  memsize   : 456 B
  ISeq  bytes     : 216 B

== STEP 2 (+ fizzbuzz) ==
  Class memsize   : 256 B
  Method memsize  : 160 B
  ISeq  memsize   : 1112 B
  ISeq  bytes     : 932 B

== STEP 3 (+ fizzbuzz_recursive) ==
  Class memsize   : 256 B
  Method memsize  : 240 B
  ISeq  memsize   : 3128 B
  ISeq  bytes     : 1720 B

Remote result (first 10) => [1, 2, "Fizz", 4, "Buzz", "Fizz", 7, 8, "Fizz", "Buzz", 11, "Fizz", 13, 14, "FizzBuzz", 16, 17, "Fizz", 19, "Buzz", "Fizz", 22, 23, "Fizz", "Buzz", 26, "Fizz", 28, 29, "FizzBuzz"]
== STEP 4 (+ fetch_fizzbuzz_remote) ==
  Class memsize   : 528 B
  Method memsize  : 320 B
  ISeq  memsize   : 4592 B
  ISeq  bytes     : 2424 B

== STEP 5 (+ tiny2) ==
  Class memsize   : 528 B
  Method memsize  : 400 B
  ISeq  memsize   : 5048 B
  ISeq  bytes     : 2640 B

== STEP 6 (+ tiny3) ==
  Class memsize   : 528 B
  Method memsize  : 480 B
  ISeq  memsize   : 5504 B
  ISeq  bytes     : 2856 B

== STEP 7 (+ tiny10) ==
  Class memsize   : 912 B
  Method memsize  : 1040 B
  ISeq  memsize   : 8696 B
  ISeq  bytes     : 4372 B
```

## まとめ
rubyではクラスを定義すると、256(byte)、メソッド1つあたり80(byte)ほど消費することがわかりました。
特に意味を持たないメソッドでも、クラスやメソッドの占有する領域は増える傾向にあり、apiコールをする様なロジックを組むとクラスの占有する領域が増加する傾向にあることがわかりました。
再帰呼び出しは意外とメモリを食う事も分かり、今後実用的な関数など比較をしていきたいと感じました。
