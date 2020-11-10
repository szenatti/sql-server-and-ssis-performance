# Introduction

This documents provides guidance and start point to review performance issue between SQL Environments, such as On-premises and Azure IaaS servers.

> This document does not cover Azure SQL (PaaS).

# Azure - SQL Server IaaS

First of all, review if the SQL Server and IaaS Server are configured as per:

- Reference: https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices

# SSIS 

Compare SSIS configuration between two environments and get SSIS packages performance metrics (Duration).

- Reference: https://www.red-gate.com/simple-talk/sql/bi/comparing-ssis-catalog-contents-using-dbfit-framework/

## Compare between multiple SSIS Catalog environments
```sql
CREATE procedure [dbo].[GetSSISEnvironmentDetails] (@FolderName varchar(50))
  as
  Begin
  	SELECT
  	Convert(varchar(255),E.name) AS EnvironmentName
  	,   Convert(varchar(255),F.name) AS FolderName
  	,   Convert(varchar(255),P.name) AS ProjectName
  	, Convert(varchar(255),EV.name) as EnvironmentVariableName
  	, Convert(varchar(255),EV.Description) as EnvironmentVariableDescription
  	, Convert(varchar(255),EV.type) as EnvironmentVariableType
  	, Convert(varchar(255),EV.sensitive) as EnvironmentVariableSensitive
  	, Convert(varchar(255),EV.Value) as EnvironmentVariableValue
  	FROM
  		SSISDB.catalog.environments AS E
  		INNER JOIN
  			SSISDB.catalog.folders AS F
  			ON F.folder_id = E.folder_id
  		INNER JOIN 
  			SSISDB.catalog.projects AS P
  			ON P.folder_id = F.folder_id
  		INNER JOIN
  			SSISDB.catalog.environment_references AS ER
  			ON ER.project_id = P.project_id
  		INNER JOIN SSISDB.[catalog].[environment_variables] EV
  			on E.environment_id = EV.environment_id
  	Where F.name = @FolderName 
  	ORDER BY 
  		ER.reference_id;
  End	
  GO
```
## Identifying missing SSIS packages in the target servers
```sql
CREATE procedure [dbo].[GetSSISPackageDetails] (@FolderName varchar(50))
  as
  Begin
  	Select Convert(varchar(255),FOl.name) as FolderName,
  	Convert(varchar(255),PRJ.name) as ProjectName,
  	Convert(varchar(255),PRJ.description) as ProjectDescription,
  	Convert(varchar(255),PKG.name) as PackageName
  	from SSISDB.[catalog].[projects] PRJ
  			INNER JOIN SSISDB.catalog.folders FOL
  				ON PRJ.folder_id = FOl.folder_id
  			INNER JOIN SSISDB.catalog.[packages] PKG
  				ON PRJ.project_id = PKG.project_id
  	Where FOl.name = @FolderName 
  End
  GO
```
## Comparing Package Parameters
```sql
CREATE procedure [dbo].[GetPackageParameterDetails] (@ProjectName varchar(50))
  as
  Begin
  	Select 
  		Convert(varchar(255),object_name) as PackageName,
  		Convert(varchar(255),parameter_name) ParameterName,
  		Convert(varchar(255),data_type) as ParameterDataType,
  		Convert(varchar(255),default_value) as ParameterValue
  	from SSISDB.[catalog].[object_parameters] PARA
  			INNER JOIN SSISDB.[catalog].[projects] PRJ
  				on PARA.project_id = PRJ.project_id
  	where 
  		PRJ.name = @ProjectName
  		and object_type=30 --  Package Parameter
  End
```

## Project Parameter comparison
```sql
CREATE procedure [dbo].[GetProjectParameterDetails] (@ProjectName varchar(50))
  as
  Begin
  	Select 
  		Convert(varchar(255),object_name) as ProjectName,
  		Convert(varchar(255),parameter_name) ParameterName,
  		Convert(varchar(255),data_type) as ParameterDataType,
  		Convert(varchar(255),default_value) as ParameterValue
  	from SSISDB.[catalog].[object_parameters] PARA
  			INNER JOIN SSISDB.[catalog].[projects] PRJ
  				on PARA.project_id = PRJ.project_id
  	where 
  		PRJ.name = @ProjectName
  		and object_type=20 --  Project Parameter
  End
```

## Package Parameters referenced from environment variable
```sql
Create procedure [dbo].[GetPackageParameterReferenceByEnvironmentDetails] (@ProjectName varchar(50))
  as
  Begin
  	Select 
  		Convert(varchar(255),object_name) as PackageName,
  		Convert(varchar(255),parameter_name) ParameterName,
  		Case	
  			When PARA.value_type='V'
  				then 'Direct Parameter'
  			When PARA.value_type='R'
  				then 'Parameter Referenced from Environment'
  			else 'NA'
  		End as 'ParameterType',
  		Convert(varchar(255),referenced_variable_name) as EnvironmentVariableName,
  		Convert(varchar(255),data_type) as ParameterDataType,
  		Convert(varchar(255),default_value) as ParameterValue
  	from SSISDB.[catalog].[object_parameters] PARA
  			INNER JOIN SSISDB.[catalog].[projects] PRJ
  				on PARA.project_id = PRJ.project_id
  	where 
  		PRJ.name = @ProjectName
  		and object_type=30 --  Package Parameter
  End
```

## Job Duration

```sql
SELECT 
   t.last_run_date,
   t.last_run_time,
   t.last_run_duration RunDuration,
   t.last_run_outcome,
   t.*,
   tt.*
FROM msdb..sysjobsteps t 
   INNER JOIN msdb..sysjobs tt ON t.job_id = tt.job_id
WHERE tt.name = '?????' 
order by t.last_run_date desc, t.last_run_time desc
```

```sql
use SSISDB
go

declare @foldername nvarchar(260)
declare @projectname nvarchar(260)
declare @packagename nvarchar(260)

set @foldername = 'Folder1'
set @projectname = 'Project1'
set @packagename = '???????.dtsx'

DECLARE @ExecIds table(execution_id bigint);

insert into @ExecIds
SELECT execution_id
FROM catalog.executions
WHERE folder_name = @foldername
AND project_name = @projectname
AND package_name = @packagename
AND status = 7
go

SELECT es.execution_id, e.executable_name, ES.execution_duration
FROM catalog.executable_statistics es, catalog.executables e
WHERE
es.executable_id = e.executable_id AND
es.execution_id = e.execution_id AND
es.execution_id in (select * from @ExecIds)
ORDER BY e.executable_name,es.execution_duration DESC
go 

-- first compute the average and standard deviation for the duration spent by each task. In the query, we define a “slower than usual” task as one whose duration is greater than average + standard deviation (i.e. es.execution_duration > (AvgDuration.avg_duration + AvgDuration.stddev))

With AverageExecDudration As (
select executable_name, avg(es.execution_duration) as avg_duration,STDEV(es.execution_duration) as stddev
from catalog.executable_statistics es, catalog.executables e
where
es.executable_id = e.executable_id AND
es.execution_id = e.execution_id AND
es.execution_id in (select * from @ExecIds)
group by e.executable_name
)
select es.execution_id, e.executable_name, ES.execution_duration, AvgDuration.avg_duration, AvgDuration.stddev
from catalog.executable_statistics es, catalog.executables e,
AverageExecDudration AvgDuration
where
es.executable_id = e.executable_id AND
es.execution_id = e.execution_id AND
es.execution_id in (select * from @ExecIds) AND
e.executable_name = AvgDuration.executable_name AND
es.execution_duration > (AvgDuration.avg_duration + AvgDuration.stddev)
order by es.execution_duration desc

-- Zoom into this specific execution, and identify the time spent in each phase of the data flow task

declare @probExec bigint
set @probExec = 188

-- Identify the component’s total and active time

select package_name, task_name, subcomponent_name, execution_path,
SUM(DATEDIFF(ms,start_time,end_time)) as active_time,
DATEDIFF(ms,min(start_time), max(end_time)) as total_time
from catalog.execution_component_phases
where execution_id = @probExec
group by package_name, task_name, subcomponent_name, execution_path
order by active_time desc

declare @component_name nvarchar(1024)
set @component_name = '?????????'

-- See the breakdown of the component by phases

select package_name, task_name, subcomponent_name, execution_path,phase,
SUM(DATEDIFF(ms,start_time,end_time)) as active_time,
DATEDIFF(ms,min(start_time), max(end_time)) as total_time
from catalog.execution_component_phases
where execution_id = @probExec AND subcomponent_name = @component_name
group by package_name, task_name, subcomponent_name, execution_path, phase
order by active_time desc

```

# SQL Server Database

Check tables size, index fragmentation and SQL Agent Jobs history between environments:

- (OnPrem and Azure) Get Tables Size

```sql
SELECT
s.Name AS SchemaName,
t.Name AS TableName,
p.rows AS RowCounts,
CAST(ROUND((SUM(a.used_pages) / 128.00), 2) AS NUMERIC(36, 2)) AS Used_MB,
CAST(ROUND((SUM(a.total_pages) - SUM(a.used_pages)) / 128.00, 2) AS NUMERIC(36, 2)) AS Unused_MB,
CAST(ROUND((SUM(a.total_pages) / 128.00), 2) AS NUMERIC(36, 2)) AS Total_MB
FROM sys.tables t
INNER JOIN sys.indexes i ON t.OBJECT_ID = i.object_id
INNER JOIN sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
GROUP BY t.Name, s.Name, p.Rows
ORDER BY s.Name, t.Name
```

- (OnPrem and Azure) Get Tables Index Fragmentation status (Source and Target DBs)

```sql
SELECT S.name as 'Schema',
T.name as 'Table',
I.name as 'Index',
DDIPS.avg_fragmentation_in_percent,
DDIPS.page_count
FROM sys.dm_db_index_physical_stats (DB_ID(), NULL, NULL, NULL, NULL) AS DDIPS
INNER JOIN sys.tables T on T.object_id = DDIPS.object_id
INNER JOIN sys.schemas S on T.schema_id = S.schema_id
INNER JOIN sys.indexes I ON I.object_id = DDIPS.object_id
AND DDIPS.index_id = I.index_id
WHERE DDIPS.database_id = DB_ID()
and I.name is not null
```

- (OnPrem and Azure) SQL Agent Job History

```sql
USE msdb  
GO  
  
EXEC dbo.sp_help_jobhistory  
    @start_run_date > 20201020,
    @mode = N'FULL' ;  
GO  
```

- Check list:
    * (OnPrem and Azure) Compare if SSIS packages are running on 32bit or 64 bits, look at SSIS Package and SQL Agent Jobs
    * (OnPrem and Azure) Compare SSISDB Properties, such as disk location, size, configuration, recovery mode...
    * (OnPrem and Azure) Extract SSIS Log/Audit table from ETL database
    * (Azure) In Azure Portal check Server metrics (Memory, CPU, Disk...)
        * https://docs.microsoft.com/en-us/azure/monitoring/infrastructure-health/vmhealth-windows/winserver-memory-pctcommitted
		* Enable Azure Monitor for Azure VM
    * (OnPrem and Azure) Export SSIS Packages from both environments and compare
    * (Azure) EventViewer, Windows Update, Scheduler, SQL Logs
    * (OnPrem and Azure) Windows Version, Virtual Memory***
        * https://serverfault.com/questions/930424/windows-server-2016-pagefile-does-not-increase
