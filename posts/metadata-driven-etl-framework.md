# Metadata-Driven ETL Framework: Orchestrating 1,000+ Packages

*Published: January 2025 | By: Data Engineering Professional*

## Overview

Building a metadata-driven controller and logging framework to orchestrate 1,000+ stored procedures and SSIS packages, enabling automated execution, monitoring, retry logic, and seamless migration to Azure. This framework reduces manual intervention by ~80% and significantly improves troubleshooting visibility.

## The Challenge

**Business Requirements:**
- Orchestrate 1,000+ SSIS packages and stored procedures
- Reduce manual runs by ~80%
- Improve visibility and troubleshooting
- Enable seamless migration to Azure
- Implement centralized logging and monitoring

**Technical Challenges:**
- Managing dependencies across 1,000+ packages
- Coordinating execution schedules
- Handling failures and retries
- Monitoring and logging at scale
- Hybrid on-premises and cloud execution

---

## Architecture Overview

### Framework Architecture

```
┌─────────────────────────────────────────────────────┐
│           Metadata Database (SSISConfig)             │
│  • Package Configuration                             │
│  • Schedules & Dependencies                          │
│  • Credentials (Key Vault References)                │
│  • Retry Policies                                     │
│  • Execution Logs                                    │
└──────────────────┬──────────────────────────────────┘
                   │
       ┌───────────┴───────────┐
       │                       │
┌──────▼──────────┐   ┌────────▼────────┐
│  T-SQL          │   │  Azure Data     │
│  Orchestrator   │   │  Factory        │
│  (On-Prem)      │   │  (Cloud)        │
└──────┬──────────┘   └────────┬────────┘
       │                       │
┌──────▼───────────────────────▼────────┐
│      Execution Layer                   │
│  • SSIS Packages (On-Prem/Cloud)      │
│  • Stored Procedures                   │
│  • Azure-SSIS Integration Runtime     │
└──────────────────┬────────────────────┘
                   │
┌──────────────────▼────────────────────┐
│      Logging & Monitoring              │
│  • ApplicationInstance                 │
│  • PackageExecution                    │
│  • LogEntry (Detailed)                 │
│  • Azure Monitor / Log Analytics        │
└────────────────────────────────────────┘
```

### Key Components

**Metadata Database (SSISConfig):**
- Centralized configuration store
- Package metadata and dependencies
- Schedule and retry policies
- Execution history and logging

**Orchestrator:**
- T-SQL controller for queue management
- Azure Data Factory pipelines
- Priority-based execution
- Dependency resolution

**Execution Layer:**
- SSIS packages (on-premises and cloud)
- Stored procedures
- Azure-SSIS Integration Runtime
- Self-Hosted Integration Runtime

**Logging & Monitoring:**
- ApplicationInstance tracking
- PackageExecution logs
- Detailed step/task logs
- Azure Monitor integration

---

## Metadata Model Design

### Core Tables

**Packages Table:**
```sql
CREATE TABLE dbo.Packages (
  PackageID      INT IDENTITY PRIMARY KEY,
  PackageName    NVARCHAR(260),
  ProjectName    NVARCHAR(260),
  DeploymentMode VARCHAR(20), -- 'SSISDB'|'FileSystem'
  IsEnabled      BIT DEFAULT 1,
  NodeCount      INT DEFAULT 1,
  TimeoutMinutes INT DEFAULT 120,
  RetryPolicy    NVARCHAR(MAX), -- JSON retry configuration
  LastModifiedBy NVARCHAR(100),
  LastModifiedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);
```

**Orchestrator Queue:**
```sql
CREATE TABLE dbo.OrchQueue (
  QueueID       INT IDENTITY PRIMARY KEY,
  PackageID     INT REFERENCES dbo.Packages(PackageID),
  ScheduledFor  DATETIME2,
  Priority      TINYINT DEFAULT 5, -- 1=Highest, 10=Lowest
  Status        VARCHAR(20) DEFAULT 'Pending', -- Pending, Running, Completed, Failed
  LockedBy      NVARCHAR(100) NULL,
  LockedAt      DATETIME2 NULL
);
```

**Application Instance:**
```sql
CREATE TABLE dbo.ApplicationInstance (
  AppInstanceID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  StartedAt     DATETIME2,
  FinishedAt    DATETIME2 NULL,
  Status        VARCHAR(20), -- Running, Completed, Failed
  Initiator     NVARCHAR(100),
  Remarks       NVARCHAR(4000),
  CorrelationID NVARCHAR(100) -- For log correlation
);
```

**Package Execution Log:**
```sql
CREATE TABLE dbo.PackageExecution (
  ExecID         BIGINT IDENTITY PRIMARY KEY,
  AppInstanceID  UNIQUEIDENTIFIER REFERENCES dbo.ApplicationInstance(AppInstanceID),
  QueueID        INT REFERENCES dbo.OrchQueue(QueueID),
  PackageID      INT REFERENCES dbo.Packages(PackageID),
  StartedAt      DATETIME2,
  FinishedAt     DATETIME2,
  Status         VARCHAR(20), -- Running, Success, Failed, Cancelled
  RowsProcessed  BIGINT NULL,
  DurationSeconds INT NULL,
  ErrorCode      INT NULL,
  ErrorMessage   NVARCHAR(4000) NULL,
  RetryCount     INT DEFAULT 0,
  ExecutionMode VARCHAR(20) -- 'OnPrem'|'Azure'
);
```

**Detailed Log Entry:**
```sql
CREATE TABLE dbo.LogEntry (
  LogID       BIGINT IDENTITY PRIMARY KEY,
  ExecID      BIGINT REFERENCES dbo.PackageExecution(ExecID),
  LoggedAt    DATETIME2 DEFAULT SYSUTCDATETIME(),
  Level       VARCHAR(10), -- INFO, WARN, ERROR
  Component   NVARCHAR(200),
  Message     NVARCHAR(MAX),
  StackTrace  NVARCHAR(MAX) NULL
);
```

---

## Orchestrator Implementation

### T-SQL Controller

**Core Orchestrator Stored Procedure:**
```sql
CREATE PROCEDURE dbo.usp_Orchestrator_ProcessNext
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @QueueID INT;
    DECLARE @PackageID INT;
    DECLARE @AppInstanceID UNIQUEIDENTIFIER = NEWID();
    
    BEGIN TRANSACTION;
    
    -- Lock and select next item from queue
    SELECT TOP 1 
        @QueueID = QueueID,
        @PackageID = PackageID
    FROM dbo.OrchQueue WITH (UPDLOCK, ROWLOCK)
    WHERE Status = 'Pending' 
        AND ScheduledFor <= GETUTCDATE()
        AND IsEnabled = 1
    ORDER BY Priority ASC, ScheduledFor ASC;
    
    IF @QueueID IS NOT NULL
    BEGIN
        -- Create application instance
        INSERT INTO dbo.ApplicationInstance (AppInstanceID, StartedAt, Status, Initiator)
        VALUES (@AppInstanceID, GETUTCDATE(), 'Running', SYSTEM_USER);
        
        -- Update queue status
        UPDATE dbo.OrchQueue
        SET Status = 'Running',
            LockedBy = SYSTEM_USER,
            LockedAt = GETUTCDATE()
        WHERE QueueID = @QueueID;
        
        COMMIT TRANSACTION;
        
        -- Execute package (on-premises or queue for ADF)
        EXEC dbo.usp_ExecutePackage 
            @PackageID = @PackageID,
            @AppInstanceID = @AppInstanceID,
            @QueueID = @QueueID;
    END
    ELSE
    BEGIN
        COMMIT TRANSACTION;
    END
END;
```

### Package Execution Logic

**On-Premises Execution:**
```sql
CREATE PROCEDURE dbo.usp_ExecutePackage
    @PackageID INT,
    @AppInstanceID UNIQUEIDENTIFIER,
    @QueueID INT
AS
BEGIN
    DECLARE @ExecID BIGINT;
    DECLARE @PackageName NVARCHAR(260);
    DECLARE @StartTime DATETIME2 = GETUTCDATE();
    DECLARE @ExitCode INT;
    
    SELECT @PackageName = PackageName 
    FROM dbo.Packages 
    WHERE PackageID = @PackageID;
    
    -- Log execution start
    INSERT INTO dbo.PackageExecution 
        (AppInstanceID, QueueID, PackageID, StartedAt, Status)
    VALUES 
        (@AppInstanceID, @QueueID, @PackageID, @StartTime, 'Running');
    
    SET @ExecID = SCOPE_IDENTITY();
    
    -- Execute SSIS package via dtexec or SQL Agent
    -- Capture exit code
    
    DECLARE @EndTime DATETIME2 = GETUTCDATE();
    DECLARE @DurationSeconds INT = DATEDIFF(SECOND, @StartTime, @EndTime);
    
    IF @ExitCode = 0
    BEGIN
        UPDATE dbo.PackageExecution
        SET Status = 'Success',
            FinishedAt = @EndTime,
            DurationSeconds = @DurationSeconds
        WHERE ExecID = @ExecID;
        
        UPDATE dbo.OrchQueue
        SET Status = 'Completed'
        WHERE QueueID = @QueueID;
    END
    ELSE
    BEGIN
        UPDATE dbo.PackageExecution
        SET Status = 'Failed',
            FinishedAt = @EndTime,
            DurationSeconds = @DurationSeconds,
            ErrorCode = @ExitCode,
            ErrorMessage = 'Package execution failed'
        WHERE ExecID = @ExecID;
        
        -- Apply retry policy
        EXEC dbo.usp_HandleRetry @QueueID, @PackageID;
    END
END;
```

---

## Azure Data Factory Integration

### Metadata-Driven Pipeline

**Pipeline Structure:**
```json
{
  "name": "OrchestratorMain",
  "properties": {
    "activities": [
      {
        "name": "GetPackagesToRun",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "SqlSource",
            "sqlReaderQuery": "SELECT * FROM dbo.OrchQueue WHERE Status='Pending' AND ScheduledFor <= GETUTCDATE()"
          }
        }
      },
      {
        "name": "ForEachPackage",
        "type": "ForEach",
        "dependsOn": [{"activity": "GetPackagesToRun"}],
        "typeProperties": {
          "items": {
            "value": "@activity('GetPackagesToRun').output.value",
            "type": "Expression"
          },
          "isSequential": false,
          "batchCount": 5,
          "activities": [
            {
              "name": "StartAzureSSISIR",
              "type": "WebActivity",
              "condition": "@equals(item().ExecutionMode, 'Azure')",
              "typeProperties": {
                "url": "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.DataFactory/factories/{factoryName}/integrationRuntimes/{irName}/start?api-version=2018-06-01",
                "method": "POST"
              }
            },
            {
              "name": "ExecuteSSISPackage",
              "type": "ExecuteSSISPackage",
              "typeProperties": {
                "packageLocation": {
                  "type": "SSISDB",
                  "packagePath": "@concat(item().ProjectName, '/', item().PackageName)"
                },
                "runtime": {
                  "type": "Managed",
                  "computeType": "General",
                  "coreCount": item().NodeCount
                },
                "projectParameters": {
                  "Environment": "Production"
                }
              }
            },
            {
              "name": "UpdateExecutionLog",
              "type": "SqlServerStoredProcedure",
              "typeProperties": {
                "storedProcedureName": "dbo.usp_UpdateExecutionLog",
                "storedProcedureParameters": {
                  "ExecID": {"value": "@activity('ExecuteSSISPackage').output.executionId", "type": "String"},
                  "Status": {"value": "@activity('ExecuteSSISPackage').output.status", "type": "String"}
                }
              }
            },
            {
              "name": "StopAzureSSISIR",
              "type": "WebActivity",
              "condition": "@equals(item().ExecutionMode, 'Azure')",
              "typeProperties": {
                "url": "https://management.azure.com/.../integrationRuntimes/{irName}/stop?api-version=2018-06-01",
                "method": "POST"
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

## Key Features

### Centralized Logging

**Three-Level Logging:**
1. **ApplicationInstance**: Logical application run
2. **PackageExecution**: Individual package execution
3. **LogEntry**: Detailed step/task messages

**Correlation IDs:**
- Trace execution across all levels
- Join logs for SLA tracking
- Incident investigation
- Performance analysis

### Retry Logic

**Configurable Retry Policies:**
```sql
-- Retry policy JSON structure
{
  "maxRetries": 3,
  "retryDelaySeconds": 60,
  "backoffMultiplier": 2,
  "retryOnErrors": [500, 503, 504],
  "exponentialBackoff": true
}
```

**Retry Implementation:**
```sql
CREATE PROCEDURE dbo.usp_HandleRetry
    @QueueID INT,
    @PackageID INT
AS
BEGIN
    DECLARE @RetryCount INT;
    DECLARE @MaxRetries INT;
    DECLARE @RetryPolicy NVARCHAR(MAX);
    
    SELECT @RetryPolicy = RetryPolicy
    FROM dbo.Packages
    WHERE PackageID = @PackageID;
    
    -- Parse retry policy and determine if retry allowed
    -- Update queue with new scheduled time
    -- Increment retry count
END;
```

### Cost Optimization

**Azure-SSIS IR Management:**
- Start IR only when needed
- Stop IR after execution completes
- Scale nodes based on workload
- Monitor idle time and costs

**PowerShell Automation:**
```powershell
# Start Azure-SSIS IR
Start-AzDataFactoryV2IntegrationRuntime `
    -ResourceGroupName "rg-etl" `
    -DataFactoryName "adf-etl" `
    -Name "Azure-SSIS-IR"

# Stop Azure-SSIS IR after execution
Stop-AzDataFactoryV2IntegrationRuntime `
    -ResourceGroupName "rg-etl" `
    -DataFactoryName "adf-etl" `
    -Name "Azure-SSIS-IR"
```

---

## Key Achievements

### Operational Improvements

**Automation:**
- ✅ **~80% reduction** in manual package runs
- ✅ **Automated scheduling** and execution
- ✅ **Self-healing** retry mechanisms
- ✅ **Centralized** configuration management

**Visibility:**
- ✅ **Comprehensive logging** at all levels
- ✅ **Real-time monitoring** and alerts
- ✅ **Correlation IDs** for troubleshooting
- ✅ **Performance metrics** and analytics

**Migration:**
- ✅ **Seamless migration** to Azure
- ✅ **Hybrid execution** support
- ✅ **Cost optimization** through IR management
- ✅ **Scalable architecture**

---

## Best Practices

### Framework Best Practices

1. **Single Source of Truth**: Metadata DB for all configuration
2. **Idempotent Packages**: Safe to re-run packages
3. **Centralized Logging**: Correlation IDs for troubleshooting
4. **Secrets Management**: Azure Key Vault for credentials
5. **Small, Focused Packages**: Modular and maintainable
6. **CI/CD Integration**: Automated deployment
7. **Concurrency Control**: Limit parallelism to protect resources
8. **Schema Validation**: Validate before heavy processing

### Common Traps & Solutions

**Trap**: Hard-coded file paths in packages  
**Solution**: Parameterize all paths using metadata

**Trap**: Secrets in package configs  
**Solution**: Use Azure Key Vault references

**Trap**: Running Azure-SSIS IR 24x7  
**Solution**: Automate start/stop based on schedule

**Trap**: Database locking with parallel packages  
**Solution**: Limit concurrency and implement locking strategies

**Trap**: Full loads instead of incremental  
**Solution**: Implement change data capture (CDC)

---

## Monitoring & Observability

### Azure Monitor Integration

**Metrics:**
- Package execution duration
- Success/failure rates
- Retry counts
- Resource utilization

**Alerts:**
- Failed package executions
- Long-running packages
- High retry rates
- Cost threshold alerts

**Log Analytics:**
- Centralized log analysis
- Query execution logs
- Performance trends
- Error pattern analysis

---

## Migration Checklist

### Quick Migration Steps

1. **Inventory**: Catalog all packages and stored procedures
2. **Tag**: Classify by risk, dependencies, and frequency
3. **Parameterize**: Replace hard-coded values with parameters
4. **Metadata**: Implement SSISConfig database
5. **Orchestrator**: Build controller and logging procedures
6. **Deploy**: Deploy packages to Azure-SSIS IR
7. **ADF Pipelines**: Build metadata-driven ADF pipelines
8. **Monitor**: Set up monitoring and alerts
9. **Cutover**: Gradual migration with validation
10. **Optimize**: Tune performance and costs

---

## Conclusion

A metadata-driven ETL framework provides:
- **Centralized control** for 1,000+ packages
- **Automated execution** reducing manual effort
- **Comprehensive logging** improving troubleshooting
- **Seamless migration** to cloud platforms
- **Cost optimization** through intelligent resource management

**Key Takeaways:**
1. Metadata-driven approach scales to thousands of packages
2. Centralized logging dramatically improves troubleshooting
3. Automated retry logic reduces manual intervention
4. Cost optimization through intelligent IR management
5. Hybrid execution enables gradual cloud migration

---

*This framework enables enterprise-scale ETL orchestration. For related content, see [Enterprise Data Warehouse Migration](./enterprise-data-warehouse-migration.html) and [ETL Automation Solution](./etl-automation-solution.html).*

