# Personal Expense Tracker

CS 665 Introduction to Database Systems | Project Check-in 1 | Fall 2026<br>
Abishek Kumar Giri | Wichita State University

## 1 Problem Definition and Mobile Scope

Small daily purchases are easy to forget, which makes it difficult to see where money went. This Android application will let a person record each expense, place it in a category, review the expense history, filter records by category or date, and see a total amount spent. The target user is an individual who wants a simple personal spending record.

**Scope for this semester.** The application will store one local user profile and its categories and expenses on the Android device. Users can add, edit, and delete expenses, create categories, and view filtered spending history and a total. The project does not include bank connections, payments, account sign-in, cloud synchronization, receipt scanning, or budget predictions. The planned implementation is Kotlin with a local SQLite database through Android Room.

## 2 Initial Database Design

The database has three tables. The User table represents the local profile, Category stores that user’s category names, and Expense stores individual purchases. Each expense belongs to exactly one user and one category owned by that same user.

| Table | Columns and data types | Key information |
| --- | --- | --- |
| User | `user_id INTEGER`; `name TEXT` | PK: `user_id` |
| Category | `category_id INTEGER`; `user_id INTEGER`; `category_name TEXT` | PK: `category_id`; FK: `user_id` → `User.user_id` |
| Expense | `expense_id INTEGER`; `user_id INTEGER`; `category_id INTEGER`; `amount_cents INTEGER`; `expense_date TEXT`; `description TEXT` | PK: `expense_id`; FK: `user_id` → `User.user_id`; FK: (`category_id`, `user_id`) → `Category.(category_id, user_id)` |

**Relationships.** One user has many categories. One user has many expenses. One category has many expenses. An expense references both its user and its category. The two-column foreign key ensures that the selected category belongs to the expense’s user.

**Data choices.** Money is stored as integer cents, so $12.50 is stored as 1250. Dates are stored as ISO text (YYYY-MM-DD), allowing date-range comparisons. The app validates dates before insertion. For the first version, the single local profile has `user_id` 1; this is a data record, not a sign-in system.

## 3 SQL Schema

The following SQLite statements create the initial tables. Foreign-key enforcement must be enabled for each SQLite connection when using the SQL directly. Room will be evaluated when implementing the Android persistence layer.

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE User (
    user_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE Category (
    category_id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    category_name TEXT NOT NULL,
    UNIQUE (user_id, category_name),
    UNIQUE (category_id, user_id),
    FOREIGN KEY (user_id) REFERENCES User(user_id)
);

CREATE TABLE Expense (
    expense_id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    category_id INTEGER NOT NULL,
    amount_cents INTEGER NOT NULL CHECK (amount_cents > 0),
    expense_date TEXT NOT NULL,
    description TEXT,
    FOREIGN KEY (user_id) REFERENCES User(user_id),
    FOREIGN KEY (category_id, user_id)
        REFERENCES Category(category_id, user_id)
);
```

The unique pair (`category_id`, `user_id`) is the referenced key for Expense’s two-column foreign key. This prevents an expense assigned to user 1 from using a category owned by user 2. Category names are unique within a user’s profile. The SQL below uses `user_id = 1` to represent the active local profile.

## 4 Five Core Queries

Each query states the Android feature it supports. The SQL columns match the listed relational algebra attributes. Here π selects columns, σ filters rows, and ⋈ joins matching rows. The `expense_id` column is included when a result must retain distinct expense records. In Query 5, `e` and `c` are short names for Expense and Category.

### Query 1 Expense history

**Feature:** show the local user’s expenses, including amount, date, and description.

```sql
SELECT expense_id, amount_cents, expense_date, description
FROM Expense
WHERE user_id = 1;
```

**Relational algebra:** `π_{expense_id, amount_cents, expense_date, description}(σ_{user_id=1}(Expense))`

### Query 2 Categories

**Feature:** populate the category list and the Add Expense category picker.

```sql
SELECT category_id, category_name
FROM Category
WHERE user_id = 1;
```

**Relational algebra:** `π_{category_id, category_name}(σ_{user_id=1}(Category))`

### Query 3 Filter by category

**Feature:** show this user’s expenses in a selected category, such as Food (`category_id = 2`).

```sql
SELECT expense_id, amount_cents, expense_date, description
FROM Expense
WHERE user_id = 1 AND category_id = 2;
```

**Relational algebra:** `π_{expense_id, amount_cents, expense_date, description}(σ_{user_id=1 ∧ category_id=2}(Expense))`

### Query 4 Filter by date

**Feature:** show expenses entered for a date range chosen on the history screen.

```sql
SELECT expense_id, amount_cents, expense_date, description
FROM Expense
WHERE user_id = 1
  AND expense_date >= '2026-09-01'
  AND expense_date < '2026-10-01';
```

**Relational algebra:** `π_{expense_id, amount_cents, expense_date, description}(σ_{user_id=1 ∧ expense_date≥'2026-09-01' ∧ expense_date<'2026-10-01'}(Expense))`

### Query 5 History with category names

**Feature:** display each expense together with a readable category name.

```sql
SELECT e.expense_id, e.description, e.amount_cents,
       e.expense_date, c.category_name
FROM Expense AS e
JOIN Category AS c
  ON e.category_id = c.category_id
 AND e.user_id = c.user_id
WHERE e.user_id = 1;
```

**Relational algebra:** `π_{e.expense_id, e.description, e.amount_cents, e.expense_date, c.category_name}(σ_{e.user_id=1}(Expense e ⋈_{e.category_id=c.category_id ∧ e.user_id=c.user_id} Category c))`

**Spending total.** The home screen will additionally run `SELECT COALESCE(SUM(amount_cents), 0) FROM Expense WHERE user_id = 1;` and divide the result by 100 for display. `SUM` is an aggregate from extended relational algebra, so this supporting SQL is separate from the five basic relational algebra examples.

## 5 Generative AI Utilization Plan

I plan to use ChatGPT as a tutor and code review assistant while I learn SQL, relational algebra, and Android database integration. I will first write my own schema and queries, test them with sample rows, and compare the output with the expected result. When I am stuck, I will ask for an explanation or a hint. I will revise the solution myself and retest it before adding it to the project. I will keep a short record of important corrections so I can explain my choices without relying on generated answers.

**Agent and purpose:** ChatGPT will help explain database concepts, examine a specific SQL error, and check whether my relational algebra expresses the same query as my SQL. For Android implementation, I will use it to interpret Room errors after attempting the fix myself. I will verify suggestions against my working database and the official documentation.

### Example prompts I intend to use

1. I wrote a relational algebra expression for expenses in a selected category. Explain what each operator does and give me a hint if the expression is wrong, without writing the complete answer.
2. Here is my SQL JOIN between Expense and Category, along with sample rows and the result I expected. Help me find why the category name is wrong.
3. Why does Expense use both `user_id` and `category_id` in its foreign key to Category? Show one valid row and one invalid row.
4. Compare my SQL date filter with my relational algebra expression. Tell me which condition is missing and let me correct it.
5. I received this SQLite or Room error after writing my own code. Explain the cause and suggest a small debugging step before showing a solution.

## 6 Implementation and Verification Plan

I will create sample categories and expenses, run the five queries, and verify that their outputs match the intended screens. I will test an invalid category/user combination to confirm the foreign key rejects it. After the database design works, I will connect the same operations to a small Android interface for entering and viewing expenses. The total-spending calculation will be checked against the sample expense amounts.

## References

- Android Developers. “Save data in a local database using Room.” https://developer.android.com/training/data-storage/room
- SQLite Documentation. “SQLite Foreign Key Support.” https://www.sqlite.org/foreignkeys.html
- SQLite Documentation. “PRAGMA statements.” https://www.sqlite.org/pragma.html#pragma_foreign_keys
