![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Header_890x170.png "Mikes Data Work")        

# Create Modern Drive Alerts With Pure SQL HTML And CSS
**Post Date: June 4, 2021**


## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)      

## About-Process


<p>Here is another example on how to create an SQL HTML & CSS Alert for Drive Space.  All that is required is plug in your SMTP Server Name, and of course your email as the primary recipient.  Next... Throw this logic into a Job and put it on a regular schedule.  This can be modified to check space every minute, or once a day.  The charting is created using direct HTML Tables and this actually works in Outlook without any additional modification.</p>

![Modern SQL HTML Email Alert For Drive Space]( https://mikesdatawork.files.wordpress.com/2021/06/image002.png "Modern SQL HTML Email Alert For Drive Space")
 
     
## SQL-Logic
```SQL
use [master];
set nocount on

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- configure database mail and create SQLAlert profile with SMTP server mailer.company.com

if (select sum(cast([value_in_use] as int)) from master.sys.configurations where [configuration_id] in ('518', '16386', '16388')) <> 3
begin
	exec master..sp_configure	'show advanced options',		1 reconfigure
	exec master..sp_configure	'ole automation procedures',	1 reconfigure with override
	exec master..sp_configure	'database mail xps',			1 reconfigure with override
end
 
if not exists(select * from msdb.dbo.sysmail_profile where  name = 'sqlalerts')  
	begin 
	execute msdb.dbo.sysmail_add_profile_sp 
		@profile_name = 'sqlalerts', 
		@description  = 'sqldatabasemail'; 
	end
   
	if not exists(select * from msdb.dbo.sysmail_account where  name = 'sqlalerts@company.com') 
	begin 
	execute msdb.dbo.sysmail_add_account_sp 
	@account_name            = 'sqlalerts@company.com', 
	@email_address           = 'sqlalerts@company.com', 
	@mailserver_name         = 'mailer.company.com', 
	@mailserver_type         = 'smtp', 
	@port                    = '25', 
	@use_default_credentials =  0 , 
	@enable_ssl              =  0 ; 
end 
   
if not exists
(
select * from msdb.dbo.sysmail_profileaccount pa 
inner join msdb.dbo.sysmail_profile p on pa.profile_id = p.profile_id 
inner join msdb.dbo.sysmail_account a on pa.account_id = a.account_id   
where p.name = 'sqlalerts' 
and a.name = 'sqlalerts@company.com'
)  
begin 
execute msdb.dbo.sysmail_add_profileaccount_sp 
    @profile_name = 'sqlalerts', 
    @account_name = 'sqlalerts@company.com', 
    @sequence_number = 1 ; 
end

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- capture database server info

use master;
set nocount on

declare @server_name	varchar(255)
declare @domain			varchar(255)
declare	@os				varchar(255)
set		@server_name	= (select @@servername)
declare @domain_info	table ([key] varchar(255), [domain] varchar(255))
declare	@sql_version	varchar(255)
declare @sql_port		varchar(25)
set		@sql_port		= (select cast([local_tcp_port] as varchar) from sys.dm_exec_connections where [session_id] = @@spid)
insert into @domain_info exec master..xp_regread 'hkey_local_machine', 'system\currentcontrolset\services\tcpip\parameters', N'domain'
set		@sql_version	= (left(@@version, 25))
set		@domain			= (select [domain] from @domain_info)
set		@os				= (select case [windows_release] when '6.2' then 'Windows Server 2012' when '6.3' then 'Windows Server 2012 R2' when '10.0' then 'Windows Server 2016' else 'OS_Version: ' + [windows_release] end from sys.dm_os_windows_info)

declare @sql_instances table ([count] int identity(1,1), [value] nvarchar(100), [instance_name] nvarchar(100), [data] nvarchar(100))
insert into @sql_instances
exec master..xp_regread @rootkey = 'hkey_local_machine', @key = 'software\microsoft\microsoft sql server', @value_name = 'installedinstances'
declare @sql_count	varchar(3)  set	@sql_count	= (select count(*) from @sql_instances)

declare @server_info	table ([server_name] varchar(255), [sql_connection_string] varchar(255), [sql_version] varchar(255), [os_version] varchar(255))
insert into @server_info select @server_name, @server_name + '.' + @domain + ',' + @sql_port, @sql_version, @os 

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- capture drive info

use [master];
set nocount on

declare @drive_info table 
(
	[id]		int identity(1,1) primary key clustered
,	[drive]     varchar(255)	null
,	[output]	varchar(1000)	null
,	[object]	varchar(100)	null
,	[amount]	bigint			null
);

declare @drives table ([drive] varchar(255) not null, [mb_free] bigint not null);
insert into @drives ([drive], [mb_free]) exec master..xp_fixeddrives;

declare @drive varchar(100);
declare quick_cursor cursor local forward_only static read_only
for
select drv.drive from @drives drv order by drv.drive;
open quick_cursor; fetch next from quick_cursor into @drive;
while @@fetch_status = 0
begin
    declare	@cmd varchar(1000);
    set		@cmd = 'fsutil volume diskfree ' + @drive + ':';
    insert into @drive_info ([output]) exec master..xp_cmdshell @cmd;
    update	@drive_info set drive = @drive + ':' where drive is null;
    fetch next from quick_cursor into @drive;
end
close quick_cursor; deallocate quick_cursor;

delete from @drive_info where [output] is null;
update @drive_info set [object] = left([output], charindex(':', [output]) - 1), [amount] = right([output], charindex(':', reverse([output])) - 1)
declare @drive_report table ([drive] varchar(25), [total] int, [mb_used] int, [mb_free] int, [percent_free] int, [percent_diff] int, [browse_drive] varchar(255))
 
;with pivot_cte as 
(
	select 
		[pivot].[drive]
	,	[pivot].[total # of bytes             ]
	,	[pivot].[total # of free bytes        ]
	from 
		@drive_info
	pivot 
	(
	sum([amount])
	for [object] in (
		[total # of free bytes        ]
	,	[total # of bytes             ]
	)
	) [pivot]
)
insert into @drive_report
select 
	[drive]
,	[total]			= cast(replace(cast(format(sum(pivot_cte.[total # of bytes             ] / 1048576), '#,###') as varchar), ',','') as int)
,	[mb_used]		= cast(replace(cast(format(sum(pivot_cte.[total # of bytes             ] / 1048576), '#,###') as varchar), ',','') as int) - cast(replace(cast(format(sum(pivot_cte.[total # of free bytes        ] / 1048576), '#,###') as varchar), ',','') as int)
,	[mb_free]		= cast(replace(cast(format(sum(pivot_cte.[total # of free bytes        ] / 1048576), '#,###') as varchar), ',', '') as int)
,	[percent_free]	= ceiling(replace(cast(format(convert(double precision, sum(pivot_cte.[total # of free bytes        ])) / convert(double precision, sum(pivot_cte.[total # of bytes             ])), '0.00%') as varchar), '%', ''))
,	[percent_diff]	= 100 - ceiling(replace(cast(format(convert(double precision, sum(pivot_cte.[total # of free bytes        ])) / convert(double precision, sum(pivot_cte.[total # of bytes             ])), '0.00%') as varchar), '%', ''))
,	[browse_drive]	= '\\' + @server_name + '\' + replace([drive], ':', '') + '$\'
from [pivot_cte] group by [drive]

select * from @drive_report

----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- set table framework for XML percent categories

declare	@html_body				nvarchar(max)
declare @html_table_server_info nvarchar(max)
declare @html_table_instances	nvarchar(max)
declare	@html_table_drives		nvarchar(max)

set		@html_table_server_info = 
cast(
	(select
		[server_name]			as 'td'
	,   ''
	,	[sql_connection_string]		as 'td'
	,   ''
	,	[sql_version]		as 'td'
	,   ''
	,	[os_version]		as 'td'
	,	''
	from  @server_info 
	--order by [percent] asc
	for xml raw('tr')
,   elements, type)
as nvarchar(max)
	)

set		@html_table_instances = 
cast(
	(select
		case when [instance_name] = 'MSSQLSERVER' then @server_name else [instance_name] end as 'td'
	,   ''
	from  @sql_instances 
	--order by [percent] asc
	for xml raw('tr')
,   elements, type)
as nvarchar(max)
	)

set		@html_table_drives = 
cast(
	(select
		[drive]			as 'td'
	,   ''
	,	[total]			as 'td'
	,	''
	,	[mb_used]		as 'td'
	,   ''
	,	[mb_free]		as 'td'
	,   ''
	,   cast([percent_free] as varchar) + '%'	as 'td'
	,   ''
	,	cast('<table style="border-collapse: collapse; border: none;" cellpadding="0" cellspacing="0" width="250"><td bgcolor="#CA9321" style="width:' + cast([percent_diff] as varchar) + '%; border: none; background-color:#CA9321; float:left; height:12px"></td><td bgcolor="#1D1D1D" style="width:' + cast([percent_free] as varchar) + '%; border: none; background-color:#1D1D1D; float:left; height:12px;"></td></table>' as xml) as 'td'
	,	''
	,	[browse_drive]	as 'td'
	,	''
	from  @drive_report 
	--order by [percent] asc
	for xml raw('tr')
,   elements, type)
as nvarchar(max)
	)

----------------------------------------------------------------------------------------
-- create html for email
set @html_body =
'<html>
<head>
<style>																				
BODY {background-color:#1A1B20; line-height:1px; -webkit-text-size-adjust:none; color: #bbb; font-family: sans-serif;}
H1 {font-size: 90%; color: #bbb;}
H2 {font-size: 90%;	color: #bbb;}
H3 {color: #bbb;}

TABLE, TD, TH {
	font-size: 87%;
	border: 1px solid #bbb;
	border-collapse: collapse;}
			
TH {
	font-size: 87%;
	text-align: left;
	background-color: #1A1B20;
	color: #f8ab0c;
	padding: 4px;
	padding-left: 7px;
	padding-right: 7px;}

TD {
	font-size: 87%;
	padding: 4px;
	padding-left: 7px;
	padding-right: 7px;
	max-width: 100px;
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;}

ul {list-style: none;}

ul li::before {
	content: "\2022";  
	color: red; 
	font-weight: bold; 
	display: inline-block; 
	width: 1em; 
	margin-left: -1em;}

hr {
border: none;
height: 1px;
background: #ca9321;}

a {
  color: 	#1E90FF;
  text-decoration: none;
  font-weight: normal;
  }


</style>
</head>
<body>
<p style="font-family: sans-serif; font-size: 20px; color: #f8ab0c;">Space Alert - Server:  ' + @server_name + '.' + @domain + ',' + @sql_port + '</p>
<hr width=100% color=#f8ab0c>

<p>Server Info</p>
<table border = 1>
<tr>
	<th> SERVER NAME	</th>
	<th> SQL CONNECTION STRING	</th>
	<th> SQL VERSION	</th>
	<th> OS VERSION		</th>
</tr>'
+ @html_table_server_info+ '</table>

<p> ' + @sql_count + ' SQL Instance(s) found on this system</p>
<table border = 1>
<tr>
	<th> SQL INSTANCE		</th>
</tr>'
+ @html_table_instances+ '</table>

<p style="color:red">Space Red</p>

<p>Drive Space</p>
<table border = 1>
<tr>
	<th> DRIVE		</th>
	<th> MB TOTAL	</th>
	<th> MB USED	</th>
	<th> MB FREE	</th>
	<th> % FREE		</th>
	<th> GRAPH		</th>
	<th> BROWSE DRIVE </th>
</tr>'
+ @html_table_drives + '</table>

</body>
</html>'

----------------------------------------------------------------------------------------
-- set message subject.
declare 	@message_subject		varchar(255)
set 		@message_subject		= 'Drive Space on : ' + @server_name

----------------------------------------------------------------------------------------
-- send email.
 
exec	msdb.dbo.sp_send_dbmail
			@profile_name	= 'sqlalerts'
		,	@recipients		= 'database.admin@company.com'
		,	@subject		= @message_subject
		,	@body			= @html_body
		,	@body_format	= 'html';

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

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/Mikes_Data_Work_Social.png "Mikes Data Work")
