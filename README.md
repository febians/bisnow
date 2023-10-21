#### *Q1:  In leads, find all contacts with duplicate contact_employments.*
*Remove the duplicates, keeping the employment with the most data.*
*For example: One has a title, one does not. Pull the employments with a title. If neither employment has a title, keep the most recently created employment.*

#### **My Approach**:

My approach is not to write a very complex query or accomplish the job in one query. But to find the optimum way to solve the problem. Further my approach can be broken down and executed in multiple iterations or smaller batches. This helps reduce the time queries spend in the buffer. Additionally, my approach supports multithreading and can be stopped and restarted at any time without affecting the data. It also allows for application-level partitioning. For example, you can process batches of 10,000 records and clean them before moving on to the next batch.

Assumptions: 
* Latest record has the highest id
* Job Titles are known/limited set: Application has the list of Job Titles or we’ll write one simple select query to fetch ids (run once throughout the lifetime of the application) of all not-null Job Titles. Let’s call it MASTER_JOB_LIST.

Query for MASTER_JOB_LIST:
```SELECT id FROM employments where job_title is not null;```

Data I ran this on: 

![image](https://github.com/febians/bisnow/assets/40207605/3dca80e9-3037-4831-bfad-afaf8ceb44dc)

Step 1) Find all duplicate combinations: 

```SELECT *, COUNT(id) as C,
CONCAT(GROUP_CONCAT(id, "-", employment_id )) as ID_EMPID
FROM contact_employments group by contact_id
HAVING C > 1;
```

Result: 

![image](https://github.com/febians/bisnow/assets/40207605/216f87c8-191c-4dba-8ceb-c00e5a19692e)

Step 2) How to use the result set: 

Repeat for every row: 

  From the right most pair in ID_EMPID - check if the EMPID is in MASTER_JOB_LIST. 
  
  * if found: keep this record and delete all rows from `contact_employments`with all other ids. 
  * if none found: delete all rows from `contact_employments`with all ids except the right most (latest). 


Example: EmpID 4 is NULL. Then the most recent record with a valid title will be 9-2. Therefore delete ID [5,6,10,11,12]

---

### Q2: *Schema update & Query:*
*In leads, quickly lookup exactly one email address assuming the table contact_emails contains 30 million records. Assume the table needs optimization AND a query like SELECT * FROM contact_emails WHERE email = '<email address here>' will not provide the performance we need.*

### **My Approach:**

Create a unique index / hash index on email field in database. 

Pros: 

- Reading records will be fast.

Cons: 

- Creation of a unique/hash index will be time consuming and memory intensive.
- Slower writes for new records as it has to create its index and check for duplicates.

**My Assumption**: I need to reach the record as quickly as possible, thus: 

Create a hash MD5 on the email id and store it in DB. And create an index or hashmap on this value. Example febian@gmail.com is hashed to: 07356e8d96d3ac76e5442f71e053d56c

As the application can create a MD5 for an email, I can directly query “Select id, email from *contact_emails where hashmap_field = "*07356e8d96d3ac76e5442f71e053d56c” and email = “febian@gmail.com”. 

On this assumption, I am not going to create a unique index on email field. Instead I’ll create an index on the hashmap_field. 

*Please note: This assessment is based on a limited understanding of the business and could be improved with a better understanding.*

---
### Q3: Next, we want to track the number of active contacts over time. 

* Criteria to determine active contacts:

  * contacts.deleted_at is null
  * Has an employment
  * Has an email
  * Contact has an employment in one of the following industries:
    * Real estate
    * Media and Real Estate

* Create a table, contacts_active, on schema leads to track the number of contacts. The table should include:
  * Contact count
  * Date

#### **My Approach**:
_I am not exactly sure what the second part of the question is saying here. I have created the solution to the best of my understanding. However, if this is not what you were looking for, please let me know so I can update this correctly. 
_

The result count of the query below is the total number (3 active contacts in the screenshot below) of active contacts. Sample query output: 

![image](https://github.com/febians/bisnow/assets/40207605/ab990322-df5f-45fd-974e-a6cca11d0e1d)

```
SELECT contacts.id as CONTACT_ID, email as EMAIL,
GROUP_CONCAT(DISTINCT employments.id) as EIDs, COUNT(DISTINCT employments.id) as COUNT_OF_EIDs,
GROUP_CONCAT(DISTINCT employments.employment_id) as JOB_POSITION_IDs, COUNT(DISTINCT employments.employment_id) as COUNT_OF_POSITIONs,
first_name as FIRST_NAME, last_name as LAST_NAME
FROM contact_emails AS emails
inner join contact_employments as employments on emails.contact_id = employments.contact_id
inner join contacts as contacts on contacts.id = emails.contact_id
WHERE employments.employment_id in (2,3,4)
group by contacts.id;
```

○ Insert active contacts into the contacts_active table

#### **My Approach**:

Depending on how often the tables are updated, creating a View might be an option over creating a Table. Views are not recommended if there are frequent DML changes. 

```
CREATE VIEW CONTACTS_ACTIVE AS
SELECT contacts.id as CONTACT_ID, email as EMAIL,
GROUP_CONCAT(DISTINCT employments.id) as EIDs, COUNT(DISTINCT employments.id) as COUNT_OF_EIDs,
GROUP_CONCAT(DISTINCT employments.employment_id) as JOB_POSITION_IDs, COUNT(DISTINCT employments.employment_id) as COUNT_OF_POSITIONs,
first_name as FIRST_NAME, last_name as LAST_NAME
FROM contact_emails AS emails
inner join contact_employments as employments on emails.contact_id = employments.contact_id
inner join contacts as contacts on contacts.id = emails.contact_id
WHERE employments.employment_id in (2,3,4)
group by contacts.id;
```

To create the table use: 
```
CREATE TABLE `active_contacts` (
  `id` bigint(20) NOT NULL,
  `contact_id` bigint(11) NOT NULL,
  `eids` varchar(255) NOT NULL,
  `count_of_eids` bigint(20) NOT NULL,
  `job_positions` varchar(255) NOT NULL,
  `count_of_job_positions` int(11) NOT NULL,
  `created_on` int(11) NOT NULL COMMENT 'EPOCH',
  `modified_on` int(11) NOT NULL COMMENT 'EPOCH',
  `modified_by` int(11) NOT NULL COMMENT 'user id as foreign key'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

ALTER TABLE `active_contacts`
  ADD PRIMARY KEY (`id`),
  ADD KEY `contact id` (`contact_id`);

ALTER TABLE `active_contacts`
  ADD CONSTRAINT `contact id` FOREIGN KEY (`contact_id`) REFERENCES `contacts` (`id`);
COMMIT;

```


Creating Stored Procedure:
```
DELIMITER //
CREATE PROCEDURE PRD_CONTACTS_ACTIVE()
BEGIN

SELECT [contacts.id](http://contacts.id/) as CONTACT_ID, email as EMAIL,
GROUP_CONCAT(DISTINCT [employments.id](http://employments.id/)) as EIDs, COUNT(DISTINCT [employments.id](http://employments.id/)) as COUNT_OF_EIDs,
GROUP_CONCAT(DISTINCT employments.employment_id) as JOB_POSITION_IDs, COUNT(DISTINCT employments.employment_id) as COUNT_OF_POSITIONs,
first_name as FIRST_NAME, last_name as LAST_NAME
FROM contact_emails AS emails
inner join contact_employments as employments on emails.contact_id = employments.contact_id
inner join contacts as contacts on [contacts.id](http://contacts.id/) = emails.contact_id
WHERE employments.employment_id in (2,3,4)
group by [contacts.id](http://contacts.id/);
END;
//
DELIMITER
```
Assumptions: 

Create the table only once. The procedure will run only once. However, additional checks should be created for updating records, as the active_contacts table will not be updated.

* Option 1: Create triggers when relevant information about the contacts are added or updated. 
* Option 2: Application layer to update the records in the relevant tables and active_contacts. 
* Option 3: Run the stored procedure as often as required to update the active_contacts. Note: Users will not have access to the latest data at all times. 
---
---
**Non-Assignment related General Performance improvement suggestions:** 

* Dates are in Date format - could be converted to EPOCH for better performance.
* Improve application logic to not allow duplicate generation.
* Use EXPLAIN query to find any optimization opportunities - such as indexes or removing of nested queries.
* Setup slow query logging.
* Setup MAX buffer size & MAX execution time.
* Setup a QA DB with mock data to match business scenarios for query testing & optimization. This will also help developers and create all possible scenarios.
* ***Average*** Query execution time should be monitored
* Put checks before data insertion to avoid any dedupe.
* In the DB - James Farer is a Manager at Bisnow and there is another row of James Farer at Bisnow. How would you want to treat the record if the records were like Record 1: James Farer - Manager at NULL. Record 2: James Farer NULL at Bisnow.
* List an assumption of the database engine: The provided structures are in INNODB with row level locking - which is good enough.
* Version you are writing the SQL for: All the queries provided should be compatible all versions of SQL.
* If you assume ANSI standard, you could list ANSI standard as your assumption: NA.
