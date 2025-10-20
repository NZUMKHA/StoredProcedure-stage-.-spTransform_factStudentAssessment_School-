T4 Senior CIE Dashboard Control Implementation
Date: October 21, 2025
Author: Umar Khattak (INTERNAL\u.khattak)
Issue: T4.SEN.CIE.EQ control not working - grades not being hidden/shown based on LockedFlag
Server: KC-SQL-AGS-01 (ETL/Data Warehouse Server)
Database: KCDW
Procedure Modified: stage.spTransform_factStudentAssessment_School
________________________________________
🎯 Problem Statement
Initial Issue
When Geoff (Academic Head) added the control entry T4.SEN.CIE.EQ to the uludashboardDataControl table in Production, the lock/unlock functionality did not work. The grades were not being hidden when LockedFlag = 1, and teacher comments were not updating during ETL runs.
Root Cause Discovered
The ETL stored procedure stage.spTransform_factStudentAssessment_School on KC-SQL-AGS-01 was missing T4 Senior flag support. Investigation revealed:
1.	Procedure last modified: July 11, 2023 by m.kannangara (Madawa)
2.	Control codes added throughout 2025 by G.Smith (Geoff) as each term arrived
3.	T4 Junior support was partially added but T4 Senior (A&E and CIE) was never implemented
4.	T4.SEN.CIE.EQ control entry was added Oct 20, 2025 by u.khattak, but procedure couldn't process it
________________________________________
📊 How the Control System Works
Control Flow
Production Database (KC-SQL-SVR-09)
  └─ uludashboardDataControl table
       ↓
ETL Step 10: Extract to Warehouse
  └─ stage.Extract_Synergetic_luDashboardDataControl (KC-SQL-AGS-01)
       ↓
ETL Step 74: Transform Comments (stage.spTransform_factStudentAssessment_School)
  └─ Reads LockedFlag from control table
  └─ IF LockedFlag = 1 THEN hide grades ('') 
  └─ IF LockedFlag = 0 THEN show grades
       ↓
Power BI Dashboards show/hide data based on transformed results
Control Table Schema
Table: uludashboardDataControl (Production: KC-SQL-SVR-09.Synergetic_NZNTH_KINGSCOL_PRD)
Column	Type	Purpose
Code	VARCHAR(15)	Unique identifier (e.g., T4.SEN.CIE.EQ)
Description	VARCHAR(50)	Human-readable name
LockedFlag	BIT	1 = Hide grades/lock comments, 0 = Show grades/allow updates
ActiveFlag	BIT	1 = Dashboard visible, 0 = Dashboard hidden
ModifiedDate	DATETIME	Last change timestamp
ModifiedUser	VARCHAR(100)	Who made the change
Existing Control Codes (as of Oct 21, 2025)
T1.JUN.A&E       - Term 1 Junior A&E
T1.SEN.A&E       - Term 1 Senior A&E
T1.SEN.CIE.EQ    - Term 1 Senior CIE Equivalent Grade
T2.JUN.A&E       - Term 2 Junior A&E
T2.SEN.A&E       - Term 2 Senior A&E
T2.SEN.CIE.EQ    - Term 2 Senior CIE Equivalent Grade
T3.JUN.A&E       - Term 3 Junior A&E
T3.SEN.A&E       - Term 3 Senior A&E
T3.SEN.CIE.EQ    - Term 3 Senior CIE Equivalent Grade
T4.JUN.A&E       - Term 4 Junior A&E (EXISTED - working)
T4.SEN.CIE.EQ    - Term 4 Senior CIE Eqv Grade (NEW - wasn't working)
REP.COM          - Report Comments control
________________________________________
🔧 Solution Implemented
Files Modified
•	Primary: stage.spTransform_factStudentAssessment_School on KC-SQL-AGS-01
•	Version: Changed from 1.0.6 to 1.0.7
Changes Made (5 modifications)
1. Added T4 Senior Flag Declarations (Line 73)
Before:
DECLARE @T1SenAEFlag BIT, @T1SenCIEFlag BIT, @T2SenAEFlag BIT, @T2SenCIEFlag BIT, @T3SenAEFlag BIT, @T3SenCIEFlag BIT
After:
DECLARE @T1SenAEFlag BIT, @T1SenCIEFlag BIT, @T2SenAEFlag BIT, @T2SenCIEFlag BIT, @T3SenAEFlag BIT, @T3SenCIEFlag BIT, @T4SenAEFlag BIT, @T4SenCIEFlag BIT
2. Added T4 Senior Code Constants (Line 77)
Before:
DECLARE @T1SenAECode VARCHAR(15) = 'T1.SEN.A&E',@T1SenCIECode VARCHAR(15) = 'T1.SEN.CIE.EQ', @T2SenAECode VARCHAR(15) = 'T2.SEN.A&E', @T2SenCIECode VARCHAR(15) = 'T2.SEN.CIE.EQ', @T3SenAECode VARCHAR(15) = 'T3.SEN.A&E', @T3SenCIECode VARCHAR(15) = 'T3.SEN.CIE.EQ'
After:
DECLARE @T1SenAECode VARCHAR(15) = 'T1.SEN.A&E',@T1SenCIECode VARCHAR(15) = 'T1.SEN.CIE.EQ', @T2SenAECode VARCHAR(15) = 'T2.SEN.A&E', @T2SenCIECode VARCHAR(15) = 'T2.SEN.CIE.EQ', @T3SenAECode VARCHAR(15) = 'T3.SEN.A&E', @T3SenCIECode VARCHAR(15) = 'T3.SEN.CIE.EQ', @T4SenAECode VARCHAR(15) = 'T4.SEN.A&E', @T4SenCIECode VARCHAR(15) = 'T4.SEN.CIE.EQ'
3. Added T4 Senior Flag Reads (After Line 86)
Added:
SELECT @T4SenAEFlag = LockedFlag FROM [stage].[Extract_Synergetic_luDashboardDataControl] WHERE Code = @T4SenAECode
SELECT @T4SenCIEFlag = LockedFlag FROM [stage].[Extract_Synergetic_luDashboardDataControl] WHERE Code = @T4SenCIECode
4. Added T4 Conditions to CASE Statement (Line ~468)
Before:
WHEN (AssessAreaResultGroup = 'T3.SEN.ORD' AND @T3SenCIEFlag = 1 AND r.FileYear = DATEPART(YYYY,GETDATE())) THEN ''
ELSE  AssessResultsResult 
END,
After:
WHEN (AssessAreaResultGroup = 'T3.SEN.ORD' AND @T3SenCIEFlag = 1 AND r.FileYear = DATEPART(YYYY,GETDATE())) THEN ''
WHEN (AssessAreaResultGroup = 'T4.SEN.ORD.A&E' AND @T4SenAEFlag = 1 AND r.FileYear = DATEPART(YYYY,GETDATE())) THEN ''
WHEN (AssessAreaResultGroup = 'T4.SEN.ORD' AND @T4SenCIEFlag = 1 AND r.FileYear = DATEPART(YYYY,GETDATE())) THEN ''
ELSE  AssessResultsResult 
END,
5. Updated Version Header (Line ~32)
Added:
Changes: 
Author:		 u.khattak
Date:		 21/10/2025
Version:	 1.0.7
Modifications: Added T4 Senior CIE and A&E flag support (T4.SEN.CIE.EQ, T4.SEN.A&E)
________________________________________
💾 Backup Information
Backup Table Created
Table Name: stage.Backup_spTransform_factStudentAssessment_School_20251021
Server: KC-SQL-AGS-01
Database: KCDW
Created: 2025-10-21 10:10:30.280
Original Procedure Modified Date: 2023-07-11 08:40:06.110
Code Length: 79,602 characters
Backup Table Schema
CREATE TABLE stage.Backup_spTransform_factStudentAssessment_School_20251021 (
    BackupDateTime DATETIME,
    ServerName VARCHAR(100),
    DatabaseName VARCHAR(100),
    ProcedureName VARCHAR(200),
    OriginalCreateDate DATETIME,
    LastModifiedDate DATETIME,
    ProcedureCode NVARCHAR(MAX)
)
Retrieve Backup Query
USE KCDW;
GO

SELECT 
    BackupDateTime,
    ServerName,
    DatabaseName,
    ProcedureName,
    OriginalCreateDate,
    LastModifiedDate,
    LEN(ProcedureCode) AS CodeLength
FROM stage.Backup_spTransform_factStudentAssessment_School_20251021;
________________________________________
🔄 Rollback Procedure
If Changes Need to be Reverted
Step 1: Verify Backup Exists
USE KCDW;
GO

-- Check backup table exists
IF OBJECT_ID('stage.Backup_spTransform_factStudentAssessment_School_20251021', 'U') IS NOT NULL
    PRINT '✓ Backup exists'
ELSE
    PRINT '✗ WARNING: Backup not found!';

-- View backup details
SELECT 
    BackupDateTime,
    OriginalCreateDate,
    LastModifiedDate
FROM stage.Backup_spTransform_factStudentAssessment_School_20251021;
Step 2: Extract Backup Code
-- Get the original procedure code
SELECT ProcedureCode 
FROM stage.Backup_spTransform_factStudentAssessment_School_20251021;
Step 3: Execute Backup Code
1.	Copy the entire ProcedureCode result
2.	Open a new query window in SSMS
3.	Paste the code
4.	Execute (F5)
Step 4: Verify Rollback
-- Check procedure modification date changed
SELECT 
    name AS ProcedureName,
    modify_date AS LastModified
FROM sys.procedures
WHERE SCHEMA_NAME(schema_id) = 'stage' 
    AND name = 'spTransform_factStudentAssessment_School';

-- Verify T4 Senior flags are removed
SELECT 
    CASE WHEN definition LIKE '%@T4SenCIEFlag%' THEN 'T4 Still Exists' 
         ELSE 'T4 Removed - Rollback Successful' 
    END AS RollbackStatus
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School');
________________________________________
✅ Verification Steps
Post-Deployment Validation
1. Verify Procedure Modified Successfully
USE KCDW;
GO

-- Check modification date changed
SELECT 
    name AS ProcedureName,
    create_date AS FirstCreated,
    modify_date AS LastModified,
    DATEDIFF(HOUR, modify_date, GETDATE()) AS HoursSinceModification
FROM sys.procedures
WHERE SCHEMA_NAME(schema_id) = 'stage' 
    AND name = 'spTransform_factStudentAssessment_School';
Expected: LastModified should be October 21, 2025
2. Verify T4 Support Added
-- Comprehensive check for all T4 components
SELECT 
    'T4 Senior A&E Flag Declared' AS CheckType,
    CASE WHEN definition LIKE '%@T4SenAEFlag%' THEN '✓ EXISTS' ELSE '✗ MISSING' END AS Status
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School')

UNION ALL

SELECT 
    'T4 Senior CIE Flag Declared',
    CASE WHEN definition LIKE '%@T4SenCIEFlag%' THEN '✓ EXISTS' ELSE '✗ MISSING' END
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School')

UNION ALL

SELECT 
    'T4 Senior A&E Code Declared',
    CASE WHEN definition LIKE '%T4.SEN.A&E%' THEN '✓ EXISTS' ELSE '✗ MISSING' END
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School')

UNION ALL

SELECT 
    'T4 Senior CIE Code Declared',
    CASE WHEN definition LIKE '%T4.SEN.CIE.EQ%' THEN '✓ EXISTS' ELSE '✗ MISSING' END
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School')

UNION ALL

SELECT 
    'T4 Senior CIE CASE Condition',
    CASE WHEN definition LIKE '%T4.SEN.ORD'' AND @T4SenCIEFlag%' THEN '✓ EXISTS' ELSE '✗ MISSING' END
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('stage.spTransform_factStudentAssessment_School');
Expected: All checks should show ✓ EXISTS
3. Verify Control Table Has Entry
-- Switch to Production database
-- Server: KC-SQL-SVR-09
USE Synergetic_NZNTH_KINGSCOL_PRD;
GO

SELECT 
    Code,
    Description,
    CASE WHEN LockedFlag = 1 THEN 'Locked (Grades Hidden)' 
         ELSE 'Unlocked (Grades Visible)' END AS LockStatus,
    CASE WHEN ActiveFlag = 1 THEN 'Active (Dashboard Visible)' 
         ELSE 'Inactive (Dashboard Hidden)' END AS ActiveStatus,
    ModifiedDate,
    ModifiedUser
FROM uludashboardDataControl
WHERE Code = 'T4.SEN.CIE.EQ';
Expected: Should return 1 row with T4.SEN.CIE.EQ
________________________________________
🧪 Testing Plan
Test Scenario 1: Lock Grades (Hide from Dashboard)
-- On KC-SQL-SVR-09 Production
UPDATE uludashboardDataControl
SET LockedFlag = 1,
    ModifiedDate = GETDATE(),
    ModifiedUser = 'INTERNAL\Your.Username'
WHERE Code = 'T4.SEN.CIE.EQ';
Action: Run ETL job or wait for scheduled run
Expected Result: T4 Senior CIE grades should be blank ('') in Power BI dashboards for current year
Test Scenario 2: Unlock Grades (Show in Dashboard)
-- On KC-SQL-SVR-09 Production
UPDATE uludashboardDataControl
SET LockedFlag = 0,
    ModifiedDate = GETDATE(),
    ModifiedUser = 'INTERNAL\Your.Username'
WHERE Code = 'T4.SEN.CIE.EQ';
Action: Run ETL job or wait for scheduled run
Expected Result: T4 Senior CIE grades should be visible in Power BI dashboards
Test Scenario 3: Monitor ETL Execution
-- On KC-SQL-AGS-01
USE msdb;
GO

-- Check recent ETL job runs
SELECT TOP 5
    j.name AS JobName,
    h.step_name AS StepName,
    h.run_date,
    h.run_time,
    CASE h.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
    END AS Status,
    h.run_duration AS DurationSeconds,
    LEFT(h.message, 200) AS MessagePreview
FROM sysjobs j
INNER JOIN sysjobhistory h ON j.job_id = h.job_id
WHERE j.name = 'KCDW - ETL'
    AND h.step_name = 'ETL - Transform_dimComment'
ORDER BY h.run_date DESC, h.run_time DESC;
Expected: Step should succeed without errors
________________________________________
📋 ETL Job Details
Relevant ETL Steps
Step	Name	Purpose
10	ETL - Extract_Synergetic_luDashboardDataControl	Extracts control table from Production to Warehouse
74	ETL - Transform_dimComment	Calls spTransform_factStudentAssessment_School
82	ETL - Load_dimComment	Loads processed comment data
ETL Schedule
•	Job Name: KCDW - ETL
•	Server: KC-SQL-AGS-01
•	Schedule: Daily execution
•	Duration: 2-4 hours typical
•	Most Failure-Prone Steps: 92, 142, 143
Manual ETL Trigger (if needed)
-- On KC-SQL-AGS-01
USE msdb;
GO

-- Start the ETL job manually
EXEC dbo.sp_start_job @job_name = 'KCDW - ETL';

-- Check job status
EXEC dbo.sp_help_job @job_name = 'KCDW - ETL';
________________________________________
🗂️ Key System Locations
Servers
•	KC-SQL-SVR-09 - Production Database (Synergetic)
•	KC-SQL-AGS-01 - ETL/Data Warehouse (KCDW)
•	KC-SQL-AGS-02 - Analytics Server (SSAS)
Databases
•	Production: Synergetic_NZNTH_KINGSCOL_PRD on KC-SQL-SVR-09
•	Warehouse: KCDW on KC-SQL-AGS-01
•	Development: Dashboards on KC-SQL-SVR-09
Key Tables
•	Control Table (Prod): Synergetic_NZNTH_KINGSCOL_PRD.dbo.uludashboardDataControl
•	Control Table (Warehouse): KCDW.stage.Extract_Synergetic_luDashboardDataControl
•	Backup Table: KCDW.stage.Backup_spTransform_factStudentAssessment_School_20251021
Key Procedures
•	Modified: KCDW.stage.spTransform_factStudentAssessment_School
•	Related: DDS.LoadMerge_factStudentAssessment
Access Paths
•	SSMS: KC-SQL-AGS-01 → Databases → KCDW → Programmability → Stored Procedures → stage
•	Remote Access: Citrix → KC-SQL-MGMT-01 → SSMS
________________________________________
👥 Key Personnel
Technical Contacts
•	Umar Khattak (INTERNAL\u.khattak) - Current Data Analyst
•	Madawa Kannangara (m.kannangara) - Previous Developer (left 2023)
•	Tim Foote - Original Procedure Author (2017)
Business Contacts
•	Geoff Smith (INTERNAL\G.Smith) - Academic Head 
o	Manages control flags
o	Locks/unlocks dashboards for data release
o	Access via: Synergetic → System → Lookup Table Maintenance
Responsibilities
•	Umar: Database development, ETL maintenance, Power BI dashboards
•	Geoff: Control flag management, dashboard release timing
•	Academic Staff: Consume dashboards, enter comments
________________________________________
📝 Historical Context
Timeline
•	2017-04-29: Original procedure created by Tim Foote
•	2018-03-22 to 2018-08-27: Multiple enhancements by m.kannangara (versions 1.0.1 - 1.0.6) 
o	Added A&E and CIE control flag functionality
o	Implemented T1, T2, T3 support
•	2023-07-11: Last modification by m.kannangara before leaving
•	2025-02-24: T4.JUN.A&E added by Geoff
•	2025-04-10 to 2025-09-18: T1, T2, T3 controls added by Geoff throughout the year
•	2025-10-20: T4.SEN.CIE.EQ added by Umar
•	2025-10-21: T4 Senior support added to procedure by Umar
Why T4 Was Missed
1.	Madawa left in mid-2023 before T4 implementation was needed
2.	T4 Junior support was partially added (likely by someone else)
3.	T4 Senior support was never completed
4.	Documentation was incomplete - this gap wasn't obvious
5.	Pattern broke: new terms needed code changes, but this wasn't documented
Lessons Learned
1.	Annual term additions require procedure updates
2.	Control table entries alone are insufficient
3.	Need documentation for yearly maintenance tasks
4.	This document now serves as the reference for future terms
________________________________________
🔮 Future Maintenance
Adding New Term Controls (e.g., T5, T6, etc.)
Step 1: Add Control Entry in Production
-- On KC-SQL-SVR-09
USE Synergetic_NZNTH_KINGSCOL_PRD;
GO

-- Add new term control
INSERT INTO uludashboardDataControl (Code, Description, LockedFlag, ActiveFlag, ModifiedDate, ModifiedUser)
VALUES 
    ('T5.JUN.A&E', 'T5 Junior A&E', 1, 1, GETDATE(), 'INTERNAL\Your.Username'),
    ('T5.SEN.A&E', 'T5 Senior A&E', 1, 1, GETDATE(), 'INTERNAL\Your.Username'),
    ('T5.SEN.CIE.EQ', 'T5 CIE Eqv Gr', 1, 1, GETDATE(), 'INTERNAL\Your.Username');
Step 2: Update ETL Procedure
Follow the same pattern as T4 implementation:
1.	Add flag declarations
2.	Add code constants
3.	Add flag reads
4.	Add CASE statement conditions
5.	Update version number
Step 3: Test
1.	Run ETL manually
2.	Verify flags work (lock/unlock)
3.	Check Power BI dashboards
________________________________________
🚨 Known Issues & Warnings
Critical Notes
1.	Never skip backup - Always create backup before modifying procedures
2.	Test in Dashboards first - Never test directly in Production
3.	Two CASE statements - The procedure has TWO locations where grades are processed (current and past students)
4.	Collation issues - Always use COLLATE DATABASE_DEFAULT when joining
5.	Year filter required - CASE conditions check r.FileYear = DATEPART(YYYY,GETDATE()) to only affect current year
Common Mistakes to Avoid
1.	❌ Only updating one CASE statement (need to update both)
2.	❌ Forgetting to add both A&E and CIE variants
3.	❌ Not adding the flag read SELECT statements
4.	❌ Missing the version number update in header
5.	❌ Testing in Production without Dashboards validation first
________________________________________
📖 Additional Resources
Architecture Reference
See: Kings College Master Architecture Reference.docx for:
•	Complete system architecture
•	ETL troubleshooting
•	Power BI integration details
•	Common issues and solutions
Useful Queries
See STEP validation queries throughout this document
Related Documentation
•	Control table maintenance guide
•	ETL job monitoring procedures
•	Power BI refresh schedules
•	Dashboard deployment process
________________________________________
✨ Summary
This implementation adds T4 Senior (A&E and CIE) support to the Kings College assessment ETL process, enabling Geoff to control the visibility of T4 Senior grades and manage comment updates through the uludashboardDataControl table. The solution follows the existing pattern established for T1-T3 terms and ensures consistency across all term controls.
Status: ✅ IMPLEMENTED - October 21, 2025
Backup: ✅ SECURED - stage.Backup_spTransform_factStudentAssessment_School_20251021
Testing: ⏳ PENDING - Awaiting next ETL run and user validation
________________________________________
Document maintained by: Umar Khattak (INTERNAL\u.khattak)
Last updated: October 21, 2025

✨ Summary
This implementation adds T4 Senior (A&E and CIE) support to the Kings College assessment ETL process, enabling Geoff to control the visibility of T4 Senior grades and manage comment updates through the uludashboardDataControl table. The solution follows the existing pattern established for T1-T3 terms and ensures consistency across all term controls.
Status: ✅ IMPLEMENTED - October 21, 2025
Backup: ✅ SECURED - stage.Backup_spTransform_factStudentAssessment_School_20251021
Testing: ⏳ PENDING - Awaiting next ETL run and user validation
________________________________________
📝 Implementation Log - October 21, 2025
Session Timeline
10:10 AM - Created backup of original procedure
•	Backup confirmed with 79,602 characters
•	Original last modified: 2023-07-11
11:00 AM - Reviewed and prepared modifications
•	Confirmed all 5 change locations identified correctly
•	Used GitHub for version control of changes
11:19 AM - Successfully executed modified procedure
Commands completed successfully.
Completion time: 2025-10-21T11:19:47.4173202+13:00
11:20 AM - Verification completed
-- Verification Results:
Status: ✅ T4 Senior Support CONFIRMED
ModifiedAt: 2025-10-21 11:19:47.377

-- Control Table Status:
T4.SEN.CIE.EQ - Exists (LockedFlag=1, ActiveFlag=1)
T4.SEN.A&E - Not yet created (can be added when needed)
Key Decisions Made
1.	Used ALTER PROCEDURE instead of DROP/CREATE
o	Preserved existing permissions
o	No need to re-grant EXECUTE permissions
o	ETL can continue running without permission changes
2.	T4.SEN.A&E Not Required Yet
o	Procedure supports it when added
o	Geoff can add via Synergetic when needed
o	No additional code changes required
Implementation Verification
✅ Stored Procedure Modified - Version 1.0.7 active ✅ Backup Secured - Recovery procedure available if needed ✅ T4.SEN.CIE.EQ Control - Currently locked (grades hidden) ✅ Permissions Preserved - Using ALTER maintained all grants ⏳ ETL Testing - Awaiting next scheduled run
Next Steps
1.	Monitor next ETL execution for errors
2.	Verify T4 Senior CIE grades are hidden in dashboards
3.	Inform Geoff that controls are functional
4.	Document any issues that arise during ETL
Notes
•	GitHub repository updated with modified procedure
•	No SSAS or Visual Studio updates required
•	Power BI permissions remain unchanged (ALTER preserved them)
•	T4.SEN.A&E can be added to control table when needed
Document maintained by: Umar Khattak (INTERNAL\u.khattak)
Last updated: October 21, 2025
Version: 1.1




Version: 1.0

