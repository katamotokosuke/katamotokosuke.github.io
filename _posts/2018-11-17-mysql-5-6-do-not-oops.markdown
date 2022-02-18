---
layout: post
title: mysql 5.6~でのalter tableでやらかさない
which_category: tech
---

# はじめに
mysql5.6の話をします。それ以上のバージョンでも通じる話ではあると思います。
どんなシステムでも本番稼働中にindexを貼りたい、テーブル定義(カラム名、型、並び順...etc)の変更をしたいことは往々してあると思います。そんな時は```alter table```コマンドを叩きたい気分になります。安直にコマンドをぶっ叩くとやらかしてしまう可能性があります。今後もやらかさないためにまとめておきます。

# 環境
```
mysql> status;
--------------
mysql  Ver 14.14 Distrib 5.6.42, for Linux (x86_64) using  EditLine wrapper

Connection id:		3
Current database:	
Current user:		root@localhost
SSL:			Not in use
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server version:		5.6.42 MySQL Community Server (GPL)
Protocol version:	10
Connection:		Localhost via UNIX socket
Server characterset:	latin1
Db     characterset:	latin1
Client characterset:	latin1
Conn.  characterset:	latin1
UNIX socket:		/var/run/mysqld/mysqld.sock
Uptime:			2 min 34 sec

Threads: 2  Questions: 10  Slow queries: 0  Opens: 67  Flush tables: 1  Open tables: 60  Queries per second avg: 0.064
--------------
```



# 5.6で追加されたやらかさない技術

結論から言うとonline DDLと呼ばれる機能です。これは2つの要素から成り立っています。
1. 高速index作成(こちらは5.5から存在する)
2. alter table実行中にDML処理が可能にする拡張


## 高速index作成とは
すでに書きましたが5.5からある機能ではあります。
そもそも```alter table```は結構忙しい処理です。ざっくり言うと
1. ```alter table```の内容を元に新テーブル定義されたテンポラリテーブルを作成
2. 既存テーブルからテンポラリテーブルにデータを登録
3. 旧テーブルを削除
4. テンポラリテーブルの名前を旧テーブル名にrename

このような手順を踏んでいます。

セカンダリインデックスのadd dropに関してはテーブルコピーのようなことをせずに速度に最適化が行われています。

## DMLの実行が可能とは
ざっくり言うと以前まではread(SELECT)は可能ですが、write(INSERT, UPDATE, DELETE)をブロックされていたのがどちらもブロックされなくなったということです。
とはいえすべてができるようになったわけではありません。詳しくはドキュメントを参照してみてください。 https://dev.mysql.com/doc/refman/5.6/ja/innodb-create-index-overview.html

## というわけで
online DDLがあるとメンテモードにせずにDMLをぶっ叩くことができるようになるということです。しかしwriteブロックする場合もあるのでそちらは依然として注意が必要です。


# やらかす事象
とはいえonline DDLがあったとしてもやらかす時はやらかします。例えば実行時間の長いtransaction中のプロセスがある平気でやらかします。なぜかと言うとmysqlにはメタデータのロックという機構があり、それにより```alter table```がロックされて解放するまで待ちの状態になり、その後のDMLコマンドも待ち状態になって事故ってしまうケースがあります。

## メタデータのロック？？
主な目的としてはデータの一貫性を確保するためです。
明示、暗黙的問わず開始されたトランザクションはメタデータロックを保持し、利用されてるテーブルに対してDDLを実行を禁止します。 またこれはトランザクション終了まで保持されます。
詳しくは: https://dev.mysql.com/doc/refman/5.6/ja/metadata-locking.html



## 試す
実際にメタデータロックを意図的に起こしてみたいと思います。
とりあえずmysqlが動く環境を作りましょう。

```
$ docker pull mysql:5.6.42
```

```
$ docker run --name mysql-tx -e MYSQL_ROOT_PASSWORD=secret -d mysql:5.6.42
```

めちゃくちゃ適当ですが、できました。


```
$ docker exec -it mysql-tx bash
```

からの

実験なのでrootでmysqlに繋ぎにいきます。普段はそんなことしないようにしてください。

```
$ mysql -u root -p
```


testデータベースを作ります。

```
mysql> create database test;
```


```
mysql> use test;
```

テーブルを作ります。

```
mysql> create table users (id int, name text, age int);
```

適当にデータを登録しておきます。

```
mysql> insert into users (id, name, age) values (1, "a", 22), (2, "b", 23), (3, "c", 24), (4, "d", 25);
```


お待たせしました。ここからがメタロックの実験です。

まず遅いselectを実行してみましょう。

```
mysql> select sleep(20), id  from users;
```


このクエリが実行中はメタデータロックを保持していますので、別セクションからDDLを叩くとロックされているはずです。実際にやっています。
別の端末からmysql-txにログインして同様にmysqlに接続して次のコマンドを叩いてみます。


```
mysql> alter table users add index idx_id(id);

```

やはり全然返ってきません（何も返ってこないので見せるものがありません。）。予定通りDDLがブロックされているみたいです。
もう一つ別のセクションから以下のコマンドを叩いて確かめてみましょう。

```
mysql> show processlist;
```



```
mysql> show processlist;
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
| Id | User | Host      | db   | Command | Time | State                           | Info                                    |
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
|  4 | root | localhost | test | Query   |   28 | Waiting for table metadata lock | alter table users add index idx_idb(id) |
|  5 | root | localhost | test | Query   |   38 | User sleep                      | select sleep(20), id  from users        |
|  6 | root | localhost | NULL | Query   |    0 | init                            | show processlist                        |
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
3 rows in set (0.00 sec)

```

ついにきました。Waiting for table metadata lock。 これが本番で発生するとすみません状態になります。


# まとめ
mysql5.6からonline DDLによってindexをサービスインさせたまま貼ることができて便利です。
とはいえindexを貼る時も長いトランザクションがあったりする時は気をつけましょう。
また、今回やらかす事象にメタデータロックだけを取り上げましたが、もちろん他にもたくさんあると思います。実際にこんなのにハマりましたみたいなのがあればぜひ教えていただきたいです。
