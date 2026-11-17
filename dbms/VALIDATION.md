# Schema checks

Checked on 17 November 2026 using PostgreSQL 17.12 on the same MacBook Pro
used for the Blinkit baseline.

## What was run

The final `03_schema.sql` ran from top to bottom on a new empty database,
`platescore_t0133_check`. It created these five tables in one transaction:

```text
platescore.users
platescore.meals
platescore.dishes
platescore.menu_items
platescore.ratings
```

Temporary test rows used two sample students, one sample BCH manager, four
meal slots on the same date, two dishes and a rating. They used
`example.invalid` emails and test-only password-hash text. These rows test
the table rules; they are not real accounts, usable login credentials or
collected feedback. Each test ran in a transaction that was rolled back.

## Results

| Check | Result |
|---|---|
| breakfast, lunch, snacks and dinner on the same date | All four accepted |
| Two dishes in one meal | Accepted through menu_items |
| Student -> rating -> meal join | Returned the expected sample student, lunch and score 4 |
| Score and comment in one INSERT | Accepted together |
| Rating without a comment | Accepted; comment is NULL |
| Scores 0 and 6 | Both rejected |
| Second rating by the same student for the same meal | Rejected by UNIQUE(user_id, meal_id) |
| Rating for an unknown student or meal | Both rejected by foreign keys |
| Two snacks entries on the same date | Rejected by UNIQUE(meal_type, meal_date) |
| Same dish twice in one meal | Rejected by the menu_items primary key |
| Role other than student/manager | Rejected |
| Mixed-case stored email | Rejected; the API must lowercase it before saving |
| Meal type brunch | Rejected |
| Comment with 301 characters | Rejected |
| NULL score | Rejected |
| Delete a user referenced by a rating | Rejected |
| Delete a dish referenced by a menu | Rejected |
| Row counts after rollback | Zero in all five tables |

Fourteen deliberately invalid operations were checked. Each failed for the
expected CHECK, UNIQUE, NOT NULL, length or foreign-key rule. The valid join
and optional-comment checks also passed.

## Reading the design

`meals` represents one dated serving, such as snacks on 17 November. The same
type is allowed again on 18 November. The unique rule uses **both** type and
date. `menu_items` is the link between a dated meal and its dishes.

`ratings` also has a combined unique rule: **both** user and meal. It does
not limit a student to one rating forever. Score and comment belong to this
same row, so one INSERT saves them together. In the future Prisma API, they
should be fields of the same create/update operation.

These checks cover the database design only. They do not prove JWT login,
password hashing, role checks, private "my ratings" access, reports or
deployment. Those need the later Express application and its own tests.
