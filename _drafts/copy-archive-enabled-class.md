---
layout: post
title: Copy Archive-Enabled Class from Oracle User Schema to SDE Master Schema
author: ujr
---

This article explains how to move an archived table (or feature class)
from an Oracle user-schema geodatabase to the master SDE geodatabase,
*keeping the archived records*. It works only for non-versioned tables.

## Background

There can be [only one geodatabase per Oracle schema][multiple].
Usually, the schema is called “SDE” and the geodatabase in this
schema is the **master (SDE) geodatabase**. A geodatabase in
the schema of a user other than the SDE user is called a
**user-schema geodatabase**. It is *not* completely independent
from the master geodatabase. ArcGIS tends to become slower if
there are user-schema geodatabases in Oracle. Probably for this
reason they are no longer fashionable and have long been discouraged.
Since version 10.7 (Pro 2.3) you can no longer create user-schema
geodatabases.

## Archiving

If archiving is enabled for a non-versioned geodatabase table *FOO*,
ArcGIS adds fields `GDB_FROM_DATE`, `GDB_TO_DATE`, `GDB_ARCHIVE_OID`
and creates a view *FOO*_EVW without the additional fields and where
`GDB_TO_DATE` is 9999-12-31, that is, showing only records that exist
at present. ArcGIS Pro and ArcMap show this view, but by the name of
the actual table.

If archiving is enabled for a versioned geodatabase table *FOO*,
a new archive table named *FOO*_H is created. It has the same
schema as the original table, plus three fields `GDB_FROM_DATE`,
`GDB_TO_DATE`, `GDB_ARCHIVE_OID`. It is not shown in the catalog.
No view is created.

This document only covers the non-versioned case.

## The Problem

There is no known way to copy an archived table from within ArcGIS
without loosing the archive. Esri describes a workflow to
[move user-schema gdb to stand-alone gdb in Oracle][esrimove],
but it will loose the archive. An [old StackExchange thread][stex]
confirms this inconvenience and proposes backup/restore on the
database level as a workaround, which is no less inconvenient.

## The Solution

If you are willing to use SQL, there is a simple solution:

1. Create new empty feature class in the master SDE schema.
2. Enable archiving (will create the view and the extra fields).
3. Use SQL to copy rows, including the extra fields.

ArcGIS will recognize the result as a populated archived
feature class.

The solution can be scripted using ArcPy along these lines:

```Python
import arcpy

user_schema_connection = "UserConnection.sde"
master_sde_connection = "MasterConnection.sde"

user_fc_name = "MyFeatureClass"
sde_fc_name = "MyFeatureClass"  # could be different

arcpy.env.workspace = user_schema_connection
sref = arcpy.Describe(user_fc_name).spatialReference

# Create empty feature class in master schema:
arcpy.env.workspace = master_sde_connection
arcpy.management.CreateFeatureclass(master_sde_connection, sde_fc_name, ..., sref, ...)
arcpy.management.AddField(sde_fc_name, ...)
# etc... create all other fields

arcpy.management.EnableArchiving(sde_fc_name)

desc = arcpy.Describe(sde_fc_name)
assert desc.IsArchived
assert not desc.IsVersioned

# Up to here we have an empty non-versioned table with archiving enabled.
# Now we use SQL to copy all rows from the user schema table to the new
# empty master schema table. Be sure to include the special archive fields!

fields = ('OBJECTID', 'SHAPE', ..., 'GDB_FROM_DATE', 'GDB_TO_DATE', 'GDB_ARCHIVE_OID')
fields = ', '.join('"'+f+'"' for f in fields)
sql = 'INSERT INTO ' + sde_fc_name + ' ( ' + fields + ' ) SELECT ' + fields + ' FROM ' user_fc_name

sde = arcpy.ArcSDESQLExecute(sdeconn)
sde.startTransaction()
sde.execute(sql)
sde.commitTransaction()

# We now have a copy of the user-schema table, with archive state
# retained. The original user-schema table could now be deleted.
```

Note: I was trying to use the old feature class as a template for
the new feature class to spare all the `AddField()` calls. However,
this resulted in a duplicate OBJECTID field. Reason: unknown.

Obviously, you will have to adjust this snippet to your
situation, and you will need appropriate connection files.
Good luck.

[multiple]: https://desktop.arcgis.com/en/arcmap/10.3/manage-data/gdbs-in-oracle/multiple-geodatabases-oracle.htm
[esrimove]: https://pro.arcgis.com/en/pro-app/help/data/geodatabases/manage-oracle/migrate-user-schema-geodatabases.htm
[stex]: https://gis.stackexchange.com/questions/32354
