# MySQLite: SQLiteデータベースを読み書きするMySQLストレージエンジン

2014/02/18

第2回 MariaDB/MySQL コミュニティイベント

中谷 翔

---

## 自己紹介

- 中谷 翔 [@laysakura](https://twitter.com/laysakura)
- 東京大学 情報理工学系研究科 修了(予定)
  - 分散ストリーム処理系
- IPA未踏: **High Performance SQLite の開発**
  - 得た知見からMySQLiteの開発に着手
- 4月からDeNA入社

---

## 発表アウトライン

1. **開発背景**
1. デモ
1. 仕組み
1. 現状
1. 評価
1. まとめ

---

## SQLiteのクエリ実行は低速

![SQLiteコンポーネント](/img/sqlite_component.png)

---

## 高速なMySQL/MariaDBのクエリ実行系を利用

![MySQLiteコンポーネント](/img/mysqlite_component.png)

SQLiteのDBファイルを扱うMySQL/MariaDBストレージエンジンエンジンを開発

---

## なぜSQLiteクエリ実行は遅いのか

![VDBE](/img/vdbe_use.png)

- [VDBE (Virtual Database Engine)](http://www.sqlite.org/vdbe.html)の機構が原因
  - SQLはVDBE命令列に変換されて実行される
  - VDBE命令間をまたいだ操作のためにメモリ操作が頻発
  - VDBEは改変が困難
    - レジスタマシンに近い低級なモデル
    - 1つのVDBE命令が複数の種類のSQLで共通利用される

---

## MySQLiteの開発経緯

- 「SQLiteを改善する」未踏では改善に踏み切れなかった
- 未踏ではSQLiteのクエリ実行系の改善は(ほぼ)諦め、ページャ部分を高速化
  - フルスキャンの合成ベンチマークで最大5.0倍の高速化
- クエリ実行系は外部のものを利用するアプローチを構想し、DeNAインターン(と趣味)でMySQLiteを実装
  - **ソートやGroup-byクエリで高速化を実現**

---

## 発表アウトライン

1. 開発背景
1. **デモ**
1. 仕組み
1. 現状
1. 評価
1. まとめ

---

## SQLite DBファイル作成

```sql
$ sqlite3 /home/nakatani/foobar.sqlite

sqlite> .schema
CREATE TABLE T0 (col0 INT);
sqlite> select * from T0;
777
333
111
888
```

---

## MySQLite を使ってSQLite DBを読む

```sql
$ mysql

mysql> use test
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ...            |
+----------------+
mysql> create table T0 engine=mysqlite file_name='/home/nakatani/foobar.sqlite,T0';
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ...            |
| T0             |
+----------------+
mysql> select * from T0;
+------+
| col0 |
+------+
|  777 |
|  333 |
|  111 |
|  888 |
+------+
```

---

## SQLite DBファイルの排他制御

![SQLite DBファイルの排他制御](/img/lock.png)

---

## 発表アウトライン

1. 開発背景
1. デモ
1. 仕組み
1. 現状
1. **評価**
1. まとめ

---

## 評価環境

| ソフトウェア |           バージョン |
|--------------|----------------------|
| SQLite       |               3.7.16 |
| MariaDB      |               10.0.2 |
| OS           | Linux 2.6.32-5-amd64 |

　

| ハードウェア | スペック                    |
|--------------|-----------------------------|
| CPU          | Intel Xeon E5530 @ 2.40 GHz |
| メモリ       | 24GB                        |

- SQLite DBファイルはファイルキャッシュに乗っている

---

## SQLiteとの速度比較1 - 設定

```sql
-- T: (key_col INT, val_col INT), 10,000,000 行

-- Sort
select * from T order by val_col limit 5;

-- Group-by
select avg(val_col) from T group by key_col;  -- key_colは5種類

-- Scan-only
select count(*) from T;
```

---

## SQLiteとの速度比較1 - 結果

![SQLiteとの速度比較1](/img/sort-group-scan.png)

- 単純なスキャンは遅く、それ以外は速い
  - スキャンが遅いのはストレージエンジン実装最適化不足 :P

---

## SQLiteとの速度比較2 - 設定

```sql
-- T: (key_col INT, val_col INT), 250,000 行
-- S: (key_col INT, val_col INT),     100 行

-- Join
select count(*) from T, S where T.key_col = S.key_col;

```

---

## SQLiteとの速度比較2 - 結果

![SQLiteとの速度比較2](/img/join.png)

- Joinで負けているが、内部表T(250,000行)のスキャンを外部表S(100行)の行数分行ったからと予想
  - 250,000行のスキャン => 0.0343秒(Scan-only結果から計算)
- スキャン実装を速くすれば勝てる(おそらく)

---

## 発表アウトライン

1. 開発背景
1. デモ
1. 仕組み
1. 現状
1. 評価
1. **まとめ**

---

## まとめ

- SQLiteの低速なクエリ実行の代わりにMySQL/MariaDBを利用するためにMySQLiteを開発
- Sortで15.3倍, Group-byで9.0倍の高速化
- Scan-onlyでは0.1倍(遅い)。これはストレージエンジン実装の問題
- Joinでは0.5倍(遅い)。Scan-onlyが遅いのと同様の理由と予想

---

## 今後の展望

- フルスキャン実装の高速化
- インデックス対応
- 書き込みトランザクション対応

---

## todo

- 背景
  - SQLiteのクエリ実行が低速
    - DB => ページ読込(Pager) => クエリプラン => クエリ実行 っていう絵
    - VDBEっていう仕組みが、各vdbe間の遷移の際にメモリコピーを多発
      <!-- vdbeのsortでの例を見せて、ループしてるから本当にメモリコピが多いことを -->
  - 未踏では、Pagerを一部改良し、フルスキャン性能を上昇
    - 合成ベンチマークで5.0倍高速化
    - でも本当はクエリ実行系を速くしたかった
  - クエリ実行系をMySQLと置き換えれば速くなるのでは! => 実際これくらい速くなった
- デモ
  - 動作を見せるためのデモ
  - **localhostで動くようにしとかなきゃ!**
- 仕組み
  - 背景で使った、MySQLとの置き換え図
  <!-- - 何でMySQLの実行系は速いの? => 調べる時間ないなぁ・・・ -->
  - SQLiteのファイル構成をチラッと話す(可視化を見せる?)
  - mmapでがっつり読んでる
  - 書き込みの工夫(実装中)
    - sqlite3 プロセスとの競合 => ファイルロック, shared lock
    - トランザクション開始時にファイルロックをとり、トランザクションcommit時にwrite, unlock
    - mysql プロセスとの競合 => 同じように
      本当はファイルロックより再粒度なロックを使いたいが、、、実装を楽にするため
  - maria db ならでは部分
- 現状
  - フルスキャンクエリのみ
  - index無理、write無理
  - proof of concept
- 評価
  - sort, group by 勝ってる系(もっと増やしたい)
  - fullscan, join 負けてる系
    join負けてるの、ずっとdisk ioがあるからかな? fullscanと同じ原因だと言って、fullscanを速くする方法も提示し、その場を収めたい
  - もうちょいまともなクエリ
  - ていうか今この瞬間にfullscanを速くしたい
