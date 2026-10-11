# How to Install and Use Laravel Telescope to Monitor Everything in Your Laravel App

*By [Lucky Ram](https://about.me/luckyram) · Laravel & Filament Developer*

When something breaks in a Laravel app, the hardest part usually isn't fixing it. It's **finding** it. Which query was slow? Which job failed? Did the email go out? What did the payment API actually send back?

[Laravel Telescope](https://laravel.com/docs/telescope) answers all of that. It's an official Laravel package that records what your app does (requests, database queries, queued jobs, emails, cache, logs, exceptions, outgoing API calls and more) and shows it in a clean dashboard at `/telescope`.

This guide walks through installing it step by step, configuring it properly, and using it to debug real problems.

---

## Why Telescope is so useful

| Without Telescope | With Telescope |
|---|---|
| Adding `dd()` and `Log::info()` everywhere to guess what happened | Click a request and see everything it did |
| "The page is slow" with no idea why | See every query, its time, and the exact file and line that ran it |
| N+1 query problems hide until production | The same query repeated 50 times is obvious at a glance |
| Sending test emails to yourself | Preview every email in the browser, no inbox needed |
| "The job failed" with no context | See the job payload, every attempt, the exception, and the request that dispatched it |
| "The partner API says they never got it" | See the exact URL, headers, body and response of every outgoing `Http::` call |
| "Did the nightly sync run?" | See when each scheduled task ran and what it printed |
| "Why can't this user see the button?" | See every gate/policy check and whether it passed |

In short, Telescope turns your app from a black box into something you can see inside. It saves hours of guesswork on every bug.

---

## What Telescope records

Telescope uses **watchers**. Each watcher listens for one kind of activity:

| Watcher | What you see |
|---|---|
| **Requests** | URL, method, status, headers, payload, response, duration, memory |
| **Queries** | Every SQL query with bindings, time, and source file/line. Slow ones are tagged |
| **Models** | Eloquent `created` / `updated` / `deleted` events and how many models were loaded |
| **Exceptions** | Stack trace, how often it happened, and the request or job that caused it |
| **Logs** | Everything written with `Log::` |
| **Jobs / Batches** | Queued jobs, payload, status, attempts, failures |
| **Mail** | Every email with a rendered HTML preview |
| **Notifications** | Notifications and the channels used |
| **Events** | Dispatched events and their listeners |
| **Cache / Redis** | Cache hits, misses, writes; Redis commands |
| **HTTP Client** | Outgoing `Http::` requests and responses |
| **Schedule** | Scheduled task runs and output |
| **Commands** | Artisan commands, arguments and exit codes |
| **Gates** | Authorization checks and results |
| **Views** | Rendered views and the data passed to them |
| **Dumps** | `dump()` output, shown in the dashboard instead of the page |

---

## Requirements

- PHP 8.2+
- Composer
- Laravel 11, 12 or 13
- A database (MySQL, PostgreSQL, SQLite, etc.) for storing Telescope entries

---

## Step 1 — Create (or open) a Laravel project

```bash
composer create-project laravel/laravel telescope-demo
cd telescope-demo
```

Set up your database in `.env`, then run:

```bash
php artisan migrate
```

Already have a project? Skip to Step 2.

---

## Step 2 — Install Telescope

**Option A: all environments**

```bash
composer require laravel/telescope
```

**Option B: local development only (recommended for most teams)**

```bash
composer require laravel/telescope --dev
```

---

## Step 3 — Publish Telescope's files and run the migration

```bash
php artisan telescope:install
php artisan migrate
```

`telescope:install` creates:

- `config/telescope.php`: watchers, storage, path, etc.
- `app/Providers/TelescopeServiceProvider.php`: authorization, filtering, tagging
- the dashboard's frontend assets

`migrate` creates the `telescope_entries`, `telescope_entries_tags` and `telescope_monitoring` tables.

---

## Step 4 — Open the dashboard

```bash
php artisan serve
```

Visit **http://localhost:8000/telescope**. Click around your app in another tab and watch requests, queries and everything else appear in real time.

---

## Step 5 — (Local only) Register Telescope just for local

If you used `--dev` in Step 2, stop Laravel from auto-loading Telescope in production.

**5.1** Remove `App\Providers\TelescopeServiceProvider::class` from `bootstrap/providers.php`.

**5.2** Register it conditionally in `app/Providers/AppServiceProvider.php`:

```php
public function register(): void
{
    if ($this->app->environment('local') && class_exists(\Laravel\Telescope\TelescopeServiceProvider::class)) {
        $this->app->register(\Laravel\Telescope\TelescopeServiceProvider::class);
        $this->app->register(TelescopeServiceProvider::class);
    }
}
```

**5.3** Turn off package auto-discovery for Telescope in `composer.json`:

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "laravel/telescope"
        ]
    }
}
```

Then run:

```bash
composer dump-autoload
```

You can also toggle Telescope anytime with:

```dotenv
TELESCOPE_ENABLED=true
```

---

## Step 6 — Secure the dashboard

In `local`, anyone can open `/telescope`. Everywhere else, access is controlled by the `viewTelescope` gate in `app/Providers/TelescopeServiceProvider.php`:

```php
protected function gate(): void
{
    Gate::define('viewTelescope', function ($user) {
        return in_array($user->email, [
            'admin@your-company.com',
        ]);
    });
}
```

Using roles (for example with spatie/laravel-permission) is cleaner:

```php
Gate::define('viewTelescope', fn ($user) => $user->hasRole('developer'));
```

Optionally move the dashboard to a less obvious URL:

```dotenv
TELESCOPE_PATH=internal/telescope
```

> ⚠️ Telescope shows request payloads, SQL bindings and email contents. **Never** leave it open on a public server.

---

## Step 7 — Configure the watchers

Open `config/telescope.php`. These are the settings I change on every project:

```php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'ignore_packages' => true,
        'ignore_paths' => [],
        'slow' => 100, // ms: queries slower than this get tagged "slow"
    ],

    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.*'],
        'hydrations' => true, // shows how many models each request loads
    ],

    Watchers\LogWatcher::class => [
        'enabled' => env('TELESCOPE_LOG_WATCHER', true),
        'level' => 'error', // only record error and above
    ],

    Watchers\RequestWatcher::class => [
        'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
        'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64), // KB
        'ignore_http_methods' => [],
        'ignore_status_codes' => [],
    ],

    Watchers\CommandWatcher::class => [
        'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
        'ignore' => ['schedule:run', 'queue:work'],
    ],

    // ...keep the rest of the defaults
],
```

Ignore noisy paths so they don't flood the dashboard:

```php
'ignore_paths' => [
    'livewire*',
    'nova-api*',
    'pulse*',
    'horizon*',
],
```

Any watcher can be turned off from `.env`, for example:

```dotenv
TELESCOPE_VIEW_WATCHER=false
TELESCOPE_CACHE_WATCHER=false
```

---

## Step 8 — Filter what gets stored (staging/production)

Locally you want everything. On staging or production you usually only want the important things. In `TelescopeServiceProvider::register()`:

```php
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

public function register(): void
{
    $this->hideSensitiveRequestDetails();

    $isLocal = $this->app->environment('local');

    Telescope::filter(function (IncomingEntry $entry) use ($isLocal) {
        return $isLocal ||
               $entry->isReportableException() ||
               $entry->isFailedRequest() ||
               $entry->isFailedJob() ||
               $entry->isScheduledTask() ||
               $entry->isSlowQuery() ||
               $entry->hasMonitoredTag();
    });
}
```

---

## Step 9 — Hide sensitive data

Telescope stores request payloads and headers, so hide anything private. This matters a lot for healthcare and finance apps:

```php
protected function hideSensitiveRequestDetails(): void
{
    if ($this->app->environment('local')) {
        return;
    }

    Telescope::hideRequestParameters([
        '_token',
        'password',
        'password_confirmation',
        'ssn',
        'card_number',
        'cvv',
        'date_of_birth',
    ]);

    Telescope::hideRequestHeaders([
        'cookie',
        'x-csrf-token',
        'x-xsrf-token',
        'authorization',
    ]);
}
```

Also:

- Use fake or anonymized data wherever Telescope records everything.
- Disable the **Views** watcher if views receive personal data.
- Treat `telescope_entries` like a log file with sensitive data: restrict access and prune it often.

---

## Step 10 — Add custom tags

Telescope auto-tags entries with the user ID and model names (e.g. `App\Models\Invoice:42`). Add your own tags to search faster:

```php
Telescope::tag(function (IncomingEntry $entry) {
    $tags = [];

    if ($entry->type === 'request') {
        $tags[] = 'status:' . $entry->content['response_status'];
    }

    if (auth()->check()) {
        $tags[] = 'role:' . auth()->user()->role;
    }

    return $tags;
});
```

Now search `status:500` or `role:admin` in the dashboard.

**Monitoring a tag:** on the **Monitoring** page, add a tag like `Auth:17`. Combined with `hasMonitoredTag()` in Step 8, Telescope records **everything** for that one user, even on staging where you normally store only errors. Perfect for debugging one customer's bug report.

---

## Step 11 — Prune old entries

Telescope writes a lot of rows. Without pruning, the table grows to gigabytes. Schedule the prune command in `routes/console.php`:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

Useful commands:

```bash
php artisan telescope:prune --hours=48     # delete entries older than 48 hours
php artisan telescope:clear                # delete all entries
php artisan telescope:pause                # stop recording
php artisan telescope:resume               # start recording again
```

Optional: store entries on a separate database connection. Add a `telescope` connection in `config/database.php`, then in `config/telescope.php`:

```php
'storage' => [
    'database' => [
        'connection' => env('TELESCOPE_DB_CONNECTION', 'telescope'),
        'chunk' => 1000,
    ],
],
```

---

## Step 12 — Use it to debug real problems

### 🐢 Find N+1 queries

Open a slow request → **Queries**. If you see this:

```sql
select * from `patients` where `patients`.`id` = 1 limit 1
select * from `patients` where `patients`.`id` = 2 limit 1
select * from `patients` where `patients`.`id` = 3 limit 1
...
```

...fix it with eager loading:

```php
// Before
$visits = Visit::all();

// After
$visits = Visit::with('patient')->get();
```

Keep them from coming back:

```php
// AppServiceProvider::boot()
Model::preventLazyLoading(! app()->isProduction());
```

### 🐌 Find slow queries

Filter the **Queries** tab by the `slow` tag. Telescope shows the file and line that ran each one, so you know exactly which index to add or which query to rewrite.

### 💥 Trace an exception to its cause

**Exceptions** → open one → follow the link to the request or job that triggered it. You'll see the payload, the logged-in user, and every query that ran before it failed.

### 📬 Preview emails

**Mail** shows recipients, subject and a rendered HTML preview. No more test emails to yourself.

### ⚙️ Debug queued jobs

**Jobs** shows payload, attempts and the exception. Telescope links each job to the request that dispatched it: request → job → API call → email.

### 🌐 Inspect third-party API calls

Every `Http::` call appears in **HTTP Client** with full URL, headers, body, response and status code:

```php
$response = Http::withToken($token)
    ->post('https://api.partner.com/v1/visits', $payload);
```

Great for payment gateways, EHR/EVV integrations, SMS providers and other partner APIs.

### ⏰ Check scheduled tasks

**Schedule** shows when each task ran and what it printed.

### 🔐 Debug permissions

**Gates** shows every `Gate::allows()` / policy check and whether it passed.

### 🖨️ Dump without breaking the page

`dump($data)` output appears in the **Dumps** tab instead of breaking your JSON or layout.

---

## Common issues

| Problem | Fix |
|---|---|
| `/telescope` returns 404 | Check `TELESCOPE_ENABLED=true`, that the provider is registered, then run `php artisan optimize:clear` |
| `/telescope` returns 403 outside local | Add your user to the `viewTelescope` gate (Step 6) |
| Dashboard loads but is empty | Check the watchers are enabled and that `Telescope::filter()` isn't filtering everything out |
| `telescope_entries` table doesn't exist | Run `php artisan migrate` |
| Dashboard assets look broken after an update | Run `php artisan telescope:publish` |
| Database growing too fast | Schedule `telescope:prune` (Step 11) and disable noisy watchers |
| App slower with Telescope on | Disable heavy watchers (Views, Cache, Redis, Events, Models) or filter entries (Step 8) |

---

## Should you run Telescope in production?

Telescope is built mainly for **development and staging**. In production it adds a database write for almost every event. If you do run it there:

1. Lock down the `viewTelescope` gate.
2. Store only exceptions, failed jobs, failed requests and monitored tags.
3. Disable noisy watchers.
4. Prune every few hours and use a separate DB connection.
5. Hide all sensitive parameters and headers.

### Telescope vs other tools

| Tool | Best for |
|---|---|
| **Telescope** | Debugging: "what exactly happened in this request or job?" |
| **[Laravel Pulse](https://laravel.com/docs/pulse)** | Production overview: slow endpoints, slow queries, busy users, queue throughput |
| **[Horizon](https://laravel.com/docs/horizon)** | Redis queue dashboard and workers |
| **Sentry / Flare / Nightwatch** | Production error tracking and alerts |

My usual setup: **Telescope locally and on staging**, **Pulse + an error tracker in production**, **Horizon** when using Redis queues.

---

## Quick checklist

- [ ] Install Telescope (`--dev` for local only) and run migrations
- [ ] Register the provider only where you want it
- [ ] Lock down the `viewTelescope` gate and change `TELESCOPE_PATH`
- [ ] Set the slow query threshold and log level
- [ ] Add `Telescope::filter()` for non-local environments
- [ ] Hide sensitive parameters and headers
- [ ] Add custom tags
- [ ] Schedule `telescope:prune`

---

## Wrapping up

Once you're used to clicking from a slow request to its queries, from a failed job to the API call that broke it, or from a bug report to everything one user did, it's hard to go back to `dd()` and `Log::info()`. Telescope is one of the most useful tools in the Laravel ecosystem, and it takes about five minutes to set up.

If this helped, give the repo a ⭐ and feel free to connect:

- 📧 contact.luckyram@gmail.com
- 💼 [LinkedIn](https://www.linkedin.com/in/lucky-ram)
- 🌐 [about.me/luckyram](https://about.me/luckyram)

**References:** [Laravel Telescope docs](https://laravel.com/docs/telescope) · [GitHub: laravel/telescope](https://github.com/laravel/telescope) · [Packagist: laravel/telescope](https://packagist.org/packages/laravel/telescope)
