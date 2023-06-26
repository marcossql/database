select *
from Alert_Parameter
where Nm_Alert = 'CPU Utilization'

update Alert_Parameter
set Vl_Parameter = 40
where Nm_Alert = 'CPU Utilization'

-- Rodei as queries das outras conexões

update Alert_Parameter
set Vl_Parameter = 85
where Nm_Alert = 'CPU Utilization'




--Conferir a Versão do SQL Server
SELECT @@version


-- Quantidade de cores disponíveis para o SQL Server
SELECT current_tasks_count,runnable_tasks_count,* 
FROM sys.dm_os_schedulers
WHERE scheduler_id < 255
AND status = 'VISIBLE ONLINE'

--Já peguei casos do cliente ter 8 sockets configurados no SQL Server Standard, e o SQL estar utilizando apenas 4.
	 

--------------------- Algumas dicas para se fazer quando tiver um gargalo de CPU

-- Conferir no task Manager se tem outro processo consumindo muita CPU além do SQL Server


-- Validar a o que está rodando no banco de dados
exec sp_whoisactive @get_task_info =2, @get_plans = 1, @delta_interval = 1, @show_sleeping_spids = 0, @get_outer_command = 1


-- Queries nesse momento utilizando os cores de CPU
SELECT 
a.scheduler_id ,
b.session_id,
 (SELECT TOP 1 SUBSTRING(s2.text,statement_start_offset / 2+1 , 
      ( (CASE WHEN statement_end_offset = -1 
         THEN (LEN(CONVERT(nvarchar(max),s2.text)) * 2) 
         ELSE statement_end_offset END)  - statement_start_offset) / 2+1))  AS sql_statement
FROM sys.dm_os_schedulers a 
INNER JOIN sys.dm_os_tasks b on a.active_worker_address = b.worker_address
INNER JOIN sys.dm_exec_requests c on b.task_address = c.task_address
CROSS APPLY sys.dm_exec_sql_text(c.sql_handle) AS s2 



-- Se for uma procedure, rodar um sp_recompile

-- Limpar o PlanCache
GO  
SELECT plan_handle, st.text  
FROM sys.dm_exec_cached_plans   
CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS st  
WHERE text LIKE N'%Gols_Jogadores_SQLGOL%';  
GO  
-- Remove the specific plan from the cache.  
DBCC FREEPROCCACHE (0x0600070002DAC81B200AF7C81402000001000000000000000000000000000000000000000000000000000000);  
GO  
DBCC FREEPROCCACHE

-- Atualizar as estatísticas das tabelas envolvidas

--Analisar a possibilidade de criação de um índice ou reescrita das queries envolvidas


--Queries que mais consomem CPU no traces
SELECT *
FROM Traces..Queries_Profile
ORDER BY CPU DESC


--Queries do Cache com maior consumo de CPU
if object_id('tempdb..#Temp_Trace') is not null drop table #Temp_Trace

sELECT TOP 50 total_worker_time ,  sql_handle,execution_count,last_execution_time,last_worker_time
into #Temp_Trace
FROM sys.dm_exec_query_stats A
where last_elapsed_time > 20
 and last_execution_time > dateadd(ss,-600,getdate()) --ultimos 5 min
order by A.total_worker_time desc

select distinct A.*, B.*, DB.name
from #Temp_Trace A
cross apply sys.dm_exec_sql_text (sql_handle) B
join sys.databases DB on B.dbid = DB.database_id
order by 1 DESC


--Consumo de CPU por base de dados
WITH DB_CPU_Statistics
AS
(SELECT pa.DatabaseID, DB_NAME(pa.DatabaseID) AS [Database Name], SUM(qs.total_worker_time/1000) AS [CPU_Time_Ms]
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY (SELECT CONVERT(INT, value) AS [DatabaseID]
FROM sys.dm_exec_plan_attributes(qs.plan_handle)
WHERE attribute = N'dbid') AS pa
GROUP BY DatabaseID)
SELECT ROW_NUMBER() OVER(ORDER BY [CPU_Time_Ms] DESC) AS [CPU Ranking],
[Database Name], [CPU_Time_Ms] AS [CPU Time (ms)],
CAST([CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPU Percent]
FROM DB_CPU_Statistics
WHERE DatabaseID <> 32767 -- ResourceDB
ORDER BY [CPU Ranking] OPTION (RECOMPILE);

