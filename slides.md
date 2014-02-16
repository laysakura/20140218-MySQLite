# MySQLite: SQLiteデータベースを読み書きするMySQLストレージエンジン

2014/02/18

第2回 MariaDB/MySQL コミュニティイベント

中谷 翔

---

## todo

- 背景
- デモ
- 現状
  - フルスキャンクエリのみ
  - index無理、write無理
  - proof of concept
- 評価
  - sort, group by 勝ってる系(もっと増やしたい)
  - fullscan, join 負けてる系
    join負けてるの、ずっとdisk ioがあるからかな? fullscanと同じ原因だと言って、fullscanを速くする方法も提示し、その場を収めたい
  - もうちょいまともなクエリ
