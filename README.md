# PlateScore

A feedback and analytics app for the Rishihood University hostel mess,
operated by BCH. The daily meals are breakfast, lunch, snacks and dinner.

| Role | Name | Enrollment No. | GitHub |
|---|---|---|---|
| Group lead | Pranjal Garg | 2501010342 | pjlgrg |

Team T0133, Section C. This repository currently contains the CP1 and CP2
database design. The Express API and frontend are planned work.

Students will log in, give a dated meal a score from 1 to 5, and see their own
ratings. Managers will publish the weekly menu and read feedback reports.
The planned stack is Node.js, Express, Prisma, PostgreSQL and JWT with student
and manager roles. Passwords will be hashed in the API, never stored as plain
text. There will be five tables: users, meals, dishes, menu_items and ratings.

## Checkpoint files

- [Normalization from UNF to 3NF/BCNF](dbms/01_normalization.md)
- [ER diagram](dbms/02_er_diagram.png) and [editable Mermaid source](dbms/02_er_diagram.mmd)
- [PostgreSQL tables and constraints](dbms/03_schema.sql)

## Run the schema

Install PostgreSQL 15 or newer, start it, then run from this repository:

```sh
createdb platescore
psql -X -v ON_ERROR_STOP=1 -d platescore -f dbms/03_schema.sql
```

Use a new empty database. The script creates a schema called `platescore`
and commits all five tables together. It stops on an error. To list them:

```sh
psql -d platescore -c '\dt platescore.*'
```

The SQL was checked on PostgreSQL 17.12. The load/check steps and the meaning
of each rule are in [dbms/VALIDATION.md](dbms/VALIDATION.md).

The separate [DE repository](https://github.com/pjlgrg/DE_PROJECT_2026) contains
the assigned Blinkit project and its 30-query benchmark. PlateScore's own SQL
reports, API, authentication and deployment are not part of these CP1/CP2 files.
