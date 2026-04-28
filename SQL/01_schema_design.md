# Relational Schema Design & Normalization

> Pattern-first. Each normalization form gets a broken example and a fixed one. Classic interview schemas come last — study the reasoning, not just the tables.

---

## Normal Forms

### 1NF — Atomic Values, No Repeating Groups

**The rule:** Every column holds one indivisible value. No comma-separated lists, no array-in-a-cell.

**Bad:**

```sql
CREATE TABLE orders (
    order_id   INT PRIMARY KEY,
    customer   TEXT,
    products   TEXT  -- "widget,gadget,gizmo"  ← violation
);
```

**Problem:** You cannot query "find all orders containing widget" without a `LIKE '%widget%'` hack. You cannot join to a products table. You cannot count items.

**Fixed:**

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id)
);

CREATE TABLE order_items (
    order_id    INT REFERENCES orders(order_id),
    product_id  INT REFERENCES products(id),
    quantity    INT NOT NULL DEFAULT 1,
    PRIMARY KEY (order_id, product_id)
);
```

---

### 2NF — No Partial Dependencies (applies to composite PKs)

**The rule:** Every non-key column must depend on the *entire* primary key, not just part of it.

**Bad:**

```sql
CREATE TABLE order_items (
    order_id      INT,
    product_id    INT,
    quantity      INT,
    product_name  TEXT,  -- depends only on product_id, not the composite PK
    unit_price    NUMERIC,  -- same violation
    PRIMARY KEY (order_id, product_id)
);
```

**Problem:** If you rename a product, you update thousands of order_item rows. `product_name` belongs to the product, not to the order-product pair.

**Fixed:**

```sql
CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    product_name  TEXT NOT NULL,
    unit_price    NUMERIC NOT NULL
);

CREATE TABLE order_items (
    order_id    INT REFERENCES orders(order_id),
    product_id  INT REFERENCES products(product_id),
    quantity    INT NOT NULL DEFAULT 1,
    -- price_at_purchase is ok here: it's a snapshot, specific to this order
    price_snapshot NUMERIC NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

---

### 3NF — No Transitive Dependencies

**The rule:** Non-key columns must not depend on other non-key columns.

**Bad:**

```sql
CREATE TABLE employees (
    emp_id        INT PRIMARY KEY,
    emp_name      TEXT,
    dept_id       INT,
    dept_name     TEXT,  -- depends on dept_id, not emp_id
    dept_location TEXT   -- same violation
);
```

**Problem:** If the HR department moves to a new building, you update every employee row in that department. Two rows can have the same `dept_id` but different `dept_name` — instant inconsistency.

**Fixed:**

```sql
CREATE TABLE departments (
    dept_id       INT PRIMARY KEY,
    dept_name     TEXT NOT NULL,
    dept_location TEXT
);

CREATE TABLE employees (
    emp_id   INT PRIMARY KEY,
    emp_name TEXT NOT NULL,
    dept_id  INT REFERENCES departments(dept_id)
);
```

---

### BCNF — Boyce-Codd Normal Form (stricter 3NF)

**The rule:** Every determinant must be a candidate key. Catches edge cases 3NF misses when there are overlapping composite candidate keys.

**Bad (a rare but real case):**

```sql
-- A teacher can teach multiple subjects.
-- A subject has exactly one room.
-- A teacher is assigned to exactly one room per subject they teach.
-- But a room is used for only one subject at a time.
-- Candidate keys: (teacher, subject) and (teacher, room)
CREATE TABLE teaching (
    teacher  TEXT,
    subject  TEXT,
    room     TEXT  -- room → subject, but room is not a candidate key alone
);
```

**Problem:** `room` determines `subject`, but `room` alone is not a candidate key. If you add a new room-subject pairing, you repeat subject info redundantly.

**Fixed:** Decompose so that every determinant is a key.

```sql
CREATE TABLE room_subjects (
    room    TEXT PRIMARY KEY,
    subject TEXT NOT NULL
);

CREATE TABLE teacher_rooms (
    teacher TEXT,
    room    TEXT REFERENCES room_subjects(room),
    PRIMARY KEY (teacher, room)
);
```

---

## When to Denormalize Intentionally

Normalization is the default. Denormalize only when you have a measured performance problem and a clear trade-off.

| Scenario | Denormalization technique | Trade-off |
|---|---|---|
| Read-heavy reporting (dashboards, analytics) | Precomputed aggregate columns (`total_order_value` on `orders`) | Writes must update two places |
| High-join query on hot path | Store redundant foreign-key label (`user_name` on `posts`) | Updates to username need two tables |
| Time-series / immutable events | Flatten all fields into one wide event row | Data grows fast; no normalization benefit for immutable data |
| Full-text search fields | Duplicate a `search_text` column combining multiple fields | Sync complexity |
| Sharded systems | Embed data that would require cross-shard joins | Duplication is the cost of horizontal scale |

**Signal to denormalize:** a query that joins 5+ tables runs in the hot path of every page load. **Not** "joins feel slow in development."

---

## Common Schema Patterns

### One-to-Many

The default pattern. Parent has many children; each child belongs to one parent.

```sql
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    email      TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE posts (
    post_id    SERIAL PRIMARY KEY,
    user_id    INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    title      TEXT NOT NULL,
    body       TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### Many-to-Many (Junction Table)

Two entities where each can relate to many of the other. The junction table holds the relationship plus any attributes of the relationship itself.

```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL
);

CREATE TABLE courses (
    course_id SERIAL PRIMARY KEY,
    title     TEXT NOT NULL
);

CREATE TABLE enrollments (
    student_id  INT REFERENCES students(student_id),
    course_id   INT REFERENCES courses(course_id),
    enrolled_at DATE NOT NULL DEFAULT CURRENT_DATE,
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

---

### Self-Referential (Org Chart, Category Trees)

A row references another row in the same table.

```sql
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    manager_id INT REFERENCES employees(emp_id)  -- NULL for the CEO
);

-- Querying the hierarchy requires a recursive CTE (see file 03)
```

For deep trees with frequent subtree queries, consider the **closure table** pattern instead:

```sql
CREATE TABLE category_paths (
    ancestor_id   INT REFERENCES categories(category_id),
    descendant_id INT REFERENCES categories(category_id),
    depth         INT NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);
-- Every node stores a row for each of its ancestors (including itself at depth 0).
-- "Get all descendants of category 5" = WHERE ancestor_id = 5 ORDER BY depth
```

---

### Polymorphic Associations

One table has a foreign key that can point to rows in different tables.

```sql
-- Anti-pattern: generic FK columns
CREATE TABLE comments (
    comment_id    SERIAL PRIMARY KEY,
    body          TEXT NOT NULL,
    commentable_type TEXT,  -- 'post' or 'video' or 'photo'
    commentable_id   INT    -- points to post_id, video_id, or photo_id
    -- No FK constraint possible here. DB cannot enforce referential integrity.
);
```

| Approach | Pros | Cons |
|---|---|---|
| Polymorphic FK (`type` + `id`) | Simple to add new types | No referential integrity, hard to query |
| Separate join tables (`post_comments`, `video_comments`) | Full FK enforcement | More tables, duplicated structure |
| Shared base table (`commentable` parent) | FK works, cleaner queries | Requires schema planning up front |
| JSONB column | Total flexibility | No constraints, no joins, hard to index |

**Preferred for interviews:** separate join tables or a shared parent table. Polymorphic FKs are a pragmatic choice in application code (Rails STI, etc.) but you should know the trade-offs.

---

## Classic Interview Schemas

### 1. E-Commerce System

**Entities:** users, products, categories, orders, order_items, reviews

```sql
CREATE TABLE users (
    user_id       SERIAL PRIMARY KEY,
    email         TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name     TEXT,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    name          TEXT NOT NULL,
    parent_id     INT REFERENCES categories(category_id)  -- self-referential
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INT REFERENCES categories(category_id),
    name          TEXT NOT NULL,
    description   TEXT,
    price         NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock_qty     INT NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    user_id       INT NOT NULL REFERENCES users(user_id),
    status        TEXT NOT NULL DEFAULT 'pending'
                      CHECK (status IN ('pending','confirmed','shipped','delivered','cancelled')),
    total_amount  NUMERIC(10, 2) NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    order_id        INT REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id      INT REFERENCES products(product_id),
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10, 2) NOT NULL,  -- snapshot at time of order
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE reviews (
    review_id   SERIAL PRIMARY KEY,
    product_id  INT NOT NULL REFERENCES products(product_id),
    user_id     INT NOT NULL REFERENCES users(user_id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    body        TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, user_id)  -- one review per user per product
);
```

**Key decisions to explain in an interview:**
- `unit_price` on `order_items` is a snapshot — product price can change, order history must not.
- `total_amount` on `orders` is a denormalization: it can be computed from `order_items`, but storing it avoids an aggregate on every order list page.
- `UNIQUE (product_id, user_id)` on reviews prevents review bombing.
- Self-referential `categories` supports unlimited nesting.

---

### 2. Social Network

**Entities:** users, follows (self-referential many-to-many), posts, likes, comments

```sql
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    username   TEXT UNIQUE NOT NULL,
    email      TEXT UNIQUE NOT NULL,
    bio        TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Asymmetric follow graph: follower follows followee
CREATE TABLE follows (
    follower_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    followee_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id <> followee_id)  -- cannot follow yourself
);

CREATE TABLE posts (
    post_id    SERIAL PRIMARY KEY,
    user_id    INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE likes (
    user_id    INT REFERENCES users(user_id) ON DELETE CASCADE,
    post_id    INT REFERENCES posts(post_id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    post_id     INT NOT NULL REFERENCES posts(post_id) ON DELETE CASCADE,
    user_id     INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    parent_id   INT REFERENCES comments(comment_id),  -- threaded replies
    body        TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

**Key decisions:**
- `follows` is directional — (A→B) and (B→A) are separate rows. "Is mutual follow?" = check both rows exist.
- `CHECK (follower_id <> followee_id)` is a cheap constraint that avoids an application-level check.
- `parent_id` on comments enables threaded replies (self-referential). Depth is usually limited to 2 in practice (UI constraint), so this is sufficient.
- Feed generation ("posts from people I follow") is a classic read-scaling problem — `follows` + `posts` join works for small scale; fan-out-on-write (precomputed feed table) for large scale.

---

### 3. Hotel Booking System

**Entities:** hotels, rooms, room_types, guests, bookings

```sql
CREATE TABLE hotels (
    hotel_id   SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    city       TEXT NOT NULL,
    country    TEXT NOT NULL,
    star_rating SMALLINT CHECK (star_rating BETWEEN 1 AND 5)
);

CREATE TABLE room_types (
    room_type_id  SERIAL PRIMARY KEY,
    hotel_id      INT NOT NULL REFERENCES hotels(hotel_id),
    name          TEXT NOT NULL,  -- 'Standard', 'Deluxe', 'Suite'
    base_price    NUMERIC(10, 2) NOT NULL,
    max_guests    SMALLINT NOT NULL
);

CREATE TABLE rooms (
    room_id      SERIAL PRIMARY KEY,
    hotel_id     INT NOT NULL REFERENCES hotels(hotel_id),
    room_type_id INT NOT NULL REFERENCES room_types(room_type_id),
    room_number  TEXT NOT NULL,
    floor        SMALLINT,
    UNIQUE (hotel_id, room_number)
);

CREATE TABLE guests (
    guest_id    SERIAL PRIMARY KEY,
    email       TEXT UNIQUE NOT NULL,
    full_name   TEXT NOT NULL,
    phone       TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE bookings (
    booking_id   SERIAL PRIMARY KEY,
    room_id      INT NOT NULL REFERENCES rooms(room_id),
    guest_id     INT NOT NULL REFERENCES guests(guest_id),
    check_in     DATE NOT NULL,
    check_out    DATE NOT NULL,
    total_price  NUMERIC(10, 2) NOT NULL,
    status       TEXT NOT NULL DEFAULT 'confirmed'
                     CHECK (status IN ('confirmed', 'cancelled', 'checked_in', 'checked_out')),
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    CHECK (check_out > check_in)
);

-- Prevent double-booking: no two confirmed bookings for the same room with overlapping dates
-- Enforced via exclusion constraint (PostgreSQL) or application-level serialized transaction
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings ADD CONSTRAINT no_double_booking
    EXCLUDE USING GIST (
        room_id WITH =,
        daterange(check_in, check_out, '[)') WITH &&
    )
    WHERE (status = 'confirmed');
```

**Key decisions:**
- `room_types` separates the concept of "what kind of room" from "which physical room." Price lives on the type, not each room.
- `EXCLUDE` constraint using `btree_gist` is the PostgreSQL way to prevent overlapping date ranges with a DB-level guarantee. This is a great detail to mention in an interview — it shows you know DB constraints beyond FK and UNIQUE.
- `total_price` is stored on `bookings` — prices and discount logic can change; the agreed price at booking time must be immutable.
- `CHECK (check_out > check_in)` catches bad input at the DB layer.

---

### 4. Task Management App (Jira-like)

**Entities:** users, projects, issues, issue_links, comments, attachments, labels

```sql
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    email      TEXT UNIQUE NOT NULL,
    full_name  TEXT NOT NULL,
    avatar_url TEXT
);

CREATE TABLE projects (
    project_id  SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    key         TEXT UNIQUE NOT NULL,  -- e.g., 'PROJ', 'BACKEND'
    owner_id    INT REFERENCES users(user_id),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE issues (
    issue_id     SERIAL PRIMARY KEY,
    project_id   INT NOT NULL REFERENCES projects(project_id),
    reporter_id  INT NOT NULL REFERENCES users(user_id),
    assignee_id  INT REFERENCES users(user_id),  -- nullable: unassigned
    parent_id    INT REFERENCES issues(issue_id),  -- subtasks
    title        TEXT NOT NULL,
    description  TEXT,
    type         TEXT NOT NULL CHECK (type IN ('bug', 'feature', 'task', 'epic')),
    status       TEXT NOT NULL DEFAULT 'open'
                     CHECK (status IN ('open', 'in_progress', 'review', 'done', 'cancelled')),
    priority     TEXT NOT NULL DEFAULT 'medium'
                     CHECK (priority IN ('low', 'medium', 'high', 'critical')),
    story_points SMALLINT,
    due_date     DATE,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Many-to-many self-referential: "blocks", "is blocked by", "duplicates"
CREATE TABLE issue_links (
    link_id      SERIAL PRIMARY KEY,
    from_issue   INT NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    to_issue     INT NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    link_type    TEXT NOT NULL CHECK (link_type IN ('blocks', 'duplicates', 'relates_to')),
    CHECK (from_issue <> to_issue),
    UNIQUE (from_issue, to_issue, link_type)
);

CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    issue_id    INT NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    author_id   INT NOT NULL REFERENCES users(user_id),
    body        TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE attachments (
    attachment_id  SERIAL PRIMARY KEY,
    issue_id       INT NOT NULL REFERENCES issues(issue_id) ON DELETE CASCADE,
    uploader_id    INT NOT NULL REFERENCES users(user_id),
    filename       TEXT NOT NULL,
    storage_key    TEXT NOT NULL,  -- S3 key or equivalent
    file_size_kb   INT,
    content_type   TEXT,
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE labels (
    label_id   SERIAL PRIMARY KEY,
    project_id INT NOT NULL REFERENCES projects(project_id),
    name       TEXT NOT NULL,
    color      TEXT,
    UNIQUE (project_id, name)
);

CREATE TABLE issue_labels (
    issue_id  INT REFERENCES issues(issue_id) ON DELETE CASCADE,
    label_id  INT REFERENCES labels(label_id) ON DELETE CASCADE,
    PRIMARY KEY (issue_id, label_id)
);
```

**Key decisions:**
- `parent_id` on `issues` handles subtasks (one level is usually enough; epics-as-parent via `type = 'epic'` is common).
- `issue_links` is a separate table for cross-issue relationships (blocks, duplicates). It is directional; "is blocked by" is just the reverse direction read.
- `attachments` stores only metadata — actual files live in object storage (S3). Never store blobs in relational DBs for anything at scale.
- `labels` are scoped to `project_id` — label "bug" in Project A and label "bug" in Project B are different rows. Prevents global label pollution.
- `updated_at` on `issues` and `comments` enables activity feeds sorted by last-modified.
