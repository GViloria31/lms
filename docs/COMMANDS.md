# Command Cheat Sheet

Every command and code snippet I use in this project, grouped by purpose: what it does and when to use it.
Look things up here; for the reasoning behind each step, see [LEARNING_LOG.md](LEARNING_LOG.md).

### Artisan: generate files (`make:`)
`make:` copies a Laravel template (stub) and fills in the name/namespace. It writes the **structure**; I write the **logic**.

| Command | Creates | Notes |
|---|---|---|
| `php artisan make:migration add_role_to_users_table` | `database/migrations/<timestamp>_add_role_to_users_table.php` | Name ending in `_to_users_table` → Laravel pre-fills `Schema::table('users')` |
| `php artisan make:enum Enums/Role --string` | `app/Enums/Role.php` | `Enums/` creates the folder + namespace. `--string` = string-backed (needed to store in DB) |
| `php artisan list make` | — | Show every `make:` command |
| `php artisan make:enum --help` | — | Show options for any command |

### Artisan: database
| Command | What it does |
|---|---|
| `php artisan migrate` | Run migrations that haven't run yet (only new ones) |
| `php artisan migrate:rollback` | Undo the last batch |
| `php artisan migrate:status` | Show which migrations ran |
| `php artisan migrate:fresh` | ⚠️ Drop ALL tables and rebuild (deletes data, dev only) |
| `php artisan migrate --path=database/migrations/<file>.php` | Run one specific migration (rarely needed) |

### Tinker: interactive PHP console for testing code
```bash
php artisan tinker          # start (prompt changes to >)
exit                        # quit
```
```php
// test the Role enum + User model
$u = new App\Models\User;            // create a user object (MUST run first)
$u->role = App\Enums\Role::Admin;    // full class name needed; Tinker doesn't know my `use` lines
$u->isAdmin();                       // true
$u->role->value;                     // "admin"
$u->role->label();                   // "Administrator"
```
```php
// work with REAL rows in the database
App\Models\User::count();                     // how many users exist
App\Models\User::first();                     // first user, or null if the table is empty
App\Models\User::all();                       // every user
App\Models\User::find(1);                     // user with id 1 (null if not found)

// create test users with the factory (fake name/email, password = "password")
$u = App\Models\User::factory()->create();                                   // normal user (role default 'user')
$a = App\Models\User::factory()->create(['role' => App\Enums\Role::Admin]);  // admin
App\Models\User::factory()->count(5)->create();                              // 5 users at once

$u->role;            // App\Enums\Role::User  (an enum object, thanks to the cast)
$u->isAdmin();       // false
$a->isAdmin();       // true
```
- `new User` = object in memory only (not saved). `factory()->create()` = saved to the DB (shows in phpMyAdmin).
- `factory()->make()` = like `create()` but **not** saved.

One-liner without entering Tinker:
```bash
php artisan tinker --execute='$u = new App\Models\User; $u->role = App\Enums\Role::Admin; var_dump($u->isAdmin()); echo $u->role->label();'
```
**Tinker rules:**
- Variables are forgotten after `exit`. Re-run lines from the top in each new session.
- Restart Tinker after changing a PHP file.
- Pasting in the VS Code terminal can add hidden characters → `PARSE ERROR ... ASCII 27`. Type by hand, or set `"terminal.integrated.ignoreBracketedPasteMode": true` in VS Code settings.

### PHP checks
| Command | What it does |
|---|---|
| `php -l path/to/File.php` | **Lint**: check a file for syntax errors without running it. Use when you get a ParseError |
| `php -v` | Show PHP version |

### Code style (Laravel Pint)
**What it is:** a code formatter that comes with Laravel (Composer installed it in `vendor/`). It fixes code **style** (spacing, indentation, blank lines, imports) to Laravel's standard. It never changes what the code does, so it's always safe to run.
**Why use it:** consistent code is easier to read and review, and teams expect it. Run it before every commit.

**How to use it:**
- You **never open or edit** the `vendor/bin/pint` file. It's a program you **run** from the terminal, like `php artisan`.
- Open the terminal (`` Ctrl+` ``) and make sure you're in the project folder (the prompt ends in `\lms>`).
- Path: PowerShell `.\vendor\bin\pint` · Git Bash `./vendor/bin/pint` · works anywhere `php vendor/bin/pint`
- Read the output: `"result":"passed"` = all good. `"fail"` + a list of `fixers` = style problems found (see the rules table below).

| Part of the command | Meaning |
|---|---|
| `.\` | Start from the current folder (PowerShell) |
| `vendor\bin\pint` | Path to the Pint program |
| `--test` | Check only, don't edit |
| `--dirty` | Only files changed since the last commit |
| `-v` | Verbose: list each fix |

**Error:** `The term '.\vendor\bin\pint' is not recognized` → wrong folder. `cd` into `lms`, or use `php vendor/bin/pint`.

| Command | What it does |
|---|---|
| `.\vendor\bin\pint --test` | Report style problems only, don't change files |
| `.\vendor\bin\pint` | Fix all files |
| `.\vendor\bin\pint --dirty` | Fix only files changed since last commit (use before every commit) |
| `.\vendor\bin\pint app/Models/User.php` | Fix one file (or a folder) |
| `.\vendor\bin\pint -v` | Show exactly what changed |
| `git diff` | See what Pint changed (red = removed, green = added) |

Common rules in its output:
| Rule | Meaning |
|---|---|
| `no_trailing_whitespace` | Removed spaces at line ends |
| `statement_indentation` | Fixed indentation |
| `single_blank_line_at_eof` | One empty line at end of file |
| `blank_line_after_namespace` | Blank line after `namespace ...;` |
| `class_attributes_separation` | Blank line between methods/properties |
| `no_extra_blank_lines` | Removed extra empty lines |

**Habit:** `pint --dirty` → `git add .` → `git commit`

### Git
| Command | What it does |
|---|---|
| `git status` | What changed? (use constantly) |
| `git add .` | Stage all changes |
| `git commit -m "Message"` | Save a snapshot (imperative message: "Add …", "Fix …") |
| `git log --oneline --graph --all` | Visual history |
| `git branch` | List branches (`*` = current) |
| `git switch -c feature/name` | Create + switch to a new branch |
| `git switch main` | Switch to an existing branch |
| `git push -u origin feature/name` | First push of a branch (links it to GitHub) |
| `git push` / `git pull` | Upload / download commits |
| `git diff` | Exact line changes before committing |
| `git restore file` | ⚠️ Discard uncommitted changes to a file |

### Run the app
| Command | What it does |
|---|---|
| `php artisan serve` | Start the dev server at http://localhost:8000 (XAMPP MySQL must be running) |
| `php artisan config:clear` | Clear cached config after editing `.env` |
| `npm run build` | Build CSS/JS assets |

### Reading errors
| Error | Meaning | Fix |
|---|---|---|
| `Parse error: ... expecting "function"` | PHP can't read the file (syntax) | Check file + line number; `php -l` the file |
| `Attempt to assign property "x" on null` | The variable is empty; it was never set | Find the line that should create it and check it ran |
| `User::first()` returns `null` | No rows in the table yet (migrations create tables, not data) | Create a row: `User::factory()->create()` |
| `Class "Role" not found` | Short name used without import | Use the full name `App\Enums\Role` or add `use App\Enums\Role;` |

**Debugging lesson:** the place where an error *appears* isn't always where the bug *is*. Read the file name and line number.
