---
path: devide-route-definition-file-of-laravel
date: 2020-11-03T02:36:05.991Z
title: Laravelでroute定義ファイルを分割する
description: Webアプリケーションを作成していくと、だんだんとroute定義ファイルが肥大していきます。この記事ではLaravelでそのroute定義ファイルを分割する方法について紹介します。
category: Laravel
cover: laravel.jpg
author: hideyoshi
---
こんにちは、[GAOGAO](https://gaogao.asia/)にてスタートアップスタジオのエンジニアをしております [@mass-min](https://twitter.com/masumi_sugae) と申します。GAOGAOでは秀吉と呼ばれています。どうぞよろしくお願いいたします。

## Route定義ファイルの肥大化を防ぐ

Webアプリの開発を進めていくに従って、ページはどんどん増えていきます。最初はユーザーがアクセスする画面しかなかったアプリケーションも、管理画面が追加されたりAPIが生えたりしてrouteの肥大化が進みます。\
ということで今回は、Laravelを使ったアプリケーション開発において、route定義ファイルが肥大化し開発速度に影響を及ぼすことがないよう、適切に分ける方法をご紹介します。

## routes/web.phpを分割する

APIのパスは routes/api.php に書くのが良いでしょう。今回のケースでは、ユーザーがアクセスするパスと管理画面用のパスとでroute定義ファイルを分けることを考えます。

### 1. 管理画面用のroute定義ファイルを作成

管理画面用のrouteだけを定義するファイルを作成します。ユーザーがアクセスするページ用のroute定義ファイルが web.php なので、ここでは webAdmin.php という名前でファイルを作成します。

routes/webAdmin.php

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return 'This is admin page.';
});
```

### 2. サービスプロバイダにroute定義ファイルを登録

app/Providers/RouteServiceProvider.php 

## 最後に

弊社GAOGAOは現在副業含めて30名以上のエンジニアの方が参画し、グローバル（シンガポール、バンコク、ホーチミン、US、日本など）で15件以上お客様の開発のお手伝いをさせていただいております。

もし、グローバルでスキルを試してみたいというエンジニアの方(デザイナーの方も)いましたらお気軽に [@tejitak](https://twitter.com/tejitak) までご連絡いただければ幸いです！世界中で「モノつくり」の連鎖を起こすことができる世界を実現するための仕組みを是非一緒に作っていきましょう！