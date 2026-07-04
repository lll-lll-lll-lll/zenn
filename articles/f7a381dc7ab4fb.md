---
title: "Postgresqlで安全にDROP COLUMNを実行できるのか"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postgresql"]
published: false
---
# 背景
PostgreSQLで不要になったカラムを削除する際に、DROP COLUMNって物理的に削除しちゃうの？
ACCESS EXCLUSIVEロックがかかるのって書いてるしちゃんと調べないと怖いなーって思ったので調べてみました。

# pg_attributeとは
docs
https://www.postgresql.jp/docs/17/catalog-pg-attribute.html
attisdroppedを有効にすることで、論理的に列を削除します。

更新箇所
https://github.com/postgres/postgres/blob/master/src/backend/catalog/dependency.c#L322-L396
https://github.com/postgres/postgres/blob/master/src/backend/catalog/dependency.c#L390

drop cloumn
呼び出し箇所
https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/tablecmds.c#L5299-L5303

実装箇所
https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/tablecmds.c#L8966-L9167

