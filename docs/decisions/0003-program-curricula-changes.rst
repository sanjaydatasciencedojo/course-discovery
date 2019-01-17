Refactor the Curriculum model to relate to the Program model
============================================================

Status
======

Proposed (January 2019)

Context
=======

The structure of a Master's Degree program (as well as other programs,
e.g. MicroMasters) requires loosely that we support programs which may
consist of 0 or more Masters-level courses or 0 or more MicroMasters
programs as part of the program's curriculum.  The structure (or "curriculum")
of a program may change over time, and we should be able to track different
versions of curricula as they change.  Our discovery service should
be able to accurately model the relationship between courses, programs,
and Master's degrees and the curricula thereof.  Consider the following example.

Georgia Tech. offers a MS in Analytics.  Enrollees in this program may
specialize in three different tracks:

- Analytical Tools
- Business Analytics
- Computation Data Analytics

Each of these specializations may require enrollees to complete partially
overlapping, or completely distinct courses and MicroMasters programs to
fulfill the requirement of the specialized degree.

So, from this example, we see that a single degree may offer 1 or more
distinct specializations, and each specialization may require the completion
of different courses or MicroMasters (i.e. other, smaller programs).

Decision
========

To accurately model data as described above, the following changes to models
in the course_metadata application are proposed:

A Program may have multiple Curricula
-------------------------------------

We can use the Curriculum model to represent the fact that a Master's
Degree may contain one or more specializations through which an enrollee
may satisfy the requirements of a degree.  Furthermore, several of these
Curricula may be currently active, and several may be inactive.  For example,
consider extending the example above as follows:

- The "Analytical Tools" speciailization is retired.
- A new specialization, "Analytics in the Quantum Age" is introduced.

Our Curriculum models should be able to accurately represent this modification
to the MS Analytics Degree.  A Curriculum model for "Analytical Tools" should
capture the fact that the specialization is no longer active, and the date
on which it became inactive (after which, presumably, no new enrollee may
participate in that specialization).  A Curriculum model for the
"Analytics in the Quantum Age" should be created, with a field to indicate
that the curriculum/specialization is now active and the date on which it
became active.

A Degree may have changing deadlines and costs
----------------------------------------------

The amount of tuition charged, the dates of deadlines, etc. will naturally
change over the lifetime of a Master's Degree.  To support this, we will
update the models that capture this data to be versioned (so that we can
capture the change history of these dimensions) and have a field indicating
the current version of the dimensions.

Use only curricula to capture a Program's included courses
----------------------------------------------------------

There should be one, and preferably only one way to model the relationship
between courses and programs.  After taking actions to implement the decisions
outlined above, there will be two ways: via program curricula, and via
the ``courses`` and ``excluded_course_runs`` fields of the ``Program`` model.
These fields should be eschewed, and their data migrated into the `Curriculum``
and associated models.

Decision that we will make later
--------------------------------

We don't currently need to track ideas like prerequisites, required, or
elective courses.


Actions
=======

Relating Curricula to Programs
------------------------------

The current course_metadata design relates Curricula to Degrees via a 1-1 field.
We will change Curricula to relate to Programs via a Foreign Key field, so
that a program may consist of zero or many curricula.  Note that specializations
are just one example of a use-case supported by relating Curricula to
Programs in this fashion.

Action 1
^^^^^^^^

- Add ``program`` as a FK field in the ``Curriculum`` model, and point
  that FK at the ``Program`` model.
- Add the following fields to the ``Curriculum`` model to help
  support versioning and specialization:

  - ``name: str`` The name of this curriculum.
  - ``current: boolean`` Whether this curriculum is currently offered/available.
  - ``effective_at: datetime`` The time at which this curriculum is available.
  - ``retired_at: datetime`` The time at which this curriculum
    becomes unavailable.
  - ``version: str`` The version of this curriculum (to support cases where
    a version with the same name was previously offered, retired, and
    reintroduced, presumably with some changes to its composition).

Action 2
^^^^^^^^

There are existing ``Curriculum`` objects that are tied to existing
``Degree`` objects (roughly 10).

- We need to migrate existing curricula to reference the program objects
  associated with the current degrees referenced by the curricula.

Action 3
^^^^^^^^

Remove the existing ``degree`` 1-1 field from the ``Curriculum`` model.

Action 4
^^^^^^^^

``DegreeProgramCurriculum`` and ``DegreeCourseCurriculum`` are the bridge
models that link ``Curriculum`` objects to ``Program`` and ``Course`` models,
respectively.  These names should be changed as follows:

- Rename ``DegreeProgramCurriculum`` to ``CurriculumProgramMembership``.
- Rename ``DegreeCourseCurriculum`` to ``CurriculumCourseMembership``.

Action 5
^^^^^^^^

To support the addition/removal and versioning thereof of programs and courses
within curricula, we should add the following fields to both the
``CurriculumProgramMembership`` and ``CurriculumCourseMembership`` models:

- ``effective_at: datetime`` The time at which this program or course is
  available within the curriculum.
- ``retired_at: datetime`` The time at which this program or course is
  becomes unavailable within the curriculum.
- ``current: boolean`` Whether this program or course is currently available
  within the curriculum.
- ``version: str`` The version of this program or course's membership
  within the curriculum (to support cases where the course/program was
  previously offered, retired, and then reintroduced).

Action 6
^^^^^^^^

Support the versioning of degree costs and deadlines.  Add the following fields
to each of the ``DegreeDeadline`` and ``DegreeCost`` models.

- ``effective_at: datetime`` The time at which the deadline/cost was effective
  w.r.t. the degree.
- ``retired_at: datetime`` The time at which this deadline/cost was no longer
  effective w.r.t. the degree.
- ``current: boolean`` Whether this deadline/cost is currently effective
  w.r.t. the degree.

Using only curricula to capture a Program's included courses (Phase B)
----------------------------------------------------------------------

The following work can be completed only after the first phase of work described
above.  There are likely more actions than those described below to come, as
the clients of the course_metadata programs API may have expectations
broken if fields go away, data is re-organized, etc.

Action B.1
^^^^^^^^^^

- Migrate existing data from the ``Program.courses`` many-to-many relationship
  into a new ``Curriculum`` model related to the program (for each program
  that we have).
- Update whatever interface currently creates new programs to use ``Curriculum``
  objects instead of the ``courses`` field.
- Remove the ``Program.courses`` field.
- Optionally, add a ``Program.courses`` property that transforms course
  curriculum data into a list of course objects as currently expected of the
  Program ``courses`` interface.

Action B.2
^^^^^^^^^^

We can keep the ``Program.excluded_course_runs`` field (I think), but the
many-to-many relationship should be modified to use an explicitly defined
model to hold the bridge table data between ``Programs`` and ``CourseRuns``.
This can be achieved via the ``through`` parameter of Django's
``ManyToManyField``.
