---
path: create-and-update-enum-column
date: 2020-12-01T12:00:00.930Z
title: (MySQL×Laravel) ENUM型カラムを扱うなら、マイグレーションの面倒さを覚悟しようという話
description:
category: マイグレーション
cover: enum.jpg
author: hideyoshi
---
![image](enum.jpg)

この記事は、 [GAOGAO Advent Calendar 2020](https://qiita.com/advent-calendar/2020/gaogao) の 1日目の記事として公開されています。

こんにちは。 [GAOGAO](https://gaogao.asia/) にてスタートアップスタジオのエンジニアをしております [@mass-min](https://twitter.com/masumi_sugae) と申します。
GAOGAOでは秀吉と呼ばれています。どうぞよろしくお願いいたします。

## 結論
- Laravelでは、デフォルトでカラムの定義変更のマイグレーションは行えません( [Laravel Document migrations #Modifying Columns](https://laravel.com/docs/8.x/migrations#modifying-columns) )。
Composer で [doctrine/dbal](https://github.com/doctrine/dbal) を導入しましょう。
- ENUM型のカラムに対し格納する文字列を増やすようなマイグレーションは、 Doctrine DBAL ではサポートされていません。
生SQLを書くか、一時テーブルを利用するなどして定義を変更しましょう。

## ENUM 型カラムとは

本題に入る前に、 ENUM 型のなんたるかをおさらいしておきましょう。

ENUM 型とは、カラムの中に入る値がリストとして明示的に列挙された文字列型カラムです。

文字で言われてもいまいちよくわからないと思うので、例としてトヨタのクルマを格納するテーブル `toyota_cars` を考えてみましょう。
このテーブルは INT 型の `id` カラムと ENUM 型の `name` カラムを持ちます。簡単のためここでは2つのカラム以外を記載していませんが、本来は所有者と紐付けたり、値段や走行距離が入るものだと思ってください。
このテーブルを作成する SQL 、及びカラム定義の確認 SQL と結果は以下のようになります。

```sql
CREATE TABLE toyota_cars (
    id int AUTO_INCREMENT,
    name ENUM('86', 'supra', 'mr2', 'soarer'),
    PRIMARY KEY (id)
);
```

```sql
DESC toyota_cars;

+-------+-----------------------------------+------+-----+---------+----------------+
| Field | Type                              | Null | Key | Default | Extra          |
+-------+-----------------------------------+------+-----+---------+----------------+
| id    | int                               | NO   | PRI | NULL    | auto_increment |
| name  | enum('86','supra','mr2','soarer') | YES  |     | NULL    |                |
+-------+-----------------------------------+------+-----+---------+----------------+
2 rows in set (0.01 sec)
```

このテーブルに対して、以下の3つのレコードを insert することを考えます。

```shell
1. name: 'supra'
2. name: 'mr2'
3. name: 'corolla'
```

それぞれの INSERT 文と SQL 実行結果は以下のとおりです。

```sql
INSERT INTO toyota_cars (name) VALUES ('supra');

Query OK, 1 row affected (0.01 sec)
```

```sql
INSERT INTO toyota_cars (name) VALUES ('mr2');

Query OK, 1 row affected (0.00 sec)
```

```sql
INSERT INTO toyota_cars (name) VALUES ('corolla');

ERROR 1265 (01000): Data truncated for column 'name' at row 1
```

このように ENUM 型カラムではカラムの中に入る値がリストとして明示的に列挙されており、リストに入っていない値を突っ込もうとするとエラーが発生します。
格納される値に制限を設けることができるため、ステータスのカラムなどと相性がよいです。
また内部的には格納されるのは文字列ではなく文字列リストの index 値(つまり整数値)なので、 VARCHAR 型に比べて値を保持するのに必要なディスク容量も小さく済みます。

```sql
# 文字列指定でレコードを取得
SELECT * FROM toyota_cars WHERE name = 'supra';

+----+-------+
| id | name  |
+----+-------+
|  1 | supra |
+----+-------+
1 row in set (0.00 sec)

# 文字列ではなくindex値でも指定できる(index値は1から始まることに注意。0は空文字にあたる)
SELECT * FROM toyota_cars WHERE name = 2;

+----+-------+
| id | name  |
+----+-------+
|  1 | supra |
+----+-------+
1 row in set (0.00 sec)
```

[MySQL5.6 リファレンスマニュアル 11.4.4 ENUM 型](https://dev.mysql.com/doc/refman/5.6/ja/enum.html)

## ENUM 型カラムに格納できる値を増やす

このカラムに格納できる値を増やすことを考えます。
以下を受け付けるようにします。

```shell
name: corolla
```

カラム定義の更新 SQL とカラム定義確認 SQL 、結果は以下のとおりです。

```sql
ALTER TABLE toyota_cars
MODIFY COLUMN
    name ENUM('86', 'supra', 'mr2', 'soarer', 'corolla')
;
```

```sql
DESC toyota_cars;

+-------+---------------------------------------------+------+-----+---------+----------------+
| Field | Type                                        | Null | Key | Default | Extra          |
+-------+---------------------------------------------+------+-----+---------+----------------+
| id    | int                                         | NO   | PRI | NULL    | auto_increment |
| name  | enum('86','supra','mr2','soarer','corolla') | YES  |     | NULL    |                |
+-------+---------------------------------------------+------+-----+---------+----------------+
2 rows in set (0.00 sec)
```

追加できたようですね。では、先程できなかった以下のレコードの insert を再度試みます。

```shell
name: corolla
```

```sql
INSERT INTO toyota_cars (name) VALUES ('corolla');

Query OK, 1 row affected (0.00 sec)
```

いけましたね！

この時点で、格納されたレコードは以下のようになっています。

```sql
SELECT * FROM toyota_cars;

+----+---------+
| id | name    |
+----+---------+
|  1 | supra   |
|  2 | mr2     |
|  3 | corolla |
+----+---------+
3 rows in set (0.00 sec)
```

## Laravel のマイグレーション で ENUM 型カラムに格納できる値を増やしてみよう

テーブル作成からカラム定義変更までの一連の流れを Laravel のマイグレーション で実行してみます。
Laravel のデフォルト状態では、テーブル作成のマイグレーションしか実行できません。
カラム定義の変更マイグレーションを実行する際は [docrine/dbal](https://github.com/doctrine/dbal) というライブラリを使用します。
Composer で install しておきましょう。

```shell
$ composer require docrine/dbal
```

DBAL が install できたら、以下の2つのマイグレーションファイルを作成します。

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateToyotaCarsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('toyota_cars', function (Blueprint $table) {
            $table->integer('id')->autoIncrement();
            $table->enum('name', ['86', 'supra', 'mr2', 'soarer']);
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('toyota_cars');
    }
}

```

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class SortNameColumnOfToyotaCarsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name', ['86', 'supra', 'mr2', 'soarer', 'corolla'])->change();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name', ['86', 'supra', 'mr2', 'soarer'])->change();
        });
    }
}
```

テーブル作成のマイグレーション、及び ENUM リストの追加マイグレーションですね。
実行すると、下記のようなエラーが発生します。

```shell
$ php artisan migrate
Migrating: 2020_11_26_080427_create_toyota_cars_table
Migrated:  2020_11_26_080427_create_toyota_cars_table (15.54ms)
Migrating: 2020_11_26_081231_sort_name_column_of_toyota_cars_table

   Doctrine\DBAL\Exception

  Unknown column type "enum" requested. Any Doctrine type that you use has to be registered with \Doctrine\DBAL\Types\Type::addType(). You can get a list of all the known types with \Doctrine\DBAL\Types\Type::getTypesMap(). If this error occurs during database introspection then you might have forgotten to register all database types for a Doctrine Type. Use AbstractPlatform#registerDoctrineTypeMapping() or have your custom types implement Type#getMappedDatabaseTypes(). If the type name is empty you might have a problem with the cache or forgot some mapping information.

  at vendor/doctrine/dbal/src/Exception.php:125
    121▕     }
    122▕
    123▕     public static function unknownColumnType(string $name): self
    124▕     {
  ➜ 125▕         return new self('Unknown column type "' . $name . '" requested. Any Doctrine type that you use has ' .
    126▕             'to be registered with \Doctrine\DBAL\Types\Type::addType(). You can get a list of all the ' .
    127▕             'known types with \Doctrine\DBAL\Types\Type::getTypesMap(). If this error occurs during database ' .
    128▕             'introspection then you might have forgotten to register all database types for a Doctrine Type. Use ' .
    129▕             'AbstractPlatform#registerDoctrineTypeMapping() or have your custom types implement ' .

      +14 vendor frames
  15  database/migrations/2020_11_26_081231_sort_name_column_of_toyota_cars_table.php:18
      Illuminate\Support\Facades\Facade::__callStatic("table")

      +21 vendor frames
  37  artisan:37
      Illuminate\Foundation\Console\Kernel::handle(Object(Symfony\Component\Console\Input\ArgvInput), Object(Symfony\Component\Console\Output\ConsoleOutput))
```

`Unknown column type "enum" requested.` 、「要求された "enum" とかいうカラムの型、我々の知らない型ですねぇ」と DBAL に怒られてしまいました。
実は Laravel のデフォルトで ENUM 型カラムの作成はできるのですが、 2020/12/01 時点では DBAL を使った更新については ENUM 型はサポートされていないのです。
この記事では、2つの対処方法を示しておきます。お好みで選択してください。

### 方法1. 生SQLを書く
Laravel の DB facade では、 `statement` メソッドを使うことで生SQLが扱えます。
MySQL で ALTER TABLE 構文を使ってカラム定義を変更する例を上記で示しましたが、そこで使用した SQL をそのまま `statement` メソッドに渡して実行します。

なお、コードとしてのみやすさを担保するため PHP の文字列連結を使用しています。
改行して文字列連結する際は単語と単語の間にスペースを入れるのを忘れないでください。

また rollback 時はカラムに定義できる値が減るわけですから、rollback 後に定義できない値が入っているレコードが存在すると、マイグレーション実行時にエラーになります。
アプリケーションのルールに沿って、事前に対処するのを忘れないでください。

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

class SortNameColumnOfToyotaCarsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        DB::statement('ALTER TABLE toyota_cars'
            . ' MODIFY COLUMN'
            . ' name ENUM('
                . "'86', 'supra', 'mr2', 'soarer', 'corolla'"
            .');'
        );
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        // name = corolla のレコードが存在するとエラーになるので、適宜 name を修正する
        DB::update("UPDATE toyota_cars SET name = 'supra' WHERE name = ?", ['corolla']);

        DB::statement('ALTER TABLE toyota_cars'
            . ' MODIFY COLUMN'
            . ' name ENUM('
                . "'86', 'supra', 'mr2', 'soarer'"
            .');'
        );
    }
}
```

### 方法2. 一時カラムを利用する
2つ目の方法は、一時カラムを利用する方法です。
手はずとしては、以下のようになります。

- 現行カラムと同定義の一時カラムを作成
- 現行のカラム値で一時カラムを上書き
- 現行カラムを削除
- 定義できる文字列が追加された状態で現行と同名の新規カラムを作成
- 一時カラムの値で新規カラムを上書き
- 一時カラムを削除


```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

class SortNameColumnOfToyotaCarsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        // 現行カラムと同定義の一時カラムを作成
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name_tmp', ['86', 'supra', 'mr2', 'soarer'])
                ->nullable()
                ->after('name');
        });

        // 現行のカラム値で一時カラムを上書き
        DB::update('UPDATE toyota_cars AS tc SET tc.name_tmp = tc.name');

        // 現行カラムを削除
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->dropColumn('name');
        });

        // 定義できる文字列が追加された状態で現行と同名の新規カラムを作成
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name', ['86', 'supra', 'mr2', 'soarer', 'corolla'])
                ->nullable()
                ->after('name_tmp');
        });

        // 一時カラムの値で新規カラムを上書き
        DB::update('UPDATE toyota_cars AS tc SET tc.name = tc.name_tmp');

        // 一時カラムを削除
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->dropColumn('name_tmp');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        // name = corolla のレコードが存在するとエラーになるので、適宜 name を修正する
        DB::update("UPDATE toyota_cars SET name = 'supra' WHERE name = ?", ['corolla']);

        // 現行カラムと同定義の一時カラムを作成
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name_tmp', ['86', 'supra', 'mr2', 'soarer', 'corolla'])
                ->nullable()
                ->after('name');
        });

        // 現行のカラム値で一時カラムを上書き
        DB::update('UPDATE toyota_cars AS tc SET tc.name_tmp = tc.name');

        // 現行カラムを削除
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->dropColumn('name');
        });

        // 定義できる文字列が追加された状態で現行と同名の新規カラムを作成
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->enum('name', ['86', 'supra', 'mr2', 'soarer'])
                ->nullable()
                ->after('name_tmp');
        });

        // 一時カラムの値で新規カラムを上書き
        DB::update('UPDATE toyota_cars AS tc SET tc.name = tc.name_tmp');

        // 一時カラムを削除
        Schema::table('toyota_cars', function (Blueprint $table) {
            $table->dropColumn('name_tmp');
        });
    }
}
```

記述量は多くなりますが、「特定の RDBMS に依存するような書き方をせず、コード上ではできるだけ Laravel の機能だけで完結させたい」という場合はこちらのほうが良いかもしれません。
上記では DB facade から update 文を呼び出していますが、 Eloquent Model 経由で呼び出せば 特定の RDBMS に依存する部分を更に隠蔽することができると思います。

## まとめ
- Laravel では、デフォルトでカラムの定義変更のマイグレーションは行えません( [Laravel Document migrations #Modifying Columns](https://laravel.com/docs/8.x/migrations#modifying-columns) )。
Composer で [doctrine/dbal](https://github.com/doctrine/dbal) を導入しましょう。
- ENUM 型のカラムに対し格納する文字列を増やすようなマイグレーションは、 2020/12/01 時点では Doctrine DBAL ではサポートされていません。
生 SQL を書くか、一時カラムを利用するなどして気合いで定義を変更しましょう。

## 最後に

弊社 GAOGAO は現在副業含めて30名以上のエンジニアの方が参画し、グローバル（シンガポール、バンコク、ホーチミン、US、日本など）で15件以上お客様の開発のお手伝いをさせていただいております。

もしグローバルでスキルを試してみたいというエンジニアの方(デザイナーの方も)いましたら、お気軽にご連絡いただければ幸いです！
私ますみん( [@mass-min](https://twitter.com/masumi_sugae) )、弊社代表テジタク( [@tejitak](https://twitter.com/tejitak) )、そしてメンバー一同、皆様からのご連絡お待ちしております！！

世界中で「モノつくり」の連鎖を起こすことができる世界を実現するための仕組みを是非一緒に作っていきましょう！
