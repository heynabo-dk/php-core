# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`equites/core`: a small PHP framework library (router, controllers, ActiveRecord, config, observability, request validation). It's published as a Composer package, and apps keep their own code in separate repos. `example/` is a minimal app that wires the framework up. The functional tests run against it. README.md is the detailed per-class API reference.

PSR-4 namespaces: `Core\` → `src/Core`, `Phdantic\` → `src/Phdantic`. The library requires PHP >= 8.3 and only `ext-json`, `ext-pdo` and `ext-simplexml`. Monolog and OpenTelemetry are **dev-only** dependencies, so framework code must not hard-require them. That's why every tracing call in `src/` is guarded by `Tracer::isEnabled()`.

## Commands

Development runs inside Docker. The dev image (`.docker/Dockerfile`, PHP 8.5 because the dev `composer.lock` needs it) runs nginx and php-fpm under supervisord. The compose file uses an external network named `web` (Traefik, `core.localhost`). Create it first if it doesn't exist:

```bash
docker network create web
```

```bash
docker compose up -d --build
```

Task (`Taskfile.yml`) wraps commands that run in the container (repo mounted at `/var/www/app`):

```bash
task composer -- install
```

```bash
task tests
```

```bash
task codecept -- run Unit
```

```bash
task codecept -- run Functional RouterCest:tryRouteWithParams
```

`task tests` runs `codecept run --coverage-html`. Suites are `Unit` (pure assertions) and `Functional` (REST and PhpBrowser against `http://localhost` inside the container). Functional tests need the container's web server running. Coverage of HTTP requests comes from `c3.php`, which `example/api.php` includes. No linter or static analyser is configured.

## How a request flows

1. nginx sends every `/api*` request to `example/api.php`. `example/` is mounted as the web root `/var/www/app/public`. `api.php` loads the vendor autoloader but **includes each example controller and util by hand**, so a new example class must be added to that include list.
2. `Utils\Boot::init()` (example code, not the framework) sets up `Tracer`, a Monolog logger with an OTEL handler that is passed to `Log::setPSRLogger()`, an exception handler, and the job namespace.
3. `Core\Router::run()` walks `REQUEST_URI` segments against `config/structure.xml`:
   - Each matched element becomes the class `<namespace>\Ucfirst(name)`. A node's `child-ns` attribute adds a namespace segment for its descendants. A node's `params="a,b"` attribute means the next unmatched segments are captured as router params.
   - Only classes that exist are instantiated. Missing ones are skipped silently. `serve()` is called on the **last instantiated** controller, so if the final node has no class, its parent's controller handles the request.
   - Every controller gets all router params. A controller "owns" only the params declared on its own node. An unmatched segment throws `External("Not found", 404)`. README still says "Exception 404 -", which is out of date.
   - `preServe()` runs on each controller in path order (use it for auth and short-circuiting). Then `serve()` runs on the final controller.
4. `Core\Controller::serve()` lazily decodes the JSON body and dispatches to a method named after the HTTP verb (`GET`, `POST`, ...). It uses the `_PARAMS` suffix (`GET_PARAMS`) when the controller owns params for this request. A missing method throws `External(..., 501)`. `Controller\Json` sets the JSON header and json-encodes any return value that isn't a string.
5. **`static META()`** on a controller does two jobs:
   - `META()['methods'][VERB]['params']['json']` holds `required`/`optional` maps of `field => int|float|string|bool|array|object`. These are validated with `Phdantic::validateObject` (400 with `errors` on failure). The body is then **filtered** to the declared keys only (`Phdantic::filterObject`).
   - `Core\Router\OpenAPI` builds an OpenAPI 3 document from `Router::getNodesNested()` (the parsed XML tree) plus each class's `META()` descriptions and tags. See `example/Controllers/Openapi.php`.
6. Errors: throw `Core\Exceptions\External($msg, $httpCode)`, and use `setErrors()` to add details. `Log::handleUncaughtExceptionJSON()` turns the exception into `{"msg", "errors"}` with that status code. For codes >= 500 (except 501), it hides the message and returns `Internal Server Error`. Any non-`External` throwable becomes a 500.
7. After the response, `Boot::run()` calls `fastcgi_finish_request()`. Then `Boot::close()` runs `Core\Tasks\Queue::get()->run()`. Jobs added during the request with `Queue::get()->addJob(Job::create('Type', $params))` are created as `new <jobNamespace>\Type($params)` and `->run()` is called on them **in the same PHP process after the client has its response**. Job exceptions are logged and swallowed.

## Other framework pieces

- `Core\Config` loads `APP_ROOT . '/backend/config/config.php'` (array or object). `Core\DB::get()` is a per-request PDO singleton built from the `db` config key.
- `Core\ActiveRecord`: subclasses set `$_table` and `$_typeMap`. Writes go through `set()` for change tracking and then `save()` (insert or update). It uses `Carbon\Carbon`, which is **not** in `composer.json`, so consuming apps must provide it.
- `Core\Vault`: an in-memory key/value singleton that lasts for one request.
- All singletons and statics (Router, Queue, Tracer, Vault, DB) assume one PHP-FPM request per process lifetime.
