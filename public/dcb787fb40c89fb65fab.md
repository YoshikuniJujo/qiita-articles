---
title: Yesod + Postgresで、stack exec -- yesod develができるところまで
tags:
  - yesod
  - postgres
  - 自分用メモ
private: false
updated_at: '2017-03-15T14:17:06+09:00'
id: dcb787fb40c89fb65fab
organization_url_name: null
slide: false
ignorePublish: false
---
Yesod + Postgresで、stack exec -- yesod develができるところまで
============================================================

はじめに
-------

この記事は[YesodでpostgreSQLを使う Haskell stack](http://blog.livedoor.jp/rtabaladi_58/archives/58083990.html)を参考にしました。参考にした記事をこえる内容はありません。自分用のまとめです。

前提
----

システムにStackやPostgreSQLはインストールずみであるとします。事前にYesod + sqliteでのテストをしてあるものとします。また、PostgreSQLは起動ずみとします。

Yesodのプロジェクトの名前
-----------------------

ここではYesodのプロジェクトの名前をtestYesodとします。

必要なユーザの作成
----------------

    % psql -U postgres -d postgres
    # create user "testYesod" password 'testYesod';
    # create user "testYesod_LOWER" password 'testYesod';

必要なデータベースの作成
---------------------

    # create database "testYesod_LOWER" owner "testYesod";
    # create database "testYesod_LOWER_test" owner "testYesod";
    # \q

Yesodのプロジェクトの作成とテスト
------------------------------

    # stack new testYesod yesod-postgres && cd testYesod
    # stack build
    # stack exec -- yesod devel

これで、[http://localhost:3000](http://localhost:3000)にアクセスするとデフォルトのYesodのページを見ることができる。

    # stack test
    ...
    10 examples, 0 failures
