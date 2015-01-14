..
 This work is licensed under a Creative Commons Attribution 4.0 International
 License.

 http://creativecommons.org/licenses/by/4.0/

========
Task API
========

Task vendor extension API makes the Neutron and MidoNet integration tasks
available to the users to provide better visibility of the state of the
Neutron API requests that require updates in both Neutron and MidoNet data
storage.


Problem Description
===================

When MidoNet and Neutron are integrated, data often need to be stored in both
Neutron DB and MidoNet cluster.  Occasionally, data could become out of sync
between them when the data is committed in Neutron DB but not in the MidoNet
cluster because of a bug in the code or some system outage.  Detecting and
fixing this kind of error is costly for those involved.  There is no way for
the operators to check whether the Neutron API request was successfully handled
by the MidoNet cluster.


Proposed Change
===============

All changes to Neutron DB are stored durably in a SQL table called
midonet_tasks in the Neutron DB.  When a request comes to Neutron API to modify
a resource, the Neutron MidoNet plugin updates the corresponding Neutron
database table for that resource, and then saves this information, including
 the resource type and the action taken, in the midonet_tasks table.    These
updates are executed under one transaction, so a failure in one of them results
in a rollback.  This table provides a reliable message queue between Neutron
and the cluster.  The cluster would periodically poll this table for
unprocessed changes, and keeps track of the last processed task ID.  In each
poll interval, the cluster fetches all tasks with ID greater than the last
processed task ID.


Data Model Impact
-----------------

The following new tables are introduced.

**midonet_task_types**

+-------------------+---------+-----------------------------------------------+
| Name              | Type    | Description                                   |
+===================+=========+===============================================+
| id                | TinyInt | Unique identifier of task types               |
+-------------------+---------+-----------------------------------------------+
| name              | String  | Name of the type                              |
+-------------------+---------+-----------------------------------------------+

Initial supported types are:

 * create
 * delete
 * update

More types will be added, such as 'flush' which tells MidoNet cluster to flush
out the data.


**midonet_data_types**

+-------------------+---------+-----------------------------------------------+
| Name              | Type    | Description                                   |
+===================+=========+===============================================+
| id                | TinyInt | Unique identifier of data types               |
+-------------------+---------+-----------------------------------------------+
| name              | String  | Name of the data type                         |
+-------------------+---------+-----------------------------------------------+

Initial supported types are:

 * network
 * subnet
 * router
 * port
 * floating_ip
 * security_group
 * security_group_rule
 * router_interface


**midonet_tasks**

+----------------+------------+-----------------------------------------------+
| Name           | Type       | Description                                   |
+================+============+===============================================+
| id             | Int        | Auto-incremented ID.  This can serve as the   |
|                |            | timestamp of the entry, describing the        |
|                |            | ordering.  It provides decoupling from the    |
|                |            | system clock on the host.  This is also the   |
|                |            | primary key of the table.                     |
+----------------+------------+-----------------------------------------------+
| type_id        | TinyInt    | ID of the task defined in midonet_task_types  |
|                |            | table                                         |
+----------------+------------+-----------------------------------------------+
| data_type_id   | TinyInt    | ID of the data type defined in                |
|                |            | midonet_data_types table                      |
+----------------+------------+-----------------------------------------------+
| data           | MediumText | The actual data corresponding to the Neutron  |
|                |            | model, created or updated in JSON format.  It |
|                |            | is in JSON to provide better debuggability.   |
+----------------+------------+-----------------------------------------------+
| resource_id    | Char(36)   | Unique identifier of the Neutron resource ID  |
+----------------+------------+-----------------------------------------------+
| transaction_id | Char(40)   | ID that identifies a set of updates that      |
|                |            | occurred in a single Neutron transaction      |
+----------------+------------+-----------------------------------------------+
| created_at     | Timestamp  | Time that this entry was added, up to seconds |
+----------------+------------+-----------------------------------------------+


Plugin Impact
-------------

The midonet_tasks table is created with the alembic migration tool used in
Neutron.  It follows the Neutron model for ORM and migrations.

task data access module exposes the following methods:

::






API Impact
----------

A GET to ``/midonet/tasks`` returns the list of unprocessed tasks.
A GET to ``/midonet/tasks/<task_id>`` returns the detailed information about a
specific task.


Client Impact
-------------

None

