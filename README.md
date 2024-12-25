# Database Development

## Objective
The primary goal of this project was to design and implement a secure and efficient database management system (DBMS) for the Hospital. The project aimed to replace an outdated and insecure spreadsheet-based record-keeping system with a robust SQL database that could handle complex relationships while improving data security and accessibility.

## Problem Statement
Hospital faced significant challenges with its existing data management system, which relied on unsecured Excel spreadsheets:
-	Data Vulnerability: Sensitive patient information was prone to unauthorized access and accidental edits.
-	Scalability Issues: The spreadsheet system struggled to manage complex relationships between patients, care plans, and staff.
-	Operational Inefficiencies: Tasks like generating reports or tracking malpractice cases were manual and error-prone.
-	Compliance Risks: The lack of security measures created risks in protecting patient privacy and adhering to data protection regulations.
## Process
1.	Requirements Gathering:
-	Understand the current system and identify critical requirements.

-	Identified the Patient Care Plan as the central entity, connecting patients, staff, and medical conditions.

-	Established the need for features such as malpractice tracking, secure access controls, and automated reporting.

2.	Database Design:
-	Created Entity-Relationship Diagram (ERD):

-	Mapped out relationships between key entities, including Patient, Staff, Allergy, Condition, and Care Plan.

-	Ensured the design reflected real-world operations and supported hospital workflows.

Snapshot of ERD,

![Project Part 1 - ERD](https://github.com/user-attachments/assets/2970b6f3-6139-46e3-ae81-1d949c4e7432)

-   Created a Logical Data Model

-	Normalized the Logic Data Model to the third normal form (3NF) to eliminate redundancy and ensure data integrity.

Snapshot of Normalization,

![Project Part 1 - NORMALIZATION](https://github.com/user-attachments/assets/720edf6e-0935-4a6f-bba8-fa64181f9499)

3.	Implementation:
-	Database Schema: Developed tables for core entities in Microsoft SQL Server, with constraints like primary keys and foreign keys to enforce relationships.

-	Business Logic Automation: Created stored procedures for tasks like generating patient reports and summarizing billing data. 

-   Designed triggers to enforce business rules, such as deactivating doctors with multiple malpractice cases and logging patient deletions.

-	Security Measures: Implemented tiered access controls, limiting data modification to authorized roles such as administrators and medical staff.

4.	Data Reporting:
-	Developed SQL queries to provide actionable insights.

-	Identifying doctors with recurring malpractice cases.

-	Analyzing allergy trends and their severity among patients.

-	Monitoring care plans involving high-risk medications like opioids.

### Stored Procedure 


    IF EXISTS (SELECT NAME FROM SYSOBJECTS WHERE NAME = 'patient_report' AND TYPE = 'P') 
    DROP PROCEDURE patient_report
    GO

    CREATE PROCEDURE patient_report 
    @patient_fname varchar(50) = NULL,
    @patient_lname varchar(50) = NULL

    AS

    DECLARE
    @patid int,
    @pdob varchar(50),
    @reportdate varchar(50)


    SELECT @patid = Patient_ID, @pdob = dob ,@reportdate = GETDATE()
    FROM Patient
    WHERE @patient_fname = fname AND @patient_lname =lname

	    PRINT '*****************************************'
	    PRINT 'Patient ID: ' + convert(varchar,@patid)
	    PRINT 'Patient Name: '+@patient_fname +' ' + @patient_lname
	    PRINT 'Date of Birth: ' + @pdob
	    PRINT 'Report Date: ' + @reportdate
	    PRINT '*****************************************'
	    PRINT '  '
	    PRINT '  '
	    PRINT 'ALLERGIES'
	    PRINT '**********'
	    PRINT 'Allergy Category    Severity Level      Reaction Desc'

	
    DECLARE patient_A_cursor CURSOR FOR
    SELECT Allergy_Category,severity_level,reaction_desc
    FROM Patient_Allergy PA
    INNER JOIN Allergy A ON A.Allergy_ID = PA.Allergy_ID
    INNER JOIN Patient P ON P.Patient_ID = PA.Patient_ID
    WHERE @patient_fname = fname AND @patient_lname =lname

    DECLARE
    @slvl varchar(50),
    @acate varchar(50),
    @redesc varchar(50)



    BEGIN

    OPEN patient_A_cursor

    IF @@CURSOR_ROWS = 0
	    BEGIN
		    RAISERROR ('no patient found',10,1)
		    RETURN
	    END
    FETCH FROM patient_A_cursor INTO
    @acate,@slvl,@redesc

    WHILE @@FETCH_STATUS = 0
	    BEGIN
		    PRINT @acate + '            ' + @slvl + '                ' + @redesc
		    FETCH NEXT FROM patient_A_cursor INTO
		    @acate,@slvl,@redesc
	    END

	    CLOSE patient_A_cursor
	    DEALLOCATE patient_A_cursor
	    END

	    PRINT '  '
	    PRINT '  '
	    PRINT 'BILLING'
	    PRINT '*********'
	    PRINT 'Care ID      Bill Date      Bill Amount'


    DECLARE patient_bill_cursor CURSOR FOR
	
	    SELECT pc.Care_ID, Bill_Date,Bill_Amt
	    FROM Patient_Care pc
	    INNER JOIN Billing b ON pc.Care_ID = b.Care_ID
	    INNER JOIN Patient p ON p.Patient_ID = b.Patient_ID
	    WHERE @patient_fname = fname AND @patient_lname =lname


	    DECLARE
	    @careid smallint,
	    @bdate date,
	    @bamt money


    BEGIN

	    OPEN patient_bill_cursor

	    IF @@CURSOR_ROWS = 0
	    BEGIN
		    RAISERROR ('no patient found',10,1)
		    RETURN
	    END

	    FETCH FROM patient_bill_cursor INTO
	    @careid,@bdate,@bamt

	    WHILE @@FETCH_STATUS = 0
	    BEGIN
		    PRINT convert(varchar,@careid) + '            ' + convert(varchar,@bdate,101) + '     ' + FORMAT(@bamt,'C')
		    FETCH NEXT FROM patient_bill_cursor INTO
		    @careid,@bdate,@bamt
	    END
	    PRINT' '
	    PRINT' '
    CLOSE patient_bill_cursor
    DEALLOCATE patient_bill_cursor
    END


    DECLARE patient_total_cursor CURSOR FOR
	
	    SELECT SUM(Bill_Amt)
	    FROM Patient_Care pc
	    INNER JOIN Billing b ON pc.Care_ID = b.Care_ID
	    INNER JOIN Patient p ON p.Patient_ID = b.Patient_ID
	    WHERE @patient_fname = fname AND @patient_lname =lname


	    DECLARE
	    @totalamt money


    BEGIN

	    OPEN patient_total_cursor

	    IF @@CURSOR_ROWS = 0
	    BEGIN
		    RAISERROR ('no patient found',10,1)
		    RETURN
	    END

	    FETCH FROM patient_total_cursor INTO
	    @totalamt

	    WHILE @@FETCH_STATUS = 0
	    BEGIN
		    PRINT 'Total Bill Amount: ' + FORMAT(@totalamt,'C')
		    FETCH NEXT FROM patient_total_cursor INTO
		    @totalamt
	    END


    CLOSE patient_total_cursor
    DEALLOCATE patient_total_cursor
    END




    GO


    execute patient_report 'Karly', 'Smartman'

### Triggers



#### Trigger 1 - Code

    IF EXISTS (SELECT name FROM sysobjects WHERE name = 'mal_change_status' AND type = 'TR') 
    DROP TRIGGER mal_change_status;
    GO


    /* BEGIN TRIGGER CREATION */
    CREATE TRIGGER mal_change_status
    ON Malpractice AFTER INSERT

    AS

    DECLARE
    @staffid AS smallint,
    @malcount AS smallint

    BEGIN
    SELECT @staffid = staff_ID
    FROM Inserted

    SELECT @malcount = (SELECT COUNT(staff_ID)
    FROM Malpractice
    WHERE Settlement_Judgement = 'Guilty' AND @staffid = staff_ID)

    IF @malcount > 3
	    BEGIN
		    UPDATE Staff
		    SET [status] = 'inactive'
		    WHERE @staffid = Staff_ID
	    END
    END
    GO

#### Trigger 1 - Result

![image](https://github.com/user-attachments/assets/7950a1fd-e786-412e-a6fd-e6b4295d88e5)

#### Trigger 2 - Code

    --Modify Patient Table and Create Log Table
    ALTER TABLE Patient
    ADD status varchar (10)
    update patient set status ='active'

    create table [Log](
    patient_id smallint not null,
    fname varchar(50) not null,
    lname varchar(50) not null,
    [system_user] varchar(50) not null,
    deletion_datetime datetime)

    --Logical Delete Trigger
    IF EXISTS (SELECT name FROM sysobjects WHERE name = 'Logical_delete' AND type = 'TR') 
    DROP TRIGGER Logical_delete;
    GO


    CREATE TRIGGER Logical_delete
    ON Patient INSTEAD OF DELETE

    AS

    declare
    @patientid as smallint,
    @status as varchar(10),
    @fname varchar(50),
    @lname varchar(50),
    @system_user varchar(50),
    @deletion_date datetime

    begin
	    select @patientid = patient_id, @fname = fname, @lname = lname
	    from deleted
	    begin
		    update patient
		    set [status] = 'inactive'
		    where @patientid = Patient_ID
	
		    insert into Log (patient_id, fname, lname, [system_user], deletion_datetime)
		    values(
		    @patientid, 
		    @fname, 
		    @lname, 
		    current_user, 
		    getdate())
	    end
    end
    GO


    set implicit_transactions on
    go

    select * from patient
    where patient_id = 69
    delete from patient where patient_id = 69

    select * from patient
    where patient_id = 69

    select * from Log
    rollback
    set implicit_transactions off
    Go

#### Trigger 2 - Result

![image](https://github.com/user-attachments/assets/752a483e-a68c-4e0d-8e63-9c11c69055f6)

### Queries

#### Query 1 - Code

    select a.allergy_category as [Allergy Category], pa.severity_level as [Severity Level], count(pa.severity_level) as [Patients Affected]
    from Patient_Allergy pa
    inner join Allergy a on a.Allergy_ID = pa.Allergy_ID
    group by a.Allergy_Category, pa.severity_level
    order by count(pa.severity_level) DESC

#### Query 2 - Results:

 ![image](https://github.com/user-attachments/assets/977b7b0c-321e-4962-9a94-69a8184c1e6b)

#### Query 2 - Code

    select d.dept_desc as [Department Name], count(distinct p.patient_id) as [Number of Patients], year(pcare.Completion_Date) as [Completion Year]
    from Department d
    inner join Patient_Care pcare on d.Dept_ID = pcare.dept_id
    inner join Patient_Condition pcon on pcare.condition_id = pcon.Condition_ID
    inner join Patient p on P.Patient_ID = pcon.Patient_ID
    group by d.Dept_Desc, year(pcare.Completion_Date)
    order by count(p.Patient_ID) DESC

#### Query 2 - Results: 

![image](https://github.com/user-attachments/assets/d7d75c4c-c984-4442-8336-abc99856a951)

#### Query 3 - Code

    SELECT m.staff_ID as [Staff ID], s.fname as [First Name], s.lname as [Last Name], count(m.staff_id)
    FROM Malpractice m
    INNER JOIN Staff s
    ON m.staff_ID = s.Staff_ID
    WHERE DATEDIFF (YEAR, GETDATE(), m.malpractice_date) <= 1
    GROUP BY m.staff_ID, s.fname, s.lname
    HAVING count (m.staff_ID) > 3

#### Query 3 - Results:

![image](https://github.com/user-attachments/assets/0a259d3e-6ff9-442b-a777-edd1f752d203)

#### Query 4 - Code

    SELECT P.Patient_ID AS [Patient ID], (p.fname + ' '+ p.lname) AS [Patient Name],pc.staff_id AS [Doctor ID], pc_physician AS [Doctor Name], PC.Care_ID AS [Care Plan ID], PC.Creation_Date AS [Creation Date], M.Med_Name AS [Medication Name], PRE.Dosage, PRE.Frequency
    FROM Patient P
    INNER JOIN Patient_Condition PCO on PCO.Patient_ID = P.Patient_ID 
    INNER JOIN Patient_Care PC on PC.condition_id = PCO.Condition_ID
    INNER JOIN Prescription PRE on PRE.Care_ID = PC.Care_ID
    INNER JOIN Medication M on M.Med_ID = PRE.Med_ID
    WHERE M.IsOpioid = 1

#### Query 4 - Results:

![image](https://github.com/user-attachments/assets/1535ac56-60a7-4414-9ccf-04c411465d37)

#### Query 5 - Code

    SELECT p.Patient_ID, p.fname + ' ' + p.lname AS [Patient Full Name], hs.Admit_Date, hs.Discharge_Date, mc.MC_Category, mc.MC_Type
    FROM Hospital_Stay hs
    INNER JOIN Patient_Care pc
    ON hs.Care_ID = pc.Care_ID
    INNER JOIN Patient_Condition pcon
    ON pc.condition_id = pcon.Condition_ID
    INNER JOIN Patient p
    ON pcon.Patient_ID = p.Patient_ID
    INNER JOIN Medical_Condition mc
    ON pcon.MC_ID = mc.MC_ID
    WHERE DATEDIFF(DAY, hs.Admit_Date, hs.Discharge_Date) >7

#### Query 5 - Results:

![image](https://github.com/user-attachments/assets/3e8bf1f9-5ae8-4fd9-bfea-a0f010ebf8f6)

## Outcome
The project delivered a fully functional, secure, and scalable SQL-based database for HFHS. Key outcomes included:

-	Improved Data Security: Tiered access controls restricted unauthorized modifications and ensured compliance with data protection standards.

-	Enhanced Operational Efficiency: Automated workflows through stored procedures and triggers reduced errors and saved time.

-	Better Insights: SQL queries provided data-driven insights into patient care and staff performance, enabling informed decision-making.

-	Scalability and Maintainability: The database design supported complex relationships and ensured long-term usability for hospital operations.
