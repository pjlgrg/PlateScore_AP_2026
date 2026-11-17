# PlateScore: normalization

Pranjal Garg | T0133 | 2501010342

PlateScore is a feedback app for the Rishihood University hostel mess, operated
by BCH. There are four meals each day: breakfast, lunch, snacks and dinner.
A student rates a dated meal from 1 to 5 and can add a short comment. A manager
publishes menus and reads the reports. This design is for this one mess, so it
does not need separate university, hostel or catering-company tables.

## What one rating means

There is one score per student per meal, not one score per dish. A meal can have
several dishes. A report grouped by dish will therefore show the average meal
score **when that dish was served**. That is a useful starting point for the
manager, but it does not prove that a particular dish caused a low score.
Separate dish ratings would need a later change to the design.

## Starting data: UNF

A single feedback sheet could contain:

```text
student_id, student_name, email, password_hash, role,
meal_id, meal_type, meal_date, published_by, manager_name, is_published,
dishes [{dish_id, dish_name, description}], score, comment, rated_at
```

For example, student 11 rates lunch 31 on 17 November 2026 as 4, with the
comment "Rice was warm". The dishes list includes Paneer Do-Pyaza and
Jeera Rice. Another student rates the same lunch as 3. The same menu and
manager details are repeated in both rows. The students, scores and comments
in this example are made up to explain the design.

The dish names were checked against the Tuesday menu on
[RU Print Portal](https://ru-print.vercel.app/) on 17 November 2026. That
lunch lists Curd, Cucumber-Carrot Salad, Paneer Do-Pyaza, Lal Malka Dal Tarka,
Jeera Rice and Chapati. Tuesday snacks lists Samosa, Chutney, Hot Milk (D),
Tea (D) and Coffee Powder. A menu describes what was planned to be served;
it does not supply real student ratings.

Problems with this sheet:

- Changing a student's email needs changes in many rows.
- A menu cannot be recorded until somebody rates it.
- Deleting the only rating for a meal could also lose its menu.

## 1NF: remove the repeating dish list

Make one row for each student, meal and dish. Each cell has one value.
The key for this flattened sheet is `(student_id, meal_id, dish_id)`.
The meal score and comment repeat for each dish in that meal.

The main functional dependencies are:

```text
student_id -> student_name, email, password_hash, role
meal_id -> meal_type, meal_date, published_by, manager_name, is_published
dish_id -> dish_name, description
(student_id, meal_id) -> score, comment, rated_at
published_by -> manager_name
```

The sheet is in 1NF, but it still has partial dependencies. For example,
the dish name depends only on `dish_id`, not the whole three-column key.

## 2NF: remove partial dependencies

Separate the data by what it describes:

```text
Users(user_id, full_name, email, password_hash, role, created_at)
Meals(meal_id, meal_type, meal_date, published_by, manager_name, is_published)
Dishes(dish_id, dish_name, description)
MenuItems(meal_id, dish_id)
Ratings(user_id, meal_id, score, comment, rated_at)
```

`student_id` is now called `user_id` because managers also log in. In
`MenuItems`, both columns form the key. In `Ratings`, the score, comment and
time depend on the whole `(user_id, meal_id)` key. A student can rate different
meals, and different students can rate the same meal.

This also lets a manager create a menu before any ratings exist. Joining
`Ratings` to `Meals`, `MenuItems` and `Dishes` gives the feedback sheet again.

## 3NF: remove the transitive dependency

In `Meals`, `meal_id -> published_by -> manager_name` is a transitive
dependency. Remove `manager_name` from `Meals`. Its value is available as
`users.full_name` through the `published_by` foreign key.

The final tables in [03_schema.sql](03_schema.sql) are:

| Table | Primary key | Other keys and main dependencies |
|---|---|---|
| users | user_id | email is unique; user_id -> full_name, email, password_hash, role, created_at |
| meals | meal_id | (meal_type, meal_date) is unique; meal_id -> meal_type, meal_date, published_by, is_published |
| dishes | dish_id | dish_name is unique; dish_id -> dish_name, description |
| menu_items | (meal_id, dish_id) | Both columns are foreign keys; there are no non-key columns |
| ratings | rating_id | (user_id, meal_id) is unique; rating_id -> user_id, meal_id, score, comment, rated_at |

`rating_id` is a simple ID for the future API. Adding it does not replace the
unique rule on `(user_id, meal_id)`. Email and dish name are also candidate
keys; users enter emails in lowercase. Dish names identify the mess's dish
catalogue entries, so a different recipe needs a different entry/name.

Under these rules, each non-trivial dependency has a candidate key on the left.
The five tables meet 3NF and BCNF. Average scores and weekly totals are not
stored in these tables; reports will calculate them from the ratings.

## Relationships and rules

- One user can write many ratings. Each rating belongs to one user and one meal.
- One manager can publish many meals. Each meal records its publishing manager.
- Meals and dishes have a many-to-many relationship through `menu_items`.
- A student can rate a meal only once. The score must be from 1 to 5.
- A comment is optional and is stored with the score in the same row. One
  INSERT saves both atomically; a failed INSERT saves neither.
- Foreign keys use RESTRICT so deleting a user, dish or meal cannot silently
  remove old feedback. An unpublished meal is a draft.

The database stores roles and references, but foreign keys do not check a user's
role. The future Express API must check that the publisher is a manager, that
the rating author is a student, and that the meal is published and ready to be
rated. It must get the student's ID from JWT, and filter "my ratings" by that
ID. These access rules are not implemented by this checkpoint's SQL alone.

See [the ER diagram](02_er_diagram.png). The PostgreSQL DDL, diagram and this
document use the same five tables and column names.
