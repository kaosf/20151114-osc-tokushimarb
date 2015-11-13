% ConoHa VPSの操作をするCLIツールを作ってみた
% ka
% 2015-11-14

# Author

ka

[![Gravatar](https://gravatar.com/avatar/884be098693425b409d25aaec5091de8?s=150)](https://gravatar.com/ka000)

Website: [kaosfield](http://www.kaosfield.net)

Twitter: [ka_](https://twitter.com/ka_)

GitHub: [kaosf](https://github.com/kaosf)

# はじめに

このスライドは公開されています

このスライドのリポジトリは[https://github.com/kaosf/20151114-osc-tokushimarb](https://github.com/kaosf/20151114-osc-tokushimarb)にあります

間違いなどを見つけた場合は

<ul>
<li>メールで連絡</li>
<li>Issueで報告</li>
<li>Pull Requestで修正案を提出</li>
</ul>

などしていただけるとありがたいです (※下に行くほど嬉しいです)

# License

CC BY-NC-SA 4.0

[![CC BY-NC-SA 4.0](https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-nc-sa/4.0/)

Copyright (C) 2015 ka

# 余談

最近 NC を除いてもいいんじゃないかって気もしてる

これは BY-NC-SA で

# おしながき

一つのことを上手くやれなくても一つのことだけに集中しようという話

Ruby製のCLIツールを作る方法

net/httpライブラリをちゃんと自分の手で使ってみたという話

OpenStackのAPIを普通に扱う方法を諦めた話

ユーザが自分しか居なくても頑張れる話

などなど

# 対象者

VPSの操作をWebブラウザ上でやるの面倒くさいと思っている人

ConoHa VPSを料金抑えて使いたいと思っている人

CUIの世界に閉じこもりたい人

Rubyの話が何か聞きたい人

# 想定するレベル

レンタルサーバなりVPSなりを使ったことがある人

Rubyの文法とかgemの使い方とか入門レベルは流石に終わっている人

ターミナルでの操作を説明されたり見せられたりしても「さっきから一体何をやってるんだ？」とはならない程度の人

# ConoHa VPSを軽く紹介

AWSの天下布武に始まり

さくら GMO Kagoya その他諸々

の色んな業者がクラウドやVPSのサービスをやっている

そんな中でGMOがやってるConoHa VPSというのが2015年の5月だか6月だか(うろ覚え)の頃に  
リニューアルをやって安いわ性能は良いわで私が今現在大変気に入っている

[ConoHa - Low-cost! Easy to use! Boot up in only 40sec.](https://www.conoha.jp/en)

#

気に入っているのは以下の点

<ul>
<li>初期費用不要</li>
<li>1時間単位で課金(1.3円ずつ)</li>
<li>1ヶ月900円</li>
<li>操作用のAPIが公開されている</li>
</ul>

# 私の今までの利用形態

ノートPC(なるべくバッテリー長持ちするやつ)を常用する

開発はSSH越しに別マシン(VPSかデスクトップマシンであることが多い)で行う

## その場合の料金

さくらのVPSを使っていた

VPSの利用料金は最低でおおよそ月額1,000円程度なので開発に利用する環境の料金が年間10,000円ちょい必要

一番低いスペックで年間10,000円はかかるので次のスペックを開発用に使おうと思ったら  
4年くらいでもっと良いノートPC買えたよね？みたいになりそう

# ConoHaに変わって

VPSのコントロールパネル(VPSを作って壊してする画面)がWebブラウザで操作出来るものとして提供される  
(これは別にどこもそれほど変わらない)

しかしAPIが提供された

時間単位で課金されるなら作っては壊し作っては壊しを出来れば利用料金を1/3くらいに抑えられる？

ならばコマンドラインで作って壊して出来るようにしてしまえ

これが一番最初の動機

# 一つのことだけに集中しよう

UNIX哲学にこういうのがある

> 一つのことを、うまくやれ

[UNIX哲学 - Wikipedia](https://ja.wikipedia.org/wiki/UNIX%E5%93%B2%E5%AD%A6)

#

例えば今回作ってみたツールでは

**シェルスクリプトを実行するのが面倒なのを簡単にしたい**

これに集中した

#

最初はConoHa APIをサンプルに則りcurlしまくるシェルスクリプト群を使ってた

```bash
AUTH_TOKEN=`head -1 authtoken`
TENANT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ROOT_PASS=xxxxxxxxxx
ADMIN_PASS=$ROOT_PASS
if [ $# -lt 1 ]; then
  IMAGE_REF=fa67ec7b-b9b4-4633-9012-fc5a6303aba7 # CentOS 6.6
elif [ $1 = ubuntu ]; then
  IMAGE_REF=2b03327f-d453-4c7d-91c9-8b9924b6ea88 # Ubuntu 14.04 amd64
elif [ $1 = centos7 ]; then
  IMAGE_REF=edc9457e-e4a8-4974-8217-c254d215b460 # CentOS 7.1
fi

FLAVOR_REF=7eea7469-0d85-4f82-8050-6ae742394681 # RAM 1GB
curl -X POST \
  -H "Accept: application/json" \
  -H "X-Auth-Token: $AUTH_TOKEN" \
  -d '{"server": {"adminPass": "'$ADMIN_PASS'","imageRef": "'$IMAGE_REF'", ... }}' \
  https://compute.tyo1.conoha.io/v2/$TENANT_ID/servers 2> /dev/null | \
  ruby -rjson -e 'puts JSON.parse(STDIN.read)["server"]["id"]'
```

これを `1-create-vps.sh` として用意してた

#

認証用のtokenは `authtoken` というファイルの一行目に書いて  
以下のシェルスクリプトで書き換えるようにしてた

``` bash
curl -X POST -H "Accept: application/json" \
  -d '{"auth":{"passwordCredentials":{"username":"xxxx","password": "xxxxxx"},"tenantId":"xxxxxxxxxxx"}}' \
  https://identity.tyo1.conoha.io/v2.0/tokens 2> /dev/null | \
  ruby -rjson -e 't=JSON.parse(STDIN.read)["access"]["token"]["id"];puts t;File.open("authtoken","w").puts t'
```

JSONのパースは面倒くさいのでRubyのワンライナーで済ませていた

これを `0-authenticate.sh` として用意してた

# 余談

ただ世の中にはBashでJSONのパースも済ませてしまう人も居るらしい

[dominictarr/JSON.sh](https://github.com/dominictarr/JSON.sh)

#

で，以上のようなシェルスクリプト(Rubyの力をJSONパースに利用させてもらう)群を  
他にもたくさん作っていて，ひとまず

<ol>
<li>認証する(トークンを得て保存する)</li>
<li>VPSを作る</li>
<li>IPアドレスを取得する</li>
<li>SSH接続を開始する</li>
<li>VPSをシャットダウンする</li>
<li>VPSをイメージとして保存する</li>
<li>VPSを消す(利用料節約のため)</li>
<li>VPSを保存したイメージから再構築する</li>
</ol>

が出来るようになった

# 余談

JSONをパースするツールとしては jq という有名ドコロがある

[jq](https://stedolan.github.io/jq/)

しかしこれをインストールするよりもRubyを普段使いする自分にとっては

```
ruby -rjson -e 'puts JSON.parse(STDIN.read)["key"][1]'
```

の方が楽かな？と思った

Rubyのワンライナーで `STDIN.read` や `puts` は省略出来るとか  
何かそういうのがあったような気もしたので知ってたら教えて下さい

# 余談

最近はあんまり見かけませんが

「それ○○で出来るよ？」

というマサカリに対しては意外と

「自分はこっちの方が楽だったので」

で普通に対応出来ることがある

※もちろんきちんと比較した上でないとダメ

# ほぼフルスクラッチHTTP

net/httpライブラリをちゃんと自分の手で使ってみたという話

#

シェルスクリプトでcurl叩きまくりプログラムでもとりあえず自分のやりたいことは出来た

しかし一度何かが出来るようになるとすぐに不満が出てくる

<ul>
<li>コマンド1つでインストールしたい</li>
<li>コマンドをもっと短くしたい</li>
<li>Rubyに依存したくない</li>
<li>BashだけでJSONパースが面倒そうなら逆にRubyだけで完結したい</li>
</ul>

# そうだgem作ろう

Bashで `JSON.parse(STDIN.read)` の代わりは難しいが  
Rubyでcurlコマンドの代わりは出来そう

gemにしてしまえばコマンド一発でインストール出来る

認証は設定ファイルをホームディレクトリに置いてそれを勝手に書き換えてくれれば扱いやすい

#

ということで自分でcurlの代わりをすることになる

幸いRubyには(他の言語でもこの辺はまずサポートされていると思うが)標準で  
HTTPの各種リクエスト，レスポンスを扱うライブラリが用意されている

#

便利メソッドがちょくちょくある(curlコマンド並の手軽さを実現？)

[library net/http (Ruby 2.2.0)](http://docs.ruby-lang.org/ja/2.2.0/library/net=2fhttp.html)

しかしリクエストヘッダを `application/json` にしたかったり  
https 通信にしたかったりするので出来るだけ細かく動かしたい

# まずURI

```ruby
require 'uri'

uri = URI.parse 'https://hostname/a/b/c'
uri.class #=> URI::HTTPS
uri.host #=> "hostname"
uri.port #=> 443
uri.request_uri #=> "/a/b/c"

uri = URI.parse 'http://myhost.com/some/where.html'
uri.class #=> URI::HTTP
uri.host #=> "myhost.com"
uri.port #=> 80
uri.request_uri #=> "/some/where.html"
```

多分これが一番簡単で細かいと思います

# HTTPリクエスト

GETリクエストの場合

```ruby
require 'net/https'

https = Net::HTTP.new(uri.host, uri.port)
https.use_ssl = true
req = Net::HTTP::Get.new(uri.path)
req['Content-Type'] = 'application/json'
req['Accept'] = 'application/json'
req['X-Auth-Token'] = "0123456789abcdef0123456789abcdef"
```

`https.use_ssl = true` これでhttps通信出来る

各種リクエストヘッダの設定はHashを更新するかのように出来る

# HTTPリクエスト

POSTリクエストの場合

```ruby
require 'net/https'
require 'json'

https = Net::HTTP.new(uri.host, uri.port)
https.use_ssl = true
req = Net::HTTP::Post.new(uri.request_uri)
req['Content-Type'] = 'application/json'
req['Accept'] = 'application/json'
req['X-Auth-Token'] = "0123456789abcdef0123456789abcdef"
req.body = payload.to_json
```

`Net::HTTP::Get` クラスの代わりに `Net::HTTP::Post` クラスのインスタンスを作る

`payload`にリクエスト用のJSONの元になるHashを設定する

# payload例

```ruby
payload = {
  server: {
    adminPass: "very-secure-root-password",
    imageRef: "01234567-89ab-cdef-0123-456789abcdef",
    flavorRef: "01234567-89ab-cdef-0123-456789abcdef",
    key_name: "registered-public-key-name",
    security_groups: [
      {name: 'default'},
      {name: 'gncs-ipv4-all'}
    ]
  }
}
```

# curlの場合

数ページ前に出てきたcurlの例は以下の通り

```bash
curl -X POST \
  -H "Accept: application/json" \
  -H "X-Auth-Token: $AUTH_TOKEN" \
  -d '{"server": {"adminPass": "'$ADMIN_PASS'","imageRef": "'$IMAGE_REF'", ... }}' \
  https://compute.tyo1.conoha.io/v2/$TENANT_ID/servers 2> /dev/null | \
  ruby -rjson -e 'puts JSON.parse(STDIN.read)["server"]["id"]'
```

長くなりすぎるので ... で省略しているが

```
  -d '{"server": {"adminPass": "'$ADMIN_PASS'","imageRef": "'$IMAGE_REF'", ... }}'
```

ここでJSONを文字列として渡していた

# HTTPリクエスト

DELETEリクエストの場合

`Net::HTTP::Get` の代わりに `Net::HTTP::Delete` を用いる

後はGETリクエストのときと変わらない

# HTTPレスポンス

作っておいた `Net::HTTP::Get` などのインスタンスを

```ruby
res = https.request(req)
```

のようにこれまたあらかじめ作っておいた `Net::HTTP` のインスタンスの `request` メソッドに渡してやる

そうして受け取れる `res` が `Net::HTTPResponse` のインスタンスとなり

```ruby
res.code #=> 200, 401 など
res.body # レスポンスの本体(今回はJSONの文字列が入ってくる)
```

のように扱える

#

<ul>
<li>https通信出来る</li>
<li>GET/POST/DELETEのHTTPリクエストが出せる</li>
<li>リクエストヘッダにapplication/jsonを指定出来る</li>
<li>レスポンスを文字列で受け取れる</li>
</ul>

これが出来るようになったのでひとまず今回のcurlコマンドの代わりとしては上等である

# Ruby製のCLIツールを作る方法

まずgemの作り方は分かっているものとする

分かってなくても3行で解説出来る

1. Rubygems.org に登録する
2. Bundlerをインストールして以下を実行
```
bundle gem your_library_name
```
3. コードを書いてコミットして以下を実行
```
rake release
```

# CLIツールを作るには？

普通に `bundle gem foobar` を実行してgemを作ってもコマンドは作れない

`exe`というディレクトリを作ってそこに実行可能スクリプトを用意する

# 昔は違った

ここでややこしいのが昔はデフォルトが `exe` ディレクトリではなく  
`bin` ディレクトリだったこと

ある時期から変わった

とは言っても `*.gemspec` ファイルにその指定はあるのでそこを好きに書き換えれば良い

### 該当箇所

```ruby
spec.executables = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
```

# そうなってないかもしれない

もしあなたが今の自分の環境で同様のことを行っても  
先に述べた状態になっていないかもしれない

その場合は Bundler 自体のバージョンが古いかもしれない

```bash
gem uninstall bundler
gem install bundler
```

古い Bundler は消してしまって新しいものを積極的に取り入れよう

私は現在 1.10.6 を使用中

# 諦めも肝心

OpenStackのAPIを普通に扱う方法を諦めた話

# 事前調査

ConoHa VPSはOpenStackの技術を採用している

OpenStackは有名なのでそれを操作するRubyのgemくらい普通にあってもいいだろうと思う

調べると以下のものを見つける

[ruby-openstack/ruby-openstack](https://github.com/ruby-openstack/ruby-openstack)

# が

READMEを読んで「自分がやりたいだけのことからかけ離れてる気がする」

そして何よりも「ボリュームがパない」

READMEが462行，28kBもある

# 感想

READMEを読んでも自分がやりたいだけのことを知るのに時間かかりそう

ConoHaのAPIドキュメントのcurlのサンプルを試しまくって  
それで積んでしまった実績があるのでそれを再利用出来るだけにする方が楽そう

なので不採用

# 事前調査2

そもそもConoHaのVPSを操作するgemが自分が作らなくてもあるのではないか

以下のものを見つける

[hironobu-s/vagrant-conoha](https://github.com/hironobu-s/vagrant-conoha/)

# が

これはOpenStackを操作するVagrant用のProviderからのforkであると分かる

Vagrant(というかVMwareないしVirtualBox)を使うのが  
無理な人も居るからConoHa VPSを採用しているんだぞという節がある

#

実際に自分がやりたかったのは

```bash
vagrant up
vagrant ssh
```

の代わりとしての

```bash
conoha create
conoha ssh
```

的な操作である

# 結論

なのでやっぱり作ってしまおうと思った

あと `conoha` ってだけの名前の gem がまだなかったので儲けモンだとも思った(名前は早い者勝ち)

# ユーザが自分しか居なくても頑張れる話

# GitHubに草が生えるから頑張れる

ありがちなモチベーション

[kaosf (ka)](https://github.com/kaosf)

# こうやってこの場で発表出来る

これは素直に面白い

自分が作ったものをネタに話出来るのは楽しい

tokushima.rbでもまた話したい

# 完成が無い

よく言われることだが「どこまで作っても完成は無い」

やりたいことが次から次に出てくる

しかしその度にconoha gemを肥大化させてはUNIX哲学に反するのでレイヤーをきちんと分けていく

そういう工夫も楽しい

# ぶっちゃけ

単にものづくりは楽しいということ

個人の趣味レベルのプロダクトでソースコードを隠すメリットはそんなに無いと思う

(※今Vectorとか窓の杜とかクローズドソースで作者も連絡不能なプロダクトが大量にあるだけみたいな場になってません？)

OSSは「別に自分は損するわけじゃないし」ってだけの動機で出来る

(※ただし著作権や特許の侵害に対しては気を付けるように)

ひとときひとときの活動記録であるコミットの集合体のリポジトリを世界に公開してしまう

こんな素晴らしい日記としてツールがあるだろうか？

# 今後の展望

HTTPの実行を司る部分，CLIツールとしての機能，  
をそれぞれ  
`conoha_api_client`, `conoha`  
に分割したい

というのも複数のアカウントを切り替えなければならないことがあるので

CLIツールで

```bash
conoha userselect ka
conoha userselect other
```

のように切り替えが出来てその後の操作が全てそのユーザで行えるようにしたい

現状HTTPリクエスト出す部分とCLIツールとしての部分は最初から分離しようと思って  
分離してあるが認証関連の部分がどっぷり前者に入っている

# 今後の展望

ConoHaのAPI用のアクセストークンは有効期限が1日だけなので24時間以上開くと

```bash
conoha authenticate
```

しなければならない

操作の度にアクセストークンを取得するか？

操作の度に401が帰ってきたらアクセストークン再取得を行うようにするか？

どちらかをやりたい

# 今後の展望

現状

```bash
conoha ssh UUID
```

するのがマニュアルの通りなのだが実用的にしんどいので

```bash
conoha ssh 0
```

とすれば 0 番目のVPSに対して操作が行えるようにしてある(READMEには書いてない)

というのも複数のVPSが存在する際に順番が保証されるのかどうか分からないので

# 今後の展望

なので `Conohafile` (まるで `Vagrantfile` のように) を用意して

それが存在するディレクトリで


```bash
conoha ssh
```

すればVPSが勝手に選ばれるようにするとかやってみたい

が現状2つのVPSを頻繁に操作することも無い(一回ログインしたらそのままになる)し

```bash
conoha ssh 0
```

で困ってないので可能性は薄い

# 最後に

ちなみに作ったものは以下になります

[conoha | RubyGems.org | your community gem host](https://rubygems.org/gems/conoha)

[kaosf/conoha](https://github.com/kaosf/conoha)

#

もし良かったらGitHubでStar下さい

もし良かったら

```bash
gem install conoha
```

して下さい

(※ConoHa VPSはVPSを消しておけば維持費かかりませんよ)

#

おわり
