![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Produce STIG SQL Scan Report using basic SQL
**Post Date: July 27, 2021**

## Contents    
- [About Process](##About-Process)  
- [SQL Logic 1](#SQL-Logic-1)  
- [SQL Logic 2](#SQL-Logic-2)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>Produce STIG SQL Scan Report using basic SQL.</p>

![Find Database Details With SQL]( https://mikesdatawork.files.wordpress.com/2021/07/image001-1.png "Produce STIG SQL Scan Results Using Only SQL")
 
<p>Capture Scanned SQL Servers, and percentage categories.</p>

![Find Database Details With SQL]( https://mikesdatawork.files.wordpress.com/2021/07/image002.png "Capture Scanned SQL Servers, and percentage categories.")
 
<p>Import and reference previous scans.</p>

![Find Database Details With SQL]( https://mikesdatawork.files.wordpress.com/2021/07/image003.png "Import and reference previous scans.")
 
<p>This particular process can be done in 2 ways.
1. Importing exported .XML scans directly into tables and extrapolating contents.
2. Performing direct queries against the source tables and patching together the results while referencing hard file exports.</p>

<p>Importing formerly exported .XML scans.</p>

## SQL-Logic-1
```SQL
use [DBA];
set nocount on
set quoted_identifier on

-- create sourcepath
declare @source_path	varchar(555)
set		@source_path	= ('D:\DBProtect-XML-Exports-SQL\')

-- get XML file list from export folder
declare @file_path	varchar(555)
set		@file_path	= ('dir D:\DBPROTECT-XML-EXPORTS-SQL\\*.*')

if object_id('tempdb..#xml_source_files') is not null
	drop table	#xml_source_files
create table	#xml_source_files ([file_properties] varchar(max))
insert into		#xml_source_files exec master..xp_cmdshell @file_path
delete from		#xml_source_files where [file_properties] not like '%/%' 
				or [file_properties] like '%<%'
				or [file_properties] is null

if object_id('tempdb..#XML_Files_SQL') is not null
	drop table  #XML_Files_SQL
create table	#XML_Files_SQL ([date] datetime, [size] varchar(50), [file_name] varchar(max))
insert into		#XML_Files_SQL 
select 
	'date'		= convert(datetime, ltrim(substring ([file_properties], 1, 20)))
,	'size'		= ltrim(substring ([file_properties], 21, 18))
,	'file_name'	= ltrim(substring ([file_properties], 40, 1000))
from 
	#xml_source_files

-- set timstamp for table name
declare @timestamp	varchar(120)
set		@timestamp = (select top 1 replace(replace(cast(convert(varchar, [date], 120)  as varchar), ':','-'),' ','-') from #xml_files_sql order by [date] desc)

-- get latest XML export file name and create import destination
declare	@xml_file_name		varchar(555) 
declare	@import_destination	varchar(555)
declare	@source_id			varchar(50)
set		@xml_file_name		= (select top 1 [file_name] from #XML_Files_SQL where [file_name] not in (select [xml_file_source] from [STIG_Source_Files]) order by [date] desc)
set		@import_destination = (@timestamp  + '-' + replace(replace(replace(substring(@xml_file_name, patindex('%XML -%', @xml_file_name), patindex('% - %',@xml_file_name)),'-','_'),'_','-'),' ', ''))
set		@source_id			= (select right(@import_destination, 6))

select	'xml_file_name'	= @xml_file_name
select	'table_name'	= @import_destination
select	'source_id'		= @source_id

-- create file, table, and XML history
if object_id('dbo.STIG_Source_Files', 'U') is not null
	drop table	dbo.[STIG_Source_Files]
create table	[STIG_Source_Files] ([line_id] int identity(1,1), [source_id] varchar(6), [time] datetime default getdate(), [import_table_name] varchar(700), [xml_file_source] varchar(700), [XML_Data] XML)

if object_id('tempdb..##XMLVarcharImport') is not null
	drop table	##XMLVarcharImport
create table	##XMLVarcharImport ([xml_data] varchar(max))

-- load XML data as varchar
declare	@load_xml	varchar(max)
set		@load_xml	= ''
select	@load_xml	= @load_xml + 
'declare @XMLVarchar	varchar(max) set @XMLVarchar = (SELECT * FROM OPENROWSET (BULK ''' + @source_path + @xml_file_name + ''', single_blob) as [XMLVarchar])
insert into	 ##XMLVarcharImport
select	(@XMLVarchar)
'
exec (@load_xml) 

-- remove cognos prefix & reset simple <metadata> prefix
update ##XMLVarcharImport
set [xml_data] = replace([xml_data], substring([xml_data],0,patindex('%<metadata>%',[xml_data])), '')
update ##XMLVarcharImport
set [xml_data] =  replace([xml_data], '<metadata>', '<dataset><metadata>')

-- set proper XML
declare @XMLData XML
set		@XMLData = (select cast([xml_data] as XML) from ##XMLVarcharImport)

-- update stig_source_files history table
insert into [STIG_Source_Files] ([source_id], [xml_file_source], [import_table_name], [xml_data])
select @source_id, @xml_file_name, @import_destination, @XMLData

-- confirm import table values
declare @import_table	varchar(555)
set		@import_table	= (select [import_table_name] from [STIG_Source_Files] where [source_id] = right(@import_destination, 6))

-- capture all metadata stig column list
declare	@meta_string		varchar(max)
set		@meta_string		= (select substring([xml_data], charindex('<metadata>', [xml_data]), charindex('</metadata>',[xml_data]) - charindex('<metadata>', [xml_data]) + len('</metadata>')) from ##XMLVarcharImport)

-- create stig_columns table
if object_id('dbo.STIG_Columns', 'U') is not null
	drop table	[STIG_Columns]
create table	[STIG_Columns] ([line_id] int identity(1,1), [source_id] varchar(6), [ordinal] int, [column_name] varchar(255))
insert into		[STIG_Columns] ([source_id], [column_name])
select @source_id, value from string_split(@meta_string,'"') where value not like '%[-!#%&+,./:;<=>@`{|}~"()*\\\_\^\?\[\]\'']%' {ESCAPE '\'} and value not like '%[0-9]%'

declare @ordinal int
set		@ordinal = 0
update	[STIG_Columns] set @ordinal = [ordinal] = @ordinal + 1 where [source_id] = @source_id

-- create import table
declare @maxcolumn	int
set		@maxcolumn  = (select max([ordinal])from [stig_columns] where [source_id] = @source_id)
select	(@maxcolumn)


declare @build_import_table	varchar(max)
set		@build_import_table	= (select 'if object_id(''dbo.[' + @import_destination + ']'') is not null drop table [' + @import_destination + '] create table [' + @import_destination + '] ([Line_ID] int identity (1,1), ')
 
declare @build_import_columns varchar(max)
set		@build_import_columns = ''
select	@build_import_columns = @build_import_columns + 
case when [line_id] = @maxcolumn then '[' + replace([column_name], ' ', '_') + '] varchar(max))' else '[' + replace([column_name], ' ', '_') + '] varchar(max),' end 
from	[STIG_Columns] order by [line_id]

declare	@build_import	varchar(max)
set		@build_import	= (select (@build_import_table + @build_import_columns))
exec	(@build_import) --for xml path (''), type

-- apply souce_id suffix to stig column_names
select [column_name] from [STIG_Columns] where [source_id] = right(@import_destination, 6)

declare @main_select	varchar(555)
set		@main_select	= ('select * from [' + @import_table + ']')

-- build xml columns
declare @xml_columns	varchar(max)
set		@xml_columns	= ''
select  @xml_columns = @xml_columns +
	case 
		when [ordinal] = @maxcolumn then '[' + replace([column_name], ' ', '_') + ']' 
		else '[' + replace([column_name], ' ', '_') + '],' 
	end
from [stig_columns] where [source_id] = @source_id order by [ordinal]

-- capture xml values
declare @xml_values	varchar(max)
set		@xml_values	= ''
select	@xml_values = @xml_values + 
	case 
		when [ordinal] = @maxcolumn then 'XMLDoc.value(''(value)[' + cast(sc.[ordinal] as varchar) +']'',''varchar(max)'')' 
		else 'XMLDoc.value(''(value)[' + cast(sc.[ordinal] as varchar) +']'',''varchar(max)''),' 
	end
from [stig_columns] sc where [source_id] = @source_id order by [ordinal] asc

exec	(@main_select)

-- extrapolate XML from XMLDoc and insert into standard SQL table
declare @insert_xml_data		varchar(max)
set		@insert_xml_data		= ''
select	@insert_xml_data		= @insert_xml_data + 
'declare @xml xml set @xml = (select [xml_data] from [stig_source_files] where [source_id] = ''' + @source_id + ''') 
insert into [' + @import_destination + '] (' + @xml_columns + ') 
select ' + @xml_values + ' from @xml.nodes(''dataset/data/row'') AS XMLTable(XMLDoc)
'
exec (@insert_xml_data) --for xml path(''), type

exec (@main_select)
```
 
<p>Performing direct queries against the source tables.</p>
## SQL-Logic-2
```SQL
use master 
set nocount on
go 
----------------------------------------------------------------------------------------
-- configure database mail and create SQLAlert profile with SMTP server mailer.MyCompany.com

if (select sum(cast([value_in_use] as int)) from master.sys.configurations where [configuration_id] in ('518', '16386', '16388')) <> 3
	begin
		exec master..sp_configure	'show advanced options',		1 reconfigure
		exec master..sp_configure	'Ole Automation Procedures',	1 reconfigure with override
		exec master..sp_configure	'Database Mail XPs',			1 reconfigure with override
	end
 
IF NOT EXISTS(SELECT * FROM msdb.dbo.sysmail_profile WHERE  name = 'SQLAlerts')  
  BEGIN 
    EXECUTE msdb.dbo.sysmail_add_profile_sp 
      @profile_name = 'SQLAlerts', 
      @description  = 'SQLDatabaseMail'; 
  END
   
  IF NOT EXISTS(SELECT * FROM msdb.dbo.sysmail_account WHERE  name = 'SQLAlerts@MyCompany.com') 
  BEGIN 
    EXECUTE msdb.dbo.sysmail_add_account_sp 
    @account_name            = 'SQLAlerts@MyCompany.com', 
    @email_address           = 'SQLAlerts@MyCompany.com', 
    @mailserver_name         = 'mailer.mycompany.com', 
    @mailserver_type         = 'SMTP', 
    @port                    = '25', 
    @use_default_credentials =  0 , 
    @enable_ssl              =  0 ; 
  END 
   
IF NOT EXISTS(SELECT * 
              FROM msdb.dbo.sysmail_profileaccount pa 
                INNER JOIN msdb.dbo.sysmail_profile p ON pa.profile_id = p.profile_id 
                INNER JOIN msdb.dbo.sysmail_account a ON pa.account_id = a.account_id   
              WHERE p.name = 'SQLAlerts' 
                AND a.name = 'SQLAlerts@MyCompany.com')  
  BEGIN 
    EXECUTE msdb.dbo.sysmail_add_profileaccount_sp 
      @profile_name = 'SQLAlerts', 
      @account_name = 'SQLAlerts@MyCompany.com', 
      @sequence_number = 1 ; 
  END

----------------------------------------------------------------------------------------
-- create variables to capture server, instance and port info

declare 	@instance_name_basic		varchar(255)
declare 	@server_name_instance_name	varchar(255)
declare 	@server_time_zone			varchar(255)
set			@instance_name_basic		= (select cast(serverproperty('servername') as varchar(255)))
set			@server_name_instance_name	= (select upper(@@servername))
declare		@statements					nvarchar(max)
declare		@original					int
declare		@iid						int
declare		@maxid						int
declare		@domain						nvarchar(255)
declare		@ports      table ([PortType] nvarchar(180), Port int)
exec		master..xp_regread 'HKEY_LOCAL_MACHINE', 'SYSTEM\CurrentControlSet\services\Tcpip\Parameters', N'Domain',@Domain output

----------------------------------------------------------------------------------------
-- create and capture all other instance information found on the server
declare		@instances  table
(
    [InstanceID]    int identity(1, 1) not null primary key
,   [InstanceName]  nvarchar(180)
,   [InstanceFolder]nvarchar(50)
,   [StaticPort]    int
,   [DynamicPort]   int
);

declare		@xp_msver_platform  table ([id] int,[name] varchar(180),[InternalValue] varchar(50), [Charactervalue] varchar (50))
declare		@platform       varchar(10)
insert into @xp_msver_platform exec master..xp_msver platform
select  @platform = (select 1 from @xp_msver_platform where [Charactervalue] like '%86%')
If  @platform is NULL
Begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end
else
Begin
    insert into @instances ([InstanceName], [InstanceFolder])
    exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
end 
 
declare		@Key table ([KeyValue] int)
insert into @Key
exec master..xp_regread'HKEY_LOCAL_MACHINE', N'SOFTWARE\Wow6432Node\Microsoft\Microsoft SQL Server\Instance Names\SQL';
select  @original= [KeyValue] from @Key
If      @original=1
        insert into @instances ([InstanceName], [InstanceFolder])
        exec master..xp_regenumvalues N'HKEY_LOCAL_MACHINE', N'SOFTWARE\Wow6432Node\Microsoft\Microsoft SQL Server\Instance Names\SQL';
 
select  @maxid = MAX([InstanceID]), @iid = 1
from    @instances
while   @iid <= @maxid
    Begin
        Delete from @ports
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Microsoft\\Microsoft SQL Server\' 
        + [InstanceFolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPDynamicPorts'''
        from        @instances  Where [InstanceID] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Microsoft\\Microsoft SQL Server\' 
        + [InstanceFolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPPort'''
        from        @instances  Where [InstanceID] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Wow6432Node\Microsoft\\Microsoft SQL Server\' 
        + [InstanceFolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPDynamicPorts'''
        from        @instances  Where [InstanceID] = @iid
        insert into @ports  exec sp_executesql @statements
        select      @statements = 'exec xp_instance_regread N''HKEY_LOCAL_MACHINE'', N''SOFTWARE\Wow6432Node\Microsoft\\Microsoft SQL Server\' 
        + [InstanceFolder] + '\MSSQLServer\SuperSocketNetLib\Tcp\IPAll'', N''TCPPort'''
        from        @instances  Where [InstanceID] = @iid
        insert into @ports  exec sp_executesql @statements
        Update SI   Set [StaticPort] = p.Port, [DynamicPort] = dp.Port from @instances si
        Inner Join @ports DP On dp.[PortType] = 'TCPDynamicPorts' Join @ports p on p.[PortType] = 'TCPPort'
        Where [InstanceID] = @iid;
        Set @iid = @iid + 1
    end
				 

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- create and capture XML file source information

use [DBA];
set nocount on

-- source path share for xml files:  \\MyServerName\report_store
-- get XML file list from export folder share
declare @file_path	varchar(555)
set		@file_path	= ('dir \\MyServerName\report_store\\*.*')

if object_id('tempdb..#xml_source_files') is not null
	drop table	#xml_source_files
create table	#xml_source_files ([file_properties] varchar(max))
insert into		#xml_source_files exec master..xp_cmdshell @file_path
delete from		#xml_source_files where [file_properties] not like '%/%' 
				or [file_properties] like '%<%'
				or [file_properties] is null

if object_id('tempdb..#DBProtect_XML_Files') is not null
	drop table  #DBProtect_XML_Files
create table	#DBProtect_XML_Files ([date] datetime, [size] varchar(50), [file_name] varchar(max))
insert into		#DBProtect_XML_Files 
select 
	'date'		= convert(datetime, ltrim(substring ([file_properties], 1, 20)))
,	'size'		= ltrim(substring ([file_properties], 21, 18))
,	'file_name'	= ltrim(substring ([file_properties], 40, 1000))
from 
	#xml_source_files

--select * from #DBProtect_XML_Files where [file_name] like '%.xml' order by [date] desc

-- get job and report information from DBProtect's [AppDetective]..[AnalyticsStoredReports] table
if object_id('tempdb..#DBProtect_Stored_Reports') is not null
	drop table	#DBProtect_Stored_Reports
create table	#DBProtect_Stored_Reports
(
	[endtime]			datetime
,	[reportname]		varchar(255)
,	[jobname]			varchar(255)
,	[uuid]				varchar(255)
,	[customlabel]		varchar(255)
,	[reportfilepath]	varchar(255)
,	[Format]			varchar(255)
,	[status]			varchar(255)
)
insert into		#DBProtect_Stored_Reports
select
	[endtime]
,	[reportname]
,	[jobname]
,	[uuid]
,	[customlabel]
,	[reportfilepath]
,	[Format]
,	[status]
from
	[AppDetective]..[AnalyticsStoredReports]
where
	[jobname] like '%mssql%'
	and [format] = 'xml'
	and [reportfilepath] is not null
order by
	[endtime] desc

select top 5 
	[endtime]
,	[reportname]
,	[jobname]
,	[customlabel]
,	[uuid]
from 
	#DBProtect_Stored_Reports order by [endtime] desc

declare @dbprotect_job_name		varchar(555)
declare @dbprotect_report_name	varchar(555)
declare	@dbprotect_custom_label	varchar(555)
set		@dbprotect_job_name		= (select top 1  [jobname]		from #DBProtect_Stored_Reports order by [endtime] desc)
set		@dbprotect_report_name	= (select top 1  [reportname]	from #DBProtect_Stored_Reports order by [endtime] desc)
set		@dbprotect_custom_label = (select top 1  [customlabel]	from #DBProtect_Stored_Reports order by [endtime] desc)

--select [file_name] from #DBProtect_XML_Files where [file_name] in (select [uuid] + '.xml' from #DBProtect_Stored_Reports)

-- create source_id
declare	@source_id	varchar(8)
set		@source_id	= (select top 1 left([uuid], 8) from #DBProtect_Stored_Reports order by [endtime] desc )
--select	@source_id

-- create table name (Import Destination)
-- create import destination
declare	@import_destination varchar(555)
set		@import_destination	= 
(select 
	top 1
	replace(replace(convert(nvarchar(max), [endtime], 20), ' ', '-'), ':', '-') + '-'
+	left([uuid], 8) + '-'
+	replace(replace(replace(replace(replace([jobname], '_', '-'), ' ', '-'), ')', ''), '(', ''), '+', '-')
from 
	#DBProtect_Stored_Reports
order by 
	[endtime] desc)

--select	(@import_destination)


-- get 5 most recent files
--select top 5 * from #DBProtect_Stored_Reports order by [endtime] desc

declare	@xml_file_name	varchar(555)
declare @time_created	varchar(555)
declare @source_path	varchar(555)
set		@xml_file_name	= (select top 1 [uuid] + '.xml' from #DBProtect_Stored_Reports order by [endtime] desc)
set		@time_created	= (select top 1 left([endtime], 25) + ')' from #DBProtect_Stored_Reports order by [endtime] desc)
set		@source_path	= ('\\MyServerName\report_store\' + @xml_file_name)

--select	'source_path'	= 'XML File Source:  '	+ @source_path
--select	'xml_file_name'	= 'XML File Name:  '	+ @xml_file_name
--select	'table_name'	= 'Imported to:  ['		+ @import_destination + ']'

-- capture job name, report, label details
declare @job_details table ([job_name] varchar(555), [report_name] varchar(555), [custom_label] varchar(555))
insert into @job_details select	@dbprotect_job_name, @dbprotect_report_name, @dbprotect_custom_label


----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------


----------------------------------------------------------------------------------------
-- create and capture SQL Version info.

declare @version_info	table
(
	[Server_Name]		varchar(255)
,	[SQL_Instance]		varchar(255)
,	[SQL_Version]		varchar(255)
,	[SQL_Build]			varchar(255)
,	[SQL_Edition]		varchar(255)
)

insert into @version_info
select
	'ServerName'		= cast(serverproperty('machinename') as varchar)
,	'SQLInstance'		= cast(upper(@@servername) as varchar)
,	'SQLVersion'		=	
case 
when left(cast(serverproperty('productversion') as varchar), 4) = '14.0' then 'SQL 2017 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '13.0' then 'SQL 2016 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '12.0' then 'SQL 2014 ' + cast(serverproperty('productlevel') as varchar)
when left(cast(serverproperty('productversion') as varchar), 4) = '11.0' then 'SQL 2012 ' + cast(serverproperty('productlevel') as varchar)
end
,	'SQLBuild'			= cast(serverproperty('productversion') as nvarchar(25))
,	'SQLEdition'		= cast(serverproperty('edition') as varchar)

 

----------------------------------------------------------------------------------------
-- create HTML\xml variables

declare @HTML_BODY				nvarchar(max)
declare @xml_instance_info		nvarchar(max)
declare @xml_version_info		nvarchar(max)
declare	@XML_FILE_FOUND			nvarchar(max)
declare	@XML_SCANNED_ASSETS		nvarchar(max)
declare	@XML_PREVIOUS_SCANS		nvarchar(max)
declare	@XML_PERCENT_CATEGORIES nvarchar(max)
declare	@XML_PERCENT_RISK		nvarchar(max)
 
----------------------------------------------------------------------------------------
-- set table framework for XML file info

declare @xml_file_info  table ([source_path] varchar(555), [xml_file_name] varchar(555), [import_destination] varchar(555))
insert into @xml_file_info
select	@source_path, @xml_file_name, @import_destination

--select * from @xml_file_info


set @XML_FILE_FOUND = 
	cast(
		(select
			[source_path]			as 'td'
		,   ''
		,   [xml_file_name]			as 'td'
		,   ''
		,   [import_destination]	as 'td'
		,   ''
		from  @xml_file_info 
		--order by [database] asc
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)

----------------------------------------------------------------------------------------
-- set table framework for XML assets scanned

declare @new_import_destination_table	varchar(255)
set		@new_import_destination_table	= (select top 1 [name] from sysobjects where [xtype] = 'u' order by [crdate] desc)
declare	@query_assets					varchar(555)
set		@query_assets					= ('select replace(upper([asset]), ''.MyCompany.com'', ''''), count(*) from [' + @new_import_destination_table + '] group by [asset] order by [asset] asc' )

declare @scanned_assets	table ([server_name] varchar(555), [findings] int)
insert into @scanned_assets ([server_name], [findings])
exec	(@query_assets)

--select * from @scanned_assets order by [server_name] asc


set @XML_SCANNED_ASSETS = 
	cast(
		(select
			[server_name]		as 'td'
		,   ''
		,	[findings]			as 'td'
		,   ''
		from  @scanned_assets 
		order by [server_name] asc
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)


----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------

-- percentage of categories
select 
	[check_id]
,	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
from
	[2021-07-01-09-30-00-3883212-Financial-Services-Group-033]
group by
	[check_id]

select 
	[risk_dv]
,	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
from
	[2021-07-01-09-30-00-3883212-Financial-Services-Group-033]
group by
	[risk_dv]

select top 10 * from [2021-07-01-09-30-00-3883212-Financial-Services-Group-033]

select 
	'server_name'	= [asset]
,	'findings'		= count(*)
from
	[2021-07-01-09-30-00-3883212-Financial-Services-Group-033]
group by
	[asset] 
order by
	[asset] asc


-- percentage of details
select
	[details]
,	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
from
	[2021-07-01-09-30-00-3883212-Financial-Services-Group-033]
group by
	[details]
order by
	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0)) asc


----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------

----------------------------------------------------------------------------------------
-- set table framework for XML percent categories
declare @percent_categories	table ([line_id] int identity(1,1), [category] varchar(255), [percent] int, [percent_diff] int)
declare	@query_percent_categories				varchar(555)
set		@query_percent_categories				= 
(
'select 
	[check_id]
,	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
,	100 - convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
from [' + @new_import_destination_table + '] group by [check_id]'
)

insert into @percent_categories ([category], [percent], [percent_diff])
exec	(@query_percent_categories)

select 
	[category]
,	[percent]
,	[percent_diff]
from 
	@percent_categories


set @XML_PERCENT_CATEGORIES = 
	cast(
		(select
			[category]							as 'td'
		,   ''
		,   cast([percent] as varchar) + '%'	as 'td'
		,   ''
		,	cast('<table style="border-collapse: collapse; border: none;" cellpadding="0" cellspacing="0" width="250"><td bgcolor="#CA9321" style="width:' + cast([percent] as varchar) + '%; border: none; background-color:#CA9321; float:left; height:12px"></td><td bgcolor="#1D1D1D" style="width:' + cast([percent_diff] as varchar) + '%; border: none; background-color:#1D1D1D; float:left; height:12px;"></td></table>' as xml) as 'td'
		,	''
		from  @percent_categories 
		--order by [percent] asc
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)
 

 ----------------------------------------------------------------------------------------
-- set table framework for XML percent risk

declare @percent_risk	table ([risk_dv] varchar(555), [percent] int, [percent_diff] int)
declare	@query_percent_risk				varchar(555)
set		@query_percent_risk				= 
(
'select 
	[risk_dv]
,	convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
,	100 - convert(double precision, round(count(*) * 100.0 /sum(count(*)) over(), 0))
from [' + @new_import_destination_table + '] group by [risk_dv]'
)

insert into @percent_risk ([risk_dv], [percent], [percent_diff])
exec	(@query_percent_risk)


set @XML_PERCENT_RISK = 
	cast(
		(select
			[risk_dv]							as 'td'
		,   ''
		,   cast([percent] as varchar) + '%'	as 'td'
		,   ''
		,	cast('<table style="border-collapse: collapse; border: none;" cellpadding="0" cellspacing="0" width="250"><td bgcolor="#CA9321" style="width:' + cast([percent] as varchar) + '%; border: none; background-color:#CA9321; float:left; height:12px"></td><td bgcolor="#1D1D1D" style="width:' + cast([percent_diff] as varchar) + '%; border: none; background-color:#1D1D1D; float:left; height:12px;"></td></table>' as xml) as 'td'
		,   ''
		from  @percent_risk 
		order by [percent] asc
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)


----------------------------------------------------------------------------------------
-- set XML for previous scans

declare @previous_scans	table ([endtime] datetime, [reportname] varchar(255), [jobname] varchar(555), [customlabel] varchar(555), [filename] varchar(555))
insert into @previous_scans
select top 5 
	[endtime]
,	[reportname]
,	[jobname]
,	[customlabel]
,	[uuid] + '.xml'
from 
	#DBProtect_Stored_Reports order by [endtime] desc

--select * from @previous_scans

set @XML_PREVIOUS_SCANS = 
	cast(
		(select
			left([endtime], 25)	as 'td'
		,   ''
		,   [reportname]		as 'td'
		,   ''
		,   [jobname]			as 'td'
		,   ''
		,   [customlabel]		as 'td'
		,   ''
		,   [filename]			as 'td'
		,   ''
		from @previous_scans
		--where [path_frequency] > 1
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)


----------------------------------------------------------------------------------------
-- set XML for instance_info
set @xml_instance_info = 
	cast(
		(select
			cast(serverproperty('MachineName') as nvarchar) + '.' + lower(@domain) +  + case when [InstanceName] = 'MSSQLServer' then '' else '\' + [InstanceName] end
		+   case when [StaticPort] is null then ',' + cast([DynamicPort] as varchar) else ',' + cast([StaticPort] as varchar) end as 'td'
		,   ''
		,    cast(serverproperty('ComputerNamePhysicalNetBIOS') as varchar)		as 'td'
		,   ''
		,    [InstanceName]														as 'td'
		,   ''
		,    isnull(cast([StaticPort] as varchar), '')							as 'td'
		,   ''
		,    isnull(cast([DynamicPort] as varchar), '')							as 'td'

		from @instances 
		order by [InstanceName] asc 
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)

----------------------------------------------------------------------------------------
-- set XML version_info

set @xml_version_info = 
	cast(
		(select
			[Server_Name]		as 'td'
		,   ''
		,	[SQL_Instance]		as 'td'
		,   ''
		,	[SQL_Version]		as 'td'
		,   ''
		,	[SQL_Build]			as 'td'
		,   ''
		,	[SQL_Edition]		as 'td'

		from @version_info 
		order by [SQL_Instance] asc 
		for xml raw('tr')
	,   elements, type)
	as nvarchar(max)
		)


----------------------------------------------------------------------------------------
-- format email
set @HTML_BODY =
'<HTML>
<HEAD>
	<STYLE>																				
		BODY {background-color:#1A1B20; line-height:1px; -webkit-text-size-adjust:none; color: #bbb; font-family: sans-serif;}
		H1 {font-size: 90%; color: #bbb;}
		H2 {font-size: 90%;	color: #bbb;}
		H3 {color: #bbb;}
											
		TABLE, TD, TH {
			font-size: 87%;
			border: 1px solid #bbb;
			border-collapse: collapse;
		}
											
		TH {
			font-size: 87%;
			text-align: left;
			background-color: #1A1B20;
			color: #f8ab0c;
			padding: 4px;
			padding-left: 7px;
			padding-right: 7px;
		}
 
		TD {
			font-size: 87%;
			padding: 4px;
			padding-left: 7px;
			padding-right: 7px;
			max-width: 100px;
			overflow: hidden;
			text-overflow: ellipsis;
			white-space: nowrap;
		}

		ul {
		  list-style: none;
		  }

		ul li::before {
		  content: "\2022";  
		  color: red; 
		  font-weight: bold; 
		  display: inline-block; 
		  width: 1em; 
		  margin-left: -1em;
		  }


		hr {
		border: none;
		height: 1px;
		background: #CA9321;
		}



	</STYLE>
</HEAD>
<BODY>

<P STYLE="font-family: sans-serif; font-size: 30px; color: #f8ab0c;">
	SQL Scan Results
<P STYLE="color: #f8ab0c;">' + cast(getdate() as varchar) + '</P>
</P>
<hr width=100% color=#f8ab0c>


<ul>
  <li><p style="color: #f8ab0c;">Job Name: </p>' + (select @dbprotect_job_name)			+ '</li>
  <li><p style="color: #f8ab0c;">Report Name: </p>' + (select @dbprotect_report_name)	+ '</li>
  <li><p style="color: #f8ab0c;">Custom Label: </p>' + (select @dbprotect_custom_label) + '</li>
</ul> 

							
<p>XML File Info:</p>
<table border = 1>
<tr>
	<th> XML SOURCE PATH		</th>
	<th> XML FILE NAME			</th>
	<th> SQL TABLE DESTINATION	</th>
</tr>'

+ @XML_FILE_FOUND + '</table>
 

<p>The following ' + (select cast(count(*) as varchar) from @scanned_assets) + ' SQL Servers were scanned:<p>
 
<table border = 1>
		<tr>
			<th> SQL DATABASE SERVERS </th>
			<th> FINDINGS </th>
		</tr>'        
         
+ @XML_SCANNED_ASSETS + '</table>


<p>Category Volume:<p>
 
<table border = 1>
		<tr>
			<th> CATEGORY			</th>
			<th> PERCENT			</th>
			<th> VOLUME				</th>
		</tr>'        
         
+ @XML_PERCENT_CATEGORIES + '</table>


<p>Risk DV:<p>
 
<table border = 1>
		<tr>
			<th> RISK DV			</th>
			<th> PERCENT			</th>
			<th> VOLUME				</th>
		</tr>'        
         
+ @XML_PERCENT_RISK + '</table>


<p>Previous 5 SQL Server Scans (For Reference):<p>
 
<table border = 1>
		<tr>
			<th> TIME			</th>
			<th> REPORT NAME	</th>
			<th> JOB NAME		</th>
			<th> CUSTOM LABEL	</th>
			<th> FILE NAME		</th>
		</tr>'        
         
+ @XML_PREVIOUS_SCANS + '</table>


<p>Scan Results were imported into the following database system:</p>

<table border = 1>
		<tr>
			<th> SERVER NAME	</th>
			<th> SQL INSTANCE	</th>
			<th> SQL VERSION	</th>
			<th> SQL BUILD		</th>
			<th> SQL EDITION	</th>
		</tr>'        
         
+ @xml_version_info + '</table>
		
<p>Instances found on this system:</p>
								
<table border = 1>
		<tr>
			<th> INSTANCE CONNECTION STRING	</th>
			<th> NET BIOS NAME				</th>
			<th> INSTANCE NAME				</th>
			<th> STATIC PORT				</th>
			<th> DYNAMIC PORT				</th>
		</tr>'        
         
+ @xml_instance_info + '</table>

</BODY>
</HTML>'

----------------------------------------------------------------------------------------
-- set message subject.
declare 	@message_subject		varchar(255)
set 		@message_subject		= 'Scan Results for ' + @dbprotect_job_name

----------------------------------------------------------------------------------------
-- send email.
 
		exec	msdb.dbo.sp_send_dbmail
				@PROFILE_NAME	= 'SQLAlerts'
			,	@RECIPIENTS		= 'MyEmail@MyCompany.com'
			,	@SUBJECT		= @MESSAGE_SUBJECT
			,	@BODY			= @HTML_BODY
			,	@BODY_format	= 'HTML';

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

   
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

