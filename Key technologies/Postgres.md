Q) What is CTE?

Great question! Let’s explore CTEs (Common Table Expressions) — one of PostgreSQL’s most powerful SQL features.

⸻

📘 What is a CTE (Common Table Expression)?

A CTE is a temporary result set that you can reference within a SELECT, INSERT, UPDATE, or DELETE statement.

Think of it like creating a temporary named subquery that makes complex queries more readable and modular.

⸻

✅ Syntax:

WITH cte_name AS (
-- Some SQL query
SELECT ...
)
SELECT ...
FROM cte_name;


⸻

🔍 Why Use CTEs?

Benefit	Example
Improve readability	Break complex logic into steps
Avoid repetition	Reuse the same subquery
Enable recursion	Perform hierarchical queries
Support modular updates	Use with UPDATE, DELETE


⸻

🧪 Example 1: Basic CTE

Problem:

Get the top 2 highest-paid employees from the “Engineering” department.

WITH engineering_employees AS (
SELECT * FROM employees WHERE department = 'Engineering'
)
SELECT *
FROM engineering_employees
ORDER BY salary DESC
LIMIT 2;

🔹 Instead of repeating the filter, we isolate it with a CTE.

⸻

🧪 Example 2: Using multiple CTEs

WITH active_customers AS (
SELECT * FROM customers WHERE is_active = true
),
recent_orders AS (
SELECT * FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT ac.name, ro.order_id, ro.amount
FROM active_customers ac
JOIN recent_orders ro ON ac.id = ro.customer_id;

✅ Breaks complex logic into readable pieces.

⸻

🔁 Example 3: Recursive CTE

Problem: Find all reports (subordinates) of a manager in an employee hierarchy.

WITH RECURSIVE employee_hierarchy AS (
SELECT id, name, manager_id
FROM employees
WHERE manager_id IS NULL  -- Top level

    UNION ALL

    SELECT e.id, e.name, e.manager_id
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy;

🔄 This CTE recursively builds the hierarchy.

⸻

⚠️ Notes:
•	CTEs are not always optimized like subqueries — use EXPLAIN ANALYZE to benchmark.
•	You can nest CTEs and use them with UPDATE, INSERT, and DELETE.

⸻

Would you like to try a quick hands-on CTE challenge next?

Or shall we move to the next advanced topic: LATERAL joins or EXPLAIN + indexing?

Q) Please explain Recursive CTE in detail.

Absolutely! Recursive CTEs are one of the most powerful and elegant features in PostgreSQL for dealing with hierarchical or tree-like data (like org charts, categories, graphs).

Let’s break it down step by step with visuals and SQL examples.

⸻

🔄 What is a Recursive CTE?

A Recursive CTE (Common Table Expression) is used when you need to query hierarchical or recursive data structures such as:
•	Employee → Manager hierarchies
•	Folder → Subfolder trees
•	Graph traversal
•	Category → Subcategory structures

⸻

🧱 Basic Syntax

WITH RECURSIVE cte_name AS (
-- Anchor member (starting point)
SELECT ...

    UNION ALL

    -- Recursive member (uses cte_name itself)
    SELECT ...
    FROM cte_name
    JOIN ...
)
SELECT * FROM cte_name;


⸻

🧑‍🤝‍🧑 Use Case: Employee Hierarchy

Let’s say you have this table:

CREATE TABLE employees (
id INT,
name TEXT,
manager_id INT
);

Sample Data:

id	name	manager_id
1	Alice	NULL
2	Bob	1
3	Carol	1
4	Dave	2
5	Eve	2
6	Frank	3


⸻

🎯 Goal:

Find all subordinates (direct and indirect) under Alice.

⸻

✅ Recursive CTE in Action

WITH RECURSIVE emp_hierarchy AS (
-- Anchor member: start with Alice
SELECT id, name, manager_id
FROM employees
WHERE name = 'Alice'

    UNION ALL

    -- Recursive member: find employees whose manager is in the current result
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    INNER JOIN emp_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM emp_hierarchy;


⸻

🧠 What Happens Internally:

🔹 Iteration 1 (Anchor):
•	Finds Alice (id = 1)

🔹 Iteration 2:
•	Finds Bob and Carol (manager_id = 1)

🔹 Iteration 3:
•	Finds Dave, Eve (manager_id = 2)
•	Finds Frank (manager_id = 3)

🔹 No more matches ⇒ stops

⸻

🔚 Final Result:

id	name	manager_id
1	Alice	NULL
2	Bob	1
3	Carol	1
4	Dave	2
5	Eve	2
6	Frank	3

✅ All levels of subordinates retrieved.

⸻

🔍 Key Rules
•	Always use WITH RECURSIVE, not just WITH.
•	Use UNION ALL (not UNION) unless you want to eliminate duplicates.
•	Anchor query runs once; recursive part runs repeatedly until no new rows.

⸻

🔄 Optional: Add Depth Column

WITH RECURSIVE emp_hierarchy(id, name, manager_id, depth) AS (
SELECT id, name, manager_id, 1
FROM employees
WHERE name = 'Alice'

    UNION ALL

    SELECT e.id, e.name, e.manager_id, eh.depth + 1
    FROM employees e
    JOIN emp_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM emp_hierarchy;

🔢 Now you can see the depth/level of each employee in the hierarchy.

⸻

🛠 Practice Exercise (Optional):

Try writing a recursive CTE for:

Find the full path from a folder to all its subfolders.

Let me know if you want a sample folders table, or shall we move on to LATERAL joins or EXPLAIN & indexing next?