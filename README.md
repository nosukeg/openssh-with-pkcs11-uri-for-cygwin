<div align="right">
openssh with pkcs11-uri for cygwin 2022-09-22<br>
added openssh-9.2p1-1 with pkcs11-uri for cygwin 2023-03-11<br>
added openssh-9.3p1-1 with pkcs11-uri for cygwin 2023-03-22<br>
added openssh-9.4p1-1 with pkcs11-uri for cygwin 2023-12-01
</div>

# PKCS#11 URIs を扱える ssh を Cygwin に導入する

## はじめに
2017年に、PKCS#11 URIs が [本家](https://www.openssh.com/) の [openssh-unix-dev](https://www.openssh.com/list.html) に於いて案内されてから今日（2022/9）まで、本家で取り込まれた様子は[見受けられません](https://marc.info/?l=openssh-unix-dev&w=2&r=1&s=pkcs+uri&q=b)。取り込んでいるのは、RedHat 系ディストリビューションのようです（本家で取り込まれない理由は分かりませんが、取り込まれることを期待しています）。

## やりたい事
SSH 公開鍵認証でリモートへ接続する際に、マイナンバーカード内のユーザ認証用鍵を優先して使いたい。
~~~
$ ssh -i "pkcs11:id=%01" -o IdentityAgent=none user@remote.machine
~~~
## 背景
ssh コマンドは、-i オプションで公開鍵認証に使う秘密鍵をファイル名で指定することができますが、マイナンバーカード内の鍵を指定する方法は無いようです。

## 関連するプログラム
<sub>※ 以下、パッケージの入手は [Cygwin](https://www.cygwin.com/) トップページにある [setup-x86_64.exe](https://www.cygwin.com/setup-x86_64.exe) を利用しています。</sub>

OpenSSH で PKCS#11 URIs を使う為に、`OpenSC` と `p11-kit` を利用します。また、`p11tool` をマイナンバーカードの URI を一覧・確認するために使います。

- OpenSC の導入は[こちら](https://github.com/nosukeg/cygwin-install-bat-for-opensc)に記載しています。
- [p11-kit](https://p11-glue.github.io/p11-glue/) を利用する為に、p11-kit パッケージを導入します（モジュール構成ファイル の準備は下記[参考](https://github.com/nosukeg/openssh-with-pkcs11-uri-for-cygwin#%E5%8F%82%E8%80%83)に例があります）。
- [p11tool](https://www.gnutls.org/) を利用する為に、GnuTLS パッケージを導入します。

| 関連プログラムの関係（階層）                                |
|:-----------------------------------------------------------:|
| p11tool、ssh コマンド                                       |
| p11-kit（*PKCS#11 URIs*、*Proxy Module:* p11-kit-proxy.so） |
| OpenSC（opensc-pkcs11.dll）                                 |
| カード リーダー                                             |
| マイナンバーカード                                          |

OpenSSH を make する際は`libedit`、`libssl`、`libfido2`、`libkrb5`、`libcrypt` 等の開発ライブラリが必要です。必要に応じて `パッケージ名-devel` パッケージを導入します。

## 導入の流れ
PKCS#11 URIs を組み込んだ OpenSSH を導入する流れは、Cygwin の OpenSSH ソースパッケージに対して、Fedora の OpenSSH ソースパッケージに同梱されている pkcs11-uri パッチを適用し、cygport でインストール先を /usr/local/OpenSSH（お好みです）にしたものを作成、配置するというものです。  
インストール先変更時に pkcs11-provider の指定もしています。

- Cygwin の OpenSSH ソースパッケージの入手
- Fedora の pkcs11-uri パッチの入手
- パッチの適用
- インストール先の変更と pkcs11-provider（p11-kit-proxy.so）の指定
- cygport の実行
- /usr/local/OpenSSH への配置

## 導入手順
<sub>※ 以下、ソースパッケージ等のバージョンは本文書作成時のものです。</sub>

### Cygwin の OpenSSH ソースパッケージを入手
`openssh-9.0p1-1-src.tar.xz` を入手します。

### Fedora の pkcs11-uri パッチの入手
Cygwin の OpenSSH が 9.0 portable ですので、それに対応した Fedora のリポジトリを探し入手します。

- まず、[Fedora WIKI](https://fedoraproject.org/wiki/Fedora_Project_Wiki) の 項目 `Want to get Fedora?` にある `Fedora Package Search` を開き、`openssh を検索`します。
- ヒットした openssh を開くと、9.0p1 に対応したリリースは Rawhide である事が確認できます。
- ページ上部にあるリンク `Source` を開きます。
- リリース一覧の安定バージョンに openssh-9.0p1-2.fc38 がありますので開きます。
- 項目 RPMs の src にある openssh-9.0p1-2.fc38.src.rpm をダウンロードします。
- ダウンロードした openssh-9.0p1-2.fc38.src.rpm を確認後、適当なディレクトリで展開し、`openssh-8.0p1-pkcs11-uri.patch` を入手します（確認・展開 の為に、rpm パッケージを導入します）。
~~~
$ rpm -K openssh-9.0p1-2.fc38.src.rpm
openssh-9.0p1-2.fc38.src.rpm: sha1 md5 OK
$ mkdir rpm2cpio
$ cd rpm2cpio
$ rpm2cpio ../openssh-9.0p1-2.fc38.src.rpm|cpio -id
$ ls *pkcs11-uri*
openssh-8.0p1-pkcs11-uri.patch
~~~

### パッチの適用
Cygwin のソースパッケージ（`openssh-9.0p1-1-src.tar.xz`）に含まれるパッチと `openssh-8.0p1-pkcs11-uri.patch` が対象としているファイルに重複があるか確認すると、configure.ac と ssh-keygen.c が重複していることが分かります。Cygwin のソースパッケージ（`openssh-9.0p1-1-src.tar.xz`）に含まれるパッチに手直しをして対応します。

パッチ対象ファイル（〇印が対象）
| ファイル名                      | openssh-9.0p1-1-src.tar.xz | openssh-8.0p1-pkcs11-uri.patch | 新 |
|:--------------------------------|:--------------------------:|:-----------------------------: |:--:|
| configure.ac                    | 〇                         | 〇                             | － |
| Makefile.in                     | －                         | 〇                             | － |
| regress/agent-pkcs11.sh         | －                         | 〇                             | － |
| regress/Makefile                | －                         | 〇                             | － |
| regress/pkcs11.sh               | －                         | 〇                             | 新 |
| regress/unittests/Makefile      | －                         | 〇                             | － |
| regress/unittests/pkcs11/test.c | －                         | 〇                             | 新 |
| ssh-add.c                       | －                         | 〇                             | － |
| ssh-agent.c                     | －                         | 〇                             | － |
| ssh-config.5                    | －                         | 〇                             | － |
| ssh.c                           | －                         | 〇                             | － |
| ssh-keygen.c                    | 〇                         | 〇                             | － |
| ssh-pkcs11-client.c             | －                         | 〇                             | － |
| ssh-pkcs11.c                    | －                         | 〇                             | － |
| ssh-pkcs11.h                    | －                         | 〇                             | － |
| ssh-pkcs11-uri.c                | －                         | 〇                             | 新 |
| ssh-pkcs11-uri.h                | －                         | 〇                             | 新 |
| sk-usbhid.c                     | 〇                         | －                             | － |
| sshconnect2.c                   | 〇                         | －                             | － |
| install-sh                      | 〇                         | －                             | － |


**Cygwin ソースパッケージ（`openssh-9.0p1-1-src.tar.xz`）を展開します。**
~~~
$ [ -d openssh-9.0p1-1.src ] && rm -r openssh-9.0p1-1.src
$ tar Jxf openssh-9.0p1-1-src.tar.xz
$ cd openssh-9.0p1-1.src/
~~~

**`openssh-8.0p1-pkcs11-uri.patch` パッチを適用した openssh-portable-9.0p1.tar.bz2 を作成します。**
- openssh-8.0p1-pkcs11-uri.patch
~~~
$ tar jxf openssh-portable-9.0p1.tar.bz2
$ cd openssh-portable/
$ patch -p1 < /path/to/openssh-8.0p1-pkcs11-uri.patch
$ cd ../
$ rm openssh-portable-9.0p1.tar.bz2
$ tar jcf openssh-portable-9.0p1.tar.bz2 openssh-portable
$ rm -r openssh-portable
~~~

**Cygwin のソースパッケージ（`openssh-9.0p1-1-src.tar.xz`）に含まれるパッチを手直しします。**
- 0001-compat-code-for-fido_dev_is_winhello.patch.patch
- 0006-Defer-token-PIN-prompt-to-handle-uv-as-well-as-clien.patch.patch
~~~
$ patch < /path/to/0001-compat-code-for-fido_dev_is_winhello.patch.patch
$ patch < /path/to/0006-Defer-token-PIN-prompt-to-handle-uv-as-well-as-clien.patch.patch
~~~

### インストール先の変更と pkcs11-provider（p11-kit-proxy.so）の指定
- openssh.cygport.patch
~~~
$ patch < /path/to/openssh.cygport.patch
~~~

### cygport の実行
~~~
$ cygport openssh.cygport prep
$ cygport openssh.cygport compile
~~~

### /usr/local/OpenSSH への配置
※ 配置後、/usr/local/OpenSSH/bin、/usr/local/OpenSSH/share/man へ PATH を通し ssh コマンドを利用します。
~~~
$ [ -d /usr/local/OpenSSH ] && rm -r /usr/local/OpenSSH
$ cd openssh-9.0p1-1.x86_64/build/
$ make install
$ ln -s /usr/local/OpenSSH/bin/ssh.exe /usr/local/OpenSSH/bin/slogin
$ ln -s /usr/local/OpenSSH/share/man/man1/ssh.1 /usr/local/OpenSSH/share/man/man1/slogin.1
$ cd ../src/openssh-portable/contrib/cygwin/
$ make cygwin-postinstall prefix=/usr/local/OpenSSH sysconfdir=/usr/local/OpenSSH/etc PRIVSEP_PATH=/usr/local/OpenSSH/var/empty
~~~

# PKCS#11 URIs を扱える openssh-9.2p1-1 を Cygwin に導入する

Cygwin のソースパッケージ `openssh-9.2p1-1-src.tar.xz` には `openssh-9.0p1-1-src.tar.xz` にあった Cygwin 用パッチがありません（関係者の仕事に敬意を表します）。その為、**pkcs11-uri パッチのみの適用**になります。

ここでは pkcs11-uri パッチとして `openssh-8.0p1-pkcs11-uri.for.openssh-9.2p1-1.patch` を用意しました。これは `Fedora openssh-9.0p1-2.fc38` の pkcs11-uri パッチを `Cygwin openssh-9.2p1-1` に適用した際に生じる offset がゼロになるようにしたものです。Makefile.in で二つの fuzz が出ますので、その内容を確認します。

<sub>※ 2023-03-11 時点で Fedora Rawhide に同梱されている Fedora openssh-9.0p1-12.fc39.1 の pkcs11-uri パッチは、Fedora openssh-9.0p1-2.fc38 と同じです。</sub>

## 導入手順
<sub>※ 以下、ソースパッケージ等のバージョンは 2023-03-11 時点のものです。</sub>

### Cygwin の OpenSSH ソースパッケージを入手
`openssh-9.2p1-1-src.tar.xz` を入手します。

### Fedora の pkcs11-uri パッチの入手
[上記手順](https://github.com/nosukeg/openssh-with-pkcs11-uri-for-cygwin#fedora-%E3%81%AE-pkcs11-uri-%E3%83%91%E3%83%83%E3%83%81%E3%81%AE%E5%85%A5%E6%89%8B)に倣って、もしくは、ここにある `openssh-8.0p1-pkcs11-uri.for.openssh-9.2p1-1.patch` を入手します。

### パッチの適用
**Cygwin ソースパッケージ（`openssh-9.2p1-1-src.tar.xz`）を展開します。**
~~~
$ [ -d openssh-9.2p1-1.src ] && rm -r openssh-9.2p1-1.src
$ tar Jxf openssh-9.2p1-1-src.tar.xz
$ cd openssh-9.2p1-1.src/
~~~

**pkcs11-uri パッチを適用した openssh-portable-9.2p1.tar.bz2 を作成します（以下は `openssh-8.0p1-pkcs11-uri.for.openssh-9.2p1-1.patch` で行った例です）。**
- openssh-8.0p1-pkcs11-uri.for.openssh-9.2p1-1.patch
~~~
$ tar jxf openssh-portable-9.2p1.tar.bz2
$ cd openssh-portable/
$ patch -p1 < /path/to/openssh-8.0p1-pkcs11-uri.for.openssh-9.2p1-1.patch
$ cd ../
$ rm openssh-portable-9.2p1.tar.bz2
$ tar jcf openssh-portable-9.2p1.tar.bz2 openssh-portable
$ rm -r openssh-portable
~~~

### インストール先の変更と pkcs11-provider（p11-kit-proxy.so）の指定
- openssh-9.2p1-1.cygport.patch
~~~
$ patch < /path/to/openssh-9.2p1-1.cygport.patch
~~~

### cygport の実行
~~~
$ cygport openssh.cygport prep
$ cygport openssh.cygport compile
~~~

### /usr/local/OpenSSH への配置
※ 配置後、/usr/local/OpenSSH/bin、/usr/local/OpenSSH/share/man へ PATH を通し ssh コマンドを利用します。
~~~
$ [ -d /usr/local/OpenSSH ] && rm -r /usr/local/OpenSSH
$ cd openssh-9.2p1-1.x86_64/build/
$ make install
$ ln -s /usr/local/OpenSSH/bin/ssh.exe /usr/local/OpenSSH/bin/slogin
$ ln -s /usr/local/OpenSSH/share/man/man1/ssh.1 /usr/local/OpenSSH/share/man/man1/slogin.1
$ cd ../src/openssh-portable/contrib/cygwin/
$ make cygwin-postinstall prefix=/usr/local/OpenSSH sysconfdir=/usr/local/OpenSSH/etc PRIVSEP_PATH=/usr/local/OpenSSH/var/empty
~~~

# PKCS#11 URIs を扱える openssh-9.3p1-1 を Cygwin に導入する

内容、手順は openssh-9.2p1-1 に倣います。パッチとして次の2ファイルを用意しました。

- `openssh-8.0p1-pkcs11-uri.for.openssh-9.3p1-1.patch`
- `openssh-9.3p1-1.cygport.patch`

# PKCS#11 URIs を扱える openssh-9.4p1-1 を Cygwin に導入する

内容、手順は openssh-9.2p1-1 に倣います。パッチとして次の2ファイルを用意しました。

<sub>※ 2023-12-01 現在、Fedora には 9.4 に対応したリリースがありません。</sub><br>
<sub>※ その為、上記 9.3 用のパッチを 9.4 に適用できるように変更しています。</sub><br>
<sub>※ 変更に際して生じる offset をゼロにしたところで、Makefile.in で 二つ、ssh-agent.c で一つの fuzz が出ます。今回はこの fuzz を修正しています。</sub><br>
<sub>※ 又、ssh-pkcs11-client.c Hunk #1、ssh-pkcs11.c Hunk #48 が失敗しますので、それも修正しています。</sub><br>
<sub>※ sh-pkcs11-client.c Hunk #1 は pkcs11_add_provider() の先頭へ debug_f() を挿入するパッチでした。</sub><br>
<sub>※ ssh-pkcs11.c Hunk #48 はそのような単純なものではありませんでしたが、処理の意味を解釈せずに、字面で合わせ込んでいます。</sub><br>
<sub>※ 以上で、**私の利用範囲では不具合は無いようですが、その他の確認はしていません**。</sub><br>
<sub>※ 加えて、configure の際に、zlib のバージョンチェックがエラーとなり、失敗します。</sub><br>
<sub>※ これは zlib のバージョンが a.b.c となっている事を前提にチェックしていることが原因（現在のバージョンは 1.3）でしたので、configure.ac に a.b でもチェックが通るような修正を入れています。</sub><br>

- `openssh-8.0p1-pkcs11-uri.for.openssh-9.4p1-1.patch`
- `openssh-9.4p1-1.cygport.patch`

# マイナンバーカード内のユーザ認証用鍵でリモートへ接続してみる
- ユーザ認証用公開鍵を取り出します（取り出し方は[こちら](https://github.com/nosukeg/cygwin-install-bat-for-opensc#ssh-%E5%85%AC%E9%96%8B%E9%8D%B5%E8%AA%8D%E8%A8%BC%E3%81%AE%E6%BA%96%E5%82%99)に例があります）。
- 取り出した公開鍵を接続先へ配置します。
- マイナンバーカードの URI を確認します（下記[参考](https://github.com/nosukeg/openssh-with-pkcs11-uri-for-cygwin#%E5%8F%82%E8%80%83)に例があります）。
- 接続を試します。

  **※ マイナンバーカードの PIN 試行回数を越えないように注意します。**
~~~
$ ssh -i "pkcs11:id=%01" -o IdentityAgent=none user@remote.machine
~~~

# 参考

## p11-kit と p11tool（GnuTLS PKCS#11 tool）
ここでは モジュール、トークン、オブジェクト（暗号オブジェクト） という言葉は、次の様なものを指しているとして便宜的に使っています。
| 言葉         | 便宜的に指しているもの                                                           |
|:-------------|:---------------------------------------------------------------------------------|
| モジュール   | トークンにアクセスするための API を備えたライブラリ など                         |
| トークン     | オブジェクトを収めた ファイル、鍵束、スマートカードやセキュリティモジュール など |
| オブジェクト | 秘密鍵、公開鍵、電子証明書、PIN など                                             |

### [p11-kit](https://p11-glue.github.io/p11-glue/)
p11-kit によって、暗号オブジェクトを URI で識別（指定）する事ができるようになります。

p11-kit を利用する際は、グローバル構成ファイル `/etc/pkcs11/pkcs11.conf` 及び モジュール構成ファイル `/etc/pkcs11/modules/*.module` を準備します。

モジュール構成ファイル（.module）の例
~~~
$ cat /etc/pkcs11/modules/OpenSC.module
module: /usr/local/OpenSC/lib/opensc-pkcs11.dll
~~~
p11-kit の扱えるモジュールの確認
~~~
$ p11-kit list-modules
~~~

### [p11tool（GnuTLS PKCS#11 tool）](https://www.gnutls.org/)
p11tool は、トークンに収められている暗号オブジェクトを PKCS#11 で操作するプログラムで、暗号オブジェクトの操作は p11-kit を介して行います。

コマンド例  
  
※ コマンドを正しく動作させる為に、URI をダブルクォートで囲う必要があるようです。  
**※ マイナンバーカードの PIN 試行回数を越えないように注意します。**
~~~
$ p11tool --list-tokens
$ p11tool --list-all "pkcs11:token=JPKI%20%28User%20Authentication%20PIN%29" --login
$ p11tool --list-all "pkcs11:token=JPKI%20%28Digital%20Signature%20PIN%29" --login
~~~
