Title: OpenStack構築手順書 Newton版
Company: 日本仮想化技術<br>
Version:0.9.0<br>

# OpenStack構築手順書 Newton版

<div class="title">
バージョン：0.9.0 (2016/10/25作成) <br>
日本仮想化技術株式会社
</div>

<!-- BREAK -->

## 変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.9.0|2016/10/25|Newton版執筆開始|


````
筆者注:このドキュメントに対する提案や誤りの指摘は
Issue登録か、日本仮想化技術までメールにてお願いします。
https://github.com/virtualtech/openstack-mitaka-docs/issues
````

<!-- BREAK -->

## 目次

<!--TOC max3-->
<!-- BREAK -->

# Part.1 OpenStack 構築編
<br>
本章は、OpenStack Foundationが公開している公式ドキュメント「OpenStack Installation Guide for Ubuntu 16.04」の内容から、「Keystone,Glance,Nova,Neutron,Horizon」までの構築手順をベースに加筆したものです。
OpenStackをUbuntu Server 16.04 ベースで構築する手順を解説しています。

Canonical社が提供するCloud Archiveリポジトリーを使って、OpenStackの最新版Newtonを導入しましょう。

<!-- BREAK -->

## 1. 構築する環境について

### 1-1 環境構築に使用するOS

本書はCanonicalのUbuntu ServerとCloud Archiveリポジトリーのパッケージを使って、OpenStack Newtonを構築する手順を解説したものです。

前バージョンのOpenStack MitakaではUbuntu Serverの14.04と16.04による構築が可能でしたが、OpenStack NewtonをUbuntuで構築するには、バージョン16.04(Xenial)が必要です。

Ubuntu 15.04以降のUbuntuはSystemdに移行しています。そのため本来ならばサービスの再起動はsystemctlコマンドを使って行うべきですが、本書ではOpenStack公式のマニュアルと同様にserviceコマンドを使ってサービスの制御をします。

本書は4.4.0-45以降のバージョンのカーネルで動作するUbuntu Server 16.04.1を使ってOpenStack Newtonを構築します。インストールイメージは以下からダウンロードできます。

- <http://releases.ubuntu.com/releases/xenial/ubuntu-16.04.1-server-amd64.iso>

````
筆者注:もしここで想定するUbuntuやカーネルバージョン以外で何らかの事象が発生した場合も、
以下までご報告ください。動作可否情報をGithubで共有できればと思います。
https://github.com/virtualtech/openstack-newton-docs/issues

Ubuntu LTSとカーネルの関係については次のWikiをご覧ください。
https://wiki.ubuntu.com/Kernel/LTSEnablementStack
````


<!-- BREAK -->

### 1-2 作成するサーバー（ノード）

本書はOpenStack環境をController,Computeの2台のサーバー上に構築することを想定しています。

| コントローラー | コンピュート 
| -------------- | --------------
| RabbitMQ       | Linux KVM
| NTP            | Nova Compute
| MariaDB        | Linux Bridge Agent
| Keystone       
| Glance
| Nova
| Neutron Server
| Linux Bridge Agent
| L3 Agent
| DHCP Agent
| Metadata Agent
| Cinder

<!-- BREAK -->

### 1-3 ネットワークセグメントの設定

IPアドレスは以下の構成で構築されている前提で解説します。

|各種設定|ネットワーク|
|:---|:---|:---|
|ネットワーク|10.0.0.0/24|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

### 1-4 各ノードのネットワーク設定

各ノードのネットワーク設定は以下の通りです。
Ubuntu 16.04ではNICのデバイス名の命名規則が変わりました。
物理サーバー上のNICはハードウェアとの接続によってensXやenoX、enpXsYのような命名規則になっています。なお、仮想マシンやコンテナーではethXと認識されるようです。

本例ではens3として認識されているのを想定していますので、実際の環境に合わせて適宜読み替えてください。

+ コントローラーノード

|インターフェース|ens3|
|:---|:---|:---|
|IPアドレス|10.0.0.101|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

+ コンピュートノード

|インターフェース|ens3|
|:---|:---|:---|
|IPアドレス|10.0.0.102|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

<!-- BREAK -->

### 1-5 Ubuntu Serverのインストール

#### 1-5-1 インストール

2台のサーバーに対し、Ubuntu Serverをインストールします。要点は以下の通りです。

+ 優先ネットワークインターフェースをens3(最初の方のNIC)に指定 
 + インターネットへ接続するインターフェースはens3を使用するため、インストール中はens3を優先ネットワークとして指定
+ OSは最小インストール
 + パッケージ選択ではOpenSSH serverを追加


【インストール時の設定パラメータ例】

|設定項目|設定例|
|:---|:---|
|初期起動時のLanguage|English|
|起動|Install Ubuntu Server|
|言語|English - English|
|地域の設定|other→Asia→Japan|
|地域の言語|United States - en_US.UTF-8|
|キーボードレイアウトの認識|No|
|キーボードの言語|Japanese→Japanese|
|優先するNIC|ens3: Ethernet|
|ホスト名|それぞれのノード名(controller, compute)|
|ユーザ名とパスワード|フルネームで入力|
|アカウント名|ユーザ名のファーストネームで設定される|
|パスワード|任意のパスワード|

パスワードを入力後、Weak passwordという警告が出た場合はYesを選択するか、警告が出ないようなより複雑なパスワードを設定してください。

<!-- BREAK -->

画面の指示に従ってセットアップを続けます。

|設定項目|設定例|
|:---|:---|
|ホームディレクトリーの暗号化|任意|
|タイムゾーン|Asia/Tokyoであることを確認|
|パーティション設定|Guided - use entire disk and set up LVM|
|パーティション選択|sdaを選択|
|パーティション書き込み|Yesを選択|
|パーティションサイズ|デフォルトのまま|
|変更の書き込み|Yesを選択|
|HTTP proxy|環境に合わせて任意|
|アップグレード|No automatic updatesを推奨|
|ソフトウェア|OpenSSH serverのみ選択|
|GRUB boot loader|Yesを選択(GRUBをインストールする)|
|インストール完了|Continueを選択|

```
筆者注:
Ubuntuインストーラーは基本的にパッケージのインストール時にインターネット接続が必要です。

Ubuntuインストール時に選択した言語がインストール後も使われます。
Ubuntu Serverで日本語の言語を設定した場合、標準出力や標準エラー出力が文字化けするなど様々な問題がおきますので、言語は英語を設定されることを推奨します。
```

<!-- BREAK -->

#### 1-5-2 プロキシーの設定

外部ネットワークとの接続にプロキシーの設定が必要な場合は、aptコマンドを使ってパッケージの照会やダウンロードを行うために次のような設定をする必要があります。

+ システムのプロキシー設定

```
# vi /etc/environment
http_proxy="http://proxy.example.com:8080/"
https_proxy="https://proxy.example.com:8080/"
no_proxy=localhost,controller,compute
```

+ APTのプロキシー設定

```
# vi /etc/apt/apt.conf
Acquire::http::proxy "http://proxy.example.com:8080/";
Acquire::https::proxy "https://proxy.example.com:8080/";
```

より詳細な情報は下記のサイトの情報を確認ください。

- <https://help.ubuntu.com/community/AptGet/Howto>
- <http://gihyo.jp/admin/serial/01/ubuntu-recipe/0331>

<!-- BREAK -->

### 1-6 Ubuntu Serverへのログインとroot権限

Ubuntuはデフォルト設定でrootユーザーの利用を許可していないため、root権限が必要となる作業は以下のように行ってください。

+ rootユーザーで直接ログインできないので、インストール時に作成したアカウントでログインする。
+ root権限が必要な作業を実行する場合には、sudoを実行したいコマンドの前につけて実行する。
+ root権限で連続して作業したい場合には、sudo -iコマンドでシェルを起動する。

### 1-7 設定ファイル等の記述について

+ 設定ファイルは特別な記述が無い限り、必要な設定を抜粋したものです。
+ 特に変更の必要がない設定項目は省略されています。
+ [見出し]が付いている場合、その見出しから次の見出しまでの間に設定を記述します。
+ コメントアウトされていない設定項目が存在する場合には、値を変更してください。多くの設定項目は記述が存在しているため、エディタの検索機能で検索することをお勧めします。
+ 特定のホストでコマンドを実行する場合はコマンドの冒頭にホスト名を記述しています。

【設定ファイルの記述例】

```
controller# vi /etc/glance/glance-api.conf ←コマンド冒頭にこのコマンドを実行するホストを記述

[DEFAULT]
debug=true                      ← コメントをはずす

[database]
#connection = sqlite:////var/lib/glance/glance.sqlite   ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記

[keystone_authtoken] ← 見出し
#auth_host = 127.0.0.1 ← 既存設定をコメントアウト
auth_host = controller ← 追記

auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0 ← 追記
admin_tenant_name = service ← 変更
admin_user = glance ← 変更
admin_password = password ← 変更
```
<!-- BREAK -->

## 2. OpenStackインストール前の設定

OpenStackパッケージのインストール前に各々のノードで以下の設定を行います。

+ ネットワークデバイスの設定
+ ホスト名と静的な名前解決の設定
+ リポジトリーの設定とパッケージの更新
+ Chronyサーバーのインストール（コントローラーノードのみ）
+ Chronyクライアントのインストール
+ Python用MySQL/MariaDBクライアントのインストール
+ MariaDBのインストール（コントローラーノードのみ）
+ RabbitMQのインストール（コントローラーノードのみ）
+ 環境変数設定ファイルの作成（コントローラーノードのみ）
+ memcachedのインストールと設定（コントローラーノードのみ）

<!-- BREAK -->

### 2-1 ネットワークデバイスの設定

各ノードの/etc/network/interfacesを編集し、IPアドレスの設定を行います。

#### 2-1-1 コントローラーノードのIPアドレスの設定

```
controller# vi /etc/network/interfaces
auto ens3
iface ens3 inet static
      address 10.0.0.101
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```

#### 2-1-2 コンピュートノードのIPアドレスの設定

```
compute# vi /etc/network/interfaces
auto ens3
iface ens3 inet static
      address 10.0.0.102
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```
<!-- BREAK -->

#### 2-1-3 ネットワークの設定を反映

各ノードで変更した設定を反映させるため、ホストを再起動します。

```
$ sudo reboot
```

### 2-2 ホスト名と静的な名前解決の設定

ホスト名でノードにアクセスするにはDNSサーバーで名前解決する方法やhostsファイルに書く方法が挙げられます。
本書では各ノードの/etc/hostsに各ノードのIPアドレスとホスト名を記述してhostsファイルを使って名前解決します。127.0.1.1の行はコメントアウトします。

#### 2-2-1 各ノードのホスト名の設定

各ノードのホスト名をhostnamectlコマンドを使って設定します。反映させるためには一度ログインしなおす必要があります。

（例）コントローラーノードの場合

```
controller# hostnamectl set-hostname controller
controller# cat /etc/hostname
controller
```

<!-- BREAK -->

#### 2-2-2 各ノードの/etc/hostsの設定

すべてのノードで127.0.1.1の行をコメントアウトします。
またホスト名で名前引きできるように設定します。

127.0.1.1の行はUbuntuの場合、デフォルトで設定されています。
memcachedをセットアップするノードでこの設定がされたままだと正常に動作しませんので注意してください。

（例）コントローラーノードの場合

```
controller# vi /etc/hosts
127.0.0.1 localhost
#127.0.1.1 controller ← 既存設定をコメントアウト
10.0.0.101 controller
10.0.0.102 compute
```

<!-- BREAK -->

### 2-3 リポジトリーの設定とパッケージの更新

コントローラーノードとコンピュートノードで以下のコマンドを実行し、Mitaka向けUbuntu Cloud Archiveリポジトリーを登録します。

```
# sudo add-apt-repository cloud-archive:newton
 Ubuntu Cloud Archive for OpenStack Newton
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it  ←ここでEnterキーを押下する
...
Importing ubuntu-cloud.archive.canonical.com keyring
OK
Processing ubuntu-cloud.archive.canonical.com removal keyring
gpg: /etc/apt/trustdb.gpg: trustdb created
OK
```

各ノードのシステムをアップデートします。Ubuntuではパッケージのインストールやアップデートの際にまず`apt-get update`を実行してリポジトリーの情報の更新が必要です。そのあと`apt-get -y dist-upgrade`でアップグレードを行います。カーネルの更新があった場合は再起動してください。

なお、`apt-get update`は頻繁に実行する必要はありません。日をまたいで作業する際や、コマンドを実行しない場合にパッケージ更新やパッケージのインストールでエラーが出る場合は実行してください。以降の手順では`apt-get update`を省略します。

```
# apt-get update && apt-get dist-upgrade
```

<!-- BREAK -->

### 2-4 OpenStackクライアントとMariaDBクライアントのインストール

コントローラーノードでOpenStackクライアントとPython用のMySQL/MariaDBクライアントをインストールします。依存するパッケージは全てインストールします。

```
controller# apt-get install python-openstackclient python-pymysql
```

### 2-5 時刻同期サーバーのインストールと設定

#### 2-5-1 時刻同期サーバーChronyの設定

各ノードの時刻を同期するためにChronyをインストールします。

```
# apt-get install chrony
```

#### 2-5-2 コントローラーノードの時刻同期サーバーの設定

コントローラーノードで公開NTPサーバーと同期するNTPサーバーを構築します。
適切な公開NTPサーバー(ex.ntp.nict.jp etc..)を指定します。ネットワーク内にNTPサーバーがある場合はそのサーバーを指定します。

設定を変更した場合はNTPサービスを再起動します。

```
controller# service chrony restart
```

<!-- BREAK -->

#### 2-5-3 その他ノードの時刻同期サーバーの設定

コンピュートノードでコントローラーノードと同期するNTPサーバーを構築します。

```
compute# vi /etc/chrony/chrony.conf

#server 0.debian.pool.ntp.org offline minpoll 8 #デフォルト設定はコメントアウト
#server 1.debian.pool.ntp.org offline minpoll 8
#server 2.debian.pool.ntp.org offline minpoll 8
#server 3.debian.pool.ntp.org offline minpoll 8

server controller iburst
```

設定を適用するため、NTPサービスを再起動します。

```
compute# service chrony restart
```

#### 2-5-4 NTPサーバーの動作確認

構築した環境でコマンドを実行して、各NTPサーバーが同期していることを確認します。

- 公開NTPサーバーと同期しているコントローラーノード

```
controller# chronyc sources
chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* chobi.paina.jp                2   8    17     1    +47us[ -312us] +/-   13ms
^- v157-7-235-92.z1d6.static     2   8    17     1  +1235us[+1235us] +/-   45ms
^- edm.butyshop.com              3   8    17     0  -2483us[-2483us] +/-   82ms
^- y.ns.gin.ntt.net              2   8    17     0  +1275us[+1275us] +/-   35ms
```

- コントローラーノードと同期しているその他ノード

```
compute# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* controller                3   6    77    25   -509us[-1484us] +/-   13ms
```

<!-- BREAK -->

### 2-6 MariaDBのインストール

コントローラーノードにデータベースサーバーのMariaDBをインストールします。

#### 2-6-1 パッケージのインストール

apt-getコマンドでmariadb-serverパッケージをインストールします。

```
controller# apt-get install mariadb-server python-pymysql
```

Ubuntu 16.04のMariaDBパッケージを使ってインストールした場合、パスワードの設定は求められません。
設定が必要ならば、`mysql_secure_installation`コマンドを実行して設定してください。

#### 2-6-2 MariaDBの設定を変更

OpenStack用のMariaDB設定ファイル /etc/mysql/mariadb.conf.d/99-openstack.cnf を作成し、以下を設定します。

別のノードからMariaDBへアクセスできるようにするためバインドアドレスを変更します。加えて使用する文字コードをutf8に変更します。また、デフォルトの接続数ではOpenStackで利用するには適さないため、接続数を4096まで増やします。なお、この設定は規模に応じて適切な設定を行ってください。

※文字コードをutf8に変更しないとOpenStackモジュールとデータベース間の通信でエラーが発生します。

```
controller# vi /etc/mysql/conf.d/openstack.cnf

[mysqld]
bind-address = 10.0.0.101                   ←controllerのIPアドレス
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

<!-- BREAK -->

#### 2-6-3 MariaDBサービスの再起動

変更した設定を反映させるためMariaDBのサービスを再起動します。

```
controller# service mysql restart
```

#### 2-6-4 MariaDBのセットアップ

MariaDBデータベースのセキュリティーを設定するにはmysql_secure_installationコマンドを実行します。
MariaDBのrootユーザーのパスワードを設定したり、デフォルトで作られるユーザーやデータベースの削除など行えますので必要に応じて行ってください。

````
controller# mysql_secure_installation
````

<!-- BREAK -->

### 2-7 RabbitMQのインストール

OpenStackは、オペレーションやステータス情報を各サービス間で連携するためにメッセージブローカーを使用しています。OpenStackではRabbitMQ、ZeroMQなど複数のメッセージブローカーサービスに対応しています。
本書ではRabbitMQをインストールする例を説明します。

#### 2-7-1 パッケージのインストール

コントローラーノードにrabbitmq-serverパッケージをインストールします。

```
controller# apt-get install rabbitmq-server
```

#### 2-7-2 openstackユーザーの作成とパーミッションの設定

RabbitMQにアクセスするためのユーザーとしてopenstackユーザーを作成し、必要なパーミッションを設定します。
以下コマンドはRabbitMQのパスワードをpasswordに設定する例です。

```
controller# rabbitmqctl add_user openstack password
controller# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 2-7-3 RabbitMQサービスのログを確認

+ ログの確認

メッセージブローカーサービスが正常に動いていないと、OpenStackの各コンポーネントは正常に動きません。RabbitMQサービスの再起動と動作確認を行い、確実に動作していることを確認します。

```
controller# tailf /var/log/rabbitmq/rabbit@controller.log
...
 =INFO REPORT==== 24-Oct-2016::13:41:30 ===

started TCP Listener on 10.0.0.101:5672  ←待受IPとポートを確認

 =INFO REPORT==== 24-Oct-2016::13:41:30 ===

Server startup complete; 0 plugins started.
```

※新たなエラーが表示されなければ問題ありません。

<!-- BREAK -->

### 2-8 環境変数設定ファイルの作成

#### 2-8-1 admin環境変数設定ファイルの作成

adminユーザー用の環境変数設定ファイルを作成します。

```
controller# vi ~/admin-openrc
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(admin)\$ '
```

#### 2-8-2 demo環境変数設定ファイルの作成

demoユーザー用の環境変数設定ファイルを作成します。

```
controller# vi ~/demo-openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(demo)\$ '
```

<!-- BREAK -->

### 2-9 memcachedのインストールと設定

コントローラーノードで認証機構のトークンをキャッシュするための用途でmemcachedをインストールします。

```
controller# apt-get install memcached python-memcache
```

設定ファイル /etc/memcached.conf を変更します。
既存の行 -l 127.0.0.1 をcontrollerのIPアドレスに変更します。

```
controller# vi /etc/memcached.conf
-l 10.0.0.101    ← controllerのIPアドレス
```

memcachedサービスを再起動します。

```
controller# service memcached restart
```

<!-- BREAK -->

## 3. Keystoneのインストールと設定（コントローラーノード）

各サービス間の連携時に使用する認証サービスKeystoneのインストールと設定を行います。

### 3-1 データベースを作成

MariaDBにKeystoneで使用するデータベースを作成しアクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 3-2 データベースの確認

MariaDBに keystoneユーザーでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -u keystone -p
Enter password:  ← MariaDBのkeystoneパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
```

<!-- BREAK -->

### 3-3 パッケージのインストール

apt-getコマンドでkeystoneおよび必要なパッケージをインストールします。

```
controller# apt-get install keystone
```

### 3-4 Keystoneの設定変更

keystoneの設定ファイルを変更します。

```
controller# vi /etc/keystone/keystone.conf
...
[database]
#connection = sqlite:////var/lib/keystone/keystone.db    ← 既存設定をコメントアウト
connection = mysql+pymysql://keystone:password@controller/keystone  ← 追記
...

[token]
...
provider = fernet          ← 追記 
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/keystone/keystone.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 3-5 データベースに展開

```
controller# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

<!-- BREAK -->

### 3-6 Fernet キーの初期化

keystone-manage コマンドで Fernet キーを初期化します。

```
controller# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
controller# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

### 3-7 Identityサービスのデプロイ

次のコマンドを実行してIdentityサービスをデプロイします。

```
controller# keystone-manage bootstrap --bootstrap-password password \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### 3-8 Apache Webサーバーの設定

+ コントローラーノードの /etc/apache2/apache2.confに項目ServerNameを追加して、コントローラーノードのホスト名を設定します。

```
# Global configuration
#
ServerName controller
...
```

### 3-9 サービスの再起動と不要DBの削除

+ Apache Webサーバーを再起動します。

```
controller# service apache2 restart
```

+ パッケージのインストール時に作成される不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/keystone/keystone.db
```

<!-- BREAK -->

### 3-10 サービスとAPIエンドポイントの作成

以下コマンドでサービスとAPIエンドポイントを設定します。

+ 環境変数の設定

```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=password
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://controller:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

+ サービスの作成

```
controller# openstack project create --domain default \
  --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 24ac7f19cd944f4cba1d77469b2a73ed |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```

<!-- BREAK -->

### 3-11 プロジェクトとユーザー、ロールの作成

+ demoプロジェクトの作成

```
controller# openstack project create --domain default \
  --description "Demo Project" demo

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
```

+ demoユーザーの作成

```
controller# openstack user create --domain default \
  --password-prompt demo

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aeda23aa78f44e859900e22c24817832 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

<!-- BREAK -->

+ userロールの作成

```
controller# openstack role create user

+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 997ce8d05fc143ac97d83fdfb5998552 |
| name      | user                             |
+-----------+----------------------------------+
```

+ userロールをdemoプロジェクトとdemoユーザーに追加

```
controller# openstack role add --project demo --user demo user
```
<!-- BREAK -->


### 3-12 Keystoneの動作確認

他のサービスをインストールする前に Keystone が正しく構築、設定されたか動作を検証します。

+ 一時的に設定した環境変数 OS_URL を解除します。

```
controller# unset OS_URL
```

+ セキュリティを確保するため、一時認証トークンメカニズムを無効化します。
  + /etc/keystone/keystone-paste.iniを開き、[pipeline:public_api]と[pipeline:admin_api]と[pipeline:api_v3]セクション（訳者注:..のpipeline行）から、admin_token_authを削除します。

```
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension user_crud_extension public_service
...
[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension crud_extension admin_service
...
[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
```

<!-- BREAK -->

動作確認のためadminおよびdemoテナントに対し認証トークンを要求します。
admin、demoユーザーのパスワードを入力します。

+ admin ユーザーとして認証トークンを要求します。

```
controller# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue

Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:15:39.014479Z                                     |
| id         | gAAAAABWvi9bsh7vkiby5BpCCnc-JkbGhm9wH3fabS_cY7uabOubesi-Me6IGWW |
|            | yQqNegDDZ5jw7grI26vvgy1J5nCVwZ_zFRqPiz_qhbq29mgbQLglbkq6FQvzBRQ |
|            | JcOzq3uwhzNxszJWmzGC7rJE_H0A_a3UFhqv8M4zMRYSbS2YF0MyFmp_U       |
| project_id | ed0b60bf607743088218b0a533d5943f                                |
| user_id    | 58126687cbcc4888bfa9ab73a2256f27                                |
+------------+-----------------------------------------------------------------+
```

<!-- BREAK -->

## 4. Glanceのインストールと設定

### 4-1 データベースを作成

MariaDBに glance で使用するデータベースを作成し、アクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
 IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 4-2 データベースの確認

MariaDBに glance ユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u glance -p
Enter password: ← MariaDBのglanceパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
2 rows in set (0.00 sec)
```

<!-- BREAK -->

### 4-3 ユーザー、サービス、APIエンドポイントの作成

以下のコマンドで認証情報を読み込み、ImageサービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

admin-openrcを読み込むと次のようにプロンプトが変化します。

```
controller# source admin-openrc
controller ~(admin)#
```

+ glanceユーザーの作成

```
controller# openstack user create --domain default --password-prompt glance
User Password: password  #glanceユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f4e777c4062483ab8d9edd7dff829df |
| name                | glance                           |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

+ adminロールをglanceユーザーとserviceプロジェクトに追加

```
controller# openstack role add --project service --user glance admin
```
<!-- BREAK -->

+ glanceサービスの作成

```
controller# openstack service create --name glance \
--description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

+ APIエンドポイントの作成

```
controller# openstack endpoint create --region RegionOne \
  image public http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image internal http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

### 4-4 Glanceのインストール

apt-getコマンドでglance パッケージをインストールします。

```
controller# apt-get install glance
```

<!-- BREAK -->

### 4-5 Glanceの設定変更

Glanceの設定を行います。glance-api.conf、glance-registry.confともに、[keystone_authtoken]に追記した設定以外のパラメーターはコメントアウトします。

```
controller# vi /etc/glance/glance-api.conf
...

[database]
#sqlite_db = /var/lib/glance/glance.sqlite         ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記
...
[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/  ← 変更
...
[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password  ← glanceユーザーのパスワード
...
[paste_deploy]
...
flavor = keystone          ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-api.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

```
controller# vi /etc/glance/glance-registry.conf

[DEFAULT]
...
[database]
#sqlite_db = /var/lib/glance/glance.sqlite             ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance  ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password      ← glanceユーザーのパスワード

...
[paste_deploy]
flavor = keystone                ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-registry.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 4-6 データベースに展開

次のコマンドでglanceデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "glance-manage db_sync" glance
```

※ 非推奨を示すメッセージが出力されますが無視して構いません。

<!-- BREAK -->

### 4-7 Glanceサービスの再起動

設定を反映させるためGlanceサービスを再起動します。

```
controller# service glance-registry restart && service glance-api restart
```

### 4-8 ログの確認と使用しないデータベースファイルの削除

サービスの再起動後、ログを参照しGlance RegistryとGlance APIサービスでエラーが起きていないことを確認します。

```
controller# tailf /var/log/glance/glance-api.log
controller# tailf /var/log/glance/glance-registry.log
```

インストール直後は作られていない場合が多いですが、コマンドを実行してglance.sqliteを削除します。

```
controller# rm /var/lib/glance/glance.sqlite
```

### 4-9 イメージの取得と登録

Glanceへインスタンス用の仮想マシンイメージを登録します。ここでは、OpenStackのテスト環境に役立つ軽量なLinuxイメージ CirrOS を登録します。

#### 4-9-1 イメージの取得

CirrOSのWebサイトより仮想マシンイメージをダウンロードします。

```
controller# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

#### 4-9-2 イメージを登録

ダウンロードした仮想マシンイメージをGlanceに登録します。

```
controller# source admin-openrc
controller# openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 133eae9fb1c98f45894a4e60d8736619                     |
| container_format | bare                                                 |
| created_at       | 2015-03-26T16:52:10Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/cc5c6982-4910-471e-b864-1098015901b5/file |
| id               | cc5c6982-4910-471e-b864-1098015901b5                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | ae7a98326b9c455588edd2656d723b9d                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13200896                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2015-03-26T16:52:10Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

#### 4-9-3 イメージの登録を確認

仮想マシンイメージが正しく登録されたか確認します。

```
controller# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |
+--------------------------------------+--------+--------+
```

<!-- BREAK -->

## 5. Novaのインストールと設定（コントローラーノード）

### 5-1 データベースを作成

MariaDBにデータベースnovaを作成します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE nova_api;
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password:           ← MariaDBのrootパスワードpasswordを入力
```

### 5-2 データベースの確認

MariaDBにnovaユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u nova -p
Enter password: ← MariaDBのnovaパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
| nova_api           |
+--------------------+
3 rows in set (0.00 sec)
```

<!-- BREAK -->

### 5-3 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc
```

+ novaユーザーの作成

```
controller# openstack user create --domain default --password-prompt nova
User Password: password  #novaユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8a7dbf5279404537b1c7b86c033620fe |
| name                | nova                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

+ novaユーザーをadminロールに追加

```
controller# openstack role add --project service --user nova admin
```

<!-- BREAK -->

+ novaサービスの作成

```
controller# openstack service create --name nova \
  --description "OpenStack Compute" compute

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 060d59eac51b4594815603d75a00aba2 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

+ ComputeサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```

<!-- BREAK -->

### 5-4 パッケージのインストール

apt-getコマンドでNova関連のパッケージをインストールします。

```
controller# apt-get install nova-api nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler
```

### 5-5 Novaの設定変更

nova.confの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:password@controller
auth_strategy = keystone

use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
my_ip = 10.0.0.101 ←controllerのIPアドレス

[vnc]
...
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
...
api_servers = http://controller:9292

(次のページに続きます→)
```

<!-- BREAK -->

```
(→前のページからの続き)

[api_database]
...
connection = mysql+pymysql://nova:password@controller/nova_api

[database]
...
connection = mysql+pymysql://nova:password@controller/nova

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 5-6 データベースに展開

次のコマンドでnovaデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "nova-manage api_db sync" nova
controller# su -s /bin/sh -c "nova-manage db sync" nova
```

### 5-7 Novaサービスの再起動

設定を反映させるため、Novaのサービスを再起動します。

```
controller# service nova-api restart && service nova-consoleauth restart && service nova-scheduler restart && \
service nova-conductor restart && service nova-novncproxy restart
```

### 5-8 不要なデータベースファイルの削除

データベースはMariaDBを使用するため、不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/nova/nova.sqlite
```

<!-- BREAK -->

## 6. Nova Computeのインストールと設定（コンピュートノード）

ここまでコントローラーノードの環境構築を行ってきましたが、ここでコンピュートノードに切り替えて設定を行います。

### 6-1 パッケージのインストール

```
compute# apt-get install nova-compute
```

### 6-2 Novaの設定を変更

nova.confの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:password@controller
auth_strategy = keystone
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

my_ip = 10.0.0.102 ← computeのIPアドレス

[vnc]
...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
...
api_servers = http://controller:9292

(次のページに続きます→)
```

<!-- BREAK -->

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

まず次のコマンドを実行し、コンピュートノードでLinux KVMが動作可能であることを確認します。コマンド結果が1以上の場合は、CPU仮想化支援機能がサポートされています。
もしこのコマンド結果が0の場合は仮想化支援機能がサポートされていないか、設定が有効化されていないので、libvirtでKVMの代わりにQEMUを使用します。後述の/etc/nova/nova-compute.confの設定でvirt_type = qemu を設定します。

```
compute# cat /proc/cpuinfo |egrep 'vmx|svm'|wc -l
4
```

VMXもしくはSVM対応CPUの場合はvirt_type = kvmと設定することにより、仮想化部分のパフォーマンスが向上します。先のコマンドの実行結果が0になる場合はvirt_typeとしてqemuを設定してください。

```
compute# vi /etc/nova/nova-compute.conf

[libvirt]
...
virt_type = kvm
```

### 6-3 Novaコンピュートサービスの再起動

設定を反映させるため、Nova-Computeサービスを再起動します。

```
compute# service nova-compute restart
```

nova-computeサービスが起動しない場合は/var/log/nova/nova-compute.logを確認してください。`AMQP server on controller:5672 is unreachable`のようなメッセージが出る場合はcontrollerの


<!-- BREAK -->

### 6-4 コントローラーノードとの疎通確認

疎通確認はコントローラーノード上にて、admin環境変数設定ファイルを読み込んで行います。

```
controller# source admin-openrc
```

#### 6-4-1 ホストリストの確認

コントローラーノードとコンピュートノードが相互に接続できているか確認します。もし、StateがXXXになっているサービスがある場合は、該当のサービスのログを確認して対処してください。

```
controller# openstack compute service list -c Binary -c Host -c State
+------------------+----------------+-------+
| Binary           | Host           | State |
+------------------+----------------+-------+
| nova-consoleauth | controller     | up    |
| nova-scheduler   | controller     | up    | ←Novaのステータスを確認
| nova-conductor   | controller     | up    |
| nova-compute     | compute        | up    |
+------------------+----------------+-------+
```

※一覧にcomputeが表示されていれば問題ありません。Stateがupでないサービスがある場合は-cオプションを外して確認します。

#### 6-4-2 ハイパーバイザの確認

コントローラーノードよりコンピュートノードのハイパーバイザが取得可能か確認します。

```
controller# openstack hypervisor list
+----+---------------------+
| ID | Hypervisor Hostname |
+----+---------------------+
|  1 | compute             |
+----+---------------------+
```

※Hypervisor一覧にcomputeが表示されていれば問題ありません。

<!-- BREAK -->

## 7. Neutronのインストール・設定（コントローラーノード）

### 7-1 データベースを作成

MariaDBにデータベースneutronを作成し、アクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ←MariaDBのrootパスワードpasswordを入力
```

### 7-2 データベースの確認

MariaDBにneutronユーザーでログインし、データベースの閲覧が可能か確認します。

```
controller# mysql -u neutron -p
Enter password: ←MariaDBのneutronパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
2 rows in set (0.00 sec)
```

※ユーザーneutronでログイン可能でデータベースが閲覧可能なら問題ありません。

<!-- BREAK -->

### 7-3 neutronユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、neutronサービスの作成とAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc
```

+ neutronユーザーの作成

```
controller# openstack user create --domain default --password-prompt neutron
User Password: password  #neutronユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 319f34694728440eb8ffcb27b6dd8b8a |
| name                | neutron                          |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

+ neutronユーザーをadminロールに追加

```
controller# openstack role add --project service --user neutron admin
```

<!-- BREAK -->

+ neutronサービスの作成

```
controller# openstack service create --name neutron \
  --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | f71529314dab4a4d8eca427e701d209e |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

+ neutronサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  network public http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network internal http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

### 7-4 パッケージのインストール

本書ではネットワークの構成は公式マニュアルの「[Networking Option 2: Self-service networks](http://docs.openstack.org/newton/install-guide-ubuntu/neutron-controller-install-option2.html)」の方法で構築する例を示します。

```
controller# apt-get install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent
```

<!-- BREAK -->

### 7-5 Neutronコンポーネントの設定

+ Neutronサーバーの設定

neutron.confの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
controller# vi /etc/neutron/neutron.conf 

[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

(次のページに続きます→)
```

<!-- BREAK -->

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[database]
...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[nova]
...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
```

[keystone_authtoken]セクションは追記した設定以外は取り除くかコメントアウトしてください。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ ML2プラグインの設定

ml2_conf.iniの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
controller# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_vxlan]
...
vni_ranges = 1:1000

[securitygroup]
...
enable_ipset = True
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ Linuxブリッジエージェントの設定

パブリックネットワークに接続している側のNICを指定します。本書ではens3を指定します。
今回の構成の場合は「flat_networksで指定した値:パブリックネットワークに接続している側のNIC」を指定する必要があるので、「provider:ens3」と設定します。

```
controller# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:ens3 ←追記
```

local_ipは、先にphysical_interface_mappingに設定したNIC側のIPアドレスを設定します。

```
[vxlan]
enable_vxlan = True                        ←コメントをはずす
local_ip = 10.0.0.101                      ←追記
l2_population = True                       ←追記 ※
```

エージェントとセキュリティグループの設定を行います。

```
[securitygroup]
...
enable_security_group = True        ←コメントをはずす
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver 
↑ 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

#### ※ ML2プラグインのl2populationについて
OpenStack Mitaka以降のバージョンの公式手順書では、ML2プラグインのmechanism_driversとしてlinuxbridgeとl2populationが指定されています。l2populationが有効だとこれまでの動きと異なり、インスタンスが起動してもネットワークが有効化されるまで通信ができません。つまりNeutronネットワークを構築してルーターのパブリック側のIPアドレス宛にPingコマンドを実行して確認できても疎通ができません。このネットワーク有効化の有無についてメッセージキューサービスが監視して制御しています。 <br />
従って、これまでのバージョン以上にメッセージキューサービス（本例や公式手順ではしばしばRabbitMQが使用されます）が確実に動作している必要があります。このような仕組みが導入されたのは、不要なパケットがネットワーク内に流れ続けないようにするためです。<br />
<br />
ただし、弊社でESXi仮想マシン環境に構築したOpenStack環境においてl2populationが有効化されていると想定通り動かないという事象が発生することを確認してます。その他のハイパーバイザーでは確認していませんが、ネットワーク通信に支障が起きる場合はl2populationをオフに設定すると改善される場合があります。修正箇所は次の通りです。

+ controllerの/etc/neutron/plugins/ml2/ml2_conf.iniの設定変更

```
[ml2]
...
mechanism_drivers = linuxbridge,l2population
↓
mechanism_drivers = linuxbridge
```

+ controller & computeの/etc/neutron/plugins/ml2/linuxbridge_agent.iniの設定変更

```
[vxlan]
...
l2_population = true
↓
l2_population = false
```

設定変更後はl2populationの設定変更を反映させるため、controllerとcomputeノードのNeutron関連サービスを再起動するか、システムを再起動してください。

<!-- BREAK -->

+ Layer-3エージェントの設定

external_network_bridgeは単一のエージェントで複数の外部ネットワークを有効にするには値を指定してはならないため、値を空白にします。

```
controller# vi /etc/neutron/l3_agent.ini

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
```

+ DHCPエージェントの設定

```
controller# vi /etc/neutron/dhcp_agent.ini 

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

<!-- BREAK -->

+ Metadataエージェントの設定

インスタンスのメタデータサービスを提供するMetadataエージェントを設定します。

```
controller# vi /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_ip = controller  ←Metadataホストを指定
metadata_proxy_shared_secret = METADATA_SECRET
```

Metadataエージェントの`metadata_proxy_shared_secret`に指定する値と、次の手順でNovaに設定する`metadata_proxy_shared_secret`が同じ値になるように設定します。任意の値を設定すれば良いですが、思いつかない場合は次のように実行して生成した乱数を使うことも可能です。

```
controller# openssl rand -hex 10
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/metadata_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 7-6 Novaの設定を変更

Novaの設定ファイルにNeutronの設定を追記します。

```
controller# vi /etc/nova/nova.conf
...
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password   ←neutronユーザーのパスワード
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

METADATA_SECRETはMetadataエージェントで指定した値に置き換えます。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 7-7 データベースに展開

コマンドを実行して、エラーが発生せずに完了することを確認します。

```
controller# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

<!-- BREAK -->

### 7-8 コントローラーノードのNeutronと関連サービスの再起動

設定を反映させるため、コントローラーノードの関連サービスを再起動します。
まずNova APIサービスを再起動します。

```
controller# service nova-api restart
```

次にNeutron関連サービスを再起動します。

```
controller# service neutron-server restart && \
 service neutron-linuxbridge-agent restart && \
 service neutron-dhcp-agent restart && \
 service neutron-metadata-agent restart && \
 service neutron-l3-agent restart
```

### 7-9 ログの確認

ログを確認して、エラーが出力されていないことを確認します。

```
controller# tailf /var/log/nova/nova-api.log
controller# tailf /var/log/neutron/neutron-server.log
controller# tailf /var/log/neutron/neutron-metadata-agent.log
controller# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

### 7-10 使用しないデータベースファイルを削除

```
controller# rm /var/lib/neutron/neutron.sqlite
```

<!-- BREAK -->

## 8. Neutronのインストール・設定（コンピュートノード）

次にコンピュートノードの設定を行います。

### 8-1 パッケージのインストール

```
compute# apt-get install neutron-linuxbridge-agent
```

### 8-2 設定の変更

+ Neutronの設定

neutron.confの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
compute# vi /etc/neutron/neutron.conf

[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

(次のページに続きます→)
```

<!-- BREAK -->

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[database]
...
#connection = sqlite:////var/lib/neutron/neutron.sqlite ←コメントアウト
```

本書の構成では、コンピュートノードのNeutron.confにはデータベースの指定は不要です。
データベースの指定がデフォルトで存在していますが、コメントアウトしてもしなくても構いません。
次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ Linuxブリッジエージェントの設定

linuxbridge_agent.iniの設定を行います。
すでにいくつかの設定は行われているので各セクションに同じように設定がされているか、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

physical_interface_mappingsにはパブリック側のネットワークに接続しているインターフェイスを指定します。本書ではens3を指定します。
local_ipにはパブリック側に接続しているNICに設定しているIPアドレスを指定します。

```
compute# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:ens3

[vxlan]
enable_vxlan = True
local_ip = 10.0.0.102
l2_population = True  (※1)

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

※1 ML2プラグインの設定 /etc/neutron/plugins/ml2/ml2_conf.iniでl2populationを使用しない場合はl2_population = False を設定し無効化します。

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 8-3 コンピュートノードのネットワーク設定

Novaの設定ファイルの内容をNeutronを利用するように変更します。

```
compute# vi /etc/nova/nova.conf
...
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password  ←neutronユーザーのパスワード
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 8-4 コンピュートノードのNeutronと関連サービスを再起動

ネットワーク設定を反映させるため、コンピュートノードのNeutronと関連のサービスを再起動します。

```
compute# service nova-compute restart 
compute# service neutron-linuxbridge-agent restart
```

<!-- BREAK -->

### 8-5 ログの確認

エラーが出ていないかログを確認します。

```
compute# tailf /var/log/nova/nova-compute.log
compute# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

### 8-6 Neutronサービスの動作を確認

`neutron agent-list`コマンドを実行してNeutronエージェントが正しく認識されており、稼働していることを確認します。

```
controller# source admin-openrc
controller# neutron agent-list -c host -c alive -c binary
+------------+-------+---------------------------+
| host       | alive | binary                    |
+------------+-------+---------------------------+
| controller | :-)   | neutron-metadata-agent    |
| controller | :-)   | neutron-linuxbridge-agent |
| controller | :-)   | neutron-dhcp-agent        |
| controller | :-)   | neutron-l3-agent          |
| compute    | :-)   | neutron-linuxbridge-agent |
+------------+-------+---------------------------+
```

 ※コントローラーとコンピュートで追加され、neutron-linuxbridge-agentが正常に稼働していることが確認できれば問題ありません。念のためログも確認してください。

<!-- BREAK -->

## 9. Dashboardのインストールと確認（コントローラーノード）

クライアントマシンからブラウザーでOpenStack環境を操作可能なWebインターフェイスをインストールします。

### 9-1 パッケージのインストール

コントローラーノードにDashboardをインストールします。

```
controller# apt-get install openstack-dashboard
```

<!-- BREAK -->

### 9-2 Dashboardの設定を変更

インストールしたDashboardの設定を行います。
すでにいくつかの設定は行われているので同じように設定がされているか確認し、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
controller# vi /etc/openstack-dashboard/local_settings.py 
...
WEBROOT = '/horizon/'
LOGIN_URL = WEBROOT + 'auth/login/'
LOGOUT_URL = WEBROOT + 'auth/logout/'
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = False
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'
...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'controller:11211',
    },
}
...
OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
...
TIME_ZONE = "Asia/Tokyo"  ← 変更(日本の場合)
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/openstack-dashboard/local_settings.py  | grep -v "^\s*$" | grep -v "^\s*#"
```

変更した内容を反映させるため、Apacheとセッションストレージサービスを再起動します。

```
controller# service apache2 restart
```

<!-- BREAK -->

### 9-3 Dashboardにアクセス

コントローラーノードとネットワーク的に接続されているマシンからブラウザで以下URLに接続してOpenStackのログイン画面が表示されるか確認します。

※ブラウザで接続するマシンは予めDNSもしくは/etc/hostsにコントローラーノードのIPを記述しておく等コンピュートノードの名前解決を行っておく必要があります。ドメイン名でアクセスしないとVNCのコンソールが出ませんので注意してください。

```
http://controller/horizon/
```

※上記URLにアクセスしてログイン画面が表示され、ユーザーadminとdemoでログイン（パスワード:password）でログインできれば問題ありません。



<!-- BREAK -->

### 9-4 Externalネットワークの作成

OpenStack Dashboardにadminユーザーでログインして、Externalネットワークを作成します。
次のようにOpenStack Dashboardを操作して、Externalネットワークを作成してください。

1. OpenStack Dashboardにadminユーザーでログイン
2. 「管理 > システム > ネットワーク」でExternalネットワークを作成

| 項目 | 設定 |
|-----|----- |
| 名前 | external-net |
| プロジェクト | admin |
| プロバイダネットワーク種別 | フラット |
| 物理ネットワーク | provider |
| セグメントID | 214 (ヘルプに記載のある範囲を設定) |
| 管理状態 | UP |
| 共有 | チェックを入れない |
| 外部ネットワーク | チェックを入れる |

3. 「管理 > システム > ネットワーク」でExternalネットワークをクリック
4. Externalネットワーク内にサブネットを作成(DHCPは無効)

| 項目 | 設定 |
|-----|----- |
| サブネット名 | external-subnet |
| ネットワークアドレス | 10.0.0.0/24 |
| IPバージョン | IPv4  |
| ゲートウェイ | 10.0.0.1 |

| 項目 | 設定 |
|-----|----- |
| DHCP有効 | チェックを入れない |
| IPアドレス割当プール | 10.0.0.177,10.0.0.190 |

IPアドレス割当プールはネットワークアドレスで定義したネットワーク範囲全てを割り当てても良い場合は定義する必要はありません。

<!-- BREAK -->

### 9-5 インスタンスネットワークの作成

OpenStack Dashboardにdemoユーザーでログインして、インスタンスネットワークを作成します。

インスタンスネットワークはユーザーが作成したインスタンスを接続するネットワークです。外部からのアクセスは不可能ですが、同じインスタンスネットワーク上のインスタンスなどと通信が可能です。

インスタンスネットワークは管理者が共有ネットワークとして定義してユーザーに利用させることもできますが、多くの場合は各ユーザー毎にロールで定義した範囲でネットワークを自由に作成させることができます。

インスタンスネットワークとルーターを作成して、adminユーザーで作成したExternalネットワークと接続することにより、外から中への通信、つまり外部からインスタンスへのリモートアクセスなどが可能になります。

次のようにOpenStack Dashboardを操作して、インスタンスネットワークを作成してください。

1.OpenStack Dashboardにdemoユーザーでログイン
2.「プロジェクト > ネットワーク > ネットワーク」でユーザーネットワークの作成

| 項目 | 設定 |
|-----|----- |
| ネットワーク名 | user-net |
| サブネットの作成 | チェックを入れる |

| 項目 | 設定 |
|-----|----- |
| サブネット名 | user-subnet |
| ネットワークアドレス | 192.168.0.0/24 |
| IPバージョン | IPv4  |
| ゲートウェイIP | 192.168.0.1 |
| ゲートウェイなし | チェックを入れない |

| 項目 | 設定 |
|-----|----- |
| DHCP有効 | チェックを入れる |
| IPアドレス割当プール | 未定義 |
| DNSサーバー | 8.8.8.8 |

DNSサーバーを複数指定したい場合は1行毎に記述します。IPアドレス割当プールはネットワークアドレスで定義したネットワーク範囲全てを割り当てても良い場合は定義する必要はありません。

3.「プロジェクト > ネットワーク > ルーター」でルーターを作成

| 項目 | 設定 |
|-----|----- |
| ルーター名 | myrouter |
| 管理状態 | UP |
| 外部ネットワーク | external-net |

4.「プロジェクト > ネットワーク > ルーター」で作成した「myrouter」をクリックして、インターフェイスを追加

| 項目 | 設定 |
|-----|----- |
| サブネット | user-net |
| IPアドレス | 未定義 |

<!-- BREAK -->

### 9-6 フレーバーの設定

フレーバーはインスタンスに設定する性能を定義するものです。従来のバージョンでは自動生成されていましたが、Newtonではデフォルトでフレーバーは定義されていません。

1.OpenStack Dashboardにadminユーザーでログイン<br>
2.「管理 > システム > フレーバー」を選択<br>
3.「フレーバーの作成」ボタンを押下<br>
4.フレーバー名、仮想CPU数、メモリー、ストレージサイズを定義<br>
5.「フレーバーの作成」ボタンを押下<br>

CirrOSを動かすだけであれば、1vCPU,64MBメモリー,1GBストレージあれば充分です。
Ubuntuを動かす場合は、1vCPU,1GBメモリー,4GBストレージ以上のスペックが必要です。

<!-- BREAK -->

### 9-7 セキュリティグループの設定

OpenStackの上で動かすインスタンスのファイアウォール設定は、セキュリティグループで行います。ログイン後、次の手順でセキュリティグループを設定できます。

1.OpenStack Dashboardにdemoユーザーでログイン<br>
2.「プロジェクト > コンピュート > アクセスとセキュリティ」を選択<br>
3.「ルールの管理」ボタンを押下<br>
4.「ルールの追加」で許可するルールを定義<br>
5.「追加」ボタンを押下<br>

インスタンスに対してPingを実行したい場合はルールとしてすべてのICMPを、インスタンスにSSH接続したい場合はSSHをルールとしてセキュリティグループに追加してください。

セキュリティーグループは複数作成できます。作成したセキュリティーグループをインスタンスを起動する際に選択することで、セキュリティグループで定義したポートを解放したり、拒否したり、接続できるクライアントを制限することができます。

<!-- BREAK -->

### 9-8 キーペアの作成

OpenStackではインスタンスへのアクセスはデフォルトで公開鍵認証方式で行います。次の手順でキーペアを作成できます。

1.OpenStack Dashboardにdemoユーザーでログイン<br>
2.「プロジェクト > コンピュート > アクセスとセキュリティ」をクリック<br>
3.「キーペア」タブをクリック<br>
4.「キーペアの作成」ボタンを押下<br>
5.キーペア名を入力<br>
6.「キーペアの作成」ボタンを押下<br>
7.キーペア（拡張子:pem）ファイルをダウンロード<br>

<!-- BREAK -->

### 9-9 インスタンスの起動

前の手順でGlanceにCirrOSイメージを登録していますので、早速構築したOpenStack環境上でインスタンスを起動してみましょう。

1.OpenStack Dashboardにdemoユーザーでログイン<br>
2.「プロジェクト > コンピュート > イメージ」をクリック<br>
3.イメージ一覧から起動するOSイメージを選び、「インスタンスの起動」ボタンを押下<br>
4.「インスタンスの起動」詳細タブで起動するインスタンス名、フレーバー、インスタンス数を設定<br>
5.アクセスとセキュリティタブで割り当てるキーペア、セキュリティーグループを設定<br>
6.ネットワークタブで割り当てるネットワークを設定<br>
7.作成後タブで必要に応じてユーザーデータの入力（オプション）<br>
8.高度な設定タブでパーティションなどの構成を設定（オプション）<br>
9.右下の「起動」ボタンを押下<br>

<!-- BREAK -->

### 9-10 Floating IPの設定

起動したインスタンスにFloating IPアドレスを設定することで、Dashboardのコンソール以外からインスタンスにアクセスできるようになります。インスタンスにFloating IPを割り当てるには次の手順で行います。

1.OpenStack Dashboardにdemoユーザーでログイン<br>
2.「プロジェクト > コンピュート > インスタンス」をクリック<br>
3.インスタンスの一覧から割り当てるインスタンスをクリック<br>
4.アクションメニューから「Floating IPの割り当て」をクリック<br>
5.「Floating IP割り当ての管理」画面のIPアドレスで「+」ボタンをクリック<br>
6.右下の「IPの確保」ボタンをクリック<br>
7.割り当てるIPアドレスとインスタンスを選択して右下の「割り当て」ボタンをクリック<br>


### 9-11 インスタンスへのアクセス

Floating IPを割り当てて、かつセキュリティグループの設定を適切に行っていれば、リモートアクセスできるようになります。セキュリティーグループでSSHを許可した場合、端末からSSH接続が可能になります（下記は実行例）。

```
client$ ping -c4 instance-floating-ip 
client$ ssh -i mykey.pem cloud-user@instance-floating-ip 
```

その他、適切なポートを開放してインスタンスへのPingを許可したり、インスタンスでWebサーバーを起動して外部PCからアクセスしてみましょう。