# Laravel Reverb Setup on a Live Server
[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

Thanks for visiting my GitHub account!

A complete reference for setting up Laravel Reverb (WebSockets) on a production server behind Nginx, with Varnish, Supervisor, and session-based authentication from Blade views. Everything here is drawn from a real deployment and documents what actually works, including the problems encountered along the way.

---

## Table of Contents

1. Prerequisites and Packages
2. Environment Configuration
3. Laravel Application Setup
4. Channel Definitions
5. Frontend: Echo and Vite
6. Blade Integration
7. Nginx Virtual Host Configuration
8. Supervisor Configuration
9. Queue Worker
10. Deployment Commands
11. Verification and Browser Console Testing
12. Common Issues and Solutions

---

## Overview
|                                                    |
| :------------------------------------------------: |
|                  Preview                  |
| ![preview](/reverb.png) |

## 1. Prerequisites and Packages

### Server Requirements

- PHP 8.2 or higher
- Node.js 18 or higher
- Nginx with SSL
- Supervisor
- A running Laravel 11 application

### Install Reverb

```bash
composer require laravel/reverb
php artisan reverb:install
```

`reverb:install` publishes the config file at `config/reverb.php` and adds the necessary entries to your `.env`. Review `config/reverb.php` after installation — the defaults are fine for most setups.

### Install Frontend Dependencies

```bash
npm install --save-dev laravel-echo pusher-js
```

Reverb uses the Pusher protocol, so `pusher-js` is required even though you are not using Pusher's service.

---

## 2. Environment Configuration

Two separate concerns live in the `.env` file: the server side (where the Reverb process binds) and the client side (where the browser connects). These must be treated separately.

```dotenv
# Broadcasting driver
BROADCAST_CONNECTION=reverb
BROADCAST_DRIVER=reverb

# Reverb application credentials
# These are generated during reverb:install or set manually
REVERB_APP_ID=your_app_id
REVERB_APP_KEY=your_app_key
REVERB_APP_SECRET=your_app_secret

# Server side: the Reverb process binds to 0.0.0.0 on port 9000
# 0.0.0.0 means it listens on all interfaces, including localhost
# Nginx will proxy wss:// traffic from port 443 to this port internally
REVERB_HOST=0.0.0.0
REVERB_PORT=9000
REVERB_SCHEME=http

# Client side: the browser connects to your public domain over HTTPS/WSS
# Nginx proxies /app to 127.0.0.1:9000 internally
VITE_REVERB_APP_KEY=your_app_key
VITE_REVERB_HOST=your-domain.com
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https

# Sanctum stateful domains (required if using Sanctum for API auth)
SANCTUM_STATEFUL_DOMAINS=your-domain.com
SESSION_DOMAIN=your-domain.com
```

The key insight is that `REVERB_HOST=0.0.0.0` and `REVERB_PORT=9000` are internal — the PHP process listens there. The browser never connects directly to port 9000. Instead, the browser connects to `wss://your-domain.com/app/...` on port 443, and Nginx forwards that to `127.0.0.1:9000` internally.

---

## 3. Laravel Application Setup

### File: `bootstrap/app.php`

Two changes are needed in this file.

**Change 1 — Add `channels:` inside `withRouting`**

Add the `channels:` key so Laravel loads your channel definitions:

```php
->withRouting(
    web:      __DIR__ . '/../routes/web.php',
    api:      __DIR__ . '/../routes/api.php',
    commands: __DIR__ . '/../routes/console.php',
    channels: __DIR__ . '/../routes/channels.php', // <-- ADD
    health:   '/up',
)
```

**Change 2 — Add `withBroadcasting` before `withExceptions`**

Add `withBroadcasting` as a chained call. Place it after `withMiddleware` and before `withExceptions`:

```php
->withBroadcasting(
    __DIR__ . '/../routes/channels.php',
    ['middleware' => ['web', 'auth']] // session-based auth for Blade apps
)
```

For API or mobile apps that authenticate via tokens, use `['middleware' => ['auth:sanctum']]` instead.

**What `withBroadcasting` does:** It registers the `POST /broadcasting/auth` route with the middleware you specify. This is the endpoint the browser calls to authorize private and presence channel subscriptions.

**Complete example of the relevant parts:**

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web:      __DIR__ . '/../routes/web.php',
        api:      __DIR__ . '/../routes/api.php',
        commands: __DIR__ . '/../routes/console.php',
        channels: __DIR__ . '/../routes/channels.php', // <-- ADD
        health:   '/up',
        // ... your other route groups
    )
    ->withMiddleware(function (Middleware $middleware) {
        // ... your existing middleware config

        // ADD these two entries to validateCsrfTokens except list:
        $middleware->validateCsrfTokens(except: [
            'broadcasting/auth',
            '/broadcasting/*',
            // ... your other existing exceptions
        ]);
    })

    // ADD this entire block:
    ->withBroadcasting(
        __DIR__ . '/../routes/channels.php',
        ['middleware' => ['web', 'auth']]
    )

    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })->create();
```

**Note on two approaches — use only one:**

Laravel 11 offers two ways to register the broadcasting auth route. Using both simultaneously creates a duplicate route and causes errors.

Option A — `withBroadcasting` in `bootstrap/app.php` (recommended for Laravel 11):
```php
->withBroadcasting(
    __DIR__ . '/../routes/channels.php',
    ['middleware' => ['web', 'auth']]
)
```

Option B — `Broadcast::routes()` at the top of `routes/channels.php`:
```php
Broadcast::routes(['middleware' => ['web', 'auth']]);
```

If you use Option B, do not pass `channels:` to `withRouting` and do not add `withBroadcasting`. Laravel's `ApplicationBuilder` calls `Broadcast::routes()` internally when `channels:` is present, which will create a second registration on top of your own.

---

## 4. Channel Definitions

### File: `routes/channels.php`

Do not call `Broadcast::routes()` here if you are using `withBroadcasting` in `bootstrap/app.php`. Only define channels.

```php
<?php

use App\Models\Conversation;
use Illuminate\Support\Facades\Broadcast;

// Public presence channel — any authenticated user can join
Broadcast::channel('online', function ($user) {
    return [
        'id'     => $user->id,
        'name'   => $user->name,
        'avatar' => $user->avatar,
    ];
});

// Private channel — user can only subscribe to their own channel
Broadcast::channel('user.{userId}', function ($user, $userId) {
    return (int) $user->id === (int) $userId;
});

// Presence channel — user must be an active participant in the conversation
Broadcast::channel('conversation.{conversationId}', function ($user, $conversationId) {
    $conversation = Conversation::where('id', $conversationId)
        ->whereHas('participants', function ($q) use ($user) {
            $q->where('user_id', $user->id)->active();
        })
        ->first();

    if (! $conversation) {
        return false;
    }

    return [
        'id'     => $user->id,
        'name'   => $user->name,
        'avatar' => $user->avatar,
    ];
});
```

**Channel authorization logic:** The callback must return `false` to deny, or a truthy value (array or `true`) to allow. For presence channels, return an array with the user data you want exposed to other channel members. If the query returns null (conversation not found, or user not a participant), returning `false` produces a 403 on the auth endpoint — this is expected and correct behavior.

---

## 5. Frontend: Echo and Vite

### File: `resources/js/echo.js`

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';
import axios from 'axios';

window.Pusher = Pusher;
window.axios = axios;

// Required for session cookie to be sent with auth requests
axios.defaults.withCredentials = true;

const wsHost   = import.meta.env.VITE_REVERB_HOST   ?? window.location.hostname;
const port     = Number(import.meta.env.VITE_REVERB_PORT ?? (location.protocol === 'https:' ? 443 : 80));
const forceTLS = (import.meta.env.VITE_REVERB_SCHEME ?? location.protocol.replace(':', '')) === 'https';

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost,
    wsPort: port,
    wssPort: port,
    forceTLS,
    enabledTransports: ['ws', 'wss'],
});

console.log('Echo initialized');
```

This is the minimal working configuration for a Blade/session-based application. The browser connects to `wss://your-domain.com/app/{key}` on port 443. Nginx proxies this to `http://127.0.0.1:9000`.

The broadcasting auth request goes to `POST /broadcasting/auth`. Since the route is under `web` middleware, it uses the Laravel session cookie for authentication — no token headers required, and no custom `authorizer` is needed.

### File: `vite.config.js`

Make sure `echo.js` is included in your Vite entry points:

```javascript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.js',
                'resources/js/echo.js',
            ],
            refresh: true,
        }),
    ],
});
```

---

## 6. Blade Integration

### Layout file `<head>` — CSRF meta tag (required)

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    @vite(['resources/css/app.css', 'resources/js/app.js', 'resources/js/echo.js'])
</head>
```

### Example: Listening to channels in a Blade view

```blade
@push('script')
<script>
    document.addEventListener('DOMContentLoaded', function () {

        if (!window.Echo) {
            console.error('Laravel Echo not loaded');
            return;
        }

        const userId         = @json(auth()->id());
        const conversationId = @json($conversation->id ?? null);

        if (!userId) {
            console.error('User not authenticated');
            return;
        }

        // Private channel — listens for events sent directly to this user
        window.Echo.private(`user.${userId}`)
            .listen('.ConversationEvent', (e) => {
                console.log('ConversationEvent on user channel:', e);
            })
            .listen('.MessageEvent', (e) => {
                console.log('MessageEvent on user channel:', e);
            });

        // Presence channel — only join if a real conversation was passed to the view
        if (conversationId) {
            window.Echo.join(`conversation.${conversationId}`)
                .here((users) => {
                    console.log('Current users in conversation:', users);
                })
                .joining((user) => {
                    console.log('User joined:', user);
                })
                .leaving((user) => {
                    console.log('User left:', user);
                })
                .listen('.MessageEvent', (e) => {
                    console.log('MessageEvent on conversation channel:', e);
                })
                .listen('.ConversationEvent', (e) => {
                    console.log('ConversationEvent on conversation channel:', e);
                })
                .error((error) => {
                    console.error('Channel error:', error);
                });
        }

    });
</script>
@endpush
```

**Note on the fallback value:** Never use `?? 1` or any hardcoded ID as a fallback for `$conversation->id`. If `$conversation` is null, it means the controller did not pass it to the view. Use `?? null` and guard the channel subscription, otherwise you will attempt to subscribe to a conversation the current user may not belong to, which produces a 403 from the channel authorization callback.

### Passing the conversation from a controller

```php
public function show($id)
{
    $conversation = Conversation::whereHas('participants', function ($q) {
        $q->where('user_id', auth()->id())->active();
    })->findOrFail($id);

    return view('your.view', compact('conversation'));
}
```

---

## 7. Nginx Virtual Host Configuration

This setup assumes a CloudPanel-style configuration where the port 443 server block proxies to port 8080 for PHP, and Reverb listens internally on port 9000.

Only one block needs to be added to your existing vhost. The port 8080 server block (PHP/FPM) requires no changes at all.

### The block to add

Open your vhost file. In the **port 443 server block**, locate the existing `location /` block. Insert the following block **immediately before it**:

```nginx
location /app {
    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;
    proxy_set_header Host              $http_host;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "Upgrade";
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port  443;
    proxy_read_timeout    3600;
    proxy_send_timeout    3600;
    proxy_connect_timeout 3600;
}
```

This block proxies all WebSocket connections (`wss://your-domain.com/app/...`) through to Reverb running on port 9000. The three `proxy_set_header` lines for `Upgrade`, `Connection`, and `proxy_http_version 1.1` are required for the WebSocket upgrade to succeed — remove any one of them and the connection will fail.

### Where exactly to place it

It must come before `location /`. Nginx matches prefix location blocks in the order they appear. If `location /` is first, Nginx routes WebSocket traffic through Varnish instead of Reverb and the upgrade fails with a 500 error.

### Full demo vhost

The line marked `[NEW BLOCK START]` and `[NEW BLOCK END]` is the only addition. Everything else is a standard CloudPanel-generated vhost and should already exist in your file.

```nginx
server {
  listen 80;
  listen [::]:80;
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  {{ssl_certificate_key}}
  {{ssl_certificate}}
  server_name your-domain.com;
  {{root}}
  {{nginx_access_log}}
  {{nginx_error_log}}

  if ($scheme != "https") {
    rewrite ^ https://$host$uri permanent;
  }

  location ~ /.well-known {
    auth_basic off;
    allow all;
  }

  {{settings}}

  # [NEW BLOCK START] — paste this before location /
  location /app {
    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;
    proxy_set_header Host              $http_host;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "Upgrade";
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port  443;
    proxy_read_timeout    3600;
    proxy_send_timeout    3600;
    proxy_connect_timeout 3600;
  }
  # [NEW BLOCK END]

  location / {
    {{varnish_proxy_pass}}
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_hide_header X-Varnish;
    proxy_redirect off;
    proxy_max_temp_file_size 0;
    proxy_connect_timeout      720;
    proxy_send_timeout         720;
    proxy_read_timeout         720;
    proxy_buffer_size          128k;
    proxy_buffers              4 256k;
    proxy_busy_buffers_size    256k;
    proxy_temp_file_write_size 256k;
  }

  location ~* ^.+\.(css|js|jpg|jpeg|gif|png|ico|gz|svg|svgz|ttf|otf|woff|woff2|eot|mp4|ogg|ogv|webm|webp|zip|swf|map)$ {
    add_header Access-Control-Allow-Origin "*";
    expires max;
    access_log off;
  }

  if (-f $request_filename) {
    break;
  }
}

# Port 8080 server block — no changes needed here
server {
  listen 8080;
  listen [::]:8080;
  server_name your-domain.com;
  {{root}}
  try_files $uri $uri/ /index.php?$args;
  index index.php index.html;

  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_intercept_errors on;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    try_files $uri =404;
    fastcgi_read_timeout 3600;
    fastcgi_send_timeout 3600;
    fastcgi_param HTTPS "on";
    fastcgi_param SERVER_PORT 443;
    fastcgi_pass 127.0.0.1:{{php_fpm_port}};
    fastcgi_param PHP_VALUE "{{php_settings}}";
  }

  location ~* ^.+\.(css|js|jpg|jpeg|gif|png|ico|gz|svg|svgz|ttf|otf|woff|woff2|eot|mp4|ogg|ogv|webm|webp|zip|swf|map)$ {
    add_header Access-Control-Allow-Origin "*";
    expires max;
    access_log off;
  }

  if (-f $request_filename) {
    break;
  }
}
```

### After editing

```bash
sudo nginx -t        # test syntax before applying
sudo nginx -s reload
```

Always run `nginx -t` first. It validates the config and reports any syntax errors without touching the live server.

---

## 8. Supervisor Configuration

Reverb must run as a persistent background process. Supervisor manages this.

### File: `/etc/supervisor/conf.d/reverb.conf`

```ini
[program:reverb]
process_name=%(program_name)s
command=php /home/your-user/htdocs/your-domain.com/public/artisan reverb:start --host=0.0.0.0 --port=9000
autostart=true
autorestart=true
user=your-linux-user
redirect_stderr=true
stdout_logfile=/var/log/supervisor/reverb.log
```

Replace `your-user`, `your-domain.com`, and `your-linux-user` with your actual values. The `user` field should match the user that owns your application files.

### Managing Reverb via Supervisor

```bash
# Apply configuration changes
sudo supervisorctl reread
sudo supervisorctl update

# Start / stop / restart
sudo supervisorctl start reverb
sudo supervisorctl stop reverb
sudo supervisorctl restart reverb

# Check status
sudo supervisorctl status reverb

# View live logs
sudo tail -f /var/log/supervisor/reverb.log
```

After any deployment or code change that affects broadcasting, restart Reverb:

```bash
sudo supervisorctl restart reverb
```

Reverb does not reload automatically when you change `.env` or application code.

---

## 9. Queue Worker

Broadcasting events are dispatched through Laravel's queue. If the queue worker is not running, events will be queued but never broadcast.

### File: `/etc/supervisor/conf.d/queue-worker.conf`

```ini
[program:queue-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/your-user/htdocs/your-domain.com/public/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=your-linux-user
numprocs=2
redirect_stderr=true
stdout_logfile=/var/log/supervisor/queue-worker.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start queue-worker:*
sudo supervisorctl status queue-worker:*
```

---

## 10. Deployment Commands

Run these in order after any deployment:

```bash
# Clear all caches
php artisan optimize:clear

# Verify the broadcasting route is registered
php artisan route:list | grep broadcasting

# Rebuild frontend assets
npm run build

# Restart Reverb
sudo supervisorctl restart reverb

# Reload Nginx (only if you changed the vhost config)
sudo nginx -t && sudo nginx -s reload
```

Expected output from `route:list | grep broadcasting`:

```
GET|POST|HEAD   broadcasting/auth   web, auth   BroadcastController@authenticate
```

If you see two broadcasting routes, you have a conflict between `withBroadcasting` in `bootstrap/app.php` and `Broadcast::routes()` in `channels.php`. Remove one of them.

### Verify Reverb is listening on port 9000

```bash
sudo ss -tlnp | grep 9000
```

Expected output:

```
LISTEN 0  511  0.0.0.0:9000  0.0.0.0:*  users:(("php",pid=XXXXX,fd=6))
```

### Verify the WebSocket connection directly (bypassing Nginx)

```bash
curl -i http://127.0.0.1:9000/app/your_app_key \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13"
```

Expected response:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
X-Powered-By: Laravel Reverb
```

If this returns 101, Reverb is working. If it returns 500, check the Supervisor log.

### Verify the WebSocket connection through Nginx

```bash
curl -i https://your-domain.com/app/your_app_key \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13"
```

Expected response is the same: `101 Switching Protocols`. If this returns 500 but the direct curl above returns 101, the problem is in the Nginx location block for `/app`.

---

## 11. Verification in the Browser Console

Open the browser developer tools (F12), go to the Console tab, and load a page that initializes Echo.

### What a successful connection looks like

In the Console:

```
Echo initialized
```

In the Network tab, filter by `WS`. You should see a WebSocket connection to:

```
wss://your-domain.com/app/your_app_key?protocol=7&client=js&version=...
```

Click on it and check the Messages tab. You should see:

```json
{"event":"pusher:connection_established","data":"{\"socket_id\":\"...\",\"activity_timeout\":30}"}
```

This confirms the WebSocket handshake succeeded.

### Verifying channel subscription

After Echo subscribes to a private or presence channel, it sends a request to `POST /broadcasting/auth`. In the Network tab, find this request and confirm:

- Status: 200
- Response body contains `auth` key (for private channels) or `auth` + `channel_data` (for presence channels)

If you see 401, the session cookie is not being sent or the user is not authenticated.

If you see 403, the channel authorization callback returned `false`. This means either the user is not a participant in that channel's resource, or the query has a bug.

### Manual channel test in the console

```javascript
// Check what Echo instance looks like
window.Echo

// Manually listen to a private channel
window.Echo.private('user.1')
  .listen('.YourEvent', (e) => console.log(e))

// Check the socket connection state
window.Echo.connector.pusher.connection.state
// Should return: "connected"

// Check subscribed channels
window.Echo.connector.pusher.channels.channels
```

---

## 12. Common Issues and Solutions

### Issue: 404 on `/broadcasting/auth`

The broadcasting auth route is not registered. Check:

```bash
php artisan route:list | grep broadcasting
```

If the route does not appear, make sure `withBroadcasting` is present in `bootstrap/app.php` or `Broadcast::routes()` is called in `channels.php`. Also clear the route cache with `php artisan route:clear`.

---

### Issue: 401 on `/broadcasting/auth`

The user session is not being recognized. This happens when the auth endpoint uses `api` middleware (which is stateless) but the request comes from a Blade page with a session cookie.

Solution: Use `web` middleware for the broadcasting auth route, not `api` or `auth:sanctum` alone.

```php
->withBroadcasting(
    __DIR__ . '/../routes/channels.php',
    ['middleware' => ['web', 'auth']]
)
```

---

### Issue: 403 on `/broadcasting/auth`

The route is found and the user is authenticated, but the channel callback returned `false`. This is a data/logic issue, not a configuration issue.

Debug steps:

1. Add logging to the channel callback in `channels.php`:

```php
Broadcast::channel('conversation.{conversationId}', function ($user, $conversationId) {
    \Illuminate\Support\Facades\Log::info('channel auth', [
        'user_id'         => $user->id,
        'conversation_id' => $conversationId,
    ]);

    $conversation = Conversation::where('id', $conversationId)
        ->whereHas('participants', function ($q) use ($user) {
            $q->where('user_id', $user->id)->active();
        })->first();

    \Illuminate\Support\Facades\Log::info('result', ['found' => $conversation?->id]);

    if (! $conversation) {
        return false;
    }

    return ['id' => $user->id, 'name' => $user->name];
});
```

2. Trigger the subscription from the browser, then check the log:

```bash
tail -30 storage/logs/laravel.log
```

3. Common causes:
   - The conversation ID in the Blade view does not match a conversation the logged-in user belongs to. Never use a hardcoded fallback ID like `?? 1`.
   - The `active()` scope on the participants query is filtering out the participant record. Check whether `is_active = 1` or `deleted_at` conditions in the scope match the actual data.
   - The user exists but was removed from the conversation (soft-deleted participant).

---

### Issue: Two broadcasting routes registered

```
GET|POST|HEAD   broadcasting/auth   (route 1)
POST            api/broadcasting/auth   (route 2)
```

This happens when both `withBroadcasting` and `Broadcast::routes()` are active simultaneously. Laravel's `ApplicationBuilder` calls `Broadcast::routes()` automatically when you pass `channels:` to `withRouting`, so adding an explicit call creates a duplicate.

Solution: Use `withBroadcasting` in `bootstrap/app.php` exclusively. Remove `Broadcast::routes()` from `channels.php`.

---

### Issue: WebSocket connects but disconnects immediately (HTTP 500)

Reverb crashed on connection. Check the Supervisor log:

```bash
sudo tail -50 /var/log/supervisor/reverb.log
```

Common causes:
- Incorrect `REVERB_APP_KEY` or `REVERB_APP_SECRET` in `.env`
- PHP memory or timeout limits
- Port 9000 conflict with another process (check with `sudo ss -tlnp | grep 9000`)

---

### Issue: WebSocket returns 500 through Nginx but 101 directly

Nginx is not forwarding the WebSocket upgrade headers correctly.

Required headers in the `/app` location block:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection "Upgrade";
```

All three are required. Without `proxy_http_version 1.1`, the upgrade will fail because HTTP/1.0 does not support connection upgrades.

---

### Issue: WebSocket connects but events are never received

The queue worker is not running. Events are dispatched to the queue, not directly to Reverb.

Check:

```bash
sudo supervisorctl status queue-worker:*
```

If it is not running:

```bash
sudo supervisorctl start queue-worker:*
```

Also confirm the queue connection in `.env`:

```dotenv
QUEUE_CONNECTION=database
```

And that queue jobs are present:

```bash
php artisan queue:monitor
# or check the jobs table directly
php artisan tinker
>>> DB::table('jobs')->count()
```

---

### Issue: Connection works on HTTP but not HTTPS

The `forceTLS` flag in Echo must match the scheme the browser is on.

In `echo.js`:

```javascript
const forceTLS = (import.meta.env.VITE_REVERB_SCHEME ?? location.protocol.replace(':', '')) === 'https';
```

And in `.env`:

```dotenv
VITE_REVERB_SCHEME=https
VITE_REVERB_PORT=443
```

Also confirm the Nginx location block for `/app` passes `X-Forwarded-Proto`:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port  443;
```

---

### Routine restart after deployment

```bash
php artisan optimize:clear
npm run build
sudo supervisorctl restart reverb
sudo supervisorctl restart queue-worker:*
```

These four commands cover the vast majority of deployment scenarios. Clear caches first so Reverb and the queue worker pick up the new configuration.
---

## Author

Developed and maintained by [MD. Rahatul Rabbi](https://github.com/learnwithfair).

---

## Follow

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='30'>](https://github.com/learnwithfair)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='30'>](https://www.facebook.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='30'>](https://www.instagram.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='30'>](https://www.youtube.com/@learnwithfair)

---

<!-- MARKDOWN LINKS -->
[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair
