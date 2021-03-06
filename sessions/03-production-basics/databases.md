layout: true
class: inverse, middle, large

---
class: special
# Galactic Database

Authors: @martenson, @nsoranzo

.footnote[\#usegalaxy / @galaxyproject]

---
class: larger

## Please Interrupt!
Your questions are bound to be answered.

---
# Defaults

* Galaxy uses the [SQLAlchemy](http://www.sqlalchemy.org/) database abstraction layer. This allows for different database servers to be plugged in.
* By default Galaxy automatically creates and uses an [SQLite](https://sqlite.org/) database during the first startup.
  * The database is in the file `database/universe.sqlite`

---
# Choices

* SQLite
  * Useful for single-user Galaxy or development.
* **PostgreSQL**
  * The recommended standard for anything serious.
* ~~MySQL~~
  * Supported by SQLAlchemy but Galaxy is not tested against it.

---
# Configuration

`database_connection` is specified as a [database URL](http://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls) in `galaxy.ini`
  * Default SQLite: `sqlite:///./database/universe.sqlite?isolation_level=IMMEDIATE`
  * Local PostgreSQL (socket) `postgresql://<user>:<password>@/<db_name>?host=/var/run/postgresql`
  * Network PostgreSQL: `postgresql://<user>:<password>@<host>:5432/<db_name>`

---
# Tuning - Pool

If the server logs errors about not having enough database pool connections.
* `database_engine_option_pool_size = 5` (10 on Main)
* `database_engine_option_max_overflow = 10` (20 on Main)

???
Values for Main are available at https://github.com/galaxyproject/usegalaxy-playbook/blob/master/env/main/group_vars/galaxyservers/vars.yml

---
# Tuning - Server-side cursors

If large database query results are causing memory or response time issues in the Galaxy process, leave it on server.
* `database_engine_option_server_side_cursors = False` (True on Main, PostgreSQL only, recommended)

---
# Tuning - Slow query logging

Queries slower than this threshold (in s) will be logged at debug level

`slow_query_log_threshold = 0` (5000 on Main)

---
# Tuning - TS install database

Galaxy can track Tool Shed data in a separate DB.

```ini
install_database_connection = sqlite:///./database/universe.sqlite?isolation_level=IMMEDIATE
```

This allows:
* Bootstrapping fresh Galaxy instances with prebuilt/tested tool sets
* Atomic installation/rollback (esp. with SQLite)

???
All other database config options but prefixed with `install_` are also available.

---
# Migrations

Changes in the Galaxy DB model are captured incrementally in the form of [atomic scripts](https://github.com/galaxyproject/galaxy/tree/dev/lib/galaxy/model/migrate/versions).

Each script can both upgrade and downgrade a DB.

```console
$ ./manage_db.sh upgrade
$ ./manage_db.sh downgrade --version=132
```

---
# Access through model

Python script to access Galaxy's database layer **via the Galaxy models**.

```console
$ python -i scripts/db_shell.py
>>> new_user = User('foo@example.org', 'secret')
>>> new_user.username = 'foo'
>>> sa_session.add(new_user)
>>> sa_session.flush()
>>> sa_session.query(User).all()
```

???
Need to be inside the Galaxy virtualenv for this to work

---
# Other Databases

* The Reports app is hooked to the Galaxy DB to present data in useful format.
* The Tool Shed app has its own database.

---
# Exercise

[Connecting Galaxy to PostgreSQL - Exercise](https://github.com/galaxyproject/dagobah-training/blob/2018-oslo/sessions/03-production-basics/ex2-postgres.md)
