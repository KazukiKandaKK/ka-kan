---
title: "Rubyのrequireはどのような処理をしているのか覗いてみる"
emoji: "🔬"
type: "tech"
topics:
  - "ruby"
published: true
published_at: "2025-06-30 06:00"
---

## はじめに
Rubyのrequireの中身ってどうなっているんだろうという知的好奇心のもと、探ってみました。

## マシンスペック
MacBook Air M2 arm64

## 前提知識
### システムコール
こちらの記事で取り扱ってますので、適宜ご参照ください。
https://zenn.dev/ka_kan/articles/b4ac244e17008e

## 準備
### Dockerコンテナで環境構築
下記のコマンドでDocker上でubuntu24.04の仮想環境を立ち上げて接続します。
```bash
docker run -it --rm -v $(pwd):/mnt ubuntu:24.04 bash
```
コンテナ内で必要なパッケージをインストールします。
```bash
apt update && apt install -y ruby-full build-essential autoconf automake libtool gdb strace git \
  libyaml-dev libssl-dev zlib1g-dev \
  libffi-dev libgdbm-dev libreadline-dev libncurses-dev \
  pkg-config libsqlite3-dev
```
### Rubyをデバッグモードでビルド
```bash
git clone https://github.com/ruby/ruby.git
cd ruby

./autogen.sh
./configure --prefix=/usr/local/ruby-debug --enable-debug-env CFLAGS='-O0 -g3'
make -j8
make install
```
### 検証用のソースコードの作成
単純にrequireするだけのファイルを準備しておきます。
```bash
echo 'require "json"' > require_json.rb
```
## irbでプログラムを動かして調査
### requireがファイルを検索する際の検索パス一覧を確認する。
```bash
irb
pp $LOAD_PATH
["/usr/local/lib/site_ruby/3.2.0",
 "/usr/local/lib/aarch64-linux-gnu/site_ruby",
 "/usr/local/lib/site_ruby",
 "/usr/lib/ruby/vendor_ruby/3.2.0",
 "/usr/lib/aarch64-linux-gnu/ruby/vendor_ruby/3.2.0",
 "/usr/lib/ruby/vendor_ruby",
 "/usr/lib/ruby/3.2.0",
 "/usr/lib/aarch64-linux-gnu/ruby/3.2.0"]
=> 
["/usr/local/lib/site_ruby/3.2.0",
 "/usr/local/lib/aarch64-linux-gnu/site_ruby",
 "/usr/local/lib/site_ruby",
 "/usr/lib/ruby/vendor_ruby/3.2.0",
 "/usr/lib/aarch64-linux-gnu/ruby/vendor_ruby/3.2.0",
 "/usr/lib/ruby/vendor_ruby",
 "/usr/lib/ruby/3.2.0",
 "/usr/lib/aarch64-linux-gnu/ruby/3.2.0"]
```
### 通常通りrequireをしてみる
```bash
require 'json'
=> true
```

### 存在しないファイルをrequireしてみる
```bash
require 'foo_test'
<internal:/usr/lib/ruby/vendor_ruby/rubygems/core_ext/kernel_require.rb>:86:in `require': cannot load such file -- foo_test (LoadError)
	from <internal:/usr/lib/ruby/vendor_ruby/rubygems/core_ext/kernel_require.rb>:86:in `require'
	from (irb):4:in `<main>'
	from /usr/lib/ruby/gems/3.2.0/gems/irb-1.6.2/exe/irb:11:in `<top (required)>'
	from /usr/bin/irb:25:in `load'
	from /usr/bin/irb:25:in `<main>'
```
## straceでシステムコールの確認をする
staceコマンドでrequireする際に呼ばれるシステムコールの確認をします。
```bash
strace -e trace=openat,read,write,stat,mmap /usr/local/ruby-debug/bin/ruby require_json.rb |& less

# mmapやread, openatが確認できる。
```
中身を眺めてみると、下記のようにjsonに関するファイルを探して読み込んでメモリに展開している様な挙動が伺えます。処理が多く、requireひとつでも沢山の処理を行っていることが伺えます。
```bash
openat(AT_FDCWD, "/usr/local/ruby-debug/lib/ruby/3.5.0+2/aarch64-linux/json/ext/generator.so", O_RDONLY|O_CLOEXEC) = 5
read(5, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0\267\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
mmap(NULL, 197504, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_DENYWRITE, -1, 0) = 0xffff7339f000
mmap(0xffff733a0000, 131968, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 5, 0) = 0xffff733a0000
mmap(0xffff733bf000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 5, 0xf000) = 0xffff733bf000
```
## GDBでRubyの動作を追う
### gdbとは
C言語のデバッガです。ターミナル上で使用して、ステップ実行や任意の場所にブレークポイントを設定できます。
統合開発環境でブレークポイントを設定してデバックができますが、それのコマンドライン版の様なイメージです。
### 実行
```bash
gdb /usr/local/ruby-debug/bin/ruby

break rb_f_require # rb_f_require関数でブレークポイントを設定
Breakpoint 2 at 0xb9184: file load.c, line 1148.

run require_json.rb # requireをするrubyファイルを実行
Starting program: /usr/local/ruby-debug/bin/ruby require_json.rb
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
[New Thread 0xffffde5ff180 (LWP 17390)]

Thread 1 "ruby" hit Breakpoint 2, rb_f_require (obj=281474837040440, fname=281474382755000)
    at load.c:1148
1148	    return rb_require_string(fname);
```
スタックトレースを確認してみます。
```bash
bt
#0  rb_f_require (obj=281474837040440, fname=281474382755000) at load.c:1148
#1  0x0000aaaaaacc267c in ractor_safe_call_cfunc_1 (recv=281474837040440, argc=1, argv=0xfffff7b1f050, 
    func=0xaaaaaab59174 <rb_f_require>) at /ruby/vm_insnhelper.c:3607
#2  0x0000aaaaaacc32a4 in vm_call_cfunc_with_frame_ (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00, argc=1, argv=0xfffff7b1f050, stack_bottom=0xfffff7b1f048)
    at /ruby/vm_insnhelper.c:3784
#3  0x0000aaaaaacc3518 in vm_call_cfunc_with_frame (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00) at /ruby/vm_insnhelper.c:3830
#4  0x0000aaaaaacc365c in vm_call_cfunc_other (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00) at /ruby/vm_insnhelper.c:3856
#5  0x0000aaaaaacc3adc in vm_call_cfunc (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00) at /ruby/vm_insnhelper.c:3938
#6  0x0000aaaaaacc66fc in vm_call_method_each_type (ec=0xaaaaab07db30, cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00) at /ruby/vm_insnhelper.c:4763
#7  0x0000aaaaaacc70fc in vm_call_method (ec=0xaaaaab07db30, cfp=0xfffff7c1efa0, calling=0xffffffffcc00)
    at /ruby/vm_insnhelper.c:4900
#8  0x0000aaaaaacc7230 in vm_call_general (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, 
    calling=0xffffffffcc00) at /ruby/vm_insnhelper.c:4933
#9  0x0000aaaaaacc9cc8 in vm_sendish (ec=0xaaaaab07db30, reg_cfp=0xfffff7c1efa0, cd=0xaaaaab193030, 
    block_handler=0, method_explorer=mexp_search_method) at /ruby/vm_insnhelper.c:5991
#10 0x0000aaaaaacd1944 in vm_exec_core (ec=0xaaaaab07db30) at /ruby/insns.def:899
#11 0x0000aaaaaace9e18 in rb_vm_exec (ec=0xaaaaab07db30) at vm.c:2621
#12 0x0000aaaaaaceaa34 in rb_iseq_eval (iseq=0xffffdc98f8a0) at vm.c:2878
#13 0x0000aaaaaad24264 in load_with_builtin_functions (feature_name=0xaaaaaaee80d8 "gem_prelude", 
    table=0x0) at builtin.c:60
#14 0x0000aaaaaad242e8 in Init_builtin_features () at builtin.c:89
#15 0x0000aaaaaac14230 in ruby_init_prelude () at ruby.c:1772
#16 0x0000aaaaaac14388 in ruby_opt_init (opt=0xfffffffff1f8) at ruby.c:1843
#17 0x0000aaaaaac15018 in prism_script (opt=0xfffffffff1f8, result=0xffffffffddc0) at ruby.c:2232
#18 0x0000aaaaaac15e28 in process_options (argc=0, argv=0xfffffffff608, opt=0xfffffffff1f8)
    at ruby.c:2560
#19 0x0000aaaaaac175c4 in ruby_process_options (argc=2, argv=0xfffffffff5f8) at ruby.c:3196
--Type <RET> for more, q to quit, c to continue without paging--
```
stepやsで中身を追えるので確認できます。

## まとめ
今回は、requireの挙動を確認する記事でした。
単純にjsonをrequireするだけでも、Rubyのコードは複雑に動いていることが確認できました。
