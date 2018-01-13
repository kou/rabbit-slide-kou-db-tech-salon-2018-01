# メモ

## ベンチマーク

### データ

シカゴの犯罪データ。651万レコード。

https://catalog.data.gov/dataset/crimes-2001-to-present-398a4

### バージョン

MariaDBは10.3.3。Mroongaは7.10。

`innodb_buffer_pool_size`は512M。

### インデックス作成時間

Mroongaのテーブルの`block`に全文検索インデックスを作成。7秒。

```text
MariaDB> create fulltext index block_index on crimes (block);
Query OK, 0 rows affected (6.989 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

Mroongaのテーブルの`description`に全文検索インデックスを作成。6秒。

```text
MariaDB> create fulltext index description_index on crimes (description);
Query OK, 0 rows affected (5.873 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

InnoDBのテーブルの`block`に全文検索インデックスを作成。35秒。

```text
MariaDB> create fulltext index block_index on crimes2 (block);
Query OK, 0 rows affected (35.382 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

InnoDBのテーブルの`description`に全文検索インデックスを作成。1分50秒。

```text
MariaDB> create fulltext index description_index on crimes2 (description);
Query OK, 0 rows affected, 1 warning (1 min 49.443 sec)
Records: 0  Duplicates: 0  Warnings: 1
```

### 全文検索だけ

半分くらい（300万件くらい）ヒットするケース。Mroongaは1秒。

```text
MariaDB > select count(*) from crimes where match(block) against('+ave' in boolean mode);
+----------+
| count(*) |
+----------+
|  3126521 |
+----------+
1 row in set (1.066 sec)
```

半分くらい（300万件くらい）ヒットするケース。Groongaは0.3秒。

```text
MariaDB > select mroonga_command('select --table crimes --query "block:@ave" --limit 0 --output_columns _key') as response;
+----------------------------------+
| response                         |
+----------------------------------+
| [[[3126521],[["_key","Int32"]]]] |
+----------------------------------+
1 row in set (0.305 sec)
```

半分くらい（300万件くらい）ヒットするケース。InnoDBは18秒。

```text
MariaDB> select count(*) from crimes2 where match(block) against('+ave' in boolean mode);
+----------+
| count(*) |
+----------+
|  3126521 |
+----------+
1 row in set (17.563 sec)
```

4万件くらいヒットするケース。Mroongaは0.02秒。

```text
MariaDB> select count(*) from crimes where match(block) against('+milwaukee' in boolean mode);
+----------+
| count(*) |
+----------+
|    38781 |
+----------+
1 row in set (0.022 sec)
```

4万件くらいヒットするケース。Groongaは0.01秒。

```text
MariaDB> select mroonga_command('select --table crimes --query "block:@milwaukee" --limit 0 --output_columns _key') as response;
+--------------------------------+
| response                       |
+--------------------------------+
| [[[38781],[["_key","Int32"]]]] |
+--------------------------------+
1 row in set (0.010 sec)
```

4万件ヒットするケース。InnoDBは0.2秒。

```text
MariaDB> select count(*) from crimes2 where match(block) against('+milwaukee' in boolean mode);
+----------+
| count(*) |
+----------+
|    38781 |
+----------+
1 row in set (0.189 sec)
```

### 通常の検索だけ

数値の等価条件1つで、26万件ヒットするケース。Mroongaの通常バージョンは1.3秒。

```text
MariaDB> select count(*) from crimes where year = 2017;
+----------+
| count(*) |
+----------+
|   265156 |
+----------+
1 row in set (1.285 sec)
```

数値の等価条件1つで、26万件ヒットするケース。Mroongaの最適化ONバージョンは0.4秒。

```text
MariaDB> set mroonga_condition_push_down_type = all;
Query OK, 0 rows affected (0.000 sec)

MariaDB> select count(*) from crimes where year = 2017;
+----------+
| count(*) |
+----------+
|   265156 |
+----------+
1 row in set (0.395 sec)
```

数値の等価条件1つで、26万件ヒットするケース。Groongaは0.4秒。

```text
MariaDB> select mroonga_command('select --table crimes --filter "year == 2017" --limit 0 --output_columns _key') as response;
+---------------------------------+
| response                        |
+---------------------------------+
| [[[265156],[["_key","Int32"]]]] |
+---------------------------------+
1 row in set (0.361 sec)
```

数値の等価条件1つで、26万件ヒットするケース。InnoDBは1.3秒。

```text
MariaDB> select count(*) from crimes2 where year = 2017;
+----------+
| count(*) |
+----------+
|   265156 |
+----------+
1 row in set (1.304 sec)
```

数値の等価条件1つと真偽値の等価条件2つで、7000件ヒットするケース。Mroongaの通常バージョンは2.3秒。

```text
MariaDB> set mroonga_condition_push_down_type = default;
Query OK, 0 rows affected (0.000 sec)

MariaDB> select count(*) from crimes where year = 2017 and domestic = true and arrest = true;
+----------+
| count(*) |
+----------+
|     7148 |
+----------+
1 row in set (2.262 sec)
```

数値の等価条件1つと真偽値の等価条件2つで、7000件ヒットするケース。Mroongaの最適化ONバージョンは0.4秒。

```text
MariaDB> set mroonga_condition_push_down_type = all;
Query OK, 0 rows affected (0.000 sec)

MariaDB> select count(*) from crimes where year = 2017 and domestic = true and arrest = true;
+----------+
| count(*) |
+----------+
|     7148 |
+----------+
1 row in set (0.365 sec)
```

数値の等価条件1つと真偽値の等価条件2つで、7000件ヒットするケース。Groongaは0.4秒。

```text
MariaDB> select mroonga_command('select --table crimes --filter "year == 2017 && domestic == true && arrest == true" --limit 0 --output_columns _key') as response;
+-------------------------------+
| response                      |
+-------------------------------+
| [[[7148],[["_key","Int32"]]]] |
+-------------------------------+
1 row in set (0.365 sec)
```

数値の等価条件1つと真偽値の等価条件2つで、7000件ヒットするケース。InnoDBは1.6秒。

```text
MariaDB> select count(*) from crimes2 where year = 2017 and domestic = true and arrest = true;
+----------+
| count(*) |
+----------+
|     7148 |
+----------+
1 row in set (1.582 sec)
```

### 全文検索と通常の検索

数値の等価条件1つと真偽値の等価条件2つと全文検索（300万件くらいヒット）で、4000件ヒットするケース。Mroongaは0.4秒。（このときは常に最適化が効く。）

```text
MariaDB> select count(*) from crimes where year = 2017 and domestic = true and arrest = true and match(block) against('+ave' in boolean mode);
+----------+
| count(*) |
+----------+
|     3982 |
+----------+
1 row in set (0.440 sec)
```

数値の等価条件1つと真偽値の等価条件2つと全文検索（300万件くらいヒット）で、4000件ヒットするケース。Groongaは0.4秒。

```text
MariaDB> select mroonga_command('select --table crimes --filter "block @ \'ave\' && year == 2017 && domestic == true && arrest == true" --limit 0 --output_columns _key') as response;
+-------------------------------+
| response                      |
+-------------------------------+
| [[[3982],[["_key","Int32"]]]] |
+-------------------------------+
1 row in set (0.436 sec)
```

数値の等価条件1つと真偽値の等価条件2つと全文検索（300万件くらいヒット）で、4000件ヒットするケース。InnoDBは18秒。

```text
MariaDB> select count(*) from crimes2 where year = 2017 and domestic = true and arrest = true and match(block) against('+ave' in boolean mode);
+----------+
| count(*) |
+----------+
|     3982 |
+----------+
1 row in set (17.697 sec)
```

数値の等価条件1つと真偽値の等価条件2つと全文検索（4万件くらいヒット）で、10件ヒットするケース。Mroongaは0.01秒。（このときは常に最適化が効く。）

```text
MariaDB> select count(*) from crimes where year = 2017 and domestic = true and arrest = true and match(block) against('+milwaukee' in boolean mode);
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.010 sec)
```

数値の等価条件1つと真偽値の等価条件2つと全文検索（4万件くらいヒット）で、10件ヒットするケース。Groongaは0.01秒。

```text
MariaDB> select mroonga_command('select --table crimes --filter "block @ \'milwaukee\' && year == 2017 && domestic == true && arrest == true" --limit 0 --output_columns _key') as response;
+-----------------------------+
| response                    |
+-----------------------------+
| [[[10],[["_key","Int32"]]]] |
+-----------------------------+
1 row in set (0.010 sec)
```

数値の等価条件1つと真偽値の等価条件2つと全文検索（4万件くらいヒット）で、10件ヒットするケース。InnoDBは0.2秒。

```text
MariaDB> select count(*) from crimes2 where year = 2017 and domestic = true and arrest = true and match(block) against('+milwaukee' in boolean mode);
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.197 sec)
```
