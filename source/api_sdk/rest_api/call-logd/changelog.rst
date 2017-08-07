.. _call_logd_changelog:

*********************************
wazo-call-logd REST API changelog
*********************************

17.11
=====

* New endpoint for getting a single call log:

  * ``GET /cdr/<cdr_id>``


17.09
=====

* All CDR endpoints can return CSV data, provided a header ``Accept: text/csv; charset=utf-8``.
* The default value of query string ``direction`` has been changed from ``asc`` to ``desc``.


17.08
=====

* ``GET /users/me/cdr`` and ``GET /users/<user_uuid>/cdr`` accept new query string arguments:

  * ``call_direction``
  * ``number``

* ``GET /cdr`` accepts new query string arguments:

  * ``call_direction``
  * ``number``
  * ``user_uuid``
  * ``tags``

* ``GET /cdr`` has new attribute:

  * ``id``
  * ``answer``
  * ``call_direction``
  * ``tags``


17.07
=====

* New endpoints for listing call logs:

  * ``GET /users/<user_uuid>/cdr``
  * ``GET /users/me/cdr``

17.06
=====

* Call logs objects now have a new attribute ``end``
* ``GET /cdr`` has new parameters:

  * ``from``
  * ``until``
  * ``order``
  * ``direction``
  * ``limit``
  * ``offset``

* ``GET /cdr`` has new attributes:

  * ``total``
  * ``filtered``

17.05
=====

* New endpoint for listing call logs:

  * ``GET /cdr``

17.04
=====

* Creation of the HTTP daemon
