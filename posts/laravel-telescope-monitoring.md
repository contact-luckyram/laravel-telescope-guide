---
title: "Laravel Telescope: How to Monitor Everything in Your Laravel App"
author: Lucky Ram
date: 2026-10-11
tags: [laravel, php, telescope, debugging, monitoring, performance]
---

# Laravel Telescope: How to Monitor Everything in Your Laravel App

*By [Lucky Ram](https://about.me/luckyram) · Laravel & Node.js developer*

When something goes wrong in a Laravel app, the hardest part usually isn't fixing it. It's **seeing** it: which query was slow, which job failed, which email never went out, which API call to a third party timed out.

[Laravel Telescope](https://laravel.com/docs/telescope) gives you that visibility. It's an official Laravel package that records almost everything your application does (requests, queries, jobs, mail, cache, logs, exceptions and more) and shows it in a clean dashboard at `/telescope`.

In this post I'll walk through how I set up and use Telescope to monitor "everything", plus the configuration I use on real projects in healthcare and finance, where performance, security and auditability really matter.

---

## Table of contents

1. [What Telescope records](#1-what-telescope-records)
2. [Installation](#2-installation)
3. [Local-only installation (recommended for most teams)](#3-local-only-installation)
4. [Securing the dashboard](#4-securing-the-dashboard)
5. [Configuring watchers](#5-configuring-watchers)
6. [Real-world debugging workflows](#6-real-world-debugging-workflows)
7. [Filtering what gets stored](#7-filtering-what-gets-stored)
8. [Tagging entries for faster search](#8-tagging-entries)
9. [Hiding sensitive data](#9-hiding-sensitive-data)
10. [Pruning so the database doesn't explode](#10-pruning)
11. [Telescope in production: should you?](#11-telescope-in-production)
12. [Telescope vs Pulse vs external APM](#12-telescope-vs-pulse-vs-external-apm)
13. [Checklist](#13-checklist)

---

## 1. What Telescope records

Telescope works through **watchers**. Each watcher listens for one kind of activity and stores it as an "entry":

| Watcher | What you see |
|---|---|
| **Requests** | Every HTTP request: URL, method, status, headers, payload, response, duration, memory |
| **Queries** | Every SQL query, its bindings, duration, and where in your code it ran. Slow queries are flagged |
| **Models** | Model `created`, `updated`, `deleted` events (and optionally hydration counts) |
| **Exceptions** | Reportable exceptions with full stack trace and the request that caused them |
| **Logs** | Everything written through `Log::` |
| **Jobs** | Queued jobs, their payload, status (pending/processed/failed) and exceptions |
| **Batches** | Job batches and their progress |
| **Mail** | Every mailable, with a rendered preview of the email |
| **Notifications** | Notifications and the channels they went through |
| **Events** | Dispatched events and their listeners |
| **Cache** | Hits, misses, writes and forgets |
| **Redis** | Redis commands |
| **HTTP Client** | Outgoing requests made with Laravel's `Http::` client: URL, payload, response |
| **Scheduled tasks** | Runs of your scheduled commands and their output |
| **Commands** | Artisan commands, arguments, options and exit codes |
| **Gates** | Authorization checks and whether they passed or failed |
| **Views** | Rendered views and the data passed to them |
| **Dumps** | Output of `dump()` calls, shown in the dashboard instead of the browser |

Together, these give you an end-to-end picture of a single request: the request comes in, it runs 14 queries, fires 2 events, dispatches a job, and that job calls an external API and sends an email. You can click through all of it.

---

## 2. Installation

```bash
composer require laravel/telescope
php artisan telescope:install
php artisan migrate
```

`telescope:install` publishes:

- `config/telescope.php`: the main config file
- `app/Providers/TelescopeServiceProvider.php`: where you set authorization, filtering and tagging
- the dashboard assets

Run your app and open `http://your-app.test/telescope`.

---

## 3. Local-only installation

Most teams only want Telescope in development. Install it as a dev dependency:

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

Then stop Laravel from auto-registering it in every environment. Remove `TelescopeServiceProvider` from `bootstrap/providers.php` and register it manually in `AppServiceProvider`:

```php
// app/Providers/AppServiceProvider.php

public function register(): void
{
    if ($this->app->environment('local') && class_exists(\Laravel\Telescope\TelescopeServiceProvider::class)) {
        $this->app->register(\Laravel\Telescope\TelescopeServiceProvider::class);
        $this->app->register(TelescopeServiceProvider::class);
    }
}
```

And exclude it from package discovery in `composer.json`:

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "laravel/telescope"
        ]
    }
}
```

You can also switch it on and off with an environment variable:

```dotenv
TELESCOPE_ENABLED=true
```

---

## 4. Securing the dashboard

In the `local` environment, anyone can open `/telescope`. In any other environment, access is controlled by the `viewTelescope` gate in `TelescopeServiceProvider`:

```php
// app/Providers/TelescopeServiceProvider.php

protected function gate(): void
{
    Gate::define('viewTelescope', function ($user) {
        return in_array($user->email, [
            'admin@your-company.com',
        ]);
    });
}
```

Better still, check a role or permission rather than hard-coding emails:

```php
Gate::define('viewTelescope', fn ($user) => $user->hasRole('developer'));
```

You can also move the dashboard off the default path, so it's not sitting at an obvious URL:

```dotenv
TELESCOPE_PATH=internal/telescope
```

> ⚠️ Telescope shows request payloads, SQL bindings and emails. **Never** leave it open on a public server.

---

## 5. Configuring watchers

All watchers live in `config/telescope.php`. Each one can be toggled with an env variable, and several have useful options. Here's the setup I start with:

```php
'watchers' => [
    Watchers\CacheWatcher::class => [
        'enabled' => env('TELESCOPE_CACHE_WATCHER', true),
        'hidden' => [],
    ],

    Watchers\CommandWatcher::class => [
        'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
        'ignore' => ['schedule:run', 'queue:work'],
    ],

    Watchers\ExceptionWatcher::class => env('TELESCOPE_EXCEPTION_WATCHER', true),

    Watchers\JobWatcher::class => env('TELESCOPE_JOB_WATCHER', true),

    Watchers\LogWatcher::class => [
        'enabled' => env('TELESCOPE_LOG_WATCHER', true),
        'level' => 'error', // only record error and above
    ],

    Watchers\MailWatcher::class => env('TELESCOPE_MAIL_WATCHER', true),

    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.*'],
        'hydrations' => true, // shows how many models each request loads
    ],

    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'ignore_packages' => true,
        'ignore_paths' => [],
        'slow' => 100, // ms: anything slower gets tagged "slow"
    ],

    Watchers\RequestWatcher::class => [
        'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
        'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64), // KB
        'ignore_http_methods' => [],
        'ignore_status_codes' => [],
    ],

    Watchers\ClientRequestWatcher::class => env('TELESCOPE_CLIENT_REQUEST_WATCHER', true),
    Watchers\ScheduleWatcher::class => env('TELESCOPE_SCHEDULE_WATCHER', true),
    Watchers\NotificationWatcher::class => env('TELESCOPE_NOTIFICATION_WATCHER', true),
    Watchers\EventWatcher::class => env('TELESCOPE_EVENT_WATCHER', true),
    Watchers\GateWatcher::class => env('TELESCOPE_GATE_WATCHER', true),
    Watchers\RedisWatcher::class => env('TELESCOPE_REDIS_WATCHER', true),
    Watchers\ViewWatcher::class => env('TELESCOPE_VIEW_WATCHER', true),
    Watchers\DumpWatcher::class => env('TELESCOPE_DUMP_WATCHER', true),
],
```

You can also tell Telescope to ignore noisy paths entirely:

```php
'ignore_paths' => [
    'livewire*',
    'nova-api*',
    'pulse*',
    'horizon*',
],
```

---

## 6. Real-world debugging workflows

This is where Telescope pays for itself. Here are the problems I use it for most.

### 🐢 Finding N+1 queries

Open a slow request in the **Requests** tab and switch to its **Queries** section. If you see the same query repeated 50 times with different IDs:

```sql
select * from `patients` where `patients`.`id` = 1 limit 1
select * from `patients` where `patients`.`id` = 2 limit 1
select * from `patients` where `patients`.`id` = 3 limit 1
...
```

...you have an N+1 problem. Fix it with eager loading:

```php
// Before
$visits = Visit::all();

// After
$visits = Visit::with('patient')->get();
```

The **Models** tab with `hydrations` enabled also shows when a single page loads thousands of models, a common hidden performance killer.

To stop N+1 queries from slipping back in, combine Telescope with Laravel's lazy-loading guard:

```php
// AppServiceProvider::boot()
Model::preventLazyLoading(! app()->isProduction());
```

### 🐌 Spotting slow queries

Any query slower than the `slow` threshold (100 ms above) is tagged **slow**. Filter the Queries tab by that tag to get a to-do list of indexes to add or queries to rewrite. Telescope also shows the file and line that ran the query, so you know exactly where to look.

### 💥 Tracing an exception back to its cause

In the **Exceptions** tab, open an exception and you'll see the stack trace, the number of times it has happened, and a link to the **request or job** that triggered it. From there you can see the full payload, the authenticated user, and every query that ran before it failed.

### 📬 Checking emails without sending them

The **Mail** tab renders every email your app sends, with recipients, subject and HTML preview. No more sending test emails to yourself, and you can check that the right template went to the right person.

### ⚙️ Debugging queued jobs

The **Jobs** tab shows each job's payload, how many attempts it took, and the exception if it failed. Because Telescope links a job to the request that dispatched it, you can follow the whole flow: request → job → outgoing API call → email.

### 🌐 Watching third-party API calls

The **HTTP Client** tab logs every outgoing request made with `Http::`. This is gold for integrations (payment gateways, EHR/EVV aggregators, SMS providers):

```php
$response = Http::withToken($token)
    ->post('https://api.partner.com/v1/visits', $payload);
```

You'll see the exact URL, headers, request body, response body and status code, which is perfect when the partner says "we never received it".

### ⏰ Verifying scheduled tasks

The **Schedule** tab shows when each scheduled command ran and what it printed. If a nightly sync "didn't run", check here first.

### 🔐 Debugging authorization

The **Gates** tab records every `Gate::allows()` / policy check and its result. When a user says "I can't see that button", you can see exactly which ability failed.

### 🖨️ Dumping without breaking the page

Keep the **Dumps** tab open and any `dump($variable)` call shows up there instead of breaking your API JSON or page layout.

---

## 7. Filtering what gets stored

Recording everything is great locally, but outside local you usually only want the important stuff. Use `Telescope::filter()` in `TelescopeServiceProvider::register()`:

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

This keeps local fully transparent while staging only stores exceptions, failures, slow queries, scheduled tasks and anything you're actively monitoring.

---

## 8. Tagging entries

Telescope automatically tags entries with things like the authenticated user ID and Eloquent model names (e.g. `App\Models\Invoice:42`). You can add your own tags:

```php
Telescope::tag(function (IncomingEntry $entry) {
    $tags = [];

    if ($entry->type === 'request') {
        $tags[] = 'status:' . $entry->content['response_status'];
    }

    if ($tenant = tenant()?->id) {
        $tags[] = 'tenant:' . $tenant;
    }

    return $tags;
});
```

Then in the dashboard you can search `tenant:12` or `status:500` and see only the relevant entries.

### Monitoring a tag

On the **Monitoring** page you can "watch" a tag, for example `Auth:17` for one user who reports a bug. Combined with `hasMonitoredTag()` in the filter above, Telescope will record **everything** for that user even in an environment where you normally store only errors. It's a really practical way to debug one customer's issue without turning on full logging for everyone.

---

## 9. Hiding sensitive data

If you work with patient records, card data or personal information, this part matters most. Telescope stores request payloads and headers, so hide anything sensitive:

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

A few more rules I follow on healthcare and finance projects:

- Use **fake or anonymized data** in any environment where Telescope records everything.
- Don't enable the **Views** watcher where views receive personal data you don't want stored.
- Treat the `telescope_entries` table like a log file that contains sensitive data: restrict access and prune it often.

---

## 10. Pruning

Telescope writes a lot of rows. Without pruning, `telescope_entries` will grow to gigabytes. Schedule the prune command:

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

By default it removes entries older than 24 hours; `--hours=48` keeps two days. Keep entries for failed jobs or exceptions longer with `--keep-exceptions`:

```bash
php artisan telescope:prune --hours=48 --keep-exceptions
```

To wipe everything (handy after a big load test):

```bash
php artisan telescope:clear
```

You can also point Telescope at a **separate database connection** so its writes never compete with your main tables. Define a `telescope` connection in `config/database.php`, then update `config/telescope.php`:

```php
'storage' => [
    'database' => [
        'connection' => env('TELESCOPE_DB_CONNECTION', 'telescope'),
        'chunk' => 1000,
    ],
],
```

---

## 11. Telescope in production

Telescope is designed mainly as a **development and staging** tool. Running it fully in production adds a database write for nearly every event, which costs performance and storage.

If you do run it in production, do it carefully:

1. Lock down the `viewTelescope` gate.
2. Use `Telescope::filter()` to store **only** exceptions, failed jobs, failed requests and monitored tags.
3. Disable noisy watchers (`Views`, `Cache`, `Redis`, `Events`, `Models`).
4. Prune aggressively (every few hours) and use a separate DB connection.
5. Hide every sensitive parameter and header.

For ongoing production monitoring, combine it with the tools below.

---

## 12. Telescope vs Pulse vs external APM

| Tool | Best for | Granularity |
|---|---|---|
| **Telescope** | Debugging: "what exactly happened in this request/job?" | Every individual entry, full detail |
| **[Laravel Pulse](https://laravel.com/docs/pulse)** | Production overview: slow endpoints, slow queries, busiest users, queue throughput | Aggregated metrics, light overhead |
| **Horizon** | Redis queue dashboard and supervisor management | Queues and jobs |
| **Sentry / Flare / Nightwatch / New Relic** | Production error tracking, alerting and APM | Errors + traces with alerts |

My usual stack: **Telescope locally and on staging**, **Pulse + an error tracker in production**, and **Horizon** when the app uses Redis queues.

---

## 13. Checklist

- [ ] Install Telescope (`--dev` if local only) and run migrations
- [ ] Register the provider only in the environments you want
- [ ] Lock down the `viewTelescope` gate and change `TELESCOPE_PATH`
- [ ] Set the slow query threshold and log level
- [ ] Add `Telescope::filter()` for non-local environments
- [ ] Add custom tags (tenant, status, user role)
- [ ] Hide sensitive request parameters and headers
- [ ] Schedule `telescope:prune`
- [ ] Use Pulse + an error tracker for production monitoring

---

## Wrapping up

Telescope turns your Laravel app from a black box into something you can actually see inside. Once you're used to clicking from a slow request to its queries, from a failed job to the API call that broke it, or from a bug report to everything one user did, it's hard to go back to `Log::info()` and `dd()`.

If you need help setting up monitoring, fixing performance issues, or building a Laravel/Filament app, especially in healthcare or finance, feel free to reach out:

- 📧 contact.luckyram@gmail.com
- 💼 [LinkedIn](https://www.linkedin.com/in/lucky-ram)
- 🌐 [about.me/luckyram](https://about.me/luckyram)

*Happy debugging!* 🔭
