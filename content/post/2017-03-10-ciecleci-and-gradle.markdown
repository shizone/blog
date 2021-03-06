---
categories: java gradle circleci
comments: true
date: 2017-03-10T09:32:48Z
title: CircleCIでGradleのテストを並列実行する
url: /blog/2017/03/10/ciecleci-and-gradle/
---

現在開発を行っているプロジェクトでは、Spring Bootを使って開発を行っているのですが、そこでのテストをCI環境で実行できるよう設定を行ったので、その手順を書いておきます。

## CircleCIで普通にテストできるようにする

最初は並列ではなく、1つのコンテナを使ってCircleCIでテストできるように設定を行います。まずcircle.ymlを以下のように準備。

``` yml
machine:
  java:
    version: openjdk8
  timezone:
    Asia/Tokyo
  environment:
    _JAVA_OPTIONS: "-Xms512m -Xmx1024m"
    GRADLE_OPTS: '-Dorg.gradle.jvmargs="-Xmx1024m -XX:+HeapDumpOnOutOfMemoryError"'
  post:
    - sudo service postgresql stop

dependencies:
  override:
    - ./gradlew testClasses

database:
  post:
    - mysql -e 'create database [データベース名];'
    # flywayなどでのマイグレーション

test:
  override:
    - ./gradlew test
  post:
    - mkdir -p $CIRCLE_TEST_REPORTS/junit/ && find . -type f -regex ".*/build/test-results/.*xml" -exec cp {} $CIRCLE_TEST_REPORTS/junit/ \;:
```

### メモリ割り当てについて

machine.environmentでJAVA_OPTIONSに"-Xms512m -Xmx1024m"を指定しています。これはCircleCIでは1つのコンテナには4Gのメモリが割当られており、その上限をこえると、コンテナがフリーズして、10分経過するとテスト失敗になるという現象に対応するためです。合わせてGRADLE_OPTSにも同様の設定をおこなっています。

このあたりの設定も状況によっては増やせる場合もありますので、テストを実行しながら、調整してみてください。

- [Your build hit the 4G memory limit](https://circleci.com/docs/1.0/oom/)

### 使わないデータベースを止める

CircleCIではデフォルトでPostgreSQLとMySQLがインストールされたコンテナが準備されます。machine.postで使わないデータベースを止めることで、貴重なメモリの使用量を抑えることができます。

今回テスト対象のデータベースはMySQLですので、PostgreSQLを止めてメモリを節約します。

### dependenciesでライブラリをダウンロードしておく

CircleCIではdatabaseサイクルが終わったタイミングで、次回のビルドを高速に実行できるよう、依存ライブラリなどをキャッシュする仕組みがあります。

しかしGradleではテストを実行する直前まで依存ライブラリはダウンロードされず、通常のままだと依存ライブラリをキャッシュに含めることができません。

そこでdependencies.overrideにてtestClassesタスクを実行しておきます。こうすることで、依存ライブラリがダウンロードされ、databaseサイクル終了後にキャッシュが作成されるようになります。

### Spring Bootのprofileはciにする

CircleCIで動かす場合はデータベースの接続先が開発環境などとは変わるはずですので、CircleCI専用のapplication.ymlをapplication-ci.ymlとして作成します。

``` yml
spring:
  profiles:
    active: ci
  datasource:
    url: jdbc:mysql://localhost:3306/{データベース名}
    username: ubuntu
    password:
    driverClassName: com.mysql.jdbc.Driver
```

CircleCIのMySQLには上記の設定で接続可能です。次にテスト実行時に

``` bash
SPRING_PROFILES_ACTIVE=ci ./gradlew test
```

とすることで、application-ci.ymlのデータベース接続情報を使用するようになります。

### テスト実行結果を集約する
test.postにて、テスト結果のxmlを$CIRCLE_TEST_REPORTSにコピーしておきます。こうすることで、CircleCIの画面からテスト結果を簡単に見ることができます。

``` bash
test:
  post:
    - mkdir -p $CIRCLE_TEST_REPORTS/junit/ && find . -type f -regex ".*/build/test-results/.*xml" -exec cp {} $CIRCLE_TEST_REPORTS/junit/ \;:
```

- [Collecting test metadata](https://circleci.com/docs/1.0/test-metadata/#gradle-junit-results)

## 並列テストが実行できるようにする

次にCircleCI+Gradleで並列テストをすることを考えてみます。

一般的に並列テストを行う場合は、テスト対象のクラスを取得し、これをノードそれぞれに均等に割り振ることでテストを分散して実行します。

Gradleにはデフォルトではテスト対象のクラスフィルタリングする昨日はあるのですが、対象クラスを個別に指定する方法はありません。

https://docs.gradle.org/current/userguide/java_plugin.html#test_filtering

ですので今回はGradle実行時に-Pオプションを指定し、以下のようにして対象クラスを一括して渡す方法を採用しています。

``` bash
./gradlew test -PtestFiles=./src/test/java/com/example/HogeTest ./src/test/java/com/example/FugaTest ....以下テスト対象クラスを列挙
```

まずは、このオプションを組み立てつつ、gradleｗ実行する専用のシェルスクリプト（circleci.sh)を準備します。

``` bash
testFiles=$(find ./src/test -name *Test.java | sort | awk "NR % ${CIRCLE_NODE_TOTAL} == ${CIRCLE_NODE_INDEX}")
echo $testFiles
SPRING_PROFILES_ACTIVE=ci ./gradlew :webapp:test -PtestFiles="$testFiles"
```

CircleCI上でビルドに使用しているノード数は環境変数CIRCLE_NODE_TOTALから、自身のノード番号は環境変数CIRCLE_NODE_INDEXから取得できますので、これをawkから利用しつつ、テスト対象クラスを分散させます。

次にbuild.gradle内では-Pオプションで渡されたtestFilesのみをテスト対象にするよう、includeTestsMatchingを使って設定を行います。

``` bash
test {
  if (project.hasProperty("testFiles")) {
      ArrayList files = project.getProperties().get("testFiles")
              .replaceAll("./src/test/java/", "")
              .replaceAll("/", ".")
              .replaceAll(".java", "")
              .split("\\s+")
      for(String file : files) {
          println file
          filter {
              includeTestsMatching file
          }
      }
  }  
}
```

こうすることで-PtestFilesで指定されたもののみ、テストを行うことができます。

最後に、並列実行できるようcircle.ymlを修正します。

``` yml
test:
  override:
    - ./circleci.sh:
        parallel: true
  post:
    - mkdir -p $CIRCLE_TEST_REPORTS/junit/ && find . -type f -regex ".*/build/test-results/.*xml" -exec cp {} $CIRCLE_TEST_REPORTS/junit/ \;:
        parallel: true
```

テストは先ほど作成したcircle.shを実行するようにしparallel: trueを付与して並列実行するようにします。**parallel: trueはインデント4つであることに注意！**
