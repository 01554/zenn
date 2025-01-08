---
title: "DifyをDockerで立ち上げたら relation dify_setups does not exist "
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dify,docker]
published: false
---

お疲れ様です、波浪です
掲題の通りなんですが、
https://docs.dify.ai/ja-jp/getting-started/install-self-hosted/docker-compose#:~:text=docker%2Dcompose%20up%20%2Dd
この手順に従って 
`docker compose up` したら dify_setups がないとか言われ立ち往生したんで
同じ現象の人に向けて解決方法の共有です。


# 解決方法

```
~/app/dify/docker❯ brew install dos2unix

<中略>

~/app/dify/docker❯ unix2dos .env
unix2dos: converting file .env to DOS format...
```
dos2unixを入れて .envを変換したら治りました。
いやそんなんわからんよ



## 自分の環境
- MacBookPro 14インチ(M1 Pro)
- Mac OS 15.0.1（24A348）
- Dify 0.1.5


## 実際に出ていたログ

```
docker-api-1         | 2025-01-08 10:43:17,784.784 ERROR [Dummy-3] [app.py:875] - Exception on /console/api/setup [GET]
docker-api-1         | Traceback (most recent call last):
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1967, in _exec_single_context
docker-api-1         |     self.dialect.do_execute(
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/default.py", line 941, in do_execute
docker-api-1         |     cursor.execute(statement, parameters)
docker-api-1         | psycopg2.errors.UndefinedTable: relation "dify_setups" does not exist
docker-api-1         | LINE 2: FROM dify_setups
docker-api-1         |              ^
docker-api-1         |
docker-api-1         |
docker-api-1         | The above exception was the direct cause of the following exception:
docker-api-1         |
docker-api-1         | Traceback (most recent call last):
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/flask/app.py", line 917, in full_dispatch_request
docker-api-1         |     rv = self.dispatch_request()
docker-api-1         |          ^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/flask/app.py", line 902, in dispatch_request
docker-api-1         |     return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/flask_restful/__init__.py", line 489, in wrapper
docker-api-1         |     resp = resource(*args, **kwargs)
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/flask/views.py", line 110, in view
docker-api-1         |     return current_app.ensure_sync(self.dispatch_request)(**kwargs)  # type: ignore[no-any-return]
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/flask_restful/__init__.py", line 604, in dispatch_request
docker-api-1         |     resp = meth(*args, **kwargs)
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/controllers/console/setup.py", line 19, in get
docker-api-1         |     setup_status = get_setup_status()
docker-api-1         |                    ^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/controllers/console/setup.py", line 55, in get_setup_status
docker-api-1         |     return DifySetup.query.first()
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/orm/query.py", line 2728, in first
docker-api-1         |     return self.limit(1)._iter().first()  # type: ignore
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/orm/query.py", line 2827, in _iter
docker-api-1         |     result: Union[ScalarResult[_T], Result[_T]] = self.session.execute(
docker-api-1         |                                                   ^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/orm/session.py", line 2362, in execute
docker-api-1         |     return self._execute_internal(
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/orm/session.py", line 2247, in _execute_internal
docker-api-1         |     result: Result[Any] = compile_state_cls.orm_execute_statement(
docker-api-1         |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/orm/context.py", line 305, in orm_execute_statement
docker-api-1         |     result = conn.execute(
docker-api-1         |              ^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1418, in execute
docker-api-1         |     return meth(
docker-api-1         |            ^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/sql/elements.py", line 515, in _execute_on_connection
docker-api-1         |     return connection._execute_clauseelement(
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1640, in _execute_clauseelement
docker-api-1         |     ret = self._execute_context(
docker-api-1         |           ^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1846, in _execute_context
docker-api-1         |     return self._exec_single_context(
docker-api-1         |            ^^^^^^^^^^^^^^^^^^^^^^^^^^
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1986, in _exec_single_context
docker-api-1         |     self._handle_dbapi_exception(
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 2355, in _handle_dbapi_exception
docker-api-1         |     raise sqlalchemy_exception.with_traceback(exc_info[2]) from e
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/base.py", line 1967, in _exec_single_context
docker-api-1         |     self.dialect.do_execute(
docker-api-1         |   File "/app/api/.venv/lib/python3.12/site-packages/sqlalchemy/engine/default.py", line 941, in do_execute
docker-api-1         |     cursor.execute(statement, parameters)
docker-api-1         | sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "dify_setups" does not exist
docker-api-1         | LINE 2: FROM dify_setups
docker-api-1         |              ^
docker-api-1         |
docker-api-1         | [SQL: SELECT dify_setups.version AS dify_setups_version, dify_setups.setup_at AS dify_setups_setup_at
docker-api-1         | FROM dify_setups
docker-api-1         |  LIMIT %(param_1)s]
docker-api-1         | [parameters: {'param_1': 1}]
docker-api-1         | (Background on this error at: https://sqlalche.me/e/20/f405)
```

## 参考にしたページ
https://github.com/langgenius/dify/discussions/7252
