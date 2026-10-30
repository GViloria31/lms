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

## Step 2 — Role enum + User model
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
# removed the empty line inside casts() by hand

# save the step in Git (on branch feature/auth)
git status                                          # see what will be saved
git add .                                           # stage everything
git commit -m "Add Role enum and isAdmin to User model"
git log --oneline -3                                # confirm the commit (a4185f2)

# commit the log update, then back up the branch to GitHub
git add docs/LEARNING_LOG.md
git commit -m "Update learning log for step 2"      # (typo "lof" slipped into the real message, e51d6e2)
git push -u origin feature/auth                     # first push: creates the branch on GitHub + links it
git branch -vv                                      # check: shows [origin/feature/auth] = linked
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
- Typo in a commit message ("lof"), noticed after pushing. **Left it:** fixing a pushed commit means rewriting history. Next time, check with `git log --oneline -1` and fix with `git commit --amend` **before** pushing.

### Verified
- [x] DB has 2 test users: #1 `user` (`isAdmin` false), #2 `admin` (`isAdmin` true)
- [x] Code tidied (`pint --test` → `passed`)
- [x] Committed to Git (`a4185f2` on `feature/auth`)
- [x] Log committed (`e51d6e2`) and branch pushed to GitHub (`origin/feature/auth`)

---

## Step 3 — Login & register with Laravel Breeze
**Date:** 2026-10-31 · **Branch:** `feature/auth`

### Goal
Let real users register, log in and log out, using Laravel's official starter kit instead of writing auth by hand.

### Decisions
- **Blade stack, not React/Vue:** I'm focusing on Laravel first. With React I'd be learning Laravel + Inertia + React at once. Backend skills carry over later. (React is on the "Later / ideas" list.)
- **No dark mode:** it keeps the views easier to study. Postponed to "Later / ideas".
- **PHPUnit, not Pest:** it's what the project already uses.

### What I did
1. Installed the Breeze package with Composer (dev only).
2. Ran the Breeze installer with the Blade stack. It generated controllers, views, routes and tests, and ran `npm install` + `npm run build`.
3. Ran migrations: nothing to migrate.
4. Checked the new routes and ran the test suite.

### Commands
```bash
composer require laravel/breeze --dev   # download Breeze (dev dependency)
php artisan breeze:install blade        # generate auth files with the Blade stack
php artisan migrate                     # -> "Nothing to migrate"
git status                              # see what Breeze added (??) and changed (M)
php artisan route:list --except-vendor  # list the app's routes (login, register, dashboard, profile...)
php artisan test                        # run all tests -> 25 passed
php artisan serve                       # test in the browser at http://localhost:8000
#   /register -> created my account -> landed on /dashboard
#   logout -> login as factory admin (corkery.bradley@example.com / password)
#   logged out -> /dashboard redirects to /login
.\vendor\bin\pint --test                # style check -> "passed" (Breeze code already follows Laravel style)

# save + back up
git status
git add .                               # warning: CRLF will be replaced by LF (harmless, see Mistakes)
git commit -m "Install Breeze with Blade for authentication"
git log --oneline -1                    # checked the message for typos before pushing
git push                                # plain push (branch already linked by -u in Step 2)
```

### What Breeze added
| Where | What |
|---|---|
| `routes/auth.php` | All auth routes (login, register, logout, password reset, email verify) |
| `routes/web.php` | `/dashboard` and `/profile` routes, protected by the `auth` middleware |
| `app/Http/Controllers/Auth/` | One controller per auth action (e.g. `RegisteredUserController`) |
| `app/Http/Controllers/ProfileController.php` | Edit/update/delete own profile |
| `app/Http/Requests/` | Form validation classes (`LoginRequest`, `ProfileUpdateRequest`) |
| `app/View/Components/` | Layout components (`AppLayout`, `GuestLayout`) |
| `resources/views/auth/`, `layouts/`, `components/`, `profile/`, `dashboard.blade.php` | The pages |
| `tailwind.config.js`, `postcss.config.js`, `vite.config.js` | Frontend build setup (Tailwind CSS) |
| `tests/Feature/Auth/`, `ProfileTest.php` | Tests for every auth feature |

### Why
- **Why Breeze:** auth is easy to get wrong, and mistakes are security holes. Breeze is official, small, and copies the code **into my project**, so I can read and change it.
- **Why "Nothing to migrate":** Laravel's default migrations (Step 0) already created everything auth needs: `users`, `password_reset_tokens`, `sessions`. Breeze only adds code, not tables.
- **Why `--dev`:** Breeze is only the installer. After it copies the files, the app doesn't need the package to run.

### Concepts learned
- **Starter kit** = generated code you own, not a library hidden in `vendor/`.
- **Middleware `auth`** = a gate in front of a route. Not logged in → redirected to `/login`.
- **Route file** = maps a URL + HTTP method (GET/POST/PATCH/DELETE) to a controller method.
- **Feature test** = code that pretends to be a browser and checks the app responds correctly.
- **Sessions table** = how Laravel remembers who is logged in. Each browser gets a row; `user_id` is filled while logged in and `NULL` after logout.
- **Registering = always a normal user.** My account got `role = user` from the column default (Step 1). The register form has no role field, so nobody can sign up as admin. That's safe by default.
- **Timestamps are in UTC.** `created_at` showed `2026-10-30 23:05`, but my local time was `2026-10-31 07:05` (UTC+8). Laravel stores times in UTC (`config/app.php` → `timezone`), which is the standard practice. Convert to local time only when displaying.

### Mistakes / errors I hit (and the fix)
- **The installer didn't ask about dark mode or the testing framework.** Because I typed the stack (`blade`) in the command, Breeze skipped all questions and used the defaults: **no dark mode, PHPUnit**. That's what I wanted anyway. (Running `php artisan breeze:install` with no stack would ask everything.)
- **`git add .` warned: `in the working copy of 'package.json', CRLF will be replaced by LF`.** Just a warning, and the files were staged. npm saved `package.json` with Windows line endings (CRLF). The project's `.gitattributes` (`* text=auto eol=lf`) tells Git to store all text files with LF. **Nothing to fix.**
- **Files edited after `git add .` weren't in the commit.** Two doc lines were changed between `git add` and `git commit`, so they stayed unstaged. **Fix:** they went into the next commit. **Habit:** run `git status` right before `git commit`. Red ` M` = not staged, green `M ` = staged.

### Verified
- [x] Breeze installed, 20 routes listed
- [x] `php artisan test` → 25 passed (61 assertions)
- [x] Tested in the browser: register, logout, login as factory admin, /dashboard blocked when logged out
- [x] New account saved in DB: user #3, `role = user`
- [x] Code style checked (`pint --test` → passed)
- [x] Committed + pushed (`c2f3b25` on `origin/feature/auth`)

---

## Step 4 — (next)
_To be filled in._

---

## Later / ideas
Things I decided to postpone, and why. Revisit once the main features work.

- [ ] **Add dark mode.** *(decided 2026-10-31, Step 3)*
  - **What:** add Tailwind `dark:` classes (e.g. `dark:bg-gray-800 dark:text-gray-200`) so pages follow the visitor's OS dark setting.
  - **Why later:** it doubles the classes in every view, which makes the Breeze files harder to study, and every new page would need `dark:` versions too. It doesn't affect any Laravel logic.
  - **When:** after the main pages are built (courses, lessons, admin). Do it as its own step on its own branch, e.g. `feature/dark-mode`.
- [ ] **Learn React, then try React + Inertia in a future Laravel project.** *(decided 2026-10-31, Step 3)*
  - **Why later:** I chose Blade so I can focus on Laravel first. Backend skills (routes, models, auth) carry over to any frontend.

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
