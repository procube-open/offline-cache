
# 背景
- yum, dockerhub, pipy, npmjs などからソフトウェアをダウンロード・インストールすると、インストール時点での最新版がインストールされるため、構築するサーバの品質が確保できない（ ansible などの IaC でサーバの構築品質を高めようとしても、先週まで動いてたのに今週になって再構築したら動かなくなったとかインストールがエラーになったなどということは日常茶飯事）が、かと言ってすべてのソフトウェアのバージョンを指定することもできない
- AlmaLinux などの yum リポジトリでは古いモジュールは時間経過とともに削除されてしまい、パッケージのバージョンを明示的に固定できない
- お客様環境によってはサーバからインターネットへのアクセスを禁じている場合があり、オフラインでの構築が求められる

# 構成
- ssh のリモートフォワーディングでサーバからのアクセスをプロキシサーバに転送
 - https でのアクセスは偽サーバ証明書を偽CA局で生成する
 - 各サーバ、コンテナには偽CA局の証明書をトラストストアにインストールしておく
- ビルド中にプロキシキャッシュにファイルを蓄積
- dockerhub からダウンロードしたコンテナイメージを偽 dockerhub にプッシュ
- オフラインビルドでは、パブリックリポジトリにはアクセスせずに mother マシンに蓄積されたものをダウンロードする
 - インストールに必要なファイルはプロキシキャッシュからダウンロード
 - コンテナイメージは偽 dockerhub からダウンロードされる

![proxy](./img/proxy.svg#center)

# プロキシの実装技法

単純なプロキシサーバの実装だけでは、オフライン時にパブリックリポジトリを偽装することはできない。ここではプロキシサーバでパブリックリポジトリを偽装する際に発生する問題点とこのシステムでの対処方法を解説する。

## キャッシュの有効期限 問題点と対応
### 問題点

HTTPのコンテンツのキャッシュの有効性についてはクライアント側からもサーバ側からも様々なオプションで指定でき、古いコンテンツや無効なコンテンツを読み出さないような仕組みがプロトコルに規定されている。しかし、オフラインキャッシュサーバにおいては、すべてのコンテンツをキャッシュして有効にする必要がある。

### 対応

squid のオプションにより、キャッシュの有効性に関連するヘッダー類をすべて無視するようにしている。

```dotnetcli
refresh_pattern . 52560000 50% 52560000 override-expire ignore-reload ignore-no-cache ignore-no-store ignore-private reload-into-ims ignore-must-revalidate ignore-auth store-stale max-stale=100
```

## HTTP Varyヘッダー 問題点と対応
### 問題点

HTTP Request
```dotnetcli
Accept-Encoding: gzip, deflate
Accept: text/html
```

HTTP Response
```dotnetcli
Vary: Accept-Encoding, Accept
```

このレスポンスの意味は Accept ヘッダー、AcceptEncoding ヘッダーの値によってレスポンスがかわりますよという意味。 PyPi は実際に Accept が  text/html と application/json の場合がある。

Squid はレスポンスの Vary ヘッダーをみると、そこに書かれているヘッダーの値を含めてmd5ハッシュ値を作成し、キャッシュキーとする。
次回のキャッシュヒットを期待するアクセス時は AcceptやAccept-Encoding をハッシュ対象に含めないので、キーが異なりヒットしない

### 対応
Vary ヘッダーを無視して、常にAcceptとAccept-Encoding のをハッシュ対象に含めるように変更した

## CONNECT メソッド 問題点と対応
### 問題点
CONNECT メソッドの結果はキャッシュされず、毎回バックエンドに接続に行ってしまい、接続できないとエラーになるため、オフラインは実装できない。
### 対応
疑似バックエンドをnginx で構築し、squid の hosts ファイルで CONNECT メソッドを使用する（＝httpsを使用する）アクセスはその nginx に接続してメソッドが成功するようにした。

## docker registry API 問題点と対応
### 問題点
docker registry API では、パラメータをヘッダに含めるなどのロジックがあり、 URL → コンテンツのキャッシュだけでは対応できない。
### 対応
偽のdocker registry を構築し、そこに必要なコンテナイメージを push しておく対応とした。

## HTTP/2 問題点と対応
### 問題点
squid は HTTP/2 でアクセスした内容をキャッシュできない。yum の引数にURLを指定してインストールするパターンで発生した。
問題となった ansible タスクの内容は以下である。

```
    - name: Install Zabbix Repo
      yum:
        name: https://repo.zabbix.com/zabbix/5.1/rhel/8/x86_64/zabbix-release-5.1-1.el8.noarch.rpm
        disable_gpg_check: true
        state: present
      until: not download_zabbix_repo.failed
      retries: 10
      delay: 5
      register: download_zabbix_repo
```

### 対応
今のところ対応策は見つかっていない。
curl で http でアクセスしたところ、HTTP/2 ではなく HTTP1.1 になったので、 https → http に修正して回避した。
