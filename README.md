# Warp SaaS

**Turn a Filament panel into a SaaS. Every team sees only its own rows — also in queued jobs and commands. Sell plans with features and limits, run free trials, let owners invite their colleagues, hand a customer a zip of all their data when they leave, delete it for good when the time comes, and stop an expensive API from running up your bill.**

<a href="https://demo.warpui.com/saas"><img src="https://raw.githubusercontent.com/qalainau-labs/filament-warp-saas-docs/main/art/live-demo.png" alt="Try the live demo: demo.warpui.com/saas" width="592"></a>

You get a company of your own: switch the plan and watch a page appear and disappear, hit the limit of a plan, invite a colleague, let a trial run out and export the data — at **[demo.warpui.com/saas](https://demo.warpui.com/saas)**. No sign-up needed; your demo company is deleted after a day without visits.

- [What it does](#what-it-does)
- [Installation](#installation)
  - [AI coding assistants (Laravel Boost)](#ai-coding-assistants-laravel-boost)
- [Quick start](#quick-start)
- [Tenant isolation](#tenant-isolation)
- [Tenant data map](#tenant-data-map)
- [Plans, features and limits](#plans-features-and-limits)
- [Trials and account status](#trials-and-account-status)
- [Members and invitations](#members-and-invitations)
- [Data export and deletion](#data-export-and-deletion)
- [Usage quotas](#usage-quotas)
- [AI gateway](#ai-gateway)
- [Operator panel](#operator-panel)
- [Events](#events)
- [Artisan commands](#artisan-commands)
- [Translations](#translations)
- [Testing your app](#testing-your-app)
- [Not included (yet)](#not-included-yet)
- [Requirements](#requirements)
- [Support](#support)
- [License](#license)

## What it does

| You want… | Use |
| --- | --- |
| Each team to see only its own rows, in the panel *and* in jobs and commands | [`BelongsToTenant`, `WarpSaas::actingAs()`](#tenant-isolation) |
| A row that points to another team's row to be refused before it is saved | [`WarpSaas::data()`](#tenant-data-map) |
| Lite / Standard / Pro, with pages that only some plans can open and caps like “5 projects” | [`Plan`, `RequiresFeature`, `EnforcesLimit`](#plans-features-and-limits) |
| A 14-day trial with a countdown bar, a mail before it ends and a page for teams whose trial is over | [`->trial()`](#trials-and-account-status) |
| Owners who invite colleagues by email, change roles and cannot remove the last owner | [`Members` page](#members-and-invitations) |
| A zip of every table and file of a team, and a safe way to delete it all | [`warp-saas:export`, `warp-saas:delete`](#data-export-and-deletion) |
| “50 AI summaries a month”, a monthly cost per team and a budget for the whole service | [`WarpSaas::meter()`](#usage-quotas) |
| Every AI feature of the Warp plugins through one place: the models, the prices, the count per team, the plan that allows it | [`app('warp.ai')`](#ai-gateway) |
| Monthly or yearly invoices paid by bank transfer, reminders before the due date, and the payments recorded by you | [`->billing(Billing::invoice())`](#billing) |
| One screen where you see every team, change its plan, extend its trial or delete it | [`FilamentWarpSaasAdminPlugin`](#operator-panel) |

Every part:

- **uses Filament's own multi-tenancy.** The tenant is your `Team`, `Company` or `Organization` model, set with `->tenant()`. Warp SaaS keeps no tenant model of its own and adds no column to your tables.
- **keeps its state in its own tables** (`warp_saas_*`). Plan, trial and contract of a team are one row in `warp_saas_accounts`. A team without a row counts as *active*, so you can add the plugin to an app that already has teams.
- **stops things where Filament really asks.** Pages are stopped in `canAccess()`, the create page in `getCreateAuthorizationResponse()`, and the models stop writes made outside the panel (queues, imports, commands). Hiding a menu item is never the only protection.
- **works with the four databases.** Tenant keys can be integers, UUIDs or ULIDs.
- **speaks ten languages** (English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese, Korean, simplified and traditional Chinese).

## Installation

> **Warp SaaS is a commercial plugin.** A license is required. After purchase you receive credentials for the private Composer repository.

```bash
composer config repositories.qalainau composer https://qalainau.privato.pub/composer
composer config --auth http-basic.qalainau.privato.pub "your-email" "your-license-key"

composer require qalainau/filament-warp-saas
php artisan migrate
```

The migrations create five tables: `warp_saas_accounts`, `warp_saas_invitations`, `warp_saas_exports`, `warp_saas_usages` and `warp_saas_charges` (the invoices of [billing](#billing)). Later versions add columns with `update_warp_saas_*` migrations, among them `update_warp_saas_accounts_add_read_only` ([look-only mode](#look-only-mode)), `update_warp_saas_accounts_add_after_trial` ([after the trial](#after-the-trial)) and `update_warp_saas_add_cards` ([card payments](#card-payments-stripe)). Your tables are not changed. To edit the migration, publish it with `php artisan vendor:publish --tag=filament-warp-saas-migrations` **before** the first `php artisan migrate`: Warp SaaS then runs your copy instead of its own. If the tables already exist, change them with a migration of your own.

The plugin builds zip files, so PHP needs `ext-zip`. It also uses Laravel's queue: run a worker (`php artisan queue:work`) in production so exports and invitation mails are sent.

### AI coding assistants (Laravel Boost)

Warp SaaS ships instructions for AI coding assistants such as Claude Code, Cursor and Codex. If your app uses
[Laravel Boost](https://laravel.com/docs/boost), the assistant learns from them to use `BelongsToTenant` instead of writing its own tenant scope, to wrap jobs and commands in `WarpSaas::actingAs()` or `eachTenant()` instead of switching the checks off, to stop pages and creates with `RequiresFeature` and `EnforcesLimit`, and how to test it all.

- **Guidelines** (`resources/boost/guidelines/core.md`): a short summary that Boost adds to `CLAUDE.md`, `AGENTS.md` and the other agent files. It also points the assistant to this README in `vendor/`, so it reads the documentation of the version you have installed.
- **Skill** `warp-saas-development` (`resources/boost/skills/`): an example for every part, and tests. The assistant loads it when it works on Warp SaaS.

When `php artisan boost:install` asks "Which third-party AI guidelines/skills would you like to install?", select `qalainau/filament-warp-saas`. To do it without the question, add the package to `boost.json` and run the command again:

```json
{
    "packages": ["qalainau/filament-warp-saas"]
}
```

```bash
php artisan boost:install --guidelines --skills -n
```

## Quick start

Five minutes: a team panel with two plans, a model that belongs to a team, and users who can have several teams.

**1. The tenant model** gets `HasSaasAccount`, and the pivot table of its members needs a role column:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Qalainau\FilamentWarpSaas\Concerns\HasSaasAccount;

class Team extends Model
{
    use HasSaasAccount;

    protected $fillable = ['name', 'slug'];

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)->withPivot('role')->withTimestamps();
    }
}
```

```php
// team_user: team_id, user_id, role (string, nullable), timestamps
$table->string('role', 32)->nullable();
```

By default the roles are `owner` and `member` ([your own roles](#members-and-invitations) are one line). Only owners can open the *Members*, *Plan and usage* and *Export data* pages, so **the person who creates a team must be attached as `owner`**:

```php
namespace App\Filament\Pages\Tenancy;

use App\Models\Team;
use Filament\Forms\Components\TextInput;
use Filament\Pages\Tenancy\RegisterTenant;
use Filament\Schemas\Schema;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class RegisterTeam extends RegisterTenant
{
    public static function getLabel(): string
    {
        return 'Register team';
    }

    public function form(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')->required()->maxLength(255),
        ]);
    }

    protected function handleRegistration(array $data): Model
    {
        $team = Team::create([...$data, 'slug' => Str::slug($data['name']).'-'.Str::lower(Str::random(5))]);
        $team->users()->attach(auth()->user(), ['role' => 'owner']);

        return $team;
    }
}
```

**2. The data models** get `BelongsToTenant`. The table needs the foreign key of the tenant (`team_id` for `Team`, `company_id` for `Company`):

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Qalainau\FilamentWarpSaas\Concerns\BelongsToTenant;

class Project extends Model
{
    use BelongsToTenant;

    protected $fillable = ['name'];
}
```

Every query on `Project` now returns the rows of the current team only, and a new project gets the current team's `team_id`.

If the model has a Filament resource in the tenant panel, keep the `team()` `BelongsTo` relationship that Filament's own tenancy asks for (Filament throws *“The model [App\Models\Project] does not have a relationship named [team]”* without it), or set `protected static bool $isScopedToTenant = false;` on the resource and let `BelongsToTenant` do the scoping alone.

A child table with no tenant column of its own (`tasks` with only a `project_id`) gets [`BelongsToTenantThrough`](#child-tables-belongstotenantthrough) instead.

**3. The user model** says which teams a user can open:

```php
namespace App\Models;

use Filament\Models\Contracts\FilamentUser;
use Filament\Models\Contracts\HasTenants;
use Filament\Panel;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Support\Collection;
use Qalainau\FilamentWarpSaas\Concerns\AccessesSaasTenants;

class User extends Authenticatable implements FilamentUser, HasTenants
{
    use AccessesSaasTenants;

    public function teams(): BelongsToMany
    {
        return $this->belongsToMany(Team::class)->withPivot('role')->withTimestamps();
    }

    public function canAccessPanel(Panel $panel): bool
    {
        return true;
    }

    public function getTenants(Panel $panel): Collection
    {
        // The tenant menu and the first team to open leave out teams whose trial or contract has ended
        return $this->activeTenants($this->teams());
    }

    public function canAccessTenant(Model $tenant): bool
    {
        // Membership only. Do not look at the status here (see below)
        return $this->teams()->whereKey($tenant->getKey())->exists();
    }
}
```

`activeTenants()` takes a relationship or a query (the ended teams are left out in the database) or a collection (one account is read per team).

**Keep `canAccessTenant()` to membership only.** If it also said “and the team is active”, a member who opens the URL of an ended team would get a plain 404 and no explanation. Warp SaaS adds a middleware to the panel that does better: it moves the person to another active team they have (with a notice), or to a page that says “The free trial of Acme ended on November 26” with the date until which the data is kept and a button to export it. A person who is *not* a member of the team still gets the 404, so the name of the team is not shown.

**4. The panel** registers the plugin:

```php
use App\Filament\Pages\Tenancy\RegisterTeam;
use App\Models\Team;
use Qalainau\FilamentWarpSaas\FilamentWarpSaasPlugin;
use Qalainau\FilamentWarpSaas\Plans\Plan;

public function panel(Panel $panel): Panel
{
    return $panel
        ->id('app')
        ->path('app')
        ->login()
        ->tenant(Team::class, slugAttribute: 'slug')
        ->tenantRegistration(RegisterTeam::class)
        ->plugin(
            FilamentWarpSaasPlugin::make()
                ->plans([
                    Plan::make('lite')->label('Lite')->features(['projects'])->limits(['projects' => 2, 'members' => 2]),
                    Plan::make('pro')->label('Pro')->allFeatures()->limits(['projects' => null, 'members' => 50]),
                ])
                ->defaultPlan('lite')
                ->countUsing('projects', fn ($tenant): int => \App\Models\Project::query()->count())
                ->members(relationship: 'users', roleColumn: 'role')
                ->trial(days: 14, plan: 'pro')
                ->startTrialOnTenantCreated()
        );
}
```

Register a team and you have: a 14-day trial on *Pro* with a bar above every page, a *Plan and usage* page, a *Members* page with invitations, and an *Export data* page. When the trial runs out, the team can no longer be opened until you call `WarpSaas::activate($team, 'lite')` (or extend the trial); on *Lite* it is then limited to 2 projects. `countUsing()` is called with the team as the current tenant, so `Project::query()` counts the rows of that team only.

The settings you give the plugin belong to the whole app, not to the panel: a queued job or a command reads the same plans. Every setting of this README can be written on the plugin, as above, or on `app(WarpSaas::class)` in a service provider.

## Tenant isolation

`BelongsToTenant` is a global scope plus rules for creating and updating. It does what Filament's tenant scoping does inside the panel — and keeps doing it where Filament is not running.

**What is “the current tenant”?** In this order: inside `WarpSaas::actingAs()` (and inside a queued job, which carries the tenant it was dispatched in) → the tenant Filament has set for the request → the function you give to `resolveTenantUsing()`. Inside `WarpSaas::withoutTenant()` there is none.

### What `BelongsToTenant` does when it reads

| Where the query runs | What happens |
| --- | --- |
| A current tenant exists | Only that tenant's rows. |
| `WarpSaas::withoutTenant()` | Not limited. You see every tenant. |
| In a panel with `->tenant()` and the tenant is not decided yet | Nothing is returned. |
| In your operator panel: the panel with `FilamentWarpSaasAdminPlugin`, or one you name with `->operatorPanels()` | Not limited. Who may enter that panel is decided by `canAccessPanel()`. |
| In any other panel without tenancy (a staff panel, a customer portal) | The same as outside the panels: the tenant from `resolveTenantUsing()`, else `TenantNotSet`. |
| Outside the panels: a job, a command, a normal route, a tinker session | **`TenantNotSet` is thrown.** Warp SaaS does not quietly return every customer's rows. |

The last row is the one that matters. A command that says `Invoice::query()->where('paid', false)->get()` on a multi-tenant app is almost always a leak; here it fails the first time you run it, and you write what you meant:

```php
use App\Models\Invoice;
use Qalainau\FilamentWarpSaas\WarpSaas;

// One tenant at a time, each inside its own tenant context (active and trialing tenants only, read one by one)
WarpSaas::eachTenant(function ($team): void {
    Invoice::query()->where('paid', false)->each(fn (Invoice $invoice) => $invoice->remind());
});

// Or: really all tenants, written down where it happens
$unpaid = WarpSaas::withoutTenant(fn () => Invoice::query()->where('paid', false)->count());
```

### The functions

| Function | What it does |
| --- | --- |
| `WarpSaas::tenant(): ?Model` | The current tenant, or `null`. |
| `WarpSaas::actingAs($tenant, fn () => …)` | Runs the function as that tenant. Can be nested, and the previous state returns even when the function throws. Returns the function's result. |
| `WarpSaas::eachTenant($callback, bool $onlyActive = true)` | Runs the callback once per tenant inside `actingAs()`. Tenants are read one by one (`lazyById`), so a large number does not fill the memory. `$onlyActive = false` includes ended tenants. |
| `WarpSaas::withoutTenant(fn () => …)` | Runs the function without limiting by tenant. Use it where you mean “all of them”. |
| `WarpSaas::isWithoutTenant(): bool` | Whether you are inside `withoutTenant()`. |
| `->resolveTenantUsing(fn () => …)` | A function that finds the tenant when Filament has none: a panel of your own, a different guard. Example: `fn () => auth('staff')->user()?->team`. When it returns `null` (the login page of that panel, nobody signed in) there is no tenant, and a query throws `TenantNotSet` as it does outside the panels. |
| `->operatorPanels('admin')` | The ids of the panels without tenancy that may see every tenant. The panel with `FilamentWarpSaasAdminPlugin` is one without being named. Any other panel without tenancy is treated like code outside the panels. |
| `->tenantForeignKey('team_id')` | The column your data tables use. Default: the foreign key of the tenant model (`Team` → `team_id`). To use another column in one model, override `getTenantForeignKeyName()` in it. |
| `->tenantModel(Team::class)` | The tenant model. Default: the model of the `->tenant()` of the panel that has the plugin. Give it when you use no Filament panel to decide the tenant, or when several panels have tenancy. |
| `->strictOutsidePanels(false)` | Outside the panels, do not throw and do not limit (the old behaviour). For the weeks in which you move an existing app to Warp SaaS, command by command. Default `true`. |

`BelongsToTenant` also gives the model `saasTenant()`, a `BelongsTo` to the tenant.

### Creating and updating

| What you do | What happens |
| --- | --- |
| Create with the foreign key empty | The current tenant is filled in. With no current tenant: in a tenant panel, or outside the panels while strict, **`TenantNotSet`**; elsewhere the row is created as it is. |
| Create with the foreign key of **another** tenant than the current one | **`TenantMismatch`**. A form value that was edited in the browser cannot put a row into someone else's team. |
| Change the foreign key of an existing row | **`TenantMismatch`**. A row cannot be moved to another tenant. |
| Any of the above inside `withoutTenant()` | Allowed. This is how a seeder or a migration of data creates or moves rows. |

### Child tables: `BelongsToTenantThrough`

Many tables have no `team_id` of their own: the tasks of a project, the lines of an invoice, the comments on a record. Their team is the team of their parent. Without a scope, `Task::all()` returns the tasks of every customer. Give such a model `BelongsToTenantThrough` and name the relationship to the parent:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Qalainau\FilamentWarpSaas\Concerns\BelongsToTenantThrough;

class Task extends Model
{
    use BelongsToTenantThrough;

    public function tenantParent(): string
    {
        return 'project';
    }

    public function project(): BelongsTo
    {
        return $this->belongsTo(Project::class);
    }
}
```

`Task::query()` now returns only the tasks whose project belongs to the current team. It behaves exactly like [`BelongsToTenant`](#what-belongstotenant-does-when-it-reads): all rows inside `withoutTenant()`, nothing in a tenant panel before the tenant is decided, everything in the operator panel, `TenantNotSet` outside the panels.

- **The parent can be a child too.** The parent of `tenantParent()` uses `BelongsToTenant` or `BelongsToTenantThrough` (or is the tenant model itself), so `TaskStep` → `Task` → `Project` → team works at any depth.
- **A polymorphic parent** (`morphTo`) needs the list of models it can be, because the scope cannot read every type from the table on each query. A row whose type is not in the list is not returned:

```php
public function tenantParent(): string
{
    return 'commentable';
}

public function tenantParentTypes(): array
{
    return [Project::class, Task::class];
}
```

- **Only the tenant is filtered, not the parent's other scopes.** The query is `WHERE project_id IN (SELECT id FROM projects WHERE team_id = ?)`, not `whereHas()`, so a task of a soft-deleted project is still found, and the parent's own scope is not applied twice.
- **Saving is checked too, with or without the trait.** Creating a task under another team's project, or moving it there, throws `TenantMismatch` ([below](#the-reference-guard)). For this the [tenant data map](#tenant-data-map) has to know the parent: a `NOT NULL` foreign key is found on its own, a polymorphic parent is registered with `morphParent()`.
- **There is nothing to fill in when creating.** The row's team is the parent's team; pick a parent of the current team.

### Inside a Filament panel with tenant scoping

Filament adds its own global scope to the model of every resource in a panel with `->tenant()` (unless the resource sets `$isScopedToTenant = false`). That scope always uses the tenant in the URL. Inside `WarpSaas::actingAs()`, `withoutTenant()` and a queued job, Warp SaaS removes Filament's scope from the query, and a new row gets the tenant of `actingAs()` even though Filament's resource would fill in the URL's tenant. So a page of team A that counts all teams with `withoutTenant()`, or works on team B with `actingAs()`, gets what it asked for, and you can keep Filament's scoping on in your resources. Outside these functions both scopes apply, as before.

### Queued jobs

A job carries the tenant it was dispatched in, and runs as that tenant. The tenant travels in the job's payload (a `warp-saas` key added with `Queue::createPayloadUsing()`), not in Laravel's `Context`, so `Context::flush()` or a rebuilt container (a long-running worker, Octane) does not lose it, and a job dispatched from inside a job gets the tenant of that job. This is tested with the `sync` and the `database` queue (every driver builds the payload the same way), for jobs, queued listeners, notifications and mailables, on the lowest supported Laravel (12.40) too.

```php
use App\Jobs\SendReminders;
use Qalainau\FilamentWarpSaas\WarpSaas;

// In a request in the team panel the job gets the team. From a command, say which one:
WarpSaas::actingAs($team, fn () => SendReminders::dispatch());
```

- The tenant is read again when the job runs. If it has been deleted, or its trial or contract has ended, **the job is deleted and not run**, and a line is written to the log (`[warp-saas] Skipped queued job …`).
- A job dispatched with no tenant runs with no tenant. If it reads a `BelongsToTenant` model it gets `TenantNotSet`, outside the panels exactly like a command.
- Jobs that must still run for an ended tenant — the “your contract has ended” mail, an export, a deletion — implement the marker interface `Qalainau\FilamentWarpSaas\Contracts\RunsForEndedTenants`. For a mailable, a notification or a queued listener, put it on that class. A tenant that no longer exists is skipped even then.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use Qalainau\FilamentWarpSaas\Contracts\RunsForEndedTenants;

class SendContractEndedMail implements RunsForEndedTenants, ShouldQueue
{
    // …
}
```

### Good to know

- **A model event is needed for the rules to run.** `Model::query()->update([...])`, `->insert()` and `DB::table()` do not fire `creating` or `updating`, so they do not check the foreign key of a row you write, and the [reference guard](#tenant-data-map) does not see them. (The scope still limits which rows a builder `update()` touches.) Write through the model in code that handles user input, and run [`warp-saas:check-references`](#tenant-data-map) for data that arrived any other way.
- **A `sync` job dispatched from the operator panel has no tenant.** The operator panel has no tenancy, so the job carries none, and a job always counts as “outside the panels” — even when `QUEUE_CONNECTION=sync` makes it run inside that request. If it reads a `BelongsToTenant` model it fails with `TenantNotSet`. Dispatch it inside `WarpSaas::actingAs($team, …)`, or `withoutTenant()` in the job if it really works on all tenants.
- **Return the tenant, not the dispatch.** `actingAs($team, fn () => SomeJob::dispatch())` works: the job is dispatched inside the function. Code that stores the `PendingDispatch` and lets it run later runs it outside.

## Tenant data map

Exporting a team, deleting a team and refusing cross-team references all need the same knowledge: *which rows belong to which team*. Warp SaaS reads it from your database schema and from a few lines you add. All of the following are called on `WarpSaas::data()`, usually in `AppServiceProvider::boot()`.

A table is one of three things:

1. **Shared** — not owned by a tenant: `users`, `sessions`, `jobs`, `cache`, `failed_jobs`, `migrations`, `password_reset_tokens`, `job_batches`, `cache_locks` and the tables of the tenant and user models are shared from the start. Anything else you list with `shared()`.
2. **Tenant-owned** — it has the foreign key of the tenant (`team_id`), or the pair `tenant_type` + `tenant_id` (Warp plugins use this pair).
3. **Tenant data by its parent** — no tenant column, but a `NOT NULL` foreign key to a table that is tenant data (`invoice_lines.invoice_id`), or a polymorphic parent you register with `morphParent()`.

A table that is none of these is reported by the [test](#testing-your-app), so a new table cannot silently fall outside the export, the deletion and the checks.

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

WarpSaas::data()
    ->shared(['countries', 'currencies'])
    ->referable(['tax_rates'])
    ->morphParent('comments', 'commentable_type', 'commentable_id')
    ->reference('tasks', 'assignee_id', 'users')
    ->morphReference('activity_log', 'subject_id', 'subject_type')
    ->files('deliveries', ['photo_path' => 'public', 'signature_path' => 'signature_disk'],
        accept: fn ($team, string $disk, string $path): bool => str_starts_with($path, "teams/{$team->getKey()}/"))
    ->notAReference('payments', 'external_id', 'The ID of the payment at Stripe')
    ->tenantFiles(fn ($team) => [['disk' => 'public', 'path' => $team->logo_path]])
    ->indexExempt('team_settings', 'One row per team');
```

| Method | Use it when |
| --- | --- |
| `shared(array $tables)` | A table is the same for every tenant: master data, settings, logs. |
| `referable(array $tables)` | A shared table that tenant data may point to with a foreign key (a table of countries). Also shared. By default tenant data may only point to tenant data, the tenant and the users. |
| `morphParent($table, $typeColumn, $idColumn)` | The owner of the row is a polymorphic parent (comments, attachments, activity). |
| `reference($table, $column, $targetTable)` | A column holds the id of a row but has **no foreign key** (kept off on purpose, circular, or a string column). The reference guard and the check command follow it. |
| `morphReference($table, $idColumn, $typeColumn)` | The same for a polymorphic pointer that is not the parent (the subject of a log line). |
| `files($table, array $columns, ?Closure $accept = null)` | A column holds a file path. Value: a disk name (`'public'`) or the name of a column that holds the disk. A JSON array of paths is read as several files. Exports include these files and deletion removes them. The path is used as it is in the row, so if users can type it (or a bug writes another team's path), pass `accept: fn (Model $tenant, string $disk, string $path): bool` — a file it refuses is neither exported nor deleted (it must return `true` itself; several accepts on one table must all agree). `accept` gets the path as export and deletion use it (`\\` turned into `/`, no leading `/`), and a path with a `..` segment is refused before `accept` is called, so `tenants/1/../2/secret.txt` cannot slip past a `str_starts_with()` check. |
| `notAReference($table, $column, $reason)` | A column ends with `_id` but points to nothing in your database (an ID at an outside service). Needed so the test does not report it. |
| `tenantFiles(Closure $callback)` | Files that belong to a tenant but are not in a row column (the logo). Return `[['disk' => …, 'path' => …], …]`. Can be called several times. |
| `indexExempt($table, $reason)` | A tenant table that has a few rows per tenant and needs no index that starts with the tenant column. |
| `guardReferences(bool $enabled = true)` | Turns the [reference guard](#the-reference-guard) off or on. |
| `withoutGuard(fn () => …)` | Pauses the guard for one block (importing data in which a row comes before the row it points to). |
| `flush()` | Forgets what was read from the schema. Call it after you create or drop tables in a test. Your registrations stay. |

**Tables of other Warp plugins are already in the map.** If `warp_scheduler_*`, `warp_capacity_*`, `warp_flow_*`, `warp_print_*` and the other Warp tables exist in your database, their pointers, polymorphic parents and file columns are registered for you, so an export includes the files of Warp Print and a deletion finds the rows of Warp Scheduler. What is not known to Warp SaaS are pointers *from* a plugin table *to your own tables* (the `employee_id` of a Warp Capacity booking points to your `employees`): register those with `reference()`.

### The reference guard

On every create and update through Eloquent, Warp SaaS checks that the registered pointers of the row (foreign keys, `reference()`, `morphReference()`) point to rows of **the same tenant**, and throws `Qalainau\FilamentWarpSaas\Exceptions\CrossTenantReference` if not.

When there is a current tenant, it also checks that **the row itself belongs to the current tenant** — by its tenant column, or through its parent for a child table — and throws `TenantMismatch` if not: a task created in team A under a project of team B, or moved to it, is refused even when the model has no trait. A row of another team that was read inside `withoutTenant()` cannot be saved outside it either. This check is skipped inside `withoutTenant()`, and when the current tenant is of another model than the map's tenant model. On create all pointers are checked; on update only the columns that changed (all of them if the tenant column or the parent changed). It also runs inside `withoutTenant()`: seeing all tenants is not a licence to link them. A shared table and a table with no pointers cost nothing after the schema has been read once.

```php
// $order belongs to Acme, $customer belongs to Globex
Order::create(['customer_id' => $customer->id]); // CrossTenantReference — "orders.customer_id points to customers #12 of tenant 7, but the row belongs to tenant 3"

// In team 3, under a project of team 7
Task::create(['project_id' => $globexProject->id]); // TenantMismatch — "Cannot create a row of [tasks] for tenant [7] (its tenant comes from tasks.project_id → projects) while the current tenant is [3]"
```

Checking the row's tenant reads the parent at most once per save (and only on create or when the parent changes); a table with its own tenant column costs nothing.

**It runs before your model's own `creating` and `updating`.** The check runs at `saving` when the row's tenant is already known there, so a refused row never reaches a `creating` listener that numbers it or writes a log line; it runs again at `creating` or `updating` only if something changed the row after `saving` (the tenant column set by `BelongsToTenant`, a listener that fills a pointer). Listeners on your model's own `saving` still run first: if one of them has side effects, call the check yourself before them:

```php
static::saving(function (Order $order): void {
    WarpSaas::data()->checkReferences($order); // throws CrossTenantReference or TenantMismatch
    $order->number ??= OrderNumber::next();
});
```

### Checking data that is already there

```bash
php artisan warp-saas:check-references
php artisan warp-saas:check-references --table=orders
```

Reads every tenant-data table (1,000 rows at a time) and prints each row that points to another tenant's row. The exit code is 1 when it finds any, so it can run before a deploy. Run it after an import, after you add the plugin to an app with data, and now and then: it is the only way to see violations made with `DB::table()` or raw SQL.

## Plans, features and limits

A **plan** is a set of *features* (“inventory”, “reports”) and *limits* (“5 projects”, “10 members”). A team has one plan (its account says which), can be given extra features, and can have its own limit values.

```php
use Qalainau\FilamentWarpSaas\FilamentWarpSaasPlugin;
use Qalainau\FilamentWarpSaas\Plans\Plan;

FilamentWarpSaasPlugin::make()
    ->plans([
        Plan::make('lite')
            ->label('Lite')
            ->description('For a small team that is just starting.')
            ->price(29, 'USD')
            ->features(['projects'])
            ->limits(['projects' => 2, 'members' => 2]),
        Plan::make('standard')
            ->label('Standard')
            ->price(79, 'USD')
            ->features(['projects', 'reports'])
            ->limits(['projects' => 5, 'members' => 5]),
        Plan::make('pro')
            ->label('Pro')
            ->price(1990, 'USD', 'year')
            ->allFeatures()
            ->limits(['projects' => null, 'members' => 50]),
    ])
    ->defaultPlan('lite')
    ->features(['projects' => 'Projects', 'reports' => 'Reports'])
    ->metrics(['projects' => 'Projects'])
    ->countUsing('projects', fn ($tenant): int => \App\Models\Project::query()->count())
    ->contactUrl('mailto:sales@example.com?subject=Plan');
```

Plan keys, feature keys and role keys are strings or `BackedEnum`s (the `->value` is stored), so your existing enums can be passed as they are: `Plan::make(PlanType::Pro)`, `->features([Module::Inventory])`.

### `Plan`

| Method | What it does |
| --- | --- |
| `Plan::make(string\|BackedEnum $key)` | The key is what is stored in `warp_saas_accounts.plan`. |
| `Plan::fromArray($key, array $definition)` | From `['label' =>, 'description' =>, 'price' =>, 'features' =>, 'limits' =>]`. `'features' => '*'` or `'all_features' => true` means all. `->plans()` also accepts such arrays keyed by the plan key. |
| `label(string\|Closure\|null)` | The name on screen. Default: the label of a `HasLabel` enum, or the key turned into words (`lite` → *Lite*). |
| `description(string\|Closure\|null)` | A line under the name on the *Plan and usage* page. |
| `price(int\|float\|null $amount, ?string $currency = null, string $per = 'month')` | The price, shown on the page. With [billing](#billing) turned on it is also what an invoice charges — only when `$per` is `month` or `year` (any other word is only shown). |
| `features(array $features)` | The features of the plan. Replaces what was set before. |
| `allFeatures(bool $condition = true)` | Every feature, including those added later. |
| `limits(array $limits)` | `['projects' => 5, 'members' => null]`. `null` and a metric left out mean *no limit*. Replaces what was set before. |
| `meter($meter, $limit, $budget)` | The monthly [usage quota](#usage-quotas) of this plan, written from the plan's side. |

### Settings

| Setting | What it does |
| --- | --- |
| `plans(array $plans)` | The plans. Replaces the previous list. **With no plan registered, nothing is switched or limited**: `hasFeature()` is `true` for everyone. |
| `defaultPlan(string\|BackedEnum\|null $key)` | The plan of a team whose account has no plan (or a plan you removed). Default: the first plan. |
| `features(array\|string $features)` | The list of features with labels, for the *Plan and usage* page and the operator panel: `['projects' => 'Projects']`, `['projects', 'reports']`, or the class name of an enum (a `HasLabel` enum gives its label). `allFeatures()` plans cover these. |
| `metrics(array\|string $labels)` | Labels of the limits (`members` is called *Members* without being registered). |
| `countUsing(string $metric, Closure $callback)` | How to count a metric: `fn (Model $tenant): int`. It runs **as that tenant**, so a `BelongsToTenant` model can be counted directly. You decide whether deleted or inactive rows count. `members` is built in: members plus pending invitations. A `countUsing('members', …)` of yours is used instead (when your app keeps pending members somewhere else). |
| `contactUrl(string\|Closure\|null $url)` | Where “Ask about changing the plan” and “Contact us” go (a `mailto:` or a URL, or `fn (?Model $tenant): ?string`). Without it the buttons are not shown. |

### Asking and changing, in code

| Call | Result |
| --- | --- |
| `WarpSaas::hasFeature('reports')` | Whether the current tenant can use the feature (plan features plus added ones). A second argument takes another tenant. `false` when there is no current tenant. |
| `WarpSaas::featuresOf($tenant = null): array` | The keys of all features the tenant can use. |
| `WarpSaas::planOf($tenant = null): ?Plan` | The plan (the account's plan, else the default plan). |
| `WarpSaas::limit('projects')` | A `Limit` for the current tenant (a second argument takes another). |
| `WarpSaas::changePlan($tenant, 'pro', force: false)` | Changes the plan. **Refuses with `PlanChangeBlocked`** when the team already has more than the new plan allows (it says which limit and how many); `force: true` changes it anyway. `UnknownPlan` for a key you did not register. Fires `PlanChanged`. |
| `WarpSaas::grantFeature($tenant, 'accounting')` / `revokeFeature(…)` | Adds a feature outside the plan for this team only (an add-on), or takes it away again. A feature that is in the plan is not removed by `revokeFeature()`. |
| `WarpSaas::setLimit($tenant, 'projects', 20)` / `resetLimit(…)` | Gives this team another limit than its plan (`null` = no limit), or goes back to the plan's. |
| `WarpSaas::withoutLimits(fn () => …)` | Inside, `CountsTowardLimit` does not check (seeders, imports, moving data). |
| `WarpSaas::isOwner($user = null, $tenant = null)` / `roleOf(…)` | Whether the user (default: the signed-in one) has an owner role in the tenant / which role. |

### `Limit`

`WarpSaas::limit('projects')` returns an object that counts again at every call (it keeps no number):

| Method | Result |
| --- | --- |
| `limit(): ?int` | The limit: the team's own value, else the plan's. `null` is no limit. |
| `used(): int` | The count from `countUsing()`. An exception if no counter is registered for the metric. |
| `remaining(): ?int` | How many more fit (`null` when there is no limit). |
| `isReached(): bool` | Whether no more fit. |
| `canAdd(int $count = 1): bool` | Whether `$count` more fit. |
| `ensureCanAdd(int $count = 1): void` | Throws `LimitReached` if not. Its message can be shown to the user: *“On this plan, “Projects” is limited to 2. You have 2 now.”* (for `members`: *“…up to 3 members. There are 3 members now.”*). The HTTP status is 403. |
| `message(): string` | That sentence. |
| `whileLocked(Closure $callback)` | Opens a transaction, locks the team's account row and runs the function. **Count and add inside it** so two requests at the same moment cannot both pass. |

```php
use App\Models\Site;
use Qalainau\FilamentWarpSaas\Plans\Limit;
use Qalainau\FilamentWarpSaas\WarpSaas;

WarpSaas::limit('sites')->whileLocked(function (Limit $limit) use ($data): Site {
    $limit->ensureCanAdd();

    return Site::create($data);
});
```

### Stopping a page: `RequiresFeature`

Add it to a Filament resource or page. Return the feature it needs (a string, an enum, or an array meaning *any one of these*):

```php
use BackedEnum;
use Filament\Pages\Page;
use Qalainau\FilamentWarpSaas\Concerns\RequiresFeature;

class Reports extends Page
{
    use RequiresFeature;

    public static function requiredFeature(): string|BackedEnum|array
    {
        return 'reports';
    }
}
```

The trait overrides **`canAccess()`**, which is the entrance Filament really asks: the menu item disappears, every page of the resource returns 403, and so does every Livewire call on it, even when someone types the URL. With no current tenant nobody gets in. If you write your own `canAccess()` in the class, call `static::canAccessRequiredFeature()` from it.

For routes outside Filament, a middleware: `Route::get('/api/stock', …)->middleware('warp-saas.feature:inventory')`. Several features separated by commas mean *any one*. The current tenant comes from `WarpSaas::tenant()` (see `resolveTenantUsing()`); without one the answer is 403.

### Stopping a create: `EnforcesLimit` and `CountsTowardLimit`

```php
use Filament\Resources\Resource;
use Qalainau\FilamentWarpSaas\Concerns\EnforcesLimit;

class ProjectResource extends Resource
{
    use EnforcesLimit;

    public static function limitMetric(): string
    {
        return 'projects';
    }
}
```

```php
use Illuminate\Database\Eloquent\Model;
use Qalainau\FilamentWarpSaas\Concerns\BelongsToTenant;
use Qalainau\FilamentWarpSaas\Concerns\CountsTowardLimit;

class Project extends Model
{
    use BelongsToTenant, CountsTowardLimit;

    public static function limitMetric(): string
    {
        return 'projects';
    }
}
```

- `EnforcesLimit` (on the **resource**) overrides `getCreateAuthorizationResponse()`. When the team is at its limit, the create page answers 403, and the *New project* button in the list stays visible but cannot be pressed, with the sentence “On this plan, “Projects” is limited to 2. You have 2 now.” as its tooltip. If you write `getCreateAuthorizationResponse()` yourself, return `static::limitAuthorizationResponse($parentResponse)` from it.
- `CountsTowardLimit` (on the **model**) checks in `creating`, so a queued job, an import or an API controller cannot go over the limit either. Inside a transaction (Filament's create page runs in one) the account row is locked first, so concurrent creates cannot both pass. Outside a transaction wrap the create in `->whileLocked()` as above. It checks nothing when the plan has no limit for the metric, when no tenant can be found, or inside `withoutLimits()`.
- **Rows that do not count.** If `countUsing()` counts only some rows (active technicians, published projects), say the same on the model: `public function countsTowardLimit(): bool { return (bool) $this->is_active; }`. A row that does not count can be created at the limit, and the limit is checked when an update makes it count (inactive → active). Without the method every row counts and only creates are checked.

> **Do not override `canCreate()` to enforce a limit.** In Filament the create page does not call `canCreate()` of your resource class the way the table button does; it asks `getCreateAuthorizationResponse()`. A limit written in `canCreate()` hides the button and leaves the page (and its URL) open. `EnforcesLimit` uses the right method, and `CountsTowardLimit` covers what is not a Filament page.

### The *Plan and usage* page

Added to the tenant panel for owners: the plan with its price and description, the features of the plan (and those added for this team), each limit with the current count, the state of the trial or contract with its dates, “this month's usage” of the [usage quotas](#usage-quotas), and an *Ask about changing the plan* button when `contactUrl()` is set. A team changes its plan by asking you — or, in the [operator panel](#operator-panel), by pressing a button.

Replace or remove it, and move all the pages of the plugin in the menu:

```php
FilamentWarpSaasPlugin::make()
    ->planPage(false)                     // remove; or ->planPage(MyPlanPage::class), a class that extends PlanAndUsage
    ->membersPage(false)                  // or a class that extends Members
    ->exportPage(false)                   // or a class that extends DataExport
    ->navigationGroup('Account')
    ->navigationSort(90)
    ->navigationIcon(Heroicon::OutlinedCreditCard)
```

**Showing the invoice history on your own page.** If you remove the plugin's page with `->planPage(false)` and build your own *Subscription* page, put the invoice history on it with two public helpers, `ChargeHistory::section()` and `InvoiceLinks::forTenant()`:

```php
use Filament\Facades\Filament;
use Qalainau\FilamentWarpSaas\Billing\ChargeHistory;
use Qalainau\FilamentWarpSaas\Billing\InvoiceLinks;
use Qalainau\FilamentWarpSaas\Models\Charge;

// inside an infolist / schema of your page; $tenant is Filament::getTenant()
ChargeHistory::section(
    tenant: $tenant,                                  // only this team's invoices are listed
    name: (string) Filament::getTenantName($tenant),  // shown in the description
    link: fn (Charge $charge): ?string => InvoiceLinks::forTenant($charge, $tenant, Filament::getCurrentPanel()),
);
```

`ChargeHistory::section(Model $tenant, string $name, Closure $link, string $key = 'warp_saas_charges', bool $cards = false): Section` returns a section with the latest 24 invoices (what for, total with tax, due date, status, invoice link). `$link` receives a `Charge` and returns a URL or `null`. `InvoiceLinks::forTenant(Charge $charge, Model $tenant, ?Panel $panel = null, bool $ended = false): ?string` returns the `http(s)` link your `issueInvoiceUsing()` gave, or the plugin's invoice page when `seller()` is set, or `null`.

## Trials and account status

The state of a team is decided in one place, `Account::status()`, in this order:

| State | When | What the team can do |
| --- | --- | --- |
| **Ended** | `ended_at` is in the past, **or** the contract was never started (`activated_at` empty) and `trial_ends_at` is in the past | Nothing. People are moved to another team or to the ended page. Data is kept until the end of the retention period. |
| **Active** | `activated_at` is set. Also a team with no account row, and a row with no dates | Everything its plan allows. The trial date, if any, is not looked at any more. |
| **Trialing** | `activated_at` is empty and `trial_ends_at` is in the future | Everything its plan allows, with the bar above the pages. |

`Active` is the default, so existing teams keep working when you add the plugin.

```php
use App\Models\Team;
use Qalainau\FilamentWarpSaas\WarpSaas;

$team = Team::find(1);

$team->accountStatus();    // AccountStatus::Trialing  (Active, Trialing, Ended — a HasLabel enum)
$team->isActive();         // true for Active and Trialing
$team->onTrial();          // true for Trialing
$team->trialDaysLeft();    // 14 today, 0 on the last day, null when not on trial
$team->trialLastDay();     // CarbonImmutable, the last day in the team's time zone

Team::whereAccountActive()->get();   // a query scope: Active and Trialing teams
```

`HasSaasAccount` also gives `account()` (the `Account`, a new unsaved one when the team has no row; it reads the database once per model — use `refreshAccount()` to read again), and `saasAccount()`, the relationship, for `with('saasAccount')` in a list.

### Settings

| Setting | What it does |
| --- | --- |
| `trial(int $days = 14, string\|BackedEnum\|null $plan = null)` | The default length and the plan during the trial (`null` = the default plan). |
| `startTrialOnTenantCreated(bool $enabled = true)` | When a tenant model is created (from the Filament registration page, a seeder, a command) and it has no account row yet, start a trial. |
| `trialReminderDays(array $days = [3, 1])` | How many days before the last day the owners get a mail (`0` = on the last day). |
| `timezoneUsing(?Closure $callback)` | `fn (Model $tenant): ?string`. The time zone of a team, for the days of the trial. Default: `config('app.timezone')`. |
| `trialBanner(bool\|Closure $enabled = true)` | The bar above the pages while a team is on trial. Default on. A function decides page by page: `fn (Panel $panel, Model $tenant, ?Authenticatable $user): bool` (by name). The settings belong to the whole app, so use it to leave the bar out of one of the panels that have the plugin: `->trialBanner(fn (Panel $panel) => $panel->getId() !== 'field')`. |
| `dataRetention(int $days = 30)` | How long the data of an ended team is kept. Shown on the ended page; `warp-saas:purge` deletes after it. |
| `contactUrl(…)` | The *Contact us* link in the bar (owners only) and on the ended page. |

### The five actions

All are static and change the account row under a lock. Each one returns the `Account`, and the model you pass (and the current tenant model) is updated, so `$team->accountStatus()` is right on the next line.

| Call | What it does |
| --- | --- |
| `WarpSaas::startTrial($tenant, ?int $days = null, $plan = null, $until = null)` | Starts a trial (defaults from `trial()`). Throws `AccountAlreadyActive` when the team already has a contract. On an ended team it starts again: `ended_at` and the “mail sent” date are cleared. Event `TrialStarted`. `until` gives the end instead of the days, read like `end()`: `until: '2026-11-26'` makes 25 November the last day. Not together with `days`. |
| `WarpSaas::extendTrial($tenant, int $days)` | Moves the last day `$days` later. If the trial is already over, counts from today. The reminder mails are armed again. Event `TrialExtended`. |
| `WarpSaas::activate($tenant, $plan = null)` | Starts the contract. With a plan, it is changed like `changePlan()` is — **refused with `PlanChangeBlocked` and the contract not started** if the team has more than the plan allows. Event `AccountActivated`. |
| `WarpSaas::end($tenant, DateTimeInterface\|string\|null $at = null)` | Sets when the contract ends. `null` is now. A string is read in the team's time zone; a date alone is that day at 0:00. Event `AccountEnded`. |
| `WarpSaas::resume($tenant)` | Removes the end set by `end()`. A team whose *trial* has run out is still ended: use `extendTrial()` or `activate()`. Event `AccountResumed`. |

### How dates are counted

Everything is stored in UTC and counted in the **calendar of the team's time zone** (`timezoneUsing()`).

- **`trial(days: 14)` means: today is day 0, and the last day is 14 days later.** A trial started on 12 November has 14 days left that day, 1 day left on 25 November and 0 on **26 November — the last day**. It ends at midnight at the end of the last day, so on 26 November the team still works, and at 27 November 0:00 it is ended. On the last day the bar says it is the last day, not “0 days left”.
- Calendar days, not 24-hour blocks: a day with a daylight-saving change is still one day.
- **`end($tenant, '2026-12-31')` ends the contract at the *start* of 31 December**, so 30 December is the last day. To let a team work through 31 December, pass `'2027-01-01'`. The “last day” shown to people is always the day before the end.
- The reminder mail checks the day in the team's time zone and sends **at most one mail per team per day**.

### The bar

On every page of a team that is on trial, above the content: *“The free trial of Acme has 3 days left. You can use it until the end of November 26.”* Warning colour at 3 days or fewer, info colour before. Owners get the *Contact us* link when `contactUrl()` is set; other members get a line telling them to ask an owner. The bar is drawn inside the page, so it disappears at once when you press *Start the contract* in the same session.

### Ended teams

`FilamentWarpSaasPlugin` adds a middleware to the tenant panel (also for Livewire requests, so a button pressed on an open page is caught too) and a page outside any team (`/app/account-ended/{team}`):

- A person who opens an ended team and has **another active team** is sent to it, with a notice: *“The free trial of Acme ended. You were moved to Globex.”*
- Otherwise they see the ended page: the title (“The free trial of Acme ended on November 26.” or “The contract of Acme ended on …”), until when the data is kept, a *Contact us* button, a way to sign out, and — for owners, while the data is kept — an **Export the data** button (see [export](#data-export-and-deletion)).
- A person who signs in and has only ended teams lands on the ended page instead of a 404.
- Not a member of that team: 404, the team's name is not shown.

To answer with your own page or redirect instead, give a function to `whenAccountEnded()`. Its arguments are passed by name (`$tenant`, `$account`, `$request`). Return a response to send it, or `null` to keep the behaviour above:

```php
FilamentWarpSaasPlugin::make()
    ->whenAccountEnded(fn (Team $tenant) => redirect()->route('billing.reopen', $tenant));
```

### After the trial

By default a trial that runs out ends the team (see [Ended teams](#ended-teams)). If you want a team that tried your app to become a paying customer without anyone pressing a button, say what should happen:

```php
use Qalainau\FilamentWarpSaas\Enums\AfterTrial;

FilamentWarpSaasPlugin::make()
    ->trial(days: 14, plan: 'pro')
    ->afterTrial(AfterTrial::AskOwner);
```

| `AfterTrial` | What happens to a trial that runs out |
| --- | --- |
| `End` (default) | The team ends, as before. Only you (the operator panel, or `WarpSaas::continueAfterTrial()` in code) can make it continue. |
| `Activate` | Every trial **starts** as "continue after the trial". The owners can switch it off on *Plan and usage* (*End with the trial*). |
| `AskOwner` | A trial starts as "end". An owner presses *Continue after the trial* on *Plan and usage* when they want to stay (and can press *End with the trial* to undo it until the trial ends). |

```php
WarpSaas::continueAfterTrial($team);          // the trial becomes a contract when it ends
WarpSaas::continueAfterTrial($team, false);   // the trial ends the team
```

`continueAfterTrial()` returns the `Account`. It does nothing for a team that already has a contract, and throws `NotOnTrial` for a team that is not on a trial. Setting the value it already has dispatches nothing. Owners see and change it only from their own team's *Plan and usage* page; a member who is not an owner gets a 403, and another team's account cannot be reached. `extendTrial()` keeps the choice.

**When the contract really starts.** A team whose trial is over and that was set to continue counts as *Active* at once (`$team->accountStatus()`, `Team::whereAccountActive()` and the other queries read it that way). The contract itself (`activated_at`, the event `AccountActivated`, and the first invoice when [billing](#billing) is on) is written by the first of these that runs after the trial ends: `warp-saas:notify-trials`, `warp-saas:bill`, or a person opening a page of that team. The day it is dated is the day after the last day of the trial, whenever it is written, so billing starts on the right day even if the command runs hours later. Running it twice makes one contract and one invoice.

- *Activate* with a team that is already on a trial changes nothing: the setting applies to trials started afterwards. A team that existed before you updated gets `activates_after_trial = false`, so its trial still ends it; use `WarpSaas::continueAfterTrial()` for those you want to keep.
- A team on a card ([card payments](#card-payments-stripe)) that becomes a contract without a card gets the invoice and the mail *The card payment failed*, with a button to add a card.
- Dispatches `AfterTrialChanged` (`tenant`, `account`, `continues`) when the choice changes. The trial bar, the trial mail and *Plan and usage* say which of "your contract starts and billing begins" or "the team can no longer be used" will happen. The operator list shows the choice (*After the trial: Becomes a contract / Stops*), and the operator can change it on the team's page.

### The trial mail

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('warp-saas:notify-trials')->hourly();   // one run a day is enough when all teams share one time zone
```

For each team whose number of days left is in `trialReminderDays()`, the owners get a mail that says how many days are left and which day is the last (in their own language when the user model implements `HasLocalePreference`) and the event `TrialEnding` is dispatched. `--dry-run` only lists who would get a mail. A team that has no owner is reported and skipped.

## Billing

Warp SaaS can send invoices to your customers and keep track of what they paid — **paid by bank transfer (invoice)**, without a payment provider. It is off until you turn it on; an app that does not call `billing()` works exactly as before. Card payments (Stripe) are in [their own section](#card-payments-stripe).

What it does for you, once a team starts its contract:

- Creates an invoice at the start of each month (or year) of the contract, for the price of the team's plan — with [a line for each paid feature](#lines-of-an-invoice) the team has on top of its plan.
- Emails the owners: the amount, the due date, how to pay, and a link to the invoice.
- Reminds them a few days before the due date, tells them when it is past due, and — if you want — ends the contract when it stays unpaid.
- Lets you record each payment in the operator panel.

```php
use Qalainau\FilamentWarpSaas\Billing\Billing;
use Qalainau\FilamentWarpSaas\FilamentWarpSaasPlugin;

FilamentWarpSaasPlugin::make()
    ->plans([
        Plan::make('lite')->label('Lite')->price(2980, 'JPY'),          // per month
        Plan::make('pro')->label('Pro')->price(29800, 'JPY', 'year'),   // per year
    ])
    ->billing(Billing::invoice()
        ->dueInDays(30)
        ->tax(rate: 10)
        ->paymentInstructions("Example Bank, Shibuya branch\nOrdinary 1234567  Warp Inc.")
        ->seller(name: 'Warp Inc.', address: '1-2-3 Shibuya, Tokyo', registrationNumber: 'T1234567890123'))
```

```php
// routes/console.php
Schedule::command('warp-saas:bill')->hourly();   // once a day is enough when all teams share one time zone
```

### Settings

`Billing::manual()` is the default: no invoices, nothing billing-related on the pages. `Billing::invoice()` turns invoices on and takes:

| Method | What it does |
| --- | --- |
| `dueInDays(int $days = 30)` | Days from the invoice date to the due date. |
| `dueUsing(?Closure $callback)` | `fn (CarbonImmutable $issuedOn, Model $tenant)` returns the due date instead — for example the end of next month: `$issuedOn->addMonthNoOverflow()->endOfMonth()`. |
| `tax(?float $rate = null, bool $included = false, string $rounding = 'floor')` | The tax in percent (`null` = no tax, the default). `included: true` when your prices already include tax: the invoice shows the tax inside the price. `rounding` is `floor`, `round` or `ceil`, for the tax amount. |
| `paymentInstructions(string\|Closure\|null $instructions)` | Where to pay. Shown on the team's page, in the mails and on the invoice. A function gets the tenant: `fn (Model $tenant): ?string`. |
| `remindDaysBeforeDue(array $days = [3])` | Days before the due date when the owners get a reminder. `[]` sends none. |
| `graceDays(int $days = 7)` | Days after the due date before `whenOverdue()` happens. |
| `whenOverdue(Overdue $overdue = Overdue::Notify)` | `Overdue::Notify` only dispatches `OverdueGraceEnded`. `Overdue::End` ends the contract (`end()`). `Overdue::ReadOnly` makes the team [look-only](#look-only-mode). See [unpaid invoices](#when-an-invoice-is-not-paid). |
| `issueInvoiceUsing(?Closure $callback)` | Make the invoice in your own system; see [below](#your-own-invoices). |
| `seller(string $name, ?string $address = null, ?string $registrationNumber = null)` | Your name, address and registration number for the plugin's simple invoice page. Without it, there is no invoice page (use `issueInvoiceUsing()` or send invoices yourself). |
| `mails(bool $enabled = true)` | `false` stops the three mails to the owners (invoice created, due soon, past due). The events still fire, so your app can send its own. |
| `keepChargesFor(?int $years = null)` | How long the invoices of a **deleted** team are kept. `null` keeps them (the default). See [after a team is deleted](#after-a-team-is-deleted). |
| `currency(?string $currency)` | The currency when a plan has none (`'JPY'`). The plan's currency comes first. |
| `featurePrices(array $prices)` | A price for each feature you give a team on top of its plan (`grantFeature()`): `['reports' => 19800, 'accounting' => ['amount' => 60000, 'per' => 'year']]`. Each becomes a line of the invoice; see [lines of an invoice](#lines-of-an-invoice). |
| `linesUsing(?Closure $callback)` | `fn (Model $tenant, CarbonImmutable $periodStart, CarbonImmutable $periodEnd, array $lines): array` returns the lines of the invoice instead — per seat, per use, a discount. It gets the plugin's lines and runs last. |
| `numberUsing(?Closure $callback)` | `fn (Charge $charge, int $sequence, int $yearSequence, CarbonImmutable $issuedOn): string` returns the invoice number (`WW-2026-0001`). Without it, the page shows `#000012`. See [invoice numbers](#invoice-numbers). |
| `additions(Additions $additions = Additions::Monthly)` | How a plan upgrade or a new paid feature in the middle of a period longer than a month is billed: `Monthly` (the default) month by month from the next month, `Prorated` the rest of the period at once, `NextCharge` from the next period invoice. Per team with `WarpSaas::setAdditions($team, …)`. See [changes in the middle of a period](#changes-in-the-middle-of-a-period). `prorateChanges(bool $enabled = true)` is a short way to write `Prorated` (`true`) or `NextCharge` (`false`). |
| `prepaidDiscounts(array $discounts)` | A discount for each number of months: `[12 => '10%', 24 => '15%', 36 => 50000]` — a percent (`'N%'`, above 0 and below 100) or an amount. See [prepaid discounts](#prepaid-discounts). |

On the tenant plugin, `overdueBanner(bool|Closure $enabled = true)` turns off (or decides page by page, like `trialBanner()`) the red bar that owners see while an invoice is past due.

Per team, in the operator panel or in code. **Order for a new contract:** set the price *before* `activate()` — the first invoice is made by `activate()`, so a price set afterwards only counts from the next invoice. `setPrice()` and the other setters create the team's account row when it does not exist yet (nothing else changes: the team keeps the state it had), so you can call them on a team that has never had a trial:

```php
WarpSaas::setPrice($team, 1980);               // 1. the price of this team (plan + modules)
WarpSaas::activate($team, 'pro');              // 2. the contract and its first invoice, at that price
```

All the setters:

```php
WarpSaas::setPrice($team, 1980);               // a price only for this team; null = the plan's price; 0 = free, no invoices
WarpSaas::setBillingMethod($team, ChargeMethod::Manual);   // no invoices for this team (it pays another way); null = the default
WarpSaas::setBillingPeriod($team, 24);         // bill 24 months at once, paid in advance; null = the plan's period
WarpSaas::setAdditions($team, Additions::Prorated);   // how additions in the middle of a long period are billed; null = the settings
WarpSaas::setPrepaidDiscount($team, '20%');    // its own prepaid discount: '20%', 30000, 0 for none; null = the settings
WarpSaas::changeTogether(fn () => [WarpSaas::grantFeature($team, 'exports'), WarpSaas::grantFeature($team, 'api')]);   // several changes billed as one
WarpSaas::startBilling($team);                 // a team that already had a contract before you turned billing on
WarpSaas::charge($team, 50000, 'Setup fee');   // one invoice by hand (due date from the settings, or dueOn: '2026-12-31')
WarpSaas::charge($team, lines: [               // one invoice by hand with several lines
    ['description' => 'Setup', 'amount' => 50000],
    ['description' => 'Training (per person)', 'amount' => 10000, 'quantity' => 3],
]);
WarpSaas::markPaid($charge, '2026-11-20', 'Transfer from ACME CO');
WarpSaas::voidCharge($charge, 'Sent twice');   // canceled; the same period can be invoiced again

WarpSaas::chargesFor($team)->latest('issued_on')->get();   // the invoices of one team — always read them through this
WarpSaas::nextChargeOn($team);                 // CarbonImmutable in the team's time zone, or null
WarpSaas::billingPeriodOf($team);              // 24 (months), or the plan's period: 1, or 12 for a yearly plan
WarpSaas::paidUntil($team);                    // the last day of the latest paid period (CarbonImmutable), or null
WarpSaas::additionsFor($team);                 // Additions::Monthly, Prorated or NextCharge
WarpSaas::hasOverdueCharges($team);            // an unpaid invoice past its due date
WarpSaas::upcomingLines($team);                // the lines the next invoice would have (list of ChargeLine), nothing saved
$charge->lines();                              // the lines of an invoice (list of ChargeLine)
```

Recording a payment or canceling an invoice a second time throws `ChargeAlreadySettled`.

### When invoices are made

- **The first invoice is made when the contract starts** — `activate()`, from your code or the operator panel's *Start the contract*. That day (in the team's time zone) is the **anchor**. Calling `activate()` again does not make a second one. Teams on trial get no invoices.
- **Paid in advance.** Each invoice covers one month (or year) from its first day: anchor 12 November → 12 November to 11 December, then 12 December to 11 January.
- **The end of the month does not drift.** Months are counted from the anchor: 31 January → 28 February → 31 March → 30 April.
- `warp-saas:bill` makes the invoice of a period on its first day. If the command did not run for a while, it makes **only the invoice of the current period**, not one for every period it missed. Running it twice on the same day makes nothing twice: the account row is locked, and a period has at most one invoice (a canceled one does not count).
- **Changes apply from the next invoice.** Changing the price, the billing period or the discount changes the *next* invoice. When a monthly plan becomes yearly, or the billing period changes, counting starts again from the next invoice day. A plan upgrade or a new paid feature in the middle of a longer period is billed from the next month; see [changes in the middle of a period](#changes-in-the-middle-of-a-period).
- A team whose price is `0`, whose method is `Manual`, or whose plan's price is per some other word (`'seat'`) gets no invoices. A plan without a price gets invoices only when the team has a [paid feature](#lines-of-an-invoice) or `linesUsing()` is set (they are monthly then), and an invoice whose lines add up to 0 or less is not made.
- **Ended teams get no invoices.** When an ended team is resumed (`resume()`, or `activate()` again), the time it was ended is not billed: if the current period is already invoiced nothing happens, otherwise today becomes the new anchor and one invoice is made.
- Dates (invoice date, due date, past due) are counted in the **team's time zone** (`timezoneUsing()`), like the trial.

Amounts are counted in the smallest unit of the currency (yen, cents), so there are no float errors. JPY has no decimals and USD two. With tax added, the invoice has the price, the tax and the total; with tax included, the total is the price and the tax is the part of it that is tax.

### Lines of an invoice

An invoice has one line for the plan. When you sell features on top of the plans — an add-on module, an integration — give them a price, and each team that has one gets **a line for it**, on the invoice, in the mails and on the *Plan and usage* page:

```php
Billing::invoice()
    ->featurePrices([
        'shift_attendance' => 19800,                              // the same period as the plan
        'accounting' => ['amount' => 60000, 'per' => 'year'],     // a yearly price
    ])

WarpSaas::grantFeature($team, 'shift_attendance');   // the next invoice: plan + Shift attendance
```

- Only features **you gave the team** (`grantFeature()`) and **not in its plan** become lines. A feature that the plan includes (or a plan with `allFeatures()`) costs nothing and gets no line. A price of `0` gets no line either.
- The price is in the currency of the team's price (the plan's currency, then `currency()`).
- A feature given in the middle of a period of one month is billed from the next invoice. In a longer period (a yearly plan, a prepaid period), it is billed from the next month; see [changes in the middle of a period](#changes-in-the-middle-of-a-period).

How a feature's price meets the plan's period:

| Feature | Plan | On each invoice |
| --- | --- | --- |
| a number, or `'per' => 'month'` on a monthly plan | per month | 1 × the price |
| a number on a yearly plan | per year | 1 × the price (a number means *the plan's period*) |
| `'per' => 'month'` | per year | 12 × the monthly price |
| `'per' => 'year'` | per year | 1 × the price |
| `'per' => 'year'` | per month | **once a year**: the whole price on the first invoice, then nothing for the next 11, then again |

With a [billing period](#prepaid-periods) of more than one month, the plan line and the monthly feature lines are multiplied by the months (24 × the monthly price) and a yearly price by the years (2 × the yearly price). On a period that is not whole years (3 or 6 months), a yearly feature is billed once a year, as on a monthly plan.

For the yearly feature on a monthly plan, the plugin keeps the date it is **paid until** for each feature (in the account's `meta`). Taking the feature away (`revokeFeature()`) and giving it back does not bill it again while it is paid; canceling the invoice that billed it gives the date back.

**Your own lines.** For prices the plugin cannot know — per seat, per use, a discount for one customer — return the lines yourself. You get the plugin's lines (plan + features) and what you return is exactly what is billed:

```php
Billing::invoice()->linesUsing(function (Model $tenant, CarbonImmutable $periodStart, CarbonImmutable $periodEnd, array $lines): array {
    $seats = $tenant->users()->count();

    return [
        ...$lines,
        ['description' => 'Seats', 'amount' => 500, 'quantity' => $seats],
        ['description' => 'Volume discount', 'amount' => -1000],
    ];
})
```

A line is `['description' => string, 'amount' => price of one, 'quantity' => int]` (quantity 1 when left out). A minus amount is a discount. Keep `type`, `key` and `per` on the lines you pass through, and the plugin keeps knowing them (the once-a-year feature uses them).

**In this order**, for each invoice of a period:

1. The plan line: the team's own price (`setPrice($team, 1980)`) replaces **only the plan line**; the feature lines are added as before.
2. The feature lines (`featurePrices()`).
3. `linesUsing()` gets lines 1 and 2 and has the last word.
4. Lines that add up to 0 or less make no invoice (the next invoice day still moves on).

`setPrice($team, 0)` is a **free team**: it gets no invoices at all — not for its features, and `linesUsing()` is not called.

The lines are saved on the invoice (`warp_saas_charges.lines`). Changing the prices later does not change invoices already made. Invoices made before lines existed show one line: what they were for and their amount.

**The language of the lines.** The plan and feature lines the plugin made are written when they are shown, in the language of the reader (the invoice page, the emails, *Plan and usage*): the plan line is "Standard plan" (the period is shown once, above the lines), a feature line is the feature's label, and a once-a-year feature on a monthly plan is "API access, 1 year (…)". A line whose `description` you changed in `linesUsing()`, your own lines, and lines whose plan or feature is no longer in your settings show the text that was saved. `ChargeLine::label()` gives the text to show; `description` is the saved text.

Good to know:

- **One tax rate for all lines.** The lines are added up, then the tax is added (or taken out, with `included: true`) once, as before. Lines with different tax rates are not supported.
- `linesUsing()` runs while the account row is locked, when the invoice is made, and on every view of the *Plan and usage* page (for the breakdown). Keep it quick. If it throws, the invoice is not made; `warp-saas:bill` reports it, goes on with the other teams and tries again the next time.
- `upcomingLines($team)` shows what the invoice of the current period would be with today's settings (`linesUsing()` is called for that period). It saves nothing.

### Prepaid periods

Some customers pay for one or two years at once (for example with a subsidy that wants the whole contract paid in advance). Give that team its own billing period:

```php
WarpSaas::setBillingPeriod($team, 24);     // one invoice every 24 months, paid in advance
WarpSaas::setBillingPeriod($team, null);   // back to the plan's period (monthly, or yearly for a yearly plan)
```

How additions are billed (`setAdditions()`) and the discount (`setPrepaidDiscount()`) are set on their own, and changing the period keeps them.

- From 1 to 120 months. A **yearly plan takes whole years only** (12, 24, 36 …); anything else throws `InvalidArgumentException`. If the plan later becomes yearly and the period is not whole years, the plan's period is used.
- One invoice has every line for the whole period: the plan line as *24 months × $29.00*, a monthly feature as *24 months × $10.00*, a yearly feature as *2 years × $120.00*. The invoice shows the period (first and last day), which is what a subsidy's paperwork asks for.
- **From the next invoice.** The current period stays as it is; the new period starts on the next invoice day, which becomes the new anchor. Changing the period is not prorated itself.
- `linesUsing()` gets `$periodStart` and `$periodEnd` of the whole period, so a discount for prepaying can be counted from them: `$months = (int) round($periodStart->diffInMonths($periodEnd->addDay()));`.
- `WarpSaas::paidUntil($team)` is the last day of the latest **paid** period invoice (invoices for changes and hand-made invoices do not count). The *Plan and usage* page shows the billing period (*Every 2 years*), the discount and *Paid until November 11, 2028*; the operator's team page shows the same. In the operator panel, *Set the price and how they pay* has the billing period (the plan's period, 1, 3, 6, 12, 24 or 36 months; 12, 24 and 36 for a yearly plan), how additions are billed, and the discount.
- Nothing is refunded when a team ends in the middle of a prepaid period.
- Run `php artisan migrate`: the migration `update_warp_saas_accounts_add_billing_months` adds `billing_months`, `billing_additions` and `billing_discount` to the account row.

### Prepaid discounts

```php
Billing::invoice()->prepaidDiscounts([12 => '10%', 24 => '15%', 36 => 50000])

WarpSaas::setPrepaidDiscount($team, '20%');   // this team only: '20%', 30000, or 0 for none; null = the settings
```

- A period invoice whose period has that many months gets one more line, *Prepaid discount (24 months, 15%)*, with a minus amount. A yearly plan's own period is 12 months, so `12 => '10%'` applies to yearly plans too.
- A percent is taken off the plan and feature lines (rounded half up to the smallest unit). An amount is in the currency of the team's price, and is never more than those lines (the total does not go below 0; an invoice of 0 is not made).
- The team's own discount (`setPrepaidDiscount()`) wins over the settings, for whatever period the team has. `0` or `'0%'` means no discount.
- The line is added **before** `linesUsing()`, which gets it with `'type' => 'discount'` and can change or drop it.
- On invoices for changes in the middle of a period: a percent is taken off a `Prorated` invoice too; an amount is not. `Monthly` invoices get no discount.

### Changes in the middle of a period

With a period longer than a month (a yearly plan, a prepaid period), a change billed only "from the next invoice" could be free for up to two years. So when a team **upgrades its plan** (`changePlan()`) or **gets a paid feature** (`grantFeature()`) in the middle of the period — from your code or the operator panel — the difference is billed in one of three ways (`->additions()` for the app, `WarpSaas::setAdditions($team, …)` per team):

| `Additions::` | What is billed | Example: 24 months from 12 November 2026, Reports ($10 a month) added on 20 January 2027 |
| --- | --- | --- |
| `Monthly` (default) | One invoice each month, from the next month to the end of the period, made by `warp-saas:bill` on the monthly day. | An invoice of $10.00 for 12 February to 11 March 2027, then one for 12 March to 11 April, … |
| `Prorated` | One invoice right away, for the rest of the period. | An invoice of 21 months × $10.00 = $210.00 for 12 February 2027 to 11 November 2028. |
| `NextCharge` | Nothing until the next period invoice. | Reports is on the invoice of 12 November 2028. |

- **The month of the change is free.** Months are counted from the anchor (12th to 12th here). A one-month period has nothing left, so monthly teams are not affected. Only the change itself is free for its month: with `Monthly`, what was added in earlier months is still billed that month, even when a new change lands on the monthly day before `warp-saas:bill` runs (the plugin keeps, for each slot, from which month each increase is billed, in the account's `meta`).
- **Only the difference.** For the plan and for each feature, the plugin compares the monthly price of today's settings with what this period has already billed (the period invoice and `Prorated` invoices; `Monthly` invoices count for their own month only). A plan upgrade from $29 to $49 is $20 a month. Downgrades and removed features are **not refunded**, and nothing is billed twice for the same month or change.
- Yearly prices count as 1/12 a month on a period of whole years. When that does not divide evenly, the amount is rounded (`Monthly`: $100 / 12 = $8.33 a month; `Prorated`: $100 × 9 / 12 = $75.00 at once). A yearly feature on a period that is not whole years is billed once a year at the start of a period instead.
- **`Monthly`** — schedule `warp-saas:bill` (it already makes the period invoices). It makes the invoice of the current month only: if it did not run for a month, that month is not billed later. It looks at teams with a long period whose method is `Monthly`.
- **Switching** a team to `Prorated` bills the rest of the period at once from the next month (the current month was already billed or is free). Switching to `Monthly` starts with the next month. From the next period invoice, everything is on that invoice as usual.
- **`linesUsing()` is not called** for these invoices, so per-seat lines or a discount from it are not applied (and a discount it gave on the period invoice is not billed back). For the prepaid discount, see [above](#prepaid-discounts).
- A free team (`setPrice($team, 0)`), a team on trial, an ended team and a team whose method is `Manual` get none of them. Canceling one (`voidCharge()`) lets it be billed again (`Prorated`: on the next change; `Monthly`: on the next run in that month).
- They are ordinary invoices: the number, the due date, `issueInvoiceUsing()`, the `ChargeCreated` event and the mail to the owners. They say *Added in the middle of the period, for the rest of it: …* (`Prorated`) or *… for one month: …* (`Monthly`), the invoice page says that the month of the change is not billed, and a line that comes from a yearly price says *1/12 of the yearly price a month*. They keep `['kind' => 'prorate']` or `['kind' => 'additions']` in their `meta`.
- A team's own price (`setPrice()`) counts as its plan line, but changing the price itself is applied from the next invoice.
- **Several changes at once.** Each `changePlan()` and `grantFeature()` is a change of its own, so with `Prorated` two features granted one after the other make two invoices. Wrap them to bill them as one (the operator's *Add or remove features* does this):

```php
WarpSaas::changeTogether(function () use ($team): void {
    WarpSaas::changePlan($team, 'pro');
    WarpSaas::grantFeature($team, 'exports');
});   // one invoice with both differences
```

### Invoice numbers

The plugin counts every invoice — all of them, and per year of the invoice date — in the table `warp_saas_charge_counters`. To print your own numbers, give a function:

```php
Billing::invoice()->numberUsing(fn (Charge $charge, int $sequence, int $yearSequence, CarbonImmutable $issuedOn): string =>
    'WW-'.$issuedOn->year.'-'.str_pad((string) $yearSequence, 4, '0', STR_PAD_LEFT))
// WW-2026-0001, WW-2026-0002, … and WW-2027-0001 on the first invoice of 2027
```

- The counters move up inside the transaction that makes the invoice, with the counter rows locked, so two invoices made at the same time never get the same number.
- They count even without `numberUsing()`, so setting it later goes on from where they are.
- A canceled invoice keeps its number; the number is not used again.
- The year is the year of the invoice date in the team's time zone.
- Numbers are unique (`warp_saas_charges.number` has a unique index). A number your `issueInvoiceUsing()` returns wins over `numberUsing()`.
- Without `numberUsing()`, nothing is saved and pages show the id (`#000012`), as before.

### What the team sees

On the *Plan and usage* page (owners only): how they pay, the price (with more than the plan line: each line of the next invoice, the once-a-year features, and the total per month, per year or for the billing period), the billing period and until when it is paid, the next invoice date, what is unpaid, where to pay, and the list of invoices with the period, the amount with tax, the due date, the status (*Unpaid*, *Paid*, *Canceled*, or red *Overdue*) and a link to the invoice. While an invoice is past due, owners see a red bar on every page. Members see neither. An ended team's owners still see their invoices on the ended page.

### Your own invoices

To make the invoice in your accounting system, give a function. It is called once for each new invoice, after it is saved and before the mail:

```php
Billing::invoice()->issueInvoiceUsing(function (Model $tenant, Charge $charge): array {
    $invoice = app(Accounting::class)->createInvoice($tenant, $charge->total, $charge->currency);

    return ['number' => $invoice->number, 'url' => $invoice->pdfUrl];
});
```

The number and URL are saved on the invoice and the URL becomes the link on the team's page and in the mail (only `http` and `https` links are shown). **If your function throws, the invoice is kept** and the exception is reported — you can still record its payment.

Without your own invoices, `seller()` gives a simple invoice page (seller, the team, number, date, due date, what for, amount, tax rate, tax, total, where to pay) in the team's language. The browser's print dialog saves it as a PDF. The routes of this page are only added to an app that sets `seller()` in the panel provider, so an app that makes its own invoices gets no extra routes.

### When an invoice is not paid

1. The day after the due date, `warp-saas:bill` dispatches `ChargeOverdue` once and emails the owners. **Your app can decide what to do from here** — for example make the team read-only in your own policies:

   ```php
   Event::listen(ChargeOverdue::class, function (ChargeOverdue $event): void {
       // $event->tenant, $event->charge
   });
   ```

2. When `graceDays()` more days have passed: with `Overdue::Notify` (the default) only `OverdueGraceEnded` is dispatched — a good place to tell yourself. With `Overdue::End`, the contract is ended (`end()`), people see the ended page, and `OverdueGraceEnded` is dispatched. With `Overdue::ReadOnly`, the team becomes [look-only](#look-only-mode) (the reason is *overdue*) and `OverdueGraceEnded` is dispatched.
3. When you record the payment and the team has no other invoice past due, a team that was ended **because of the unpaid invoice** is resumed automatically. A team you ended yourself is not, and neither is one you made look-only by hand.

> **A team that is ended can still open its invoices only if your app lets it into the panel.** The invoice pages of an ended team (`ended-invoice`, `account-ended`) are found through your panel's `getTenants()`. If you take ended teams out of `getTenants()`, those two pages answer 404 to everyone, so after `Overdue::End` the people of that team cannot open an invoice. Either keep ended teams in `getTenants()` (they only see the ended page), or rely on the link and the bank details in the invoice mail.

### After a team is deleted

Invoices are your records, not the team's data. Deleting a team (`warp-saas:delete`, `warp-saas:purge`) **keeps its invoices**, with the team's name copied on them, and they stay in the operator panel. A team's export zip includes a copy of its own invoices. With `keepChargesFor(years: 7)`, `warp-saas:prune-charges` deletes the invoices of deleted teams once they are older than that — invoices of teams that still exist are never deleted.

### Good to know

- The invoice table is `warp_saas_charges`, added by the migration `update_warp_saas_add_billing` (run `php artisan migrate`). It also adds `price`, `billing_method`, `next_charge_on` and `billing_anchor_on` to the account row. The migration `update_warp_saas_charges_add_lines` adds the lines, a unique index on `number` and the counter table `warp_saas_charge_counters`. If your `issueInvoiceUsing()` has ever returned the same number twice, fix those rows before you run it. The migration `update_warp_saas_accounts_add_billing_months` adds `billing_months` (the [prepaid period](#prepaid-periods)) to the account row.
- The migration `update_warp_saas_accounts_add_read_only` adds `read_only_at` and `read_only_reason` to the account row ([look-only mode](#look-only-mode); the operator's note is kept with the account's other data). `update_warp_saas_accounts_add_after_trial` adds `activates_after_trial` (`false` for every existing team). `update_warp_saas_add_cards` adds the Stripe customer and the card's brand, last four digits and expiry to the account row, and the payment reference, number of attempts and failure to the invoice ([card payments](#card-payments-stripe)). Run `php artisan migrate`.
- The price of a team is in the currency of its plan.
- In the operator list, *Overdue* counts with the app's time zone; on each team's page it uses the team's.

## Card payments (Stripe)

With `Billing::card()` the team's owners save a card once, and each invoice is charged to it when it is made. The invoice itself (the amount, tax, lines and period) is still made by Warp SaaS exactly as in [Billing](#billing); **Stripe only takes the money**. Warp SaaS does not use Laravel Cashier or Stripe Subscriptions, and the card number never reaches your app: the owner types it on Stripe's own page (Stripe Checkout).

```bash
composer require stripe/stripe-php
```

```php
use Qalainau\FilamentWarpSaas\Billing\Billing;

FilamentWarpSaasPlugin::make()
    ->billing(Billing::card()
        ->stripe()                       // reads services.stripe.secret and services.stripe.webhook_secret
        ->retryCardAfterDays([3, 5])     // the default
        ->tax(rate: 10)
        ->seller(name: 'Warp Inc.'));
```

```php
// config/services.php
'stripe' => [
    'secret' => env('STRIPE_SECRET'),
    'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
],
```

`stripe/stripe-php` is only a `suggest` of the plugin (`^19` to `^23`, the versions it was tested with). Without it (or without a secret), the first use of a card throws an exception that says what is missing.

**The Stripe API version is pinned.** Every request is sent with the API version `2026-02-25.clover` (`Billing::STRIPE_API_VERSION`), whichever version of `stripe/stripe-php` you installed; without the pin, each stripe-php release talks a newer API version, and Stripe removes parameters between versions (a Checkout Session created with the API version of stripe-php 23 no longer accepts `payment_method_types`). To use another version, pass it: `->stripe(apiVersion: '2026-02-25.clover')`. Create the webhook endpoint with **the same API version** (*Developers → Webhooks → your endpoint → API version*; for local tests, `stripe listen` sends the events in your account's default API version, so set that to the pinned version in the Stripe dashboard (*Developers → Workbench → Overview*) or create the endpoint through the API with `api_version`), because the shape of the events Stripe sends is decided by the endpoint's version, not by this plugin's requests.

### Settings

Everything in the settings of [Billing](#billing) (`dueInDays()`, `tax()`, `seller()`, `linesUsing()` …) works here too.

| Method | What it does |
| --- | --- |
| `Billing::card()` | Card first, then invoice and manual: a team can be moved to either with `WarpSaas::setBillingMethod($team, ChargeMethod::Invoice)`. The default method is the first one: `Card`. |
| `Billing::methods(ChargeMethod ...$methods)` | Choose the methods yourself; the first is the default. `Billing::methods(ChargeMethod::Invoice, ChargeMethod::Card)` bills by invoice, and a team that saves a card is moved to card payments. |
| `stripe(?string $secret = null, ?string $webhookSecret = null, ?string $apiVersion = null)` | Your Stripe keys and the Stripe API version. A key you leave out is read from `config('services.stripe.secret')` / `services.stripe.webhook_secret`; a version you leave out is `Billing::STRIPE_API_VERSION` (`2026-02-25.clover`). |
| `customerPortal(bool $enabled = true)` | Let owners manage their company’s saved card on Stripe’s customer portal. Off by default; requires Card in the payment methods. |
| `activateOnCard(bool $enabled = true)` | Start a trial company’s contract when its owner saves a card and chooses a plan. Off by default; requires Card in the payment methods. |
| `retryCardAfterDays(array $days = [3, 5])` | Days after the first failed attempt when the card is charged again. `[]` never charges again. Positive whole numbers only. |

### The customer portal

Use `Billing::card()->stripe()->customerPortal()` to show *Manage the card on Stripe* on *Plan and usage* once the company has a Stripe customer. Only owners can open it, and it always uses their current company’s customer from the database.

Enable **Customer portal** in the Stripe dashboard and allow `payment_method_update`. Add `customer.updated` and `payment_method.detached` to the webhook events: setting a new default card updates the saved card details, and removing a saved card clears them. These events keep the contract, billing method and existing invoices unchanged.

**Allow cards only in the portal.** The portal offers the payment methods turned on for your Stripe account, so it can show wallets such as Kakao Pay or Naver Pay next to cards, sometimes even selected first. Warp SaaS charges cards only: a company that picks another method thinks it changed how it pays, while the saved card stays the same. In the Stripe dashboard, open *Settings → Payment methods*, choose the configuration used by the customer portal, and leave only *Cards* on.

### The webhook

Add this endpoint in the Stripe dashboard (*Developers → Webhooks*), with the signing secret in `services.stripe.webhook_secret`:

```
POST https://your-app.example/warp-saas/stripe/webhook
```

Events to send: `checkout.session.completed`, `payment_intent.succeeded`, `payment_intent.payment_failed`, `payment_method.detached`, `customer.updated` and `customer.deleted`. The route is called `warp-saas.stripe.webhook`. It is outside the `web` middleware group (no session, no CSRF token), and is protected by the signature instead; a request with a wrong signature gets a 400 and changes nothing. In an app whose billing methods do not include `Card`, the route answers 404 without loading Stripe, so a stranger posting to it cannot fill your error log.

Each event ID is processed once, using the plugin's `warp_saas_stripe_events` table. `warp-saas:bill` removes records older than 90 days. If processing fails, the event record is removed and the route returns 500 so Stripe can retry.

**On SQLite**, Stripe often sends two events at the same moment (for example `payment_method.attached` and `customer.updated` when a card is replaced), and SQLite lets only one request write at a time. The second one can fail with `database is locked` and get a 500. Nothing is lost: Stripe sends it again later, and it is processed then. On MySQL, MariaDB and PostgreSQL this does not happen. With `stripe listen` on your machine, a failed event is not sent again by itself; resend it with `stripe events resend <event id>`.

When a saved payment method is detached, its matching account's card details are cleared. When a customer is deleted, its matching account's customer ID and card details are cleared. The contract continues with Card as its billing method, and existing invoices stay unchanged. *Plan and usage* then shows *The card was removed on Stripe. Add a card before the next invoice.* instead of *No card yet*, until a card is saved again. The next card charge fails with `no_card` and sends the owner an email with a *Pay by card* button to save a new card and pay.

### Start a contract with a card

Use `Billing::card()->stripe()->activateOnCard()` to let an owner start a contract from *Plan and usage*. Only companies currently on trial can use this action. The owner chooses a configured plan and saves a card on Stripe Checkout in setup mode. Once the card is saved, Warp SaaS activates that plan and queues the first invoice for charging to the saved card. A plan whose limits are below the current usage cannot be started; usage and the plan are checked again when Checkout completes. If that check fails, the card stays saved and the owner sees the reason. The Checkout return and webhook can arrive in either order: the same session starts the contract once.

### How a payment goes

1. **Saving the card.** An owner presses *Add a card* on *Plan and usage*. Stripe's page takes the card (a Checkout in `setup` mode for that team's own Stripe customer). When the owner comes back, Warp SaaS reads the session from Stripe and stores the brand, the last four digits and the expiry (*Visa ending in 4242 (expires 2028-04)*). The webhook does the same, and it does not matter which arrives first. A team whose method is not *Card* is moved to it (when `Card` is one of the methods), and its unpaid card invoices are charged at once.
2. **An invoice is made** (`activate()`, `warp-saas:bill`, `WarpSaas::charge()`): a queued job (`ChargeCardJob`) charges the saved card, without the customer present (an off-session PaymentIntent). The PaymentIntent's description is the invoice number (*INV-2026-0003*), so you can find the invoice from the payment in the Stripe dashboard. When it succeeds the invoice is marked paid (*Card Visa ····4242*, with the PaymentIntent id as the payment reference) and `ChargePaid` is dispatched.
3. **It failed** (declined, expired card, the bank wants the customer to confirm, no card saved): the failure is stored on the invoice, the owners get a mail with the reason and a *Pay by card* button, and `CardPaymentFailed` (`tenant`, `charge`, `failure`) is dispatched. The reason comes from Stripe's `decline_code`, then its `code`: `declined`, `insufficient_funds`, `expired`, `incorrect_cvc`, `incorrect_number`, `processing_error`, `authentication_required`, and `no_card` (no card saved). Anything Stripe sends that is not in this list is `other` (*the payment could not be completed*). The reasons are translated (`billing.card.failure.*`).
4. **Paying by hand.** *Pay by card* opens Stripe Checkout for that invoice's amount and currency, and also saves the card for next time. The return and the webhook mark it paid.
5. **Trying again.** `warp-saas:bill` charges again on the days of `retryCardAfterDays()`, counted from the first failure. The operator can press *Charge the card again* on the invoice. After the grace days `whenOverdue()` applies, as with any unpaid invoice (see [When an invoice is not paid](#when-an-invoice-is-not-paid)).

**You need a queue worker** (`php artisan queue:work`), and `warp-saas:bill` on the scheduler: the charge is a queued job, and the retries are made by the command. With `QUEUE_CONNECTION=sync` it is charged inside the request that made the invoice.

### What is checked

Before an invoice is marked paid (from the return page or the webhook), Warp SaaS checks that the signature is right, that the Stripe customer and the metadata belong to **that** team, that the invoice is still unpaid, and that the amount (in Stripe's smallest unit) and the currency are the invoice's. If anything does not match, nothing changes and the mismatch is written to the log as a warning (`[warp-saas] Ignored a card payment: …`). The same event can arrive any number of times; the invoice is paid once. Until the queue worker has made the first attempt on an invoice made for card payment, *Plan and usage* and the operator's invoice list show **Checking the payment** instead of *Unpaid*. A team cannot pay, or finish a card session for, another team's invoice: *Pay by card* on someone else's invoice answers 404 (it is looked up among the current team's unpaid invoices), and a session id of another team is not accepted. Only the owners of a team can add a card or pay.

### Refunds and chargebacks

Not handled by the plugin. Refund, and answer disputes, in the Stripe dashboard; if you want the invoice to show it, use `WarpSaas::voidCharge()` yourself. Disputes are not sent to your app by Warp SaaS.

### Deleting a team

When a team is deleted (`warp-saas:delete`, `warp-saas:purge`), its Stripe customer, and with it the saved card, is deleted too. If Stripe cannot be reached the team is still deleted, and the error is reported. The resulting `customer.deleted` webhook does nothing because the account row is already gone. Event records belong to the plugin and remain after a team is deleted.

### Testing and demos: `WarpSaas::fakeCards()`

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

$cards = WarpSaas::fakeCards();   // in a test, or in a service provider of a demo
```

It replaces Stripe with a copy inside the plugin: no network, no keys, and *Add a card* and *Pay by card* open a plain *Test card page* of the plugin instead of Stripe's. That page is available only while `fakeCards()` is on (otherwise 404). It takes four card numbers:

| Number | Result |
| --- | --- |
| `4242 4242 4242 4242` | The card is saved and every payment succeeds. |
| `4000 0000 0000 0002` | The card is declined when it is saved (*the card was declined*). |
| `4000 0000 0000 0341` | The card is saved, and every payment is declined: use it to see a failed charge, the mail and *Pay by card*. |
| `4000 0025 0000 3155` | The card is saved, but automatic payments fail because the card company asks the customer to confirm (*authentication required*). Paying on the test card page works. |

In a test, `$cards->failNextCharge('insufficient_funds')` makes the next charge fail, `$cards->charges()` lists the charges made, and `$cards->deletedCustomers()` the customers deleted. Do not turn it on in production: it never charges anyone.

## Look-only mode

A team in look-only mode can **see everything and export its data, but cannot change anything**. Use it for a team that did not pay (instead of ending it and hiding the data), for a team you stopped yourself, and for a team whose contract ended while its data is still kept.

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

WarpSaas::makeReadOnly($team, 'Waiting for the bank transfer');   // the note is shown to the team
WarpSaas::makeWritable($team);

WarpSaas::isReadOnly($team);   // true / false. Without an argument: the current team
```

| Call | What it does |
| --- | --- |
| `WarpSaas::makeReadOnly(Model $tenant, ?string $note = null)` | Starts look-only mode with the reason *manual*. It stays until `makeWritable()` (or the operator button). Event `TenantMadeReadOnly`. |
| `WarpSaas::makeWritable(Model $tenant)` | Ends it, whatever the reason. Event `TenantMadeWritable`. |
| `WarpSaas::isReadOnly(?Model $tenant = null)` | `true` for a team in look-only mode, including an ended team when `readOnlyAfterEnd()` is on. `null` is the current team. |
| `WarpSaas::whileWritable(Closure $callback)` | Runs the callback with the guard off, for your app writing as the system (a job that writes a summary, a migration of data). Can be nested; the guard comes back even if the callback throws. |

### The three reasons

| Reason | Started by | Ends when |
| --- | --- | --- |
| `manual` | `makeReadOnly()` or the operator's *Make look-only* | `makeWritable()` or the operator's *Make writable again*. |
| `overdue` | `->whenOverdue(Overdue::ReadOnly)`: an invoice is still unpaid after the due date **and** `graceDays()` | The invoice is paid or voided and no other invoice is past due: the team becomes writable by itself. A team you made look-only by hand is not touched. |
| `ended` | `->readOnlyAfterEnd()` | The contract ends and the retention period (`dataRetention()`) is over. |

```php
use Qalainau\FilamentWarpSaas\Enums\Overdue;

FilamentWarpSaasPlugin::make()
    ->billing(Billing::invoice()->graceDays(7)->whenOverdue(Overdue::ReadOnly))
    ->readOnlyAfterEnd()                       // ended teams: see and export until the data is deleted
    ->readOnlyBanner(true)                     // the bar above the pages (the default)
    ->readOnlyAbilities(['create', 'update', 'delete']);
```

| Setting | What it does |
| --- | --- |
| `Overdue::ReadOnly` | A value of `whenOverdue()`, next to `Notify` and `End`. `warp-saas:bill` makes the team look-only when the grace days are over. |
| `readOnlyAfterEnd(bool $enabled = true)` | An ended team can still open the app until the day its data is deleted, in look-only mode (and export its data). The default is the *ended* page of [Ended teams](#ended-teams). |
| `readOnlyBanner(bool\|Closure $enabled = true)` | The bar above the pages. A function decides page by page: `fn (Panel $panel, Model $tenant, ?Authenticatable $user): bool` (by name). It says why: *an invoice is past its due date* (with a link to the invoice for owners), your note, or *the data is kept until …*. |
| `readOnlyAbilities(array $abilities)` | The policy abilities that are refused. The default is `create`, `update`, `delete`, `deleteAny`, `forceDelete`, `forceDeleteAny`, `restore`, `restoreAny`, `replicate`, `reorder`, `attach`, `attachAny`, `detach`, `detachAny`, `associate`, `associateAny`, `dissociate`, `dissociateAny`. Add the names of your own abilities (`approve`); remove one to let it through. |

### What stops a change

Three places, so that a hidden button is not the only protection:

1. **`Gate::before`.** For a write ability above, on a team in look-only mode, the answer is *no*. Filament asks the gate even for a resource that has **no policy** (the check in `get_authorization_response()` runs the `before` callbacks), so the create, edit and delete buttons disappear and the URLs answer 403 for every resource, with no change in your code. It applies to the current team only: another team may be writable at the same time.
2. **Eloquent.** Saving or deleting a model that uses `BelongsToTenant` or `BelongsToTenantThrough` throws `Qalainau\FilamentWarpSaas\Exceptions\TenantReadOnly` (an `HttpException` with status 403, and a translated message) while the current team is look-only. This also stops queued jobs, commands in `WarpSaas::actingAs()`, APIs and Livewire methods that skip the buttons.
3. **The plugin's pages.** *Members* can be opened but not changed (no invitations, role changes or removing people). *Export data* and *Plan and usage* (pay an invoice, add a card) keep working.

**What it does not stop:**

- Writes that skip the models: `DB::table()`, a query builder's `->update()`, `->delete()` and `->insert()`, and `Model::query()->update([...])`. Write through the models (or check `WarpSaas::isReadOnly()` yourself).
- The plugin's own models (`Account`, `Charge`, `Export`, `Usage`, `Invitation`): billing, exports and quotas keep working.
- The operator panel: operators can always change data.
- Models that are not tenant data (a shared table, the user model).
- Rows written inside `WarpSaas::whileWritable()`.

Operators have the buttons *Make look-only* (with a note) and *Make writable again* on each team's row and page, and a *Look-only* badge in the list of states.

### Events

`TenantMadeReadOnly` (`tenant`, `account`, `reason`) and `TenantMadeWritable` (`tenant`, `account`) are dispatched when a team goes in and out of look-only mode. (An ended team shown as look-only by `readOnlyAfterEnd()` is not a change of state: it dispatches nothing.)

## Members and invitations

**Roles** are a `BackedEnum`. Without one, Warp SaaS uses `owner` and `member`. Give your own and say which are owners:

```php
use Filament\Support\Contracts\HasLabel;

enum TeamRole: string implements HasLabel
{
    case Owner = 'owner';
    case Manager = 'manager';
    case Member = 'member';

    public function getLabel(): string
    {
        return ucfirst($this->value);
    }
}
```

```php
FilamentWarpSaasPlugin::make()
    ->members(relationship: 'users', roleColumn: 'role', roles: TeamRole::class, owners: [TeamRole::Owner])
```

| Argument of `members()` | What it is |
| --- | --- |
| `relationship` | The `BelongsToMany` on the tenant model that lists its members (default `users`). Warp SaaS **stores the role in the pivot** and throws a clear error if the method is missing or is not a `BelongsToMany`. |
| `roleColumn` | The pivot column that holds the role (default `role`). |
| `roles` | The enum of roles (default `DefaultRole`: owner, member). The label comes from `HasLabel`. |
| `owners` | The roles that count as owner (default `owner`). Owners are the people who can open the three pages, are asked about the trial, and cannot all be removed. |

The meaning of a role — which role may do what in your own pages — stays in your policies. Warp SaaS gives you `WarpSaas::isOwner($user)` and `WarpSaas::roleOf($user)` to write them.

### The *Members* page

For owners, in the tenant panel: a table of members (change role, remove) and a table of pending invitations (send again, cancel), and an *Invite a member* button (email address and role). The buttons of an owner who is the last owner are disabled with the reason.

The rules, enforced in the code and not only by the buttons:

- **The last owner cannot be removed or given a lower role** (`MemberChangeBlocked`).
- **Only an owner can give or take away an owner role.** Inviting someone as an owner is also refused for a non-owner.
- **The limit is checked when the invitation is sent.** A pending invitation takes one place of the `members` limit (that is how `members` is counted); an invitation that has expired no longer does. An email that is already a member or already has a pending invitation is refused (case does not matter).
- Resending is possible once a minute, makes a new link and stops the old one.

### Restricting the team pages per role

The *Members*, *Export data* and *Plan and usage* pages are for the team's owners. When your app has its own roles on top of that ("can't use this screen", "read only"), decide per page:

```php
use Qalainau\FilamentWarpSaas\Pages\Members;

FilamentWarpSaasPlugin::make()
    ->canAccessUsing(fn (User $user, string $page): bool => $user->canOpenScreen($page))
    ->canWriteUsing(fn (User $user, string $page): bool => ! is_a($page, Members::class, true) || ! $user->isReadOnlyOn($page))
```

`$page` is the class of the page: `Pages\Members`, `Pages\DataExport` or `Pages\PlanAndUsage` (in `Qalainau\FilamentWarpSaas`), or **your own class** when you replaced one with `->membersPage()`, `->exportPage()` or `->planPage()`. Compare with `is_a($page, Members::class, true)` so both work. The *Pending invitations* table under *Members* asks with the same class as the page. Both closures receive `$user` and `$page` by name.

- `canAccessUsing()` returning `false` hides the page from the navigation, gives a 403 at its URL and refuses Livewire calls from a page that was already open. Closing *Members* also hides its table of pending invitations.
- `canWriteUsing()` returning `false` keeps *Members* open to look at, without *Invite*, *Change role*, *Remove*, *Resend* or *Revoke* (hidden, and refused if called). Starting and downloading an export are not writes.
- These rules only narrow: someone who isn't an owner still can't open the pages. With a rule set, people who aren't signed in get `false`. Without these rules nothing changes.

### The invitation flow

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

$invitation = WarpSaas::invite($team, 'ana@example.com', TeamRole::Member, by: auth()->user());
```

The link in the mail opens `/app/invitations/{token}` (the panel's path). It is a page of the tenant panel outside any team, and signing in is not needed to open it. **Opening it does not accept anything** — a mail scanner that opens every link cannot join your customers to a team. A person must press a button, and what the page offers depends on who is looking:

| Who opens the link | What the page shows |
| --- | --- |
| Signed in as the invited email address | A *Join Acme* button. It adds the person to the team and opens the team. |
| Signed in as **another** email address | “This invitation is for ana@… You are signed in as bob@… now.” with a *Sign out* button. Nothing else. |
| Not signed in, and an account with that email exists | A *Sign in to join* button; after signing in they come back to this page. |
| Not signed in, and no account exists | A small form for a name and a password and a *Create my account and join Acme* button. It creates the account (the email is marked as verified, because the link came by mail), makes the person a member, signs them in and opens the team. With `allowRegistrationFromInvitations(false)` the page asks them to create an account first and open the link again. |

An expired, cancelled, already accepted invitation, or one for a team that has ended, shows “This invitation can no longer be used”.

How the token is handled: 64 random characters; the database keeps only its **SHA-256 hash** (and the hash column is never put in an export); the link is looked up by the hash and compared with `hash_equals`; the page has a rate limit of 30 requests a minute; accepting is a button press (a POST), and the person's email address must equal the invitation's. If the person already is a member, their role is not changed by the invitation.

### Settings and calls

| Setting | What it does |
| --- | --- |
| `invitationsExpireAfter(?int $days = 7)` | How long a link works. Sending again restarts it. `null`: invitations never expire — they work until accepted or revoked and count toward the `members` limit all that time. The *Members* page then says “No expiry”, and the plugin's mail has no date line. |
| `sendInvitationsUsing(?Closure $callback)` | Send your own invitation mail instead of the plugin's. Called on `invite()` and on `resendInvitation()`; the plugin's mail is not sent. Arguments by name, ask only for what you need: `$invitation`, `$tenant`, `$url` (the plugin's accept page with the token — the only place the token appears; `null` when no panel has the plugin), `$inviter` (`null` from code), `$resent`. Runs inside `actingAs($tenant)`. |
| `invitationMailUsing(?Closure $callback)` | Add to the plugin's invitation mail instead of replacing it: `fn (MailMessage $mail, Invitation $invitation, Model $tenant)` (by name). Add lines to `$mail`, or return another `MailMessage`. Called before the last line (“If you did not expect…”), in the language of the mail, so `__()` is translated for the person invited. Example: tell each role where to sign in — `fn (MailMessage $mail, Invitation $invitation) => $invitation->role === 'technician' ? $mail->line(__('Technicians sign in at :url', ['url' => url('/field')])) : $mail`. Not called when `sendInvitationsUsing()` is set. |
| `invitedMemberPivotUsing(?Closure $callback)` | Extra pivot columns when an invitation is accepted: `fn (Invitation $invitation, Model $tenant, Authenticatable $user): array` (by name). Use it to copy app fields from `$invitation->meta` — e.g. `['department_id' => $invitation->meta('department_id')]`. The role column always comes from the invitation, and the pivot of someone who already was a member is left alone. |
| `redirectAfterInvitationUsing(?Closure $callback)` | Where a person goes after joining from the invitation page: `fn (Model $tenant, Authenticatable $user, Invitation $invitation): ?string` (by name). Return a URL, or `null` for the default — the team in the panel of the invitation page. Use it when roles use different panels: the invitation page lives in one panel, and a person whose role cannot open that panel would get a 403 or 404 right after joining. Example: `fn (Company $tenant, User $user) => $user->isTechnicianOf($tenant) ? Filament::getPanel('field')->getUrl($tenant) : null`. It is called after the person has become a member, and not when the invitation is refused. |
| `createUsersUsing(?Closure $callback)` | `fn (array $data): Authenticatable` — create the user for a person who has no account. `$data` has `name`, `email` and the **plain** `password`. Default: your auth user model with those three (password hashed, `email_verified_at` set when the column exists). Use it if your users need more fields or a different model. |
| `allowRegistrationFromInvitations(bool $allowed = true)` | With `false`, a person without an account is told to create one first and come back. |

| Call | What it does |
| --- | --- |
| `WarpSaas::invite($tenant, $email, $role = null, ?Authenticatable $by = null, array $meta = [])` | Sends the mail. No role = the first role that is not an owner. Throws `InvitationRejected` (invalid email, unknown role, owner role by a non-owner, already a member, already invited) or `LimitReached`. Event `InvitationSent`. `$by = null` is “from code” and skips the owner check. `$meta` keeps your app's fields (a department, a team inside the team) on the invitation until it is accepted — see `invitedMemberPivotUsing()`. |
| `WarpSaas::resendInvitation($invitation, ?Authenticatable $by = null)` | New link, new expiry. Once a minute. |
| `WarpSaas::revokeInvitation($invitation)` | Deletes the invitation; the link stops working. An accepted one cannot be revoked. |
| `WarpSaas::acceptInvitation($invitation, $user)` | Accepts for a user whose email matches. For your own accept screen. Event `InvitationAccepted`. |
| `WarpSaas::pendingInvitation($tenant, $email): ?Invitation` | The pending invitation of that team for that address (case ignored), without a token. For your own accept path — a signed link of yours, or the moment a person sets their password. |
| `WarpSaas::pendingInvitationsFor($email): Collection` | The pending invitations of that address in every team, oldest first. |
| `WarpSaas::removeMember($tenant, $member, ?Authenticatable $by = null)` | Removes a member. `MemberChangeBlocked` for the last owner. Event `MemberRemoved`. |
| `WarpSaas::changeRole($tenant, $member, $role, ?Authenticatable $by = null)` | Event `MemberRoleChanged` (with the previous role). |
| `WarpSaas::pruneInvitations(int $days = 30): int` | Deletes invitations that expired more than `$days` ago and were never accepted. Returns the count. |

Accepted invitations are kept as a record. Pending ones for which the link has not been used are removed by `warp-saas:prune-invitations --days=30`:

```php
Schedule::command('warp-saas:prune-invitations')->daily();
```

The invitation page uses the route name `warp-saas.invitation` of the tenant panel: `AcceptInvitation::getUrlFor($token, $panel)` builds its URL.

### Your own invitation mail and accept path

An app that already has its own invitation mail and its own link keeps them, and lets Warp SaaS keep the invitations, the limit and the membership:

```php
use Illuminate\Support\Facades\URL;
use Qalainau\FilamentWarpSaas\Models\Invitation;

FilamentWarpSaasPlugin::make()
    ->invitationsExpireAfter(null)
    ->sendInvitationsUsing(fn (Invitation $invitation, Team $tenant) => Notification::route('mail', $invitation->email)
        ->notify(new JoinTeam($tenant, URL::temporarySignedRoute('join', now()->addDays(7), ['team' => $tenant, 'email' => $invitation->email]))))
    ->invitedMemberPivotUsing(fn (Invitation $invitation): array => ['department_id' => $invitation->meta('department_id')]);

WarpSaas::invite($team, 'ana@example.com', TeamRole::Member, by: auth()->user(), meta: ['department_id' => 3]);

// Your route, behind the `signed` middleware and a signed-in user
Route::post('/join/{team}', function (Team $team, Request $request) {
    $invitation = WarpSaas::pendingInvitation($team, $request->user()->email) ?? abort(404);
    WarpSaas::acceptInvitation($invitation, $request->user());
});
```

**Who may accept is now your app's job.** `pendingInvitation()` takes no token: it only finds the invitation. Make sure the person really owns the address (a signed link sent to it, or a password they set from a mail sent to it) and that the link is yours before you call `acceptInvitation()`, which still checks that the user's email is the invitation's, that the invitation is pending and that the team has not ended (`InvitationRejected` with `unavailable` or `email_mismatch`). Change an app field of a waiting invitation with `$invitation->mergeMeta(['department_id' => 5])` (`null` removes a key); read one with `$invitation->meta('department_id')`.

## Data export and deletion

A team that leaves is owed its data. A team that asks to be forgotten is owed its deletion. Both work from the [tenant data map](#tenant-data-map): the tables are the ones the map knows as tenant data, nothing is hard-coded.

### The zip

```
acme-export-2026-11-12.zip
├── README.txt          what is inside, how the tables relate, in the team's language
├── tenant.csv          the row of the team
├── members.csv         id, name, email, role
├── tables/
│   ├── projects.csv    every tenant-data table that has rows
│   ├── tasks.csv
│   └── …
└── files/
    └── public/…        the files the rows point to (files()) and the team's own files (tenantFiles())
```

- CSV is UTF-8 with a BOM, so Excel opens it correctly. It is written row by row to a temporary file, so a table of millions of rows does not fill the memory. Arrays and objects become JSON text, booleans 1 and 0.
- **Cells that start with `=`, `+`, `-`, `@`, a tab or a line break get a `'` in front**, so a spreadsheet does not run them as a formula (CSV injection). Plain numbers such as `-12.5` are left as they are.
- **Secret columns are left out**: `password`, `remember_token`, `token`, `*_token`, `*_secret`, `token_hash`, `two_factor_*` and `app_authentication_*`. Add your own with `secretColumns()`.
- A file that is missing on the disk is counted and mentioned in `README.txt`; it does not stop the export.
- The language of `README.txt` is the team's (if the tenant model implements `HasLocalePreference`), then the person who asked, then the app's.

### The *Export data* page

For owners, in the tenant panel: *Start an export*, a table of the exports of the team (state, size, who asked, until when it can be downloaded) and a *Download* button. The zip is built by a queued job (`ExportTenantJob`); when it is ready the person who asked gets a notification in the panel (if the panel has `->databaseNotifications()`) and a mail. The table refreshes itself while an export is being made.

- Download goes through a **signed URL that is valid for 10 minutes** and is made at the moment the button is pressed. The route also requires a signed-in **owner of this team**, and it looks the export up *inside* the current team, so another team's export id gives a 404.
- Only one export runs at a time per team, and `exportsPerDay()` (3) caps how many a team can ask for in its own calendar day. Failed exports are not counted.
- Owners of an **ended** team can still export while the data is kept: the button is on the ended page.
- The file name is `<slug>-export-<date>.zip`.

### Settings

| Setting | What it does |
| --- | --- |
| `exportDisk(string $disk)` | The disk the zips are stored on. Default `local` (a private disk). **Never give a public disk**: anyone who learns a name could download it. Other disks (S3) work: the zip is built in a temporary file and then uploaded. |
| `exportDirectory(string $path = 'warp-saas/exports')` | The folder inside the disk. Each team gets its own subfolder named by a keyed hash of the tenant key (HMAC with `APP_KEY`), so it cannot be guessed from the id. The file name has 16 random characters. |
| `exportsExpireAfter(int $days = 7)` | How long a zip is kept. After this `warp-saas:prune-exports` deletes the file and the row. |
| `exportsPerDay(int $count = 3)` | Exports per team per day. |
| `secretColumns(array $patterns)` | More columns to leave out: a name (`'api_key'`) or a regular expression (`'/^iban$/'`). Added to the built-in ones. |
| `excludeFromExport(array $tables)` | Tables left out of the zip (logs, search copies). `warp_saas_exports` is always left out. |
| `keepUsersWhere(?Closure $callback)` | `fn (Authenticatable $user): bool` — a user who must not be deleted with their last team (your staff). |
| `deleteOrphanUsers(bool $enabled = true)` | Whether users who belonged only to the deleted team are deleted too. `false` if you manage users somewhere else. |

### Deletion

```bash
php artisan warp-saas:delete acme            # slug or key. Asks you to type the slug again
php artisan warp-saas:delete acme --force    # no question
php artisan warp-saas:delete acme --now      # also before the retention period is over
php artisan warp-saas:purge --dry-run        # list the ended teams whose retention is over
php artisan warp-saas:purge --force          # delete them all (for the scheduler)
```

The rules:

- **A team with an active or trialing account is refused** (`TenantDeletionRefused`; `--now` does not change that).
- Before the end of the retention period (`dataRetention()`) it is refused unless you pass `--now` — for when the customer asked for the deletion. The last day of retention is counted in the team's calendar.
- `warp-saas:delete` asks you to type the slug again; `--force` skips the question.
- The order: the tenant-data tables from child to parent, 10,000 rows per statement (no giant transaction); for a table with registered file columns, **the file of a batch is deleted before its rows**; then the users who were only in this team (not those that still appear in other teams' rows, and not those `keepUsersWhere()` keeps); the members pivot and the `warp_saas_*` rows (not the invoices in `warp_saas_charges`, which are kept — see [billing](#after-a-team-is-deleted); `WarpSaas::data()->keepOnDelete($table, $reason)` keeps a table of yours the same way); the team's own files and the folder of its exports; and **last** the account row and the team itself, in one transaction.
- Because the team row goes last, a deletion that is cut off in the middle (a timeout, a server restart) can simply be started again: the team is still *ended*, and what is left is deleted. A file whose row is already gone cannot be left behind.
- Events `TenantDeleting` (before) and `TenantDeleted` (after: type, key, name and counts of rows, files and users — the team model no longer exists).
- `purge` deletes the teams one after another; one that fails is reported and the others still go on.
- In the operator panel the same rules apply, with a confirmation that makes you type the team's name.

A button that lets a team delete itself is not included on purpose.

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('warp-saas:prune-exports')->daily();
Schedule::command('warp-saas:purge --force')->daily();   // try `warp-saas:purge --dry-run` first
```

> **Files of a tenant must live in a folder of their own.** A file column such as `photo_path` holds a path that is just a value in the database. Warp SaaS includes every path found in the rows of a team in that team's export, and deletes it when the team is deleted. If a row of team A holds the path of a file of team B — through a bug, an import, or a path edited in the browser — then A's export contains B's file and deleting A removes it. Store files under a folder per tenant (`teams/{team_id}/…`) and check the path when you save it. Warp SaaS refuses paths with `..` and disks that are not in `config/filesystems.php`, but it cannot know whose a path is.

### From code

```php
use Qalainau\FilamentWarpSaas\Export\ExportTenantData;
use Qalainau\FilamentWarpSaas\WarpSaas;

$export = WarpSaas::actingAs($team, fn () => app(ExportTenantData::class)->handle($team));
// $export->disk, $export->path, $export->size, $export->status (ExportStatus), $export->expires_at
```

`warp-saas:export {tenant}` does the same and prints the disk and path. It also leaves a row in the exports table, so the zip is deleted when it expires.

## Usage quotas

Some features cost you money every time they are used: an AI call, an SMS, a PDF render. A **meter** limits them by the month, by money, for the whole service and by the minute, and records every use.

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

// AppServiceProvider::boot()
WarpSaas::meter('ai.summary')
    ->label('Project summary (AI)')
    ->monthlyLimit(['lite' => 20, 'standard' => 300, 'pro' => 1000], trial: 10)
    ->monthlyBudget(['lite' => 1, 'standard' => 5, 'pro' => 20], trial: 0.5)
    ->perMinute(3);

WarpSaas::serviceBudget(monthly: 100, trialMonthly: 10);
```

```php
use Qalainau\FilamentWarpSaas\Exceptions\UsageLimitReached;

try {
    $summary = WarpSaas::meter('ai.summary')->attempt(
        callback: fn () => $ai->summarize($project),
        cost: fn ($result) => $result->usage->cost,   // or a number, e.g. 0.002
        meta: ['project_id' => $project->id],
        estimate: 0.01,                                // optional: what one use may cost, checked against the budget
    );
} catch (UsageLimitReached $exception) {
    Notification::make()->danger()->title($exception->getMessage())->send();   // a sentence for the user
}
```

`attempt()` **takes a seat** before it calls your function, so two people of the same team who press the button at the same moment cannot both take the last use (see [Seats](#seats-reserve-and-reservation)). Each use needs a cache store with locks (`database`, `redis`, `memcached`, `file`, `array`…, the Laravel default `database` is fine).

### `Meter`

| Method | What it does |
| --- | --- |
| `label(string\|Closure\|null)` | The name in messages and on the *Plan and usage* page. Default: the meter's name. |
| `monthlyLimit(int\|array\|null $limits, int\|false\|null $trial = false)` | Uses per team per month: one number for every plan, or `['plan' => n]`. `null` and plans left out are unlimited. `trial:` is the number for a team on trial (default: the plan's number, as if not on trial). |
| `monthlyBudget(int\|float\|array\|null $budgets, int\|float\|false\|null $trial = false)` | The same for money per team per month. The currency is yours: Warp SaaS only adds up the numbers you record. |
| `sharesBudgetWith(string\|BackedEnum\|null $meter)` | Use the money budget of another meter, shared: the budget (and the trial budget, and the team's own value) is that meter's, and it is compared with the cost of this month of that meter **and every meter that shares it**. The limit of uses stays per meter. A meter that shares cannot have its own `monthlyBudget()`, and the shared meter cannot share again (both throw `LogicException` when used). `null` goes back to the meter's own budget. |
| `budgetSpentThisMonth(?Model $tenant = null): float` | The cost compared with the budget: this meter's, or the whole group's when the budget is shared. |
| `budgetReached(?Model $tenant = null): bool` | Whether that cost has reached the budget (`false` with no budget). |
| `perMinute(?int $times)` | Uses per minute per person (per team when the person is unknown). Uses Laravel's rate limiter. |
| `ensureAvailable(?Model $tenant = null, ?Authenticatable $user = null, int $quantity = 1)` | Throws `UsageLimitReached` if the next use is not allowed. Defaults: the current tenant and signed-in user. |
| `record(?Model $tenant = null, ?Authenticatable $user = null, int\|float $cost = 0, int $quantity = 1, array $meta = [])` | Records one use (one row in `warp_saas_usages`). Negative numbers are refused. |
| `reserve(?Model $tenant = null, ?Authenticatable $user = null, int $quantity = 1, int\|float $estimate = 0, array $meta = []): Reservation` | Checks the same four stops as `ensureAvailable()` and, if they pass, **takes the seat at once**: a row is recorded with the quantity and the estimate as its cost, and counted by every check after it. Close it with `complete()` or `fail()`. See [Seats](#seats-reserve-and-reservation). |
| `attempt(?Model $tenant = null, ?Authenticatable $user = null, ?Closure $callback = null, int\|float\|Closure $cost = 0, int $quantity = 1, array\|Closure $meta = [], int\|float $estimate = 0)` | `reserve()` → your function → `complete()`, and returns the function's result. The function receives the `Reservation` (a function with no parameters works too). **If the function throws**, the cost it reported with `$reservation->cost(…)` is kept (no use is counted), or, if it reported none, the seat is removed; the exception is thrown on. Use named arguments. |
| `remaining(?Model $tenant = null): ?int` | Uses left this month (`null` if unlimited). |
| `usedThisMonth(?Model $tenant = null): array` | `['quantity' => int, 'cost' => float]`. |
| `reset(?Model $tenant = null): int` | Deletes this month's records of this meter for the tenant, so its monthly count and cost start from 0 again, and returns how many it deleted. Earlier months and other meters stay. For an operator who gives a customer a fresh month, or a demo. |
| `clearRateLimit(?Model $tenant = null, ?Authenticatable $user = null): void` | Forgets the per-minute count of one person (the logged-in person when left out), or of the tenant when there is no person, so the meter can be used again at once. |

Several features, one budget — a limit of uses for each AI feature, but one amount of money for all of them:

```php
WarpSaas::meter('ai')->monthlyBudget(['lite' => 2000, 'standard' => 6000], trial: 300);   // only the budget
WarpSaas::meter('ai.orders')->label('Read orders (AI)')->monthlyLimit(['lite' => 100, 'standard' => 300], trial: 20)->sharesBudgetWith('ai');
WarpSaas::meter('ai.invoices')->label('Read invoices (AI)')->monthlyLimit(['lite' => 100, 'standard' => 300], trial: 20)->sharesBudgetWith('ai');
```

The *Plan and usage* page leaves out a meter that only holds a shared budget (no limit of uses); when the budget is used up, every meter that shares it says so.

Where the limit comes from, in this order: the team's own value (`WarpSaas::setMeterLimit($tenant, 'ai.summary', limit: 500, budget: 10)`; `resetMeterLimit()` removes it) → `trial:` if the team is on trial → the entry for the team's plan in the meter's array → the plan's own `Plan::meter('ai.summary', limit: 50, budget: 2)` → the single number given to the meter. Pass `null` to mean unlimited, and leave `limit:` or `budget:` out of `setMeterLimit()` to leave that one alone.

### Seats: `reserve()` and `Reservation`

`ensureAvailable()` → your call → `record()` leaves a gap: two requests at the same moment both pass the check, then both record, and a team with 1 use left gets 2 (with 8 at once, 8: we measured it on MySQL and PostgreSQL). `reserve()` closes the gap. It waits for a lock per team and budget (meters that [share a budget](#meter) share the lock), checks the four stops, records the seat, and releases the lock at once — the lock is held for a few milliseconds, never while the AI answers.

```php
use Qalainau\FilamentWarpSaas\Exceptions\UsageLimitReached;

$reservation = WarpSaas::meter('ai.intake')->reserve($tenant, $user, estimate: 0.02);   // throws UsageLimitReached when used up

try {
    $response = $ai->read($file);              // the provider charges you here…
    $reservation->cost($response->cost);       // …so say what it cost as soon as you know
    $order = $parser->parse($response);        // …and this may still fail
} catch (Throwable $exception) {
    $reservation->fail();                      // keeps the reported cost (0 uses), or removes the seat when none was reported
    throw $exception;
}

$reservation->complete(cost: $response->cost, meta: ['file' => $file->name]);
```

| `Reservation` method | What it does |
| --- | --- |
| `cost(int\|float $cost): static` | Remembers what the use has cost so far. `fail()` and `complete()` use it when they get no cost. |
| `complete(int\|float\|null $cost = null, array $meta = []): Usage` | The use is done: the row gets the real cost (or the reported cost, or the estimate) and the meta (on top of the meta given to `reserve()`). |
| `fail(int\|float\|null $cost = null): ?Usage` | The use failed. With a cost (given here or reported with `cost()`), the row stays with **quantity 0** and that cost and `meta.failed = true`: it counts toward the budget, not toward the monthly limit. With no cost the row is deleted and `null` is returned. |
| `isClosed(): bool`, `usage(): Usage`, `$quantity`, `$estimate` | The state, the row, and what was reserved. |

- **The budget with an estimate:** `reserve()` stops when *cost this month + estimate > budget*. With no estimate (`0`) it stops when *cost this month ≥ budget*, as `ensureAvailable()` does. Use the estimate when one use can be expensive (a long PDF), so that the last use does not run far past the budget.
- **Closing twice** does nothing the second time: `complete()` returns the same row, `fail()` does nothing.
- **A seat that is never closed** (the PHP process died in between) stays with its estimate and `meta.reserved = true`. It is counted, so the meter errs on the side of counting too much. Nothing cleans it up; an operator can see it in `warp_saas_usages` by `meta->reserved`.
- **The lock needs a shared cache store with locks.** Without one (the `null` store), `reserve()` and `attempt()` throw a `LogicException` instead of going on without a lock. With several web servers, use a store they all share (`database`, `redis`); a `file` store only locks within one server. When the lock is busy for more than `Meter::$lockWaitSeconds` (5), `reserve()` and `attempt()` throw `Qalainau\FilamentWarpSaas\Exceptions\UsageBusy` (it extends `Illuminate\Contracts\Cache\LockTimeoutException`, so code that caught that keeps working). Its message tells the person to wait a moment and try again; uncaught in a request it becomes a 503 with `Retry-After: 5`, not a 500. No usage row is left. Through the AI gateway it becomes `AiUnavailable` with the reason `busy` and a translated `userMessage()`.
- **Do not call `reserve()` inside a database transaction** you have opened: the seat is not visible to other requests until you commit, so the gap opens again.
- `ensureAvailable()` and `record()` do not take a seat and work as before (no lock is needed).

### The four stops, in this order

`ensureAvailable()` (and `reserve()`) checks these in order and throws a message you can show:

1. **Uses this month for the team** (`monthlyLimit`): *“This month, “Project summary” can be used up to 20 times, and it has already been used 20 times. You can use it again on December 1.”*
2. **Money this month for the team** (`monthlyBudget`, or the shared budget of `sharesBudgetWith()`): *“…has reached the amount set for this month.”* The amount itself is not shown.
3. **The service-wide budget** (`serviceBudget`): the sum of all meters and all teams in the calendar month of the **app's** time zone. Teams on trial have **their own, separate budget** (`trialMonthly`), so a crowd of free trials can use up their budget without stopping paying teams. A team on trial is stopped only by the trial budget, an active team only by the monthly one. The first time a budget is reached in a month a `critical` log line is written and the event `ServiceBudgetReached` is dispatched. It is there to cap the damage of a bug or abuse, not to meter money precisely.
4. **Uses per minute** (`perMinute`): *“Wait about 12 seconds, then try again.”* The use is counted when it passes this check. `UsageLimitReached::$retryAfter` holds the seconds, and the HTTP response gets a `Retry-After` header.

`UsageLimitReached` has a `$kind` (`UsageLimitKind::MonthlyLimit`, `MonthlyBudget`, `ServiceBudget`, `PerMinute`), its status code is **429**, and `getMessage()` can be shown to the user.

### The month

“This month” for a team is the calendar month in the **team's time zone** (`timezoneUsing()`): the first of the month 0:00 to the first of the next month. The service-wide budget uses the app's time zone. Nothing is reset; uses are only counted from the start of the month.

### Good to know

- **Two requests at the same moment can both pass `ensureAvailable()`**, which does not take a seat. For a hard cap use `attempt()` or `reserve()`. The service-wide budget is not locked (it is a cap on damage, not a precise meter).
- The *Plan and usage* page shows the uses of this month and a sentence when a meter is used up. It does not show money.
- `WarpSaas::meters()` lists the registered meters.

```php
Schedule::command('warp-saas:prune-usages --months=13')->monthly();   // keeps this month and the 12 before it
```

## AI gateway

The AI features of the Warp plugins (Warp Input fills a form from pasted text, Warp Import suggests column matches, …) do not call an AI provider themselves. They ask one **gateway**, bound in the container as `warp.ai`, and show their AI buttons only when it is bound. Warp SaaS is that gateway: it picks the model, takes a [seat](#seats-reserve-and-reservation) on the team's meter, calls the provider with [Laravel AI](https://github.com/laravel/ai), works out the cost from the tokens and records it. A plugin never names a provider or a model, so you change models in one place, and every use is counted per team, per person and per feature.

### Turn it on

Warp SaaS binds `warp.ai` only when at least one model is set — an app that does not use AI keeps every AI button hidden.

```bash
php artisan vendor:publish --tag=filament-warp-saas-config   # config/filament-warp-saas.php, the 'ai' section
```

```dotenv
ANTHROPIC_API_KEY=sk-ant-...            # Laravel AI's key (config/ai.php)
WARP_AI_MODEL_FAST=claude-haiku-5-5     # quick, cheap work: filling a form, matching columns
WARP_AI_MODEL_SMART=claude-sonnet-5-5   # harder reading: drawings, long documents
WARP_AI_CURRENCY_RATE=150               # USD → the currency of your usage records (here yen)
```

```php
// config/filament-warp-saas.php
'ai' => [
    // ...
    'plans' => [
        'lite' => ['input.form_fill' => 50],          // only this feature, 50 uses a month
        'pro' => ['*' => null],                       // every AI feature, no monthly limit
    ],
    'trial_limit' => 10,                             // a team on trial: 10 uses a month of each feature
    'budget_meter' => 'ai',                          // all AI features share one money budget…
],

// AppServiceProvider::boot()
WarpSaas::meter('ai')->monthlyBudget(['lite' => 300, 'pro' => 3000], trial: 50);   // …set here, in the currency of the records
WarpSaas::meter('input.form_fill')->label('Fill a form from pasted text (AI)');
```

Then run `php artisan warp-saas:ai-check`. It stops when a model has no price, the provider key is missing or a plan in `plans` does not exist.

| Setting (`filament-warp-saas.ai.*`) | What it does |
| --- | --- |
| `provider` | The Laravel AI provider (`anthropic`, `openai`, `gemini`, …). Default `anthropic`. |
| `models.fast`, `models.smart` | The model of each tier. When one is empty the other is used. Both empty: no gateway. |
| `vision_models.fast`, `.smart` | The model for a request with images or PDFs. Empty: the tier's model. |
| `prices` | USD per million tokens: `input`, `output`, `cache_read`, `cache_write`, and `long` for a price above a number of input tokens. Anthropic's current models are filled in (2026-11-03). **A model without a price is refused** (the use is not made), so nothing is counted at no cost. |
| `currency_rate` | USD → the currency of the records. The row keeps the USD amount too (`meta.ai.usd`). |
| `plans` | The meters each plan may use, with the monthly limit (`null` = no limit, `'*'` = every meter). A plan that is not listed may use no AI. Empty: every plan may use every meter, with the limits you set with `WarpSaas::meter()`. |
| `trial_limit` | The monthly limit of each AI meter for a team on trial. `null`: the plan's. |
| `budget_meter` | A meter whose money budget every AI meter shares (`sharesBudgetWith()`). |
| `allow_without_tenant` | `false` (default): a request outside a team (an operator page, a command) is refused, because it cannot be counted. `true`: it is made without a seat or a record; listen to `AiUsed` to count it yourself. |
| `timeout` | Seconds to wait for the provider (default 60). |
| `admin_keys.anthropic`, `.openai` | Admin keys for `warp-saas:ai-reconcile` only. |

### What happens on each use

1. **A team is needed.** The team is the request's `tenant` (a queued job passes the team of the record), or the current team.
2. **The plan must allow the meter** (`plans`). The meter is `<plugin>.<feature>` (`input.form_fill`), or the name the app gives the feature in the plugin (`->aiFeature('order_intake')`).
3. **A seat is taken** with the estimate: the length of the input (one token per three characters), the files (1,600 tokens an image, 50 a KB of a PDF) and the **whole** output limit of the feature, at the model's price. Near the end of a budget the gateway stops a little early rather than late.
4. **The provider is called** with the instructions, the input, the files and the JSON schema of the answer (structured output), the output limit of the feature, and a hash of the team in `metadata.user_id` (Anthropic) or `safety_identifier` (OpenAI) — never a name or an email.
5. **The real cost** (input, cache reads and writes, output, at the price table, times `currency_rate`) replaces the estimate. The row's `meta.ai` keeps the plugin, feature, tier, provider, model, USD and token counts.
6. **When it fails** after the provider was paid (an answer that is not in the schema), the cost is kept with 0 uses; when nothing is known to be paid, the seat is removed. The plugin gets an exception with `userMessage()`, a sentence in the user's language, and shows it instead of an error page.

A plugin asks `app('warp.ai')->unavailable([...])` before it shows or enables a button. It counts nothing (not even the per-minute count) and returns `null` or the sentence to show: the plan does not include it, the monthly limit, the budget.

### Checking the cost and the models

| Command | What it does |
| --- | --- |
| `warp-saas:ai-check` | Checks the settings (above). Sends nothing to the provider. |
| `warp-saas:ai-reconcile {--month=2026-11} {--provider=}` | Compares the recorded USD of a month (UTC) with the provider's cost API: Anthropic's Usage and Cost Analytics API (falls back to the older cost report), OpenAI's Costs API. Needs an admin key; without one it shows only the recorded cost. The provider cannot split the cost per team, so the per-team numbers are Warp SaaS's records. Compare in a workspace (or project) that only the gateway uses. The providers can be a day late and revise a month for about a week. |
| `warp-ai:eval {plugin.feature} {--model=} {--case=}` | Runs a plugin's AI cases (`tests/ai-cases/<feature>.json`, where the plugin's folder is in `config('warp.ai.cases.<plugin>')`) against the real model and counts the matching values. Run it before changing a model and after changing a prompt. It costs money, so not in CI; it is not metered. |

### Testing your app

```php
use Qalainau\FilamentWarpSaas\WarpSaas;

$ai = WarpSaas::fakeAi(fn (array $request) => [
    'data' => ['fields' => ['customer' => 'Harbor Supply Co.']],
    'usage' => ['input_tokens' => 900, 'output_tokens' => 120],   // optional: the cost to record
]);

// … press the plugin's AI button in your Livewire test …

$ai->assertAsked(fn (array $request) => $request['plugin'] === 'input' && $request['feature'] === 'form_fill');
```

The gateway stays real — the team, the plan, the seat and the record — and only the provider is replaced. `fakeAi()` works without any model set. Throw from the function to test a provider error (`new AiCallFailed('…', usage: [...])` keeps its cost). To go through Laravel AI itself instead, use `GatewayAgent::fake([...])`.

### Good to know

- **The answer is data, never instructions.** The input and files are written by your users, so an answer is only used as values of the schema. A plugin checks the shape and drops what does not fit.
- **Changing a model:** set `WARP_AI_MODEL_FAST`/`SMART`, add its price if it is not in the table, run `warp-saas:ai-check`, then `warp-ai:eval` for the features you use.
- `AiUsed` is dispatched after every call (`succeeded` false when it failed). Its `request` holds what the user wrote: do not log it as is.
- Your own gateway: bind `warp.ai` yourself (`$this->app->singleton('warp.ai', MyGateway::class)`) and Warp SaaS leaves it alone.

## Operator panel

A second Filament panel **without** `->tenant()` for you, the person who runs the SaaS. It lists every team and lets you act on it.

```php
use Filament\Panel;
use Filament\PanelProvider;
use Qalainau\FilamentWarpSaas\FilamentWarpSaasAdminPlugin;

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('admin')
            ->path('admin')
            ->login()
            ->databaseNotifications()        // so you are told when an export is ready
            ->plugin(FilamentWarpSaasAdminPlugin::make());
    }
}
```

> **Who can enter this panel must be decided by `canAccessPanel()` on your user model, and only operators may pass.** The panel shows every customer and can delete them. The plugin does not know who your operators are, and a panel without a check lets any signed-in customer in.
>
> ```php
> public function canAccessPanel(Panel $panel): bool
> {
>     return $panel->getId() !== 'admin' || $this->is_admin;
> }
> ```
>
> Check the id as above: the same method is asked for the team panel too, and it must stay open for customers there. You can narrow it more with the plugin: `->authorize(fn (): bool => auth()->user()->can('manage-accounts'))` is asked before every use of the resource, every button, and every download.

The plugin throws a clear error if you put it on a panel that has `->tenant()`.

In this panel `BelongsToTenant` does not limit queries: the operator sees every team. A panel without tenancy that does **not** have this plugin is not an operator panel and is treated like code outside the panels. If you have an operator panel of your own without the plugin, name it on the tenant panel's plugin:

```php
FilamentWarpSaasPlugin::make()->operatorPanels('admin');
```

### What you can do

The resource **Accounts** lists every team: name, plan, state, trial or contract dates, number of members, created. You can search by name, filter by state and plan, and open the team's page with its details and its exports. Buttons, each with a confirmation and each checked with `authorize()`:

| Button | What it does |
| --- | --- |
| Change plan | `changePlan()`. If the team has more than the new plan allows, the form says exactly what, and you can force it. |
| Add or remove features | `grantFeature()` / `revokeFeature()` — add-ons outside the plan. |
| Change limits | `setLimit()` / `resetLimit()` per metric (members, projects, …). |
| Extend the trial | `extendTrial()`. |
| Start the contract | `activate()` with a plan. |
| Set the end date | `end()`. |
| Cancel the end | `resume()`. |
| Export the data | Starts an export like the team's own page does (as a queued job); you are told when it is ready and download it from the team's page. |
| Delete the data | Only for ended teams. Before the retention period is over you must tick “delete now”, and you must type the team's name. |
| Set the price and how they pay | With [billing](#billing): `setPrice()`, `setBillingMethod()` and `setBillingPeriod()`, from the next invoice. |
| Create an invoice | With billing: `charge()` for this team — one or more lines (what for, price of one, quantity), such as a setup fee and training. |

With billing turned on, the team's page also shows how it pays, its price (the lines of the next invoice when there is more than the plan), the next invoice date and its invoices, and a second resource, **Invoices**, lists the invoices of every team — also of deleted teams — sorted by due date, with filters for unpaid, overdue, paid and canceled (an invoice with several lines says *and 2 more lines* under what it is for). Each row has *Record the payment* (date and a note) and *Cancel the invoice* (a reason), and *Create an invoice* is above the list. Without billing the resource is not in the menu and cannot be opened.

Buttons that make no sense for the state of a team are not shown (“Extend the trial” on an active team, “Delete the data” on a team in use).

### Only the *Invoices* resource, in your own operator panel

If your app already has its own operator panel and you do not want the plugin's *Accounts* resource, register `ChargeResource` alone — `FilamentWarpSaasAdminPlugin` is not needed:

```php
$panel->resources([\Qalainau\FilamentWarpSaas\Resources\ChargeResource::class]);
```

**Register it only in an operator panel without tenancy.** In a panel that has tenancy (a team's panel) it never opens, because it lists the invoices of every team.

Without the admin plugin, `canAccess()` is true for a signed-in user (who can already open your panel through `canAccessPanel()`), billing must be turned on, and your `Charge` policy's `viewAny` is asked when you have one. **Keep the panel for operators only**, or write the policy. To decide who is an operator in another way, extend the resource and override `canAccess()` (or `isOperator()`), and register your class:

```php
class OperatorChargeResource extends ChargeResource
{
    public static function canAccess(): bool
    {
        return auth()->user()?->isFinance() === true && parent::canAccess();
    }
}
```

The *Create an invoice* button works there too (the team is searched through the tenant model of your tenant panel, or `FilamentWarpSaasAdminPlugin::make()->tenantModel(...)`). To create invoices from your team's edit page instead of the list, put the button on that page and give it the team:

```php
// in EditTeam::getHeaderActions()
ChargeResource::createChargeAction(fn ($record) => $record),
```

### Settings

| Method | What it does |
| --- | --- |
| `tenantModel(?string $class)` | The tenant model to list. Default: the model of the tenant panel that has `FilamentWarpSaasPlugin`. |
| `authorize(bool\|Closure $callback)` | Narrows who can use the resource beyond `canAccessPanel()`. |
| `accountResource(bool\|string $resource)` | `false` removes it; a class that extends `AccountResource` replaces it. |
| `chargeResource(bool\|string $resource)` | The *Invoices* resource (only shown with billing). `false` removes it; a class that extends `ChargeResource` replaces it. If your app has a policy for `Qalainau\FilamentWarpSaas\Models\Charge` with `viewAny`, it is asked too (on top of `->authorize()`), so you can keep some operators away from the invoices; without a policy nothing changes. The billing summary and history on an account's page also ask `viewAny`, and each history row and its invoice link ask `view`. |
| `tenantNameColumn(?string $column)` | The column to search and sort the name by (default `name`). `null` when the name is made by `getFilamentName()`: search and sorting by name are then not offered (the name is always shown through Filament). |
| `navigationGroup()`, `navigationSort()`, `navigationIcon()`, `navigationLabel()` | Where the resource is in the menu. |

## Events

Dispatched with Laravel's event dispatcher (`Event::listen`, a listener class, `Event::fake()`). Namespace: `Qalainau\FilamentWarpSaas\Events`.

| Event | Properties | Dispatched when |
| --- | --- | --- |
| `TrialStarted` | `tenant`, `account` | `startTrial()` (also from `startTrialOnTenantCreated()`). |
| `TrialExtended` | `tenant`, `account`, `days` | `extendTrial()`. |
| `TrialEnding` | `tenant`, `daysLeft`, `lastDay`, `recipients` | `warp-saas:notify-trials` sent a mail for a team. |
| `AccountActivated` | `tenant`, `account` | `activate()`. |
| `AccountEnded` | `tenant`, `account`, `endsAt` | `end()`. |
| `AccountResumed` | `tenant`, `account` | `resume()`. |
| `PlanChanged` | `tenant`, `previous`, `plan` | `changePlan()` changed the plan (not when it was the same). |
| `InvitationSent` | `invitation`, `by`, `resent` | `invite()` or `resendInvitation()`. |
| `InvitationAccepted` | `invitation`, `tenant`, `user`, `wasAlreadyMember` | `acceptInvitation()`. |
| `MemberRemoved` | `tenant`, `member`, `by` | `removeMember()`. |
| `MemberRoleChanged` | `tenant`, `member`, `previous`, `role`, `by` | `changeRole()` changed the role. |
| `TenantExported` | `tenant`, `export` | An export zip is ready. |
| `TenantDeleting` | `tenant` | A deletion starts. |
| `TenantDeleted` | `tenantType`, `tenantKey`, `tenantName`, `counts` | A deletion is done. The model no longer exists. |
| `ServiceBudgetReached` | `trial`, `budget`, `spent`, `month` | A service-wide budget is reached (once per month and budget). |
| `AiUsed` | `request`, `tenant`, `model`, `usage`, `usd`, `succeeded` | The [AI gateway](#ai-gateway) called the provider (also when the call failed). |
| `ChargeCreated` | `tenant`, `charge` | An invoice was made (`activate()`, `warp-saas:bill`, `charge()`), after `issueInvoiceUsing()` and before the mail. |
| `ChargePaid` | `tenant`, `charge` | `markPaid()`. |
| `ChargeVoided` | `tenant`, `charge` | `voidCharge()`. |
| `ChargeOverdue` | `tenant`, `charge` | `warp-saas:bill` found the invoice past its due date (once per invoice). |
| `OverdueGraceEnded` | `tenant`, `charge`, `action` | The invoice is still unpaid `graceDays()` after the due date (once per invoice). With `Overdue::End`, after the contract was ended. |
| `CardPaymentFailed` | `tenant`, `charge`, `failure` | A card payment failed (declined, no card, the bank wants a confirmation). `failure` is the code, for example `declined`. |
| `TenantMadeReadOnly` | `tenant`, `account`, `reason` | `makeReadOnly()`, or `Overdue::ReadOnly` after the grace days. `reason` is `manual` or `overdue`. |
| `TenantMadeWritable` | `tenant`, `account` | `makeWritable()`, or an overdue team whose unpaid invoices are gone. |
| `AfterTrialChanged` | `tenant`, `account`, `continues` | `continueAfterTrial()` changed the choice (not when it was the same). |

## Artisan commands

Put these on the scheduler of your app (`routes/console.php`). Warp SaaS does not schedule anything for you.

| Command | What it does | Schedule |
| --- | --- | --- |
| `warp-saas:notify-trials {--dry-run}` | Mails the owners of teams whose trial ends in `trialReminderDays()` days. At most one mail per team per day. First makes a contract of every finished trial that was set to [continue](#after-the-trial). | `->hourly()` (or daily) |
| `warp-saas:bill {--tenant=} {--dry-run}` | With billing: makes the invoices that are due, sends the reminders, reports invoices past due and handles them after the grace days (`Notify`, `End` or `ReadOnly`), charges failed cards again on the days of `retryCardAfterDays()`, and first makes a contract of every finished trial that was set to [continue](#after-the-trial). Nothing twice on the same day. | `->hourly()` (or daily) |
| `warp-saas:prune-charges {--dry-run}` | Deletes the invoices of deleted teams older than `keepChargesFor()`. Nothing without that setting. | `->monthly()` |
| `warp-saas:prune-exports` | Deletes zips and rows whose download period is over (`exportsExpireAfter()`). | `->daily()` |
| `warp-saas:purge {--dry-run} {--force}` | Deletes ended teams whose retention period is over. | `->daily()` with `--force` |
| `warp-saas:prune-invitations {--days=30}` | Deletes invitations that expired more than N days ago and were never accepted. | `->daily()` |
| `warp-saas:prune-usages {--months=13}` | Deletes usage records older than N months, counting this month. | `->monthly()` |
| `warp-saas:ai-check`, `warp-saas:ai-reconcile`, `warp-ai:eval` | See [AI gateway](#ai-gateway). | `ai-reconcile` monthly, after the 7th |
| `warp-saas:check-references {--table=}` | Finds rows that point to another team's rows. Exit code 1 when it finds one. | Before deploys, or `->daily()` |
| `warp-saas:export {tenant}` | Writes the zip of one team now and prints where. | By hand |
| `warp-saas:delete {tenant} {--force} {--now}` | Deletes one ended team. `{tenant}` is its slug or key. | By hand |

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('warp-saas:notify-trials')->hourly();
Schedule::command('warp-saas:bill')->hourly();
Schedule::command('warp-saas:prune-charges')->monthly();
Schedule::command('warp-saas:prune-exports')->daily();
Schedule::command('warp-saas:prune-invitations')->daily();
Schedule::command('warp-saas:purge --force')->daily();
Schedule::command('warp-saas:prune-usages')->monthly();
Schedule::command('warp-saas:check-references')->daily()->emailOutputOnFailure('ops@example.com');
```

## Translations

Everything Warp SaaS shows — pages, buttons, mails, the zip's `README.txt`, error sentences — is translated into English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese, Korean, and simplified and traditional Chinese. The language is the one Laravel is using (`app()->setLocale()`), as for the rest of your app.

To change a sentence, publish the files and edit them:

```bash
php artisan vendor:publish --tag=filament-warp-saas-translations
```

They go to `lang/vendor/filament-warp-saas/<locale>/`. Dates are written the way the language writes them (“November 26”, “11月26日”) and the year is left out for the current year.

## Testing your app

### Keep the tenant data portable

One test makes sure no new table or column slips out of the map. It runs six checks and fails with a list of sentences that say what to do:

```php
use Qalainau\FilamentWarpSaas\Testing\TenantDataAssertions;

it('keeps tenant data portable', function (): void {
    TenantDataAssertions::assertTenantDataIsPortable();
});
```

| Check | Fails when |
| --- | --- |
| `assertEveryTableIsClassified()` | A table is not shared, has no tenant column and has no parent that leads to one. |
| `assertRegisteredColumnsExist()` | A registration in `WarpSaas::data()` names a table or column that does not exist. |
| `assertEveryReferenceIsRegistered()` | A column that ends in `_id` is not a foreign key, not registered with `reference()`/`morphReference()` and not marked `notAReference()`. The guard could not follow it. |
| `assertTenantDataReferencesOnlyTenantData()` | Tenant data has a foreign key to a shared table that is not `referable()`. |
| `assertSharedTablesDoNotReferenceTenantData()` | A shared table has a foreign key to tenant data. |
| `assertTenantColumnsAreIndexed()` | A tenant table has no index starting with its tenant column (and is not `indexExempt()`). |

Each also exists as `…Problems()`, which returns the list of sentences instead of failing. The assertions flush the schema first, so they see tables created in the test. They work in Pest and PHPUnit.

### Acting as a team

In a test there is no panel request, so say who the tenant is:

```php
use App\Models\Project;
use App\Models\Team;
use Qalainau\FilamentWarpSaas\Exceptions\LimitReached;
use Qalainau\FilamentWarpSaas\Exceptions\TenantNotSet;
use Qalainau\FilamentWarpSaas\WarpSaas;

it('shows a team only its own projects', function (): void {
    $acme = Team::factory()->create();
    $globex = Team::factory()->create();

    WarpSaas::withoutTenant(function () use ($acme, $globex): void {
        Project::create(['team_id' => $acme->id, 'name' => 'Acme 1']);
        Project::create(['team_id' => $globex->id, 'name' => 'Globex 1']);
    });

    expect(WarpSaas::actingAs($acme, fn () => Project::query()->pluck('name')->all()))->toBe(['Acme 1']);
    expect(fn () => Project::query()->count())->toThrow(TenantNotSet::class);
});

it('stops the third project of a Lite team', function (): void {
    $team = Team::factory()->create();
    WarpSaas::changePlan($team, 'lite');

    WarpSaas::actingAs($team, function (): void {
        Project::create(['name' => 'One']);
        Project::create(['name' => 'Two']);

        expect(fn () => Project::create(['name' => 'Three']))->toThrow(LimitReached::class);
    });
});
```

To test a Filament page as a member, set the panel and the tenant, as the panel would:

```php
use Filament\Facades\Filament;

use function Pest\Livewire\livewire;

beforeEach(function (): void {
    $this->team = Team::factory()->create();
    $this->owner = User::factory()->create();
    $this->team->users()->attach($this->owner, ['role' => 'owner']);

    $this->actingAs($this->owner);
    Filament::setCurrentPanel('app');
    Filament::setTenant($this->team, isQuiet: true);
});

it('keeps a member out of the members page', function (): void {
    $member = User::factory()->create();
    $this->team->users()->attach($member, ['role' => 'member']);

    $this->actingAs($member);

    $this->get(Members::getUrl(tenant: $this->team, panel: 'app'))->assertForbidden();
});
```

Test the paths a member should not be able to use by their **URL or Livewire call**, not only that the button is missing: the pages, the export download of another team, a project id of another team. Warp SaaS has tests for each, and so should the pages you add.

Time-dependent code (trials, expiry) is easiest with `$this->travelTo('2026-11-26 10:00:00')`. The state is decided from `now()`.

## Not included (yet)

- Refunds and chargebacks for card payments. Do them in the Stripe dashboard ([Card payments](#card-payments-stripe)).
- Payment providers other than Stripe, and receipts or tax documents made by a provider.
- Reports and dashboards about your customers (revenue, churn, usage by team).
- Signing in as a customer from the operator panel (impersonation).
- A button on the team's side to delete its own account.
- A public sign-up form for trials. Your landing page is yours; the plugin starts the trial when the team is created.
- Custom fields per team, and a daily health-check mail.

## Requirements

- PHP 8.3+ with the `intl` and `zip` extensions
- Laravel 12 or 13
- Filament 5.8.3 or later
- SQLite, MySQL 8.0+, MariaDB 10.11+ or PostgreSQL 14+
- A queue worker for exports and mails

The interface is in English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese, Korean and Chinese (simplified and traditional).

## Support

Questions before or after buying: [warpui.com/support](https://warpui.com/support?plugin=saas).

## License

Warp SaaS is a commercial plugin. See [LICENSE.md](LICENSE.md).
