# 1_CircleCIでの自動テスト

<p style='text-align: right;'> &copy; 20220112 by Takanori Shima </p>

```
* 以下、Cloud9上でターミナルを起動してコマンドを打つ *
最終的には3つのターミナルを同時に開いておく
- MySQL用(占有)
- Laravelサーバ起動用(占有)
- artisanコマンドを随時実行用
```

## 1. 前提
以下の作業が終了していることが前提となる
```
1. 開発環境でLaravelアプリが起動している
2. アプリがGithubのリポジトリにpushされている
3. Herokuにデプロイが完了している
4. テストコードが書かれている
5. CircleCIにアカウントが作成できている
```

CircleCIのアカウント作成とGithubとの連携の作業は以下のサイトを参考にする。

https://qiita.com/sendo111/items/27b8dced1cbe9019749c

## 2. CircleCI 上で使うデータベース接続情報を設定する

CircleCI が提供する Maria DB(MySQL) の Docker イメージを使う

config/database.php 92行目附近に circle_testing という箇所の記述を追加

```
        // CircleCI テスト用
        'circle_testing' => [
            'driver' => 'mysql',
            'host' => '127.0.0.1',
            'port' => '3306',
            'database' => 'circle_test',
            'username' => 'root',
            'password' => '',
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'prefix_indexes' => true,
            'strict' => true,
            'engine' => null,
        ],


```

## 3. プロジェクトに .circleci/config.yml を配置する
開発環境にターミナルで以下を実行する

```
mkdir .circleci && touch .circleci/config.yml
```
.circleci/config.yml というファイルができるので、以下のように内容を変更
```
version: 2
jobs:
  build:
    docker:
      - image: circleci/php:7.3.0-node-browsers
      - image: circleci/mariadb:10.4
    environment:
      - DB_CONNECTION: circle_testing
    working_directory: ~/ci-demo
    steps:
      - checkout
      - run:
          name: Update apt-get
          command: sudo apt-get update
      - run:
          name: Docker php extensions install
          command: sudo docker-php-ext-install pdo_mysql
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "composer.json" }}
            - v1-dependencies-
      - run:
          name: Install PHP libraries
          command: composer install -n --prefer-dist
      - save_cache:
          paths:
            - ./vendor
          key: v1-dependencies-{{ checksum "composer.json" }}
      - run:
          name: Run PHPUnit
          command: vendor/bin/phpunit
```

## 4. Git/Github

```
git add .
git commit -m "CircleCI環境設定完了"
git log
git push origin main
```

## 5. CircleCIにプロジェクトを追加し、初回テスト自動実行
CircleCIの `Projects` メニューからアプリのGithubのリポジトリ探し、`Set Up Project` ボタンを押す

その際、自分がGithubのリポジトリにpush作成した .circleci/config.ymlを選択し、branchは`main`を選択。

初回のテストが実行される。Running状態が8分ほど続き、`Success`という状態に切り替わればテスト成功。

workflowのリンクをクリックすると、jobごとの詳細なテスト実行結果を表示することができる。

## 6. わざと自動テストが失敗するようにコードを書き換える
開発環境の以下の箇所を変更する tests/Feature/LoginControllerTest.php

```
    /** @test */
    public function ログアウトをするとログイン前の画面のリダイレクトする()
    {
        // factoryを使ってダミーユーザー作成
        $user = factory(User::class)->create();
        // そのユーザーを認証状態にセット
        $this->actingAs($user);
        // ログアウトリクエストを送る
        $response = $this->get('/logout');
        // ログアウトした画面にリダイレクトするかチェック
        $response->assertRedirect('/top'); // 変更
    }
```

Githubにpushする
```
git add .
git commit -m "CircleCIエラーテスト"
git log
git push origin main
```

この時点でCircleCIでテストが自動実行される。

statusが `Failed`となる。workflowを見ると、jobの詳細が表示される

さきほどの tests/Feature/LoginControllerTest.php をもとに戻す
```
    /** @test */
    public function ログアウトをするとログイン前の画面のリダイレクトする()
    {
        // factoryを使ってダミーユーザー作成
        $user = factory(User::class)->create();
        // そのユーザーを認証状態にセット
        $this->actingAs($user);
        // ログアウトリクエストを送る
        $response = $this->get('/logout');
        // ログアウトした画面にリダイレクトするかチェック
        $response->assertRedirect('/');
    }
```

再度Githubのpushする
```
git add .
git commit -m "CircleCIエラーを復旧する"
git log
git push origin main
```

再度テストが自動実行され、1分程度でstatus が　`Success`と表示される