layout: true
class: inverse, middle, large

---
class: special
# Storage Management

slides by @natefoo, @martenson

.footnote[\#usegalaxy / @galaxyproject]

---
class: larger

## Please Interrupt!
Questions => Answers

---
# Philosophy of Galaxy Storage

- Foster transparency and reproducibility
- Data is always created, *never overwritten*
- Data is never deleted unless *explicitly instructed*
- Even deleted data can be undeleted unless *forcibly purged*

Additionally, tools can produce large amounts of transient data while running.

---
class: normal
# Mechanics

* When files are produced they are stored in the filesystem defined by `file_path` config in `galaxy.ini`.
  * Defaults to `database/files`.

--
*  A database entry in `Dataset` table is created.
  * Among others it has:  `| id	| deleted	| purged | file_size |`

--
* To user we present the data through:
  * Entries in `HistoryDatasetAssociation` table.
    * Among others it has:  `| id	| history_id | deleted	| dataset_id |`
  * Entries in `LibraryDatasetDatasetAssociation` table.
    * Among others it has:  `| id	| library_dataset_id | deleted	| dataset_id |`

--
Any number of 'Association' records can point to a base Dataset -- this is how copying histories, history items, and libraries work without needing to copy actual file contents.

---
# Storage can grow fast

An "average" NGS analysis (by Anton): **66 GB**

10 users, 10 histories: **> 6 TB**

Solutions:

- Quotas
- Clean up deleted data (aggressively)
- Forced removal based on age available
- Data libraries for common data

---
# Data Libraries

Provide a way to conveniently share Galaxy datasets within a group of Galaxy users or with everybody that has access to a specific instance of Galaxy.

* Can import data from filesystem *without duplicating* it.

--
* Can import *whole directories* preserving the folder structure.

--
* The dataset's size does not count towards user's quota.
  * Every dataset in the library is *stored only once* no matter how many users are using it in their histories.

--
* Uses roles and groups to control permissions on library/dataset level.
  * Only admins can _create_ libraries.

---
# Libraries configuration

In `galaxy.ini`:

* `user_library_import_dir`
  * Allows authorized non-administrators to upload a directory of files.
  * Directory must contain sub-directories named the same as user's email.
  * Works well in combination with `ftp_upload_dir`.
* `allow_path_paste`
  * Admin-only, allows importing from any path that the Galaxy's user has access to.

???
If you use old library interface you can also set `library_import_dir`.
It specifies which folder admins may browse and import from.

---
# Libraries Exercise

* Create a data library

--
* Clone `https://github.com/galaxyproject/galaxy-test-data` to your home folder.

--
* Configure your instance to allow importing data from your home folder.

--
* Import to your library a subset of datasets from the newly created `~/galaxy-test-data/`.
  * Use `Link files instead of copying` for some.

--
* Import some files from the library to a new history.

---
# Data cleanup

* Can only delete (or purge) dataset when all 'associations' pointing at it have been marked `deleted`.
* Cleaning scripts are part of the codebase at `scripts/cleanup_datasets/`.
* The main script is `cleanup_datasets.py`.

---
class: normal

```shell
source /srv/galaxy/venv/bin/activate
python ./scripts/cleanup_datasets/cleanup_datasets.py ./config/galaxy.ini -d 10 -5 -r ${GALAXY_ROOT} >> ./scripts/cleanup_datasets/purge_folders.log
```

flag	| short |	description
---|---|---
--days | -d	| number of days (60) to use as a cut off; do not act on objects updated more recently than this
--info_only |	-i |	only provide info about the requested action; no changes saved to database
--remove_from_disk | -r |	remove files from disk during operations
--force_retry |	-f |	performs the requested actions, but ignores whether it might have been done before. Useful when -r wasn't used, but should have been
--delete_userless_histories |	-1 |	delete userless histories and datasets
--purge_histories |	-2 |	purge deleted histories
--purge_datasets | -3 |	purge deleted datasets
--purge_libraries |	-4 |	purge deleted libraries
--purge_folders |	-5 |	purge deleted library folders
--delete_datasets |	-6 |	mark deletable datasets as deleted and purge associated dataset instances

Note that only a single numbered flag can be used at a time.
---
# Bash wrappers

To make it easy CRONing things some usecases of `cleanup_datasets.py` come pre-wrapped.

*The ordering is of significance.*

1. `delete_userless_histories.sh`
1. `purge_histories.sh`
1. `purge_libraries.sh`
1. `purge_folders.sh`
1. `purge_datasets.sh`

* `delete_datasets.sh` - If datasets should be removed before their outer container has been deleted.

---
# Exercise - Data cleanup

* Delete and purge the new history.
* Delete and purge the new library.
* Purge all deleted datasets in your data library.

---
# Additional cleaning scripts

* `admin_cleanup_datasets.py`
  * Mark datasets as deleted that are older than specified cutoff.
  * Has email templated notification (can also be used to just send info without deleting).
  * Can be restricted to a tool.
* `pgcleanup.py`
  * Cleans up datasets in Galaxy quickly.
  * Operates directly on DB (PostgreSQL >= 9.1).
* `rename_purged_datasets.py` and `remove_renamed_datasets_from_disk`
  * Rename a dataset file by appending `_purged` to the filename.
  * Remove the renamed datasets.

---
# Other maintenance

* `update_dataset_size.py`
  * Update `dataset.size` column in the DB.

---
# Data Providers

Provide efficient access to data for viz & API

Framework provides direct link to read the raw dataset
or use data providers to adapt it

In config, assert that visualization requires a given type of data providers

Data providers process data before sending to browser - slice, filter, reformat, ...

---
class: normal

# Object Store

obsolete:
```python
>>> fh = open( dataset.file_path, 'w' )
>>> fh.write( ‘foo’ )
>>> fh.close()
>>> fh = open( dataset.file_path, ‘r’ )
>>> fh.read()
```

proper:
```python
>>> update_from_file( dataset, file_name=‘foo.txt’ )
>>> get_data( dataset )
>>> get_data( dataset, start=42, count=4096 )
```

---
# Object Store Plugins

Direct:
- Disk
- Amazon S3/OpenStack Swift
- Experimental
  - Azure
  - iRODS
  - CloudBridge → S3, Swift, Azure, GCE

Nested:
- Hierarchical
- Distributed

---
# Object Store Exercise

[Expanding File Space - Exercise](https://github.com/galaxyproject/dagobah-training/blob/2018-oslo/sessions/19-storage/ex1-objectstore.md)
