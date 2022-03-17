---
title: "Ubuntu 20.04 でpsqlコマンドが叩けるようになるまで"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ubuntu", "Ubuntu 20.04"]
published: false
---

# はじめに
Ubuntu上にPostgresをインストールしPostgresプロンプトにアクセスする手順を備忘録として書いておきます。

少し遠回りになりますがTDDで発生エラーを読みながら対応しOSやLinuxに関する知識のおさらいを行います。

※下記記事を参考に環境構築を行いました。

https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart-ja

※下記エラーに詰まっている方のために本記事を書いております。
```
postgres@ikmz:~$ psql
psql: error: could not connect to server: No such file or directory
        Is the server running locally and accepting
        connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
```

## 環境
Ubuntu 20.04 (Windows 11 Pro)

# PostgreSQLのインストール
1. PostgreSQLをインストールします

準備としてサーバのローカルパッケージを最新の状態に更新します。
```
ikmz@ikmz:~/dev/test$ sudo apt update
[sudo] password for ikmz: 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
146 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

2. 追加機能（-contrib）とともにPostgresパッケージをインストールします。
```
ikmz@ikmz:~/dev/test$ sudo apt install postgresql postgresql-contrib
Reading package lists... Done
Building dependency tree       
Reading state information... Done
postgresql is already the newest version (12+214ubuntu0.1).
postgresql-contrib is already the newest version (12+214ubuntu0.1).
0 upgraded, 0 newly installed, 0 to remove and 146 not upgraded.
```
ああ、なんか自分は以前にインストールしていたようです。初めてインストールする場合は応答が少し違うと思います。まあ、「postgresql is ... Done / postgresql-contrib is ... Done」というようなニュアンスの応答が来ればよいかと思います。

ひとまず、デフォルトのPostgresロールに関連付けられた`postgres`というユーザーアカウントが作成されました。

# PostgreSQLのロールとデータベースの使用
3. サーバー上のpostgresアカウントに切り替えます
```
ikmz@ikmz:~/dev/test$ sudo -i -u postgres
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.10.16.3-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Mar 14 09:47:55 JST 2022

  System load:  0.02               Processes:             34
  Usage of /:   6.1% of 250.98GB   Users logged in:       0
  Memory usage: 14%                IPv4 address for eth0: 172.26.86.74
  Swap usage:   0%

155 updates can be applied immediately.
94 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

This message is shown once a day. To disable it please create the
/var/lib/postgresql/.hushlogin file.
```

4. Postgresプロンプトにアクセスします
```
postgres@ikmz:~$ psql
psql: error: could not connect to server: No such file or directory
        Is the server running locally and accepting
        connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
```

Linuxで通信を行うための拠点となるソケットファイルが見当たらず通信を行えないというエラーがでますね。

## 下記手順（5～13）で対処していきます。

5. postgresqlサービスを起動してみます

service コマンドから操作できます。
```
postgres@ikmz:~$ sudo service postgresql start
[sudo] password for postgres: 
postgres is not in the sudoers file.  This incident will be reported.
```
⇒実行権限がないと怒られますね

6. DBサーバが起動しているのか確認します
```
postgres@ikmz:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
14  main    5433 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
⇒DBサーバがダウンしていることが確認できました。

7. PostgreSQLクラスタを起動します

クラスタとは1つのサーバインスタンスによって管理されるデータベースの群を指します。今回はその中でも`postgresql/12/main`を起動します。
```
postgres@ikmz:~$ pg_ctlcluster 12 main start
install: cannot change owner and permissions of ‘/var/run/postgresql’: No such file or directory
Error: Could not create /var/run/postgresql/12-main.pg_stat_tmp: No such file or directory
```
⇒実行権限がなくそのようなフォルダが存在しないと怒られてしまいます。

8. サーバを起動します
```
postgres@ikmz:~$ sudo /etc/init.d/postgresql start
[sudo] password for postgres:
postgres is not in the sudoers file.  This incident will be reported.
```
⇒実行権限がないと怒られますね

「postgres is not in the sudoers file.  This incident will be reported.」が根本的な原因のようなのでこれを解決させます。このエラーはsudoユーザーに追加していないユーザーでsudoコマンドを実行した際に表示されます。

9. postgresがsudoで実行できるようにしていきます
- Linux OS内のグループ一覧を確認します。catコマンドで/etc/groupをのぞき、grepコマンドでpostgresグループ内の内容を把握します。
- postgresユーザーでsudoを利用できるようにします。
- 変更内容を確認し、`sudo: `にpostgresが追加されていることを確認します
```
postgres@ikmz:~$ cat /etc/group | grep postgres
ssl-cert:x:119:postgres
postgres:x:120:

↓

root@ikmz:~# usermod -G sudo postgres
root@ikmz:~#

↓

postgres@ikmz:~$ cat /etc/group | grep postgres
sudo:x:27:ikmz,postgres
postgres:x:120:
```

10. 再度サーバを起動してみましょう
```
postgres@ikmz:~$ sudo /etc/init.d/postgresql start
[sudo] password for postgres: 
 * Starting PostgreSQL 12 database server                                                                                                                                                                          * Error: /usr/lib/postgresql/12/bin/pg_ctl /usr/lib/postgresql/12/bin/pg_ctl start -D /var/lib/postgresql/12/main -l /var/log/postgresql/postgresql-12-main.log -s -o  -c config_file="/etc/postgresql/12/main/postgresql.conf"  exited with status 1: 
2022-03-14 19:50:25.503 JST [15412] FATAL:  could not access private key file "/etc/ssl/private/ssl-cert-snakeoil.key": Permission denied
2022-03-14 19:50:25.503 JST [15412] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
                                                                                                                                                                                                           [fail]
 * Starting PostgreSQL 14 database server                                                                                                                                                                          * Error: /usr/lib/postgresql/14/bin/pg_ctl /usr/lib/postgresql/14/bin/pg_ctl start -D /var/lib/postgresql/14/main -l /var/log/postgresql/postgresql-14-main.log -s -o  -c config_file="/etc/postgresql/14/main/postgresql.conf"  exited with status 1: 
2022-03-14 19:50:25.688 JST [15437] FATAL:  could not access private key file "/etc/ssl/private/ssl-cert-snakeoil.key": Permission denied
2022-03-14 19:50:25.688 JST [15437] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
```

⇒秘密鍵の居場所がわかりませんと怒られますね。先ほどの手順(9)で何か必要ないところまで変更してしまったかもしれませんね。

11. この記事に沿って秘密鍵の居場所を特定し権限関係も確認します
https://stackoverflow.com/questions/34209661/fatal-could-not-access-private-key-file-etc-ssl-private-ssl-cert-snakeoil-key

```
postgres@ikmz:~$ sudo -i

root@ikmz:~# ls -la /etc/ssl/private
total 12
drwx--x--- 2 root ssl-cert 4096 Mar  9 13:45 .
drwxr-xr-x 4 root root     4096 Aug 20  2021 ..
-rw-r----- 1 root ssl-cert 1704 Mar  9 13:45 ssl-cert-snakeoil.key

root@ikmz:~# id postgres
uid=112(postgres) gid=120(postgres) groups=120(postgres),27(sudo)
```

新しいフォルダを作成して秘密鍵をmvコマンドで移動させるような方針のようでしたが、なんか怖いので別の方法で試しました。

12. 秘密鍵の所有者および権限を変更する

https://gist.github.com/GabLeRoux/0c60f9be0c28b6b41f64cb55474b0ccb

- sudo chown(change owner) で秘密鍵の所有者を変更します。
- sudo chmod(change mode) で秘密鍵のアクセス権(パーミッション)を変更します。

※chmod 740で所有者に全権限、グループに読み取り権限を与えます。他には一切権限を与えません。
```
postgres@ikmz:~$ gpasswd -a postgres ssl-cert
gpasswd: Permission denied.
postgres@ikmz:~$ sudo chown root:ssl-cert  /etc/ssl/private/ssl-cert-snakeoil.key
postgres@ikmz:~$ sudo chmod 740 /etc/ssl/private/ssl-cert-snakeoil.key
```

13. 再度DBサーバを起動します
```
postgres@ikmz:~$ sudo /etc/init.d/postgresql start
 * Starting PostgreSQL 12 database server [ OK ] 
 * Starting PostgreSQL 14 database server [ OK ] 
```

14. 再度Postgresプロンプトにアクセスします
```
postgres@ikmz:~$ psql
psql (14.2 (Ubuntu 14.2-1.pgdg20.04+1), server 12.9 (Ubuntu 12.9-0ubuntu0.20.04.1))
Type "help" for help.
```

ヨシ！(現場猫)


# おまけ
※初期導入時はpostgresというユーザーが作成されますが、パスワードがデフォルトで設定されないのでここで設定しておきます。passwordは試験用に'postgres'とします。

http://db-study.com/archives/121
```
postgres=# alter role postgres with password 'postgres';
ALTER ROLE
```

# おわりに
docker使う準備でも始めるか