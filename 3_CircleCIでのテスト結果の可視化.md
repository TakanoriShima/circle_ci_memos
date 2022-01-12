# 3_CircleCIでのテスト結果の可視化

<p style='text-align: right;'> &copy; 20220112 by Takanori Shima </p>

```
* 以下、Cloud9上でターミナルを起動してコマンドを打つ *
最終的には3つのターミナルを同時に開いておく
- MySQL用(占有)
- Laravelサーバ起動用(占有)
- artisanコマンドを随時実行用
```

以下のサイトなどを参考)

https://zenn.dev/y640/articles/qiita-20210812-873c159097a9533e88cf

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
        command: [--max_allowed_packet=32M]
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
          name: run php unit tests
          command: |
                mkdir -p phpunit 
                phpdbg -qrr vendor/bin/phpunit -d memory_limit=512M --log-junit phpunit/junit.xml --coverage-html phpunit/coverage-report
      - store_test_results:
          path:  phpunit
      - store_artifacts:
          path:  phpunit
      #heroku deploy
      - deploy:
          name: Deploy main to Heroku
          command: |
            if [ "${CIRCLE_BRANCH}" == "main" ]; then
              git push https://heroku:$HEROKU_API_KEY@git.heroku.com/$HEROKU_APP_NAME.git main
            fi
```
## 3. Git/Github

```
git add .
git commit -m "CircleCI自動テスト結果の可視化完了"
git log
git push origin main
```
