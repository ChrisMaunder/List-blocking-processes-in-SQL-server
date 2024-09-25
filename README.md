# List blocking processes in SQL server

A quick script to enable you to find the processes that are blocking in SQL server
We use this quick script to list those processes that are blocking other processes in SQL server. What you do with the information is up to you, but we suggest checking to ensure you're only locking when you need to (SET READ UNCOMMITTED or use NOLOCK, ROWLOCK hints if appropriate), and we also suggest you review your SQL setup: is your hardware fast enough? Have you separated out your logging, data and tempDB files onto separate volumes? 

Usage: `ListBlocking [@KillOrphanedProcesses = 0] `

If `@KillOrphanedProcesses` is set to 1 then the script will attempt to kill orphaned processes.

```tsql
IF NOT EXISTS (SELECT * FROM sys.objects 
WHERE object_id = OBJECT_ID(N'[dbo].[ListBlocking]') 
AND type in (N'P', N'PC'))
EXEC dbo.sp_executesql @statement = N'CREATE PROCEDURE [dbo].[ListBlocking] AS'
GO

/*===================================================================

Description:    Wrapper to sp_who2 to show only those processes that 
                are blocking. 
                
                Please see http://support.microsoft.com/kb/224453 for
                more info on locking.

===================================================================*/ 

ALTER procedure ListBlocking
(
    @KillOrphanedProcesses bit = 0
)
as

IF OBJECT_ID('tempdb..#Process') IS NOT NULL drop table #Process
IF OBJECT_ID('tempdb..#BlockingProcess') IS NOT NULL drop table #BlockingProcess

create table #Process (
    SPID int,
    Status varchar(500),
    Login varchar(500),
    Hostname varchar(500),
    BlkBy varchar(50),
    DBName varchar(500),
    Command varchar(500),
    CPUTime int,
    DiskIO int,
    LastBatch varchar(500),
    ProgramName varchar(500),
    SPID2 int,
    RequestId int
)

CREATE TABLE #BlockingProcess (
    BlockingProcessID int IDENTITY(1,1),
    ProcessID varchar(20),
    EventType varchar(100),
    Parameters varchar(100),
    EventInfo varchar(500),
    CPUTime int,
    DiskIO int,
    TransactionCount int
)

DECLARE @BlockingPID        varchar(20),
        @CPUTime            int, 
        @DiskIO             int,
        @TransactionCount   int

-- Let SQL Server get us a list of what's going on
SET NOCOUNT ON
insert into #Process exec sp_who2

-- From this list of what's going on, get those items that are blocked, and loop through them
DECLARE ProcessCursor CURSOR FAST_FORWARD FOR 
    SELECT BlkBy, SUM(CPUTime) as CPUTime, SUM(DiskIO) as DiskIO
    FROM #Process
    WHERE ISNULL(BlkBy,'') <> ''
    GROUP BY BlkBy
OPEN ProcessCursor

FETCH NEXT FROM ProcessCursor INTO @BlockingPID, @CPUTime, @DiskIO
WHILE @@FETCH_STATUS = 0
BEGIN

    -- Only valid PIDs
    if ISNULL(@BlockingPID, '') <> '' and @BlockingPID <> '0' AND @BlockingPID <> '  .'
    BEGIN

        -- For each blocked process get what information we can on the process and what's blocking it
        SET @TransactionCount = 0

        --  -2 = The blocking resource is owned by an orphaned distributed transaction.
        IF SUBSTRING(@BlockingPID, 1, 2) = '-2' 
        BEGIN
        
            -- For orphaned processes we need to get the Unit of work if we're to do anything with it
            DECLARE @UnitOfWork varchar(50)
            select top 1 @UnitOfWork = ISNULL(req_transactionUOW, '')
            from master..syslockinfo
            where req_spid = -2
                
            if @KillOrphanedProcesses = 1
            BEGIN
                if @UnitOfWork <> '' exec('KILL ''' + @UnitOfWork + '''')
                
                INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
                VALUES('', '', '- Killed UOW ' + @UnitOfWork + ' -')
            END
            ELSE
            BEGIN
                INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
                VALUES('', '', '- Orphaned. UOW = ' + @UnitOfWork + ' -')
            END
        END
        
        -- -3 = The blocking resource is owned by a deferred recovery transaction.
        ELSE IF SUBSTRING(@BlockingPID, 1, 2) = '-3' 
        BEGIN
            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            VALUES('', '', '- deferred recovery transaction -')
        END
        
        -- -4 = Session ID of the blocking latch owner could not be determined at this 
        --      time because of internal latch state transitions.
        ELSE IF SUBSTRING(@BlockingPID, 1, 2) = '-4' 
        BEGIN
            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            VALUES('', '', '- Latch owner could not be determined -')
        END

        -- Nothing unusual here. A process is being blocked by another processes, so let's get the
        -- info on the process that's doing the blocking
        ELSE
        BEGIN

            SELECT @TransactionCount = open_tran FROM master.sys.sysprocesses WHERE SPID=@BlockingPID

            INSERT INTO #BlockingProcess (EventType, Parameters, EventInfo)
            EXEC ('DBCC INPUTBUFFER(' + @BlockingPID + ') WITH NO_INFOMSGS')
        END         
        
        -- Add some aggregate info on the blocking processes
        UPDATE #BlockingProcess 
        SET ProcessID        = @BlockingPID,
            CPUTime          = @CPUTime,
            DiskIO           = @DiskIO,
            TransactionCount = @TransactionCount
        WHERE BlockingProcessID = SCOPE_IDENTITY()
    END
    
    FETCH NEXT FROM ProcessCursor INTO @BlockingPID, @CPUTime, @DiskIO
END

CLOSE ProcessCursor
DEALLOCATE ProcessCursor

-- Display the results
SELECT ProcessID, EventInfo, TransactionCount, CPUTime, DiskIO FROM #BlockingProcess

SET NOCOUNT OFF
```
