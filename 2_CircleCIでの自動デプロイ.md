# 2_CircleCIでの自動デプロイ

<p style='text-align: right;'> &copy; 20220112 by Takanori Shima </p>

```
* 以下、Cloud9上でターミナルを起動してコマンドを打つ *
最終的には3つのターミナルを同時に開いておく
- MySQL用(占有)
- Laravelサーバ起動用(占有)
- artisanコマンドを随時実行用
```

以下のサイトを参考)

https://qiita.com/seiya0429/items/83e3f5d53a01d6fbd8d0

## 1. 前提
以下の作業が終了していることが前提となる
```
1. 開発環境でLaravelアプリが起動している
2. アプリがGithubのリポジトリにpushされている
3. Herokuにデプロイが完了している
4. テストコードが書かれている
5. CircleCIにアカウントが作成できている
6. CircleCIで自動テストが実行できている
```

## 2. .circleci/config.yml の変形

以下のように内容を変更
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
      #heroku deploy
      - deploy:
          name: Deploy main to Heroku
          command: |
            if [ "${CIRCLE_BRANCH}" == "main" ]; then
              git push https://heroku:$HEROKU_API_KEY@git.heroku.com/$HEROKU_APP_NAME.git main
            fi
```

## 3. CircleCIのProjectに HEROKU_API_KEYとHEROKU_APP_NAMEを設定
以下を参考にする

https://qiita.com/seiya0429/items/83e3f5d53a01d6fbd8d0

## 4. Git/Github

```
git add .
git commit -m "CircleCI自動デプロイ設定完了"
git log
git push origin main
```

CircleCIで自動テストと自動デプロイ完了

## 5. welcome.blade.phpの変形
resources/views/welcome.blade.php を以下に変形
```
@extends('layouts.app')
@section('title', 'Schedule App')
@section('content')
    <div class="row mt-3 mb-3">
        <h1 class="col-sm-12 text-center">スケジュールアプリへようこそ！</h1>
    </div>
    <div class="row">
        <a href="/signup" class="offset-sm-1 col-sm-4 btn btn-primary">新規会員登録</a>
        <a href="/login" class="offset-sm-1 col-sm-4 btn btn-danger">ログイン</a>
    </div>
@endsection
```

## 6. Git/Github
```
git add .
git commit -m "welcome.blade.phpの変形"
git push origin main
```

CircleCIで自動テストと自動デプロイ完了

## 7. カレンダー表示の修正
app/Http/Controllers/RecordsController.php の get_calendarアクションを変更

```
    // 自分の記録のカレンダー情報の取得
    public function get_calendar()
    {
        // ログインしているユーザーの顧客全記録を取得
        // $records = DB::select('SELECT records.id, customers.name, records.customer_id, records.title, records.start, records.end, records.color, records.created_at, records.updated_at  FROM records JOIN customers ON records.customer_id=customers.id JOIN users ON customers.user_id=users.id WHERE users.id=? AND customers.delete_flag=0', [\Auth::id()]);
        
        $records = DB::table('records')
        ->join('customers', 'records.customer_id', '=', 'customers.id')
        ->join('users', 'customers.user_id', '=', 'users.id')
        ->where('customers.delete_flag', '=', 0)
        ->where('users.id', '=', \Auth::id())
        ->select('records.id', 'records.title', 'customers.name', 'records.start', 'records.end', 'records.color')
        ->get();
        
        foreach($records as $record){
            $record->title = '【' . $record->name . 'さん】:  ' . $record->title;
        }
        // var_dump($records);
        $list = array('records' => $records);
        
        // 明示的に指定しない場合は、text/html型と判断される
        header("Content-type: application/json; charset=UTF-8");
        //JSONデータを出力
        echo json_encode($list);
        exit;
    }
```

## 8. Git/Github
```
git add .
git commit -m "カレンダー表示修正"
git push origin main
```

CircleCIで自動テストと自動デプロイ完了