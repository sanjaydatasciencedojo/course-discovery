Publisher Draft Model Decisions
===============================

Status
------

Accepted

Context
-------

As we develop a new frontend for Publisher, we need to support handling a Draft
mode of Courses and Course Runs (prior to OFAC approval) that will show
potential changes before a go-live action.

- We need to be able to stage, save, and present changes without modifying the
  live course

- We want this content to be served from our REST endpoints alongside the
  Courses or Course Runs

- We want to allow for any fields on the form pages to be draft-able even if
  they are not part of the course_run or course tables

- We want to easily be able to tell which Courses or Course Runs have active
  drafts

Decision
--------

Add an additional row for each currently existing row within the tables that
need draft states. Creates will create a new "draft" row that will be the
version that is modified.

Add an additional column representing the differing state for the forms we need
history for. This will prevent us from increasing our table size, as well as
prevent us from having to modify our APIs outside of a specific query param
for unpublished data. The default manager will need to query against the
"published" states. Our schema will also be up to date via default migrations,
and any consumer of the API will be able to act directly on reading from either
publisher or draft rows.

On a successful "publish", the data of the "draft" row will be written to the
"published" row. The draft row does not need to be updated outside of this.
Consequences
------------

By choosing this solution over alternatives we miss out on a few things, as well
as open ourselves up to certain risks.

- Duplicating data across the large tables we have will be a non trivial task

- Indexes will need to be updated accordingly to accommodate the new access
  pattern we will be querying on

- Base object manager classes will need to be overridden

- Primary/Composite Primary keys will need to consider the draft/publish state

- Historical changes will not exist by default, it will be difficult to rollback
  and difficult to restore revisions

- Relations for many to many, or one to many may not be the easiest to propagate
  to the live "published" versions (update vs drop/create)
