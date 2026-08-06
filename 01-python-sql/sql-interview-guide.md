**SQL INTERVIEW PREPARATION GUIDE**

_Theory Questions +_ **Coding**_/Query Questions_

Compiled from real interview experiences at Amazon, Google, Microsoft, Meta, Flipkart, TCS, Infosys, Cognizant, Accenture, Deloitte, Capgemini, Wipro, EY, KPMG, PwC and more (2025-2026)

_Prepared for: Prashant | July 2026_

# PART 1 - THEORY / CONCEPTUAL QUESTIONS

These are the questions interviewers use to filter candidates before the coding round. They test whether you actually understand concepts, not just syntax.

## 1.1 SQL & RDBMS Fundamentals

**Q1. What is SQL and how is it different from PL/SQL?** _\[Asked at: TCS, Infosys, Wipro, Freshers\]_

**A:** SQL (Structured Query Language) is a declarative language for querying and managing relational data. PL/SQL is Oracle's procedural extension of SQL that adds loops, conditions, variables, and exception handling, allowing complex business logic to be written inside the database (stored procedures, triggers, packages).

**Q2. What is the difference between DBMS and RDBMS?** _\[Asked at: Infosys, Capgemini\]_

**A:** DBMS is software that manages data as files/records without necessarily enforcing relationships (e.g., legacy file systems). RDBMS stores data in tables with rows and columns, enforces relationships via keys, and supports normalization, ACID transactions, and constraints. MySQL, Oracle, SQL Server, PostgreSQL are RDBMS.

**Q3. What are the different types of SQL commands (DDL, DML, DCL, TCL, DQL)?** _\[Asked at: TCS Digital, Accenture\]_

**A:** DDL (CREATE, ALTER, DROP, TRUNCATE) defines schema. DML (INSERT, UPDATE, DELETE) manipulates data. DQL (SELECT) queries data. DCL (GRANT, REVOKE) controls access/permissions. TCL (COMMIT, ROLLBACK, SAVEPOINT) manages transactions.

**Q4. What is the difference between CHAR and VARCHAR (VARCHAR2)?** _\[Asked at: Toptal, Mphasis\]_

**A:** CHAR is fixed-length and pads unused space with blanks, so storage is always the declared length. VARCHAR/VARCHAR2 is variable-length and only uses the space the actual data needs, plus a small length indicator - more efficient when string lengths vary.

**Q5. What is the difference between DELETE, TRUNCATE, and DROP?** _\[Asked at: Hirist, Tech Mahindra, asked everywhere\]_

**A:** DELETE removes rows one at a time (can use WHERE), is logged, fires triggers, and can be rolled back. TRUNCATE removes all rows at once, is minimally logged and much faster, resets identity/auto-increment, and cannot use WHERE - usually cannot be rolled back after commit. DROP removes the entire table structure along with the data, indexes, and constraints permanently.

## 1.2 Keys, Constraints & Data Integrity

**Q6. What is a Primary Key and how is it different from a Unique Key?** _\[Asked at: Capgemini off-campus, asked everywhere\]_

**A:** A Primary Key uniquely identifies each row, disallows NULLs, and only one is allowed per table (can be composite). A Unique Key also enforces uniqueness but permits one NULL value and a table can have multiple unique keys.

**Q7. What is a Foreign Key and what is referential integrity?** _\[Asked at: TCS, Infosys\]_

**A:** A Foreign Key is a column (or set of columns) in one table that references the Primary Key of another table, enforcing that values in the child table must exist in the parent table. This maintains referential integrity and prevents orphaned records.

**Q8. What are the different types of constraints in SQL?** _\[Asked at: Virtusa, Wipro\]_

**A:** NOT NULL, UNIQUE, PRIMARY KEY, FOREIGN KEY, CHECK (restricts values based on a condition), and DEFAULT (assigns a default value when none is supplied).

**Q9. Explain the different types of Normalization (1NF, 2NF, 3NF, BCNF).** _\[Asked at: Asked in almost every experienced-level interview\]_

**A:** 1NF: atomic column values, no repeating groups. 2NF: 1NF plus no partial dependency on part of a composite key. 3NF: 2NF plus no transitive dependency (non-key columns depend only on the key). BCNF: a stricter version of 3NF where every determinant is a candidate key. Normalization reduces redundancy and update anomalies; denormalization is sometimes used deliberately to improve read performance in reporting/warehouse systems.

## 1.3 Joins & Set Operations

**Q10. Explain the different types of JOINs in SQL.** _\[Asked at: Google, Amazon, Meta, Stripe, TCS - the single most-asked SQL topic\]_

**A:** INNER JOIN returns only rows with matches in both tables. LEFT JOIN returns all rows from the left table plus matches from the right (NULL where no match). RIGHT JOIN is the mirror image. FULL OUTER JOIN returns all rows from both tables, with NULLs where there's no counterpart. CROSS JOIN returns the Cartesian product of both tables. SELF JOIN joins a table to itself, typically to compare rows within the same table (e.g., employee-manager hierarchy).

**Q11. What is the difference between UNION and UNION ALL?** _\[Asked at: Infosys lateral hiring, asked everywhere\]_

**A:** UNION combines result sets from two queries and removes duplicate rows (it performs an implicit sort/dedup, so it's slower). UNION ALL combines result sets and keeps all rows including duplicates, making it faster since no deduplication is done.

**Q12. What is the difference between INTERSECT, EXCEPT/MINUS, and JOIN?** _\[Asked at: Data Engineer roles\]_

**A:** INTERSECT returns rows common to both queries. EXCEPT (SQL Server/Postgres) / MINUS (Oracle) returns rows in the first query not present in the second. Unlike JOIN, these operate on full rows (all matching columns) rather than joining on a specific key.

**Q13. Can you JOIN more than two tables? What should you watch out for?** _\[Asked at: Experienced-level rounds\]_

**A:** Yes - chain multiple JOIN clauses. Watch out for: unintended row multiplication (fan-out) when joining one-to-many tables together without aggregating first, missing join conditions causing accidental cross joins, and join order affecting performance on large tables.

## 1.4 Filtering, Grouping & Aggregation

**Q14. What is the difference between WHERE and HAVING?** _\[Asked at: Infosys (Jan 2025 Data Analyst), asked everywhere\]_

**A:** WHERE filters individual rows before grouping/aggregation happens; it cannot reference aggregate functions. HAVING filters groups after GROUP BY has aggregated the data; it's used to filter on aggregate results like COUNT(\*) > 5.

**Q15. What is the order of execution of a SQL query internally?** _\[Asked at: Senior/Architect-level rounds\]_

**A:** Logical order: FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET. This is different from the written order and explains why column aliases from SELECT can't be used in WHERE but can be used in ORDER BY.

**Q16. What are aggregate functions? Name the common ones.** _\[Asked at: Freshers/Infosys\]_

**A:** Functions that operate on a set of values and return a single summary value: COUNT(), SUM(), AVG(), MIN(), MAX(). They ignore NULLs except COUNT(\*).

**Q17. How does NULL behave in comparisons and aggregations?** _\[Asked at: Tricky question - asked at senior levels\]_

**A:** NULL represents unknown/missing data. Any comparison with NULL (=, &lt;, &gt;) returns UNKNOWN, not TRUE or FALSE, so NULL = NULL is not true - use IS NULL / IS NOT NULL. Aggregate functions like SUM, AVG, COUNT(column) ignore NULLs; COUNT(\*) counts all rows regardless.

## 1.5 Subqueries & CTEs

**Q18. What is a subquery, and what's the difference between a correlated and non-correlated subquery?** _\[Asked at: Very common at all levels\]_

**A:** A subquery is a query nested inside another query, used to filter results or compute a derived value. A non-correlated subquery runs once, independently of the outer query. A correlated subquery references a column from the outer query and re-executes once per outer row, which can be slower on large tables.

**Q19. What is the difference between IN, EXISTS, and JOIN for checking related data?** _\[Asked at: Hirist, Ola backend team\]_

**A:** IN compares a value against a list/subquery result and can behave unexpectedly with NULLs. EXISTS checks only for the presence of at least one matching row and stops as soon as one is found - often faster for correlated checks on large tables. JOIN combines actual columns from both tables and is preferred when you also need columns from the second table in the output.

**Q20. What is a CTE (Common Table Expression) and why use it over a subquery?** _\[Asked at: Data Engineering & Analyst roles\]_

**A:** A CTE (WITH clause) is a named temporary result set that exists only for the duration of the query. It improves readability by breaking a complex query into logical steps, can be referenced multiple times in the same query, and supports recursion (recursive CTE) - something a plain subquery cannot do.

**Q21. What is a recursive CTE and where would you use one?** _\[Asked at: Senior Data Engineer rounds\]_

**A:** A CTE that references itself, used to walk hierarchical or graph-like data such as an organization chart, bill-of-materials, or generating a series of numbers/dates. It has an anchor member (base case) and a recursive member, combined with UNION ALL, that repeats until no new rows are produced.

## 1.6 Window Functions (now considered a 'table stakes' topic, not advanced)

**Q22. What are window functions and how are they different from GROUP BY / aggregate functions?** _\[Asked at: Amazon, Snowflake/BigQuery shops - asked constantly in 2026\]_

**A:** GROUP BY collapses multiple rows into a single aggregated row per group - you lose the individual rows. Window functions (using OVER()) compute a value across a set of related rows (the 'window') but keep every individual row in the output, attaching the computed value to each row.

**Q23. What is the difference between ROW_NUMBER(), RANK(), and DENSE_RANK()?** _\[Asked at: TCS BFSI, asked everywhere\]_

**A:** ROW_NUMBER() assigns a unique sequential number to each row with no gaps or ties. RANK() assigns the same rank to tied rows but skips subsequent rank numbers (1,2,2,4). DENSE_RANK() also assigns the same rank to ties but does not skip numbers afterward (1,2,2,3).

**Q24. What is the difference between LEAD and LAG?** _\[Asked at: Amazon, Data Analyst roles\]_

**A:** Both compare a row to another row without a self-join. LAG() fetches a value from a previous row (based on the ORDER BY in the OVER clause), commonly used for period-over-period comparisons. LEAD() fetches a value from a following row.

**Q25. What does PARTITION BY do in a window function?** _\[Asked at: Flipkart Data Engineer\]_

**A:** It divides the result set into partitions/groups (similar to GROUP BY) but the window function is then applied independently within each partition, while all rows remain in the output - unlike GROUP BY which produces one row per group.

## 1.7 Indexing, Query Optimization & Performance

**Q26. What is indexing and how does it improve query performance?** _\[Asked at: Asked at all experience levels\]_

**A:** An index is a separate data structure (typically a B-tree) that stores column values along with pointers to the corresponding rows, allowing the database to locate rows without scanning the entire table - trading some write performance and storage for much faster reads.

**Q27. What is the difference between a Clustered and a Non-Clustered index?** _\[Asked at: Senior Developer rounds\]_

**A:** A clustered index determines the physical storage order of the table's rows - there can be only one per table, usually the primary key. Range queries on it are fast because matching rows are physically adjacent. A non-clustered index is a separate structure holding pointers to the actual rows (which are stored elsewhere), so a table can have many non-clustered indexes, but random lookups via them are slower than sequential clustered-index scans.

**Q28. What is a covering index?** _\[Asked at: Senior/Architect rounds\]_

**A:** An index that contains all the columns required to satisfy a query (in the index itself or as included columns), so the database can answer the query directly from the index without a separate lookup into the base table - significantly reducing I/O.

**Q29. How would you go about optimizing a slow-running query with multiple joins?** _\[Asked at: Ola backend data team, common real-world question\]_

**A:** Check the execution plan (EXPLAIN / EXPLAIN ANALYZE) to see where time is spent. Ensure proper indexes exist on join and filter columns. Replace IN with EXISTS where appropriate for correlated checks. Break very complex queries into CTEs or temp tables. Avoid wrapping indexed columns in functions inside WHERE (this disables index usage). Filter down large tables as early as possible and join smaller, filtered result sets first.

**Q30. What are database isolation levels and why do they matter?** _\[Asked at: Senior Developer - noted as a commonly-missed topic in 2026\]_

**A:** Isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable) control how much one transaction can 'see' of another transaction's uncommitted or in-progress changes, trading off consistency against concurrency. Higher isolation reduces anomalies like dirty reads, non-repeatable reads and phantom reads but increases locking/contention.

## 1.8 Transactions, ACID & Concurrency

**Q31. What are the ACID properties of a transaction?** _\[Asked at: Asked at every experience level\]_

**A:** Atomicity - a transaction either fully completes or fully rolls back. Consistency - a transaction moves the database from one valid state to another, respecting constraints. Isolation - concurrent transactions don't interfere with each other's intermediate state. Durability - once committed, changes survive even a system crash.

**Q32. What is the difference between COMMIT, ROLLBACK, and SAVEPOINT?** _\[Asked at: TCS, freshers\]_

**A:** COMMIT permanently saves all changes made in the current transaction. ROLLBACK undoes changes made since the last COMMIT (or to a SAVEPOINT). SAVEPOINT marks an intermediate point within a transaction that you can roll back to without undoing the entire transaction.

**Q33. What is a deadlock and how can it be avoided?** _\[Asked at: Senior Developer/DBA rounds\]_

**A:** A deadlock occurs when two or more transactions each hold a lock the other needs, so neither can proceed. It can be reduced by always acquiring locks in a consistent order across transactions, keeping transactions short, using appropriate isolation levels, and letting the database's deadlock detector roll back one of the transactions automatically.

## 1.9 Views, Stored Procedures, Functions & Misc

**Q34. What is a View and what is a Materialized View?** _\[Asked at: Asked frequently at mid-to-senior levels\]_

**A:** A View is a virtual table defined by a stored SELECT query - it has no data of its own and is re-executed each time it's queried. A Materialized View physically stores the result set on disk and must be refreshed periodically, trading storage and staleness for much faster read performance on expensive queries.

**Q35. What is the difference between a Stored Procedure and a Function?** _\[Asked at: Common across MNCs\]_

**A:** A stored procedure can perform actions (INSERT/UPDATE/DELETE), doesn't have to return a value, can have input/output parameters, and cannot generally be used inside a SELECT statement. A function must return a value, is typically used within expressions/SELECT statements, and (in most RDBMS) cannot modify database state.

**Q36. What is a trigger, and when would you use one?** _\[Asked at: Cognizant, TCS\]_

**A:** A trigger is a block of code that automatically executes in response to an INSERT, UPDATE, or DELETE event on a table. Common uses: auditing/logging changes, enforcing complex business rules that constraints can't express, and maintaining denormalized/summary data.

**Q37. What is the difference between a Star Schema and a Snowflake Schema?** _\[Asked at: Data Warehousing / MDM-adjacent roles\]_

**A:** A Star Schema has a central fact table connected directly to denormalized dimension tables - simple and fast to query. A Snowflake Schema normalizes those dimension tables into sub-dimensions, reducing redundancy at the cost of more joins and complexity.

# PART 2 - CODING / QUERY-WRITING QUESTIONS

These are the hands-on questions you'll be asked to actually write and run (or write on a whiteboard/shared doc). Sample schema used throughout: Employees(emp_id, name, salary, dept_id, manager_id, hire_date), Departments(dept_id, dept_name), Orders(order_id, customer_id, order_date, amount).

## 2.1 Basic Query Questions

**Q1. Write a query to fetch all employees along with their department names.** _\[Asked at: Freshers, TCS Digital\]_

**A:** Simple INNER JOIN between Employees and Departments on dept_id.

SELECT e.name, e.salary, d.dept_name

FROM Employees e

JOIN Departments d ON e.dept_id = d.dept_id;

**Q2. Write a query to find employees whose name starts with 'A'.** _\[Asked at: Internshala/entry-level roles\]_

**A:** Use LIKE with the % wildcard.

SELECT \* FROM Employees

WHERE name LIKE 'A%';

**Q3. Write a query to fetch the top 5 highest paid employees.** _\[Asked at: Amazon, Data Analyst screens\]_

**A:** Order by salary descending and limit the result set.

SELECT name, salary

FROM Employees

ORDER BY salary DESC

LIMIT 5;

**Q4. Write a query to count the number of employees in each department.** _\[Asked at: Wipro Graduate Trainee assessment\]_

**A:** GROUP BY dept_id with COUNT(\*).

SELECT dept_id, COUNT(\*) AS emp_count

FROM Employees

GROUP BY dept_id;

**Q5. Write a query to update an employee's salary by 10%.** _\[Asked at: TCS Digital\]_

**A:** Simple UPDATE with a WHERE clause to avoid touching every row.

UPDATE Employees

SET salary = salary \* 1.10

WHERE name = 'John';

## 2.2 Aggregation & GROUP BY Scenarios

**Q6. Write a query to find departments having more than 10 employees.** _\[Asked at: Mu Sigma SQL simulation round\]_

**A:** HAVING is required here because the filter is on an aggregated value (COUNT), which WHERE cannot reference.

SELECT dept_id, COUNT(\*) AS emp_count

FROM Employees

GROUP BY dept_id

HAVING COUNT(\*) > 10;

**Q7. Write a query to find the average salary per department, only for departments with more than 5 employees, sorted by average salary descending.** _\[Asked at: GfG-style scenario question\]_

**A:** Combine GROUP BY, HAVING, and ORDER BY together.

SELECT dept_id, AVG(salary) AS avg_salary, COUNT(\*) AS emp_count

FROM Employees

GROUP BY dept_id

HAVING COUNT(\*) > 5

ORDER BY avg_salary DESC;

**Q8. Write a query to find customers who placed more than 5 orders in 2025.** _\[Asked at: GfG scenario / Data Analyst\]_

**A:** Filter by date range in WHERE, then aggregate and filter the count in HAVING.

SELECT customer_id, COUNT(\*) AS orders_2025

FROM Orders

WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01'

GROUP BY customer_id

HAVING COUNT(\*) > 5;

**Q9. Write a query to find duplicate rows/records in a table (e.g., duplicate emails).** _\[Asked at: Very frequently asked at every company\]_

**A:** GROUP BY the column(s) that should be unique and filter groups with COUNT > 1.

SELECT email, COUNT(\*) AS cnt

FROM Employees

GROUP BY email

HAVING COUNT(\*) > 1;

**Q10. Write a query to delete duplicate rows, keeping only one copy of each.** _\[Asked at: Common follow-up to the above\]_

**A:** Use ROW_NUMBER() to tag duplicates within each group, then delete everything except row number 1.

WITH ranked AS (

SELECT \*,

ROW_NUMBER() OVER (PARTITION BY email ORDER BY emp_id) AS rn

FROM Employees

)

DELETE FROM ranked WHERE rn > 1;

## 2.3 Subquery & Nth-Highest Value Problems

**Q11. Write a query to find the second highest salary from the Employees table.** _\[Asked at: Deloitte, KPMG, EY, Flipkart, PwC, TCS - the single most-asked SQL coding question overall\]_

**A:** Classic correlated-subquery approach: find the max salary that is still less than the overall max.

SELECT MAX(salary) AS second_highest

FROM Employees

WHERE salary < (SELECT MAX(salary) FROM Employees);

**Q12. Write a query to find the Nth highest salary (generalized), e.g., the 3rd highest.** _\[Asked at: Follow-up asked in almost every interview\]_

**A:** DENSE_RANK() handles ties correctly (two people tied for 2nd both count as 2nd, next distinct salary is 3rd).

SELECT salary

FROM (

SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk

FROM Employees

) t

WHERE rnk = 3;

**Q13. Write a query to find the department with the highest average salary.** _\[Asked at: TCS BFSI Division\]_

**A:** Aggregate first, then order and limit to 1 (or use RANK if ties should be shown).

SELECT dept_id, AVG(salary) AS avg_salary

FROM Employees

GROUP BY dept_id

ORDER BY avg_salary DESC

LIMIT 1;

**Q14. Write a query to find employees who earn more than the average salary of their own department.** _\[Asked at: Correlated subquery classic\]_

**A:** The subquery is correlated - it recalculates the department average per outer row's dept_id.

SELECT e.name, e.salary, e.dept_id

FROM Employees e

WHERE e.salary > (

SELECT AVG(e2.salary)

FROM Employees e2

WHERE e2.dept_id = e.dept_id

);

**Q15. Write a query to find employees who have no manager assigned (top of the hierarchy).** _\[Asked at: Self-referencing table scenario\]_

**A:** Simple NULL check on the self-referencing foreign key.

SELECT emp_id, name

FROM Employees

WHERE manager_id IS NULL;

**Q16. Write a query to display each employee along with their manager's name (self-join).** _\[Asked at: Very common self-join question\]_

**A:** Join the Employees table to itself, aliasing once for the employee and once for the manager.

SELECT e.name AS employee, m.name AS manager

FROM Employees e

LEFT JOIN Employees m ON e.manager_id = m.emp_id;

## 2.4 Window Function Scenarios

**Q17. Write a query to find the highest-paid employee in each department.** _\[Asked at: Asked by TCS BFSI Division and many product companies\]_

**A:** Use RANK() (or DENSE_RANK) partitioned by department, then filter to rank 1 in an outer query.

SELECT \*

FROM (

SELECT \*, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk

FROM Employees

) t

WHERE rk = 1;

**Q18. Write a query to fetch the top 3 highest paid employees per department.** _\[Asked at: Flipkart Reporting Analyst role\]_

**A:** DENSE_RANK() avoids skipping ranks so exactly the top 3 distinct salary tiers are returned per department.

SELECT \*

FROM (

SELECT \*, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk

FROM Employees

) t

WHERE rk <= 3;

**Q19. Write a query to calculate a running total of order amounts ordered by date.** _\[Asked at: Swiggy Analyst role screening\]_

**A:** SUM() as a window function with an ORDER BY inside OVER() gives a cumulative running total.

SELECT order_id, order_date, amount,

SUM(amount) OVER (ORDER BY order_date) AS running_total

FROM Orders;

**Q20. Write a query to fetch the 5 most recent orders per customer.** _\[Asked at: Meesho Business Data Analyst role\]_

**A:** ROW_NUMBER() partitioned by customer, ordered by date descending, then filter row_num <= 5.

SELECT \*

FROM (

SELECT \*, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn

FROM Orders

) t

WHERE rn <= 5;

**Q21. Write a query showing each employee's salary along with the salary of the previous and next employee (by hire date).** _\[Asked at: Tests LEAD/LAG understanding\]_

**A:** LAG() looks one row back, LEAD() looks one row forward, both ordered by hire_date.

SELECT name, hire_date, salary,

LAG(salary) OVER (ORDER BY hire_date) AS prev_salary,

LEAD(salary) OVER (ORDER BY hire_date) AS next_salary

FROM Employees;

**Q22. Write a query to calculate month-over-month growth in total order amount.** _\[Asked at: Data Analyst / BI roles\]_

**A:** Aggregate by month first in a CTE, then use LAG() to compare against the prior month.

WITH monthly AS (

SELECT DATE_TRUNC('month', order_date) AS mth, SUM(amount) AS total_amt

FROM Orders

GROUP BY DATE_TRUNC('month', order_date)

)

SELECT mth, total_amt,

total_amt - LAG(total_amt) OVER (ORDER BY mth) AS mom_change

FROM monthly;

## 2.5 CTE, Recursive Query & Pivot Scenarios

**Q23. Write a query using a CTE to simplify a multi-step aggregation.** _\[Asked at: Asked to test readability/structuring skills\]_

**A:** Break the problem into a named step, then query the CTE like a table.

WITH dept_avg AS (

SELECT dept_id, AVG(salary) AS avg_sal

FROM Employees

GROUP BY dept_id

)

SELECT e.name, e.salary, d.avg_sal

FROM Employees e

JOIN dept_avg d ON e.dept_id = d.dept_id

WHERE e.salary > d.avg_sal;

**Q24. Write a recursive CTE to display the full management hierarchy for a given employee.** _\[Asked at: Senior Data Engineer rounds\]_

**A:** Anchor member starts at the given employee; recursive member walks up via manager_id until it reaches NULL.

WITH RECURSIVE mgmt_chain AS (

SELECT emp_id, name, manager_id, 1 AS lvl

FROM Employees

WHERE emp_id = 101

UNION ALL

SELECT e.emp_id, e.name, e.manager_id, mc.lvl + 1

FROM Employees e

JOIN mgmt_chain mc ON e.emp_id = mc.manager_id

)

SELECT \* FROM mgmt_chain;

**Q25. Write a query to generate a series of numbers from 1 to 100 without a numbers table.** _\[Asked at: Hirist advanced round\]_

**A:** A recursive CTE (or generate_series in PostgreSQL) builds the sequence on the fly.

\-- PostgreSQL

SELECT generate_series(1, 100);

\-- SQL Server (recursive CTE)

WITH numbers AS (

SELECT 1 AS num

UNION ALL

SELECT num + 1 FROM numbers WHERE num < 100

)

SELECT \* FROM numbers OPTION (MAXRECURSION 0);

**Q26. Write a query to pivot monthly sales rows into columns (Jan, Feb, Mar...).** _\[Asked at: Asked in reporting/BI-heavy roles\]_

**A:** Conditional aggregation with CASE WHEN inside SUM turns each month's rows into their own column.

SELECT product_id,

SUM(CASE WHEN MONTH(order_date) = 1 THEN amount ELSE 0 END) AS Jan,

SUM(CASE WHEN MONTH(order_date) = 2 THEN amount ELSE 0 END) AS Feb,

SUM(CASE WHEN MONTH(order_date) = 3 THEN amount ELSE 0 END) AS Mar

FROM Orders

GROUP BY product_id;

**Q27. Write a query to create an empty copy of a table's structure (no data).** _\[Asked at: InterviewBit classic\]_

**A:** Use a WHERE clause that is always false so the SELECT INTO creates the structure but inserts zero rows.

SELECT \* INTO Employees_copy

FROM Employees

WHERE 1 = 2;

## 2.6 Real-World / Scenario-Based Problems

**Q28. Given a login/logout timestamp table, find sessions where a user was active for more than 30 minutes.** _\[Asked at: Product analytics roles (Meta/Amazon-style)\]_

**A:** Compute the difference between logout and login timestamps and filter on the interval.

SELECT user_id, login_time, logout_time,

TIMESTAMPDIFF(MINUTE, login_time, logout_time) AS session_minutes

FROM Sessions

WHERE TIMESTAMPDIFF(MINUTE, login_time, logout_time) > 30;

**Q29. Given a hospital admissions table, find patients readmitted within 30 days of a previous discharge.** _\[Asked at: Healthcare/product analytics case question\]_

**A:** Self-join the table on patient_id, matching the second admission date within 30 days of the first discharge date.

SELECT a1.patient_id, a1.discharge_date, a2.admit_date

FROM Admissions a1

JOIN Admissions a2

ON a1.patient_id = a2.patient_id

AND a2.admit_date > a1.discharge_date

AND a2.admit_date <= a1.discharge_date + INTERVAL '30 days';

**Q30. Given a social-network 'connections' table, find second-degree connections (friends of friends), excluding direct friends.** _\[Asked at: Common at product/social companies\]_

**A:** A self-join twice: first hop to direct connections, second hop to their connections, then exclude the original person and their direct friends.

SELECT DISTINCT c2.friend_id AS second_degree

FROM Connections c1

JOIN Connections c2 ON c1.friend_id = c2.user_id

WHERE c1.user_id = 1

AND c2.friend_id != 1

AND c2.friend_id NOT IN (

SELECT friend_id FROM Connections WHERE user_id = 1

);

**Q31. Find the busiest hour of the day and day of week for user sessions (product analytics style).** _\[Asked at: Common product-analytics SQL question\]_

**A:** Extract hour and weekday from the timestamp, group by both, and rank by session count.

SELECT EXTRACT(DOW FROM session_start) AS day_of_week,

EXTRACT(HOUR FROM session_start) AS hour_of_day,

COUNT(\*) AS session_count

FROM Sessions

GROUP BY 1, 2

ORDER BY session_count DESC;

**Q32. Calculate profit margin (revenue - cost) / revenue per product category.** _\[Asked at: Data analysis / finance-adjacent roles\]_

**A:** A derived-metric aggregation query grouping by category.

SELECT category,

SUM(revenue) AS total_revenue,

SUM(cost) AS total_cost,

(SUM(revenue) - SUM(cost)) / NULLIF(SUM(revenue), 0) AS margin

FROM Sales

GROUP BY category;

# APPENDIX - Interview Prep Notes

## How companies typically weight this in practice

- Product companies (Amazon, Google, Meta, Flipkart, Swiggy): heavy on window functions, CTEs, self-joins, and scenario/business-context problems on real-looking schemas. Very light on pure theory/definitions.
- Service companies (TCS, Infosys, Wipro, Accenture, Cognizant, Capgemini): mix of theory definitions (joins, keys, normalization, DELETE vs TRUNCATE) plus foundational coding (Nth highest salary, duplicates, GROUP BY/HAVING).
- Consulting/Big-4 (Deloitte, EY, KPMG, PwC): Nth highest salary and its variants come up disproportionately often, alongside CASE WHEN/pivot-style reporting queries.
- Senior/Architect-level rounds (any company): isolation levels, indexing internals (clustered vs non-clustered, covering indexes), execution plans, and query optimization trade-offs.

## Preparation checklist

- Be able to write JOINs, GROUP BY/HAVING, and subqueries cold, without hesitation.
- Be fluent in window functions (ROW_NUMBER, RANK, DENSE_RANK, LEAD, LAG, running totals) - treated as a baseline skill in 2026, not a bonus.
- Practice explaining your query out loud step by step - interviewers weigh reasoning as much as the final syntax.
- Know the SQL dialect used by the company you're interviewing with (MySQL vs PostgreSQL vs SQL Server vs Snowflake/BigQuery) - syntax for LIMIT, date functions, and window filtering (QUALIFY in BigQuery) differs.
- For senior/architect roles, be ready to discuss execution plans, indexing strategy, and how you'd optimize a real slow query - not just write correct syntax.
