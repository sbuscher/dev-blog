---
title: "Python in TSQL"
date: 2020-07-04T20:43:07-05:00
draft: true
---

# Execute Python in a SQL Script

## Prerequisites

1. SQL Server Management Studio (SSMS).
2. Client must have Python installed.
3. Client must have ArcGIS Desktop/ArcGIS Pro installed when using ArcPy. 

## Steps 

1. Create a python script and save to an accessible location (parcel-inventory.py).
```python
import arcpy

arcpy.env.workspace = r"\\gis\GISData\development\LandCadastre\Parcels\Corelogic\ParcelsDev.gdb"

for fc in arcpy.ListFeatureClasses():
    print(fc)
```
1. Open SSMS and create a new query (parcels.sql).
2. Change to command mode. In Object Explorer, right-click the server, and then click New Query, to open a new Database Engine Query Editor window. 
To turn SQLCMD scripting on by default, on the Tools menu select Options, expand Query Execution, and SQL Server, click the General page, and then check the By default open new queries in SQLCMD Mode box.
4. Add a line to execute the python script: 
```sql
!! C:\Python27\ArcGIS10.6\python.exe \\dbdeploy\DBDeploy\GISSoil\2018-06-14_EnableEnterpriseGeoDatabase\Int\parcel-inventory.py
```
5. Open a Command Prompt and execute. The *-S* option is the instance to connect. 
```
>sqlcmd -S SQL4116A -i parcels.sql