# LMS Learning Log

A record of every step I take building this LMS in Laravel: **what** I did, the **commands/code**, and **why**.
Use it to retrace my steps, review concepts, or redo the project from scratch.

---

## Step 0 — Project setup
**Date:** 2026-10-30

### What I did
Created a new Laravel project and connected it to MySQL (XAMPP).

### Commands
```bash
# inside Desktop/GG/projects
composer create-project laravel/laravel lms   # download Laravel + dependencies, generate APP_KEY
cd lms
# created an empty database "lms" in MySQL (phpMyAdmin or mysql CLI)
# edited .env:
#   DB_CONNECTION=mysql
#   DB_HOST=127.0.0.1
#   DB_PORT=3306
#   DB_DATABASE=lms
#   DB_USERNAME=root
#   DB_PASSWORD=****
php artisan migrate        # create Laravel's default tables in MySQL
npm install                # install frontend tools (Vite, Tailwind)
npm run build              # build CSS/JS assets
git init -b main           # start version control
php artisan serve          # run the app at http://localhost:8000
```

### Why
- **Composer** is PHP's package manager. It downloads Laravel and everything it depends on.
- **`.env`** holds settings and secrets (like the DB password) **outside the code**. It is in `.gitignore`, so it never goes to GitHub. `.env.example` is the shareable template.
- **Laravel ships with default migrations** (users, password_reset_tokens, sessions, cache, jobs). Running `migrate` created those tables. I didn't design them; every Laravel project starts with them.
- PHP 8.2 (XAMPP) → Composer installed **Laravel 12** (Laravel 13 needs PHP 8.3).

### Notes / gotchas
- MySQL in XAMPP must be running before using the app.
- If `.env` is edited and changes don't apply: `php artisan config:clear`.

---

## Step 1 — Add a `role` column to `users`
**Date:** 2026-10-30

### Goal
Users need a role (`admin` or `user`) so the app can show or hide features per role.

### What I did
1. Generated a new migration file.
2. Wrote the change (`up`) and the undo (`down`).
3. Ran the migration.
4. Verified in phpMyAdmin.

### Commands
```bash
php artisan make:migration add_role_to_users_table   # CREATE the migration file (does not touch the DB)
php artisan migrate                                    # RUN pending migrations (changes the DB)
```

### Code
`database/migrations/2026_10_30_071746_add_role_to_users_table.php`
```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('role')->default('user')->after('email');
    });
}

public function down(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('role');
    });
}
```
SQL equivalent:
```sql
-- up()
ALTER TABLE users ADD role VARCHAR(255) NOT NULL DEFAULT 'user' AFTER email;
-- down()
ALTER TABLE users DROP COLUMN role;
```

### Why
- **What a migration is:** a PHP file describing a database *structure* change. It's like **Git for the database**.
- **Why not phpMyAdmin:** manual changes aren't recorded. Teammates and servers wouldn't get them. With migrations, anyone runs `php artisan migrate` and gets the exact same structure.
- **Why a NEW migration instead of editing the users migration:** Laravel tracks migrations that already ran in the `migrations` table. An already-run migration is never re-run, so editing it does nothing. **Rule: never edit a migration that has run; create a new one.**
- **`up()` vs `down()`:** `up()` makes the change (`migrate`), `down()` undoes it (`migrate:rollback`). `down()` must be the exact opposite of `up()`.
- **`Schema::table` vs `Schema::create`:** `table` modifies an existing table; `create` makes a new one. Laravel picked `table` automatically because the name ends in `_to_users_table`.
- **`->default('user')`:** every new account is a normal user unless explicitly made admin. That's safe by default.
- **The filename timestamp** (`2026_10_30_071746_`) makes migrations run in creation order.

### Concepts learned
| Command | What it does |
|---|---|
| `php artisan make:migration name` | Create a new migration file |
| `php artisan migrate` | Run migrations that haven't run yet |
| `php artisan migrate:rollback` | Undo the last batch of migrations |
| `php artisan migrate:status` | Show which migrations ran |
| `php artisan migrate:fresh` | Drop ALL tables and re-run everything (dev only! deletes data) |

- **Batch:** each `migrate` run is one batch. My role migration is batch 2; Laravel's defaults are batch 1. `rollback` undoes the latest batch.
- Migrations = **structure** (tables/columns). Seeders = **data** (rows). Seeders come later.

### Verified
- [x] `role` column exists in `users`, `VARCHAR(255)`, default `user`
- [x] Migration listed in the `migrations` table (batch 2)
- [x] Tested `php artisan migrate:rollback` then `php artisan migrate`
- [x] Committed to Git

---

## Step 2 — Role enum + User model *(in progress)*
**Date:** 2026-10-31

### Goal
Turn the `role` string into a PHP enum, so the code uses `Role::Admin` instead of typing `'admin'` everywhere, and add `isAdmin()` to the User model.

### What I did
1. Generated a string-backed enum `App\Enums\Role` (`Admin`, `User`) with a `label()` method.
2. Cast the `role` column to the enum in `User::casts()`.
3. Added `isAdmin()` to the User model.
4. Tested in Tinker with real users made by the factory.
5. Checked code style with Pint.

### Commands
```bash
php artisan make:enum Enums/Role --string     # create app/Enums/Role.php
php artisan tinker                            # open the console to test
```
```php
// inside tinker
$u = App\Models\User::first();      // returned null (table empty)
$u->role;                           // null too, because $u is null
$u = App\Models\User::factory()->create();                                   // create a normal user
$u->role;                           // App\Enums\Role::User
$u->isAdmin();                      // false
$u->role->label();                  // "User"
$a = App\Models\User::factory()->create(['role' => App\Enums\Role::Admin]);  // create an admin
$a->isAdmin();                      // true
exit
```
```bash
# tidy the code with Pint (run from the lms folder; never open the pint file itself)
.\vendor\bin\pint --test -v app/Enums app/Models/User.php   # 1. check style (no changes made) -> "fail"
# fixed Role.php closing braces by hand
.\vendor\bin\pint --dirty -v                                # 2. let Pint fix the rest
git diff                                                    # 3. review what changed
.\vendor\bin\pint --test                                    # 4. re-check -> "passed"
```

### Why
- **Enum instead of strings:** typing `'admin'` everywhere invites typos (`'admn'`) that PHP won't catch. `Role::Admin` is checked by PHP, and the editor autocompletes it.
- **`--string` (backed enum):** each case has a string value (`'admin'`), which is what's actually stored in the DB column.
- **Cast (`'role' => Role::class`):** Laravel converts the DB string to the enum when reading, and back to a string when saving, automatically.
- **`isAdmin()` on the model:** the role check lives in one place. If the rule changes later, I change one method.
- **Pint:** formats code to Laravel's standard style. It only changes spacing, never behavior. I run it before every commit.

### Concepts learned
- **Enum** = a fixed list of allowed values. **Backed enum** = each value maps to a string/int (`->value`).
- **Cast** = automatic type conversion between the DB and PHP.
- **Factory** = generates fake rows for testing. `create()` saves to the DB; `make()` doesn't.
- **Tinker** = console for running PHP against the app.
- **Pint** = code style fixer (`--test` checks, plain run fixes, `--dirty` = changed files only).

### Mistakes / errors I hit (and the fix)
- `User::first()` returned `null`. The `users` table had no rows yet, because migrations only create tables, not data. **Fix:** created users with `User::factory()->create()`.
- Pint flagged style issues: indentation and trailing spaces in `Role.php`, and blank lines in `User.php`. **Fix:** lined up the `}` by hand, then ran `pint --dirty`.

### Verified
- [x] DB has 2 test users: #1 `user` (`isAdmin` false), #2 `admin` (`isAdmin` true)
- [x] Code tidied (`pint --test` → `passed`)
- [ ] Committed to Git

---

## Command Reference
All commands and snippets live in **[COMMANDS.md](COMMANDS.md)**.

---

## Template for new steps
```
## Step N — Title
**Date:**

### Goal
### What I did
### Commands
### Code
### Why
### Concepts learned
### Mistakes / errors I hit (and the fix)
### Verified
```
