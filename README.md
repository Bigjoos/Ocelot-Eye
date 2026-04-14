# Ocelot-Eye

> **Real-time log monitor for U-232 V5.5 “Chaos Edition”**  
> SSE Streaming · Multi-Log Parsing · Chart.js · Argon2id Auth

[![PHP](https://img.shields.io/badge/PHP-8.4-777BB4.svg)](https://www.php.net/)
[![Chart.js](https://img.shields.io/badge/Chart.js-4.x-FF6384.svg)](https://www.chartjs.org/)
[![SSE](https://img.shields.io/badge/Transport-Server--Sent%20Events-brightgreen.svg)]()
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Status: Production](https://img.shields.io/badge/Status-Production-brightgreen.svg)]()

-----

## What is this?

**Ocelot-Eye** is a real-time log monitoring dashboard for U-232 V5.5 tracker infrastructure. It streams live log data from multiple sources simultaneously — Ocelot announce daemon, PHP application logs, nginx access/error logs, MySQL slow query log — parses them in real time, and presents them through a live browser interface with Chart.js visualisations.

Deployed at **[logs.u-232.com](https://logs.u-232.com)**.

No WebSocket server. No Node.js. No extra infrastructure. Pure PHP streaming via **Server-Sent Events** — the browser connects, the log feed opens, and the data flows.

-----

## Inspiration

Ocelot-Eye is inspired by **glTail** — the legendary real-time log visualiser that turned server logs into a flowing, animated particle system. Where glTail was a visual spectacle, Ocelot-Eye is built for operational reality: actionable data, tracker-specific parsing, and persistent auth you can leave running in a staff tab.

-----

## Features

### 📡 Server-Sent Events Streaming

- Persistent HTTP connection — no polling, no WebSocket server needed
- Browser reconnects automatically on disconnect
- Backpressure-safe: log lines are buffered and flushed efficiently
- Works through nginx reverse proxy without special configuration

### 📋 Multi-Log Parsing

Ocelot-Eye understands the log formats that matter for U-232:

|Log Source          |What’s Parsed                                                     |
|--------------------|------------------------------------------------------------------|
|**Ocelot daemon**   |Announces, scrapes, peer events, cheat flags, interval assignments|
|**PHP application** |Errors, warnings, slow operations, auth events                    |
|**nginx access**    |Request rates, status codes, suspicious patterns                  |
|**nginx error**     |Upstream failures, connection errors                              |
|**MySQL slow query**|Query time, affected tables, calling context                      |

Each source has its own parser — structured events, not raw text dumps.

### 📊 Chart.js Visualisations

Live-updating charts built with Chart.js 4:

- **Announce rate** — announces per minute, rolling window
- **Active peers** — seeder/leecher ratio over time
- **Error rate** — PHP errors and Ocelot warnings per minute
- **HTTP status distribution** — 200/301/4xx/5xx breakdown
- **Cheat detection events** — flags fired by the Ocelot anti-cheat engine
- **Slow query frequency** — MySQL slow log events over time

Charts update as new log lines arrive — no page refresh, no polling interval.

### 🔐 Argon2id Authentication

Ocelot-Eye is a staff tool. It is not publicly accessible:

- Argon2id password hashing — consistent with U-232’s auth standard
- Session-based auth with secure, HTTPOnly, SameSite=Strict cookies
- Failed login rate limiting
- No shared credentials — individual staff accounts

### 🫧 glTail-Inspired Bubble Visualiser

Beyond the charts, a real-time bubble visualiser renders log events as animated particles — each log source is a different colour, event frequency drives bubble density, and cheat detection events create distinctive visual spikes. It’s operational, but it’s also a spectacle.

-----

## Architecture

```
Log Files (disk)
      │
      ▼
  PHP Stream Reader
  (tail -f equivalent, inotify-aware)
      │
      ▼
  Per-Source Log Parsers
  (Ocelot / nginx / PHP / MySQL)
      │
      ▼
  SSE Endpoint (stream.php)
  Content-Type: text/event-stream
      │
      ▼  (persistent HTTP connection)
  Browser
      ├── Chart.js live charts
      ├── Structured event feed
      └── Bubble visualiser
```

-----

## Requirements

- PHP **8.4+** with `pcntl` extension (for clean stream shutdown)
- nginx (recommended reverse proxy)
- Read access to Ocelot, nginx, PHP, and MySQL log files
- U-232 V5.5 staff account for auth

No Node.js. No Redis. No message queue. Log files → browser.

-----

## Installation

```bash
git clone https://github.com/Bigjoos/Ocelot-Eye.git
cd Ocelot-Eye
cp config.example.php config.php
```

Edit `config.php`:

```php
// Log file paths
define('LOG_OCELOT',   '/var/log/ocelot/ocelot.log');
define('LOG_NGINX_ACCESS', '/var/log/nginx/access.log');
define('LOG_NGINX_ERROR',  '/var/log/nginx/error.log');
define('LOG_PHP',      '/var/log/php8.4-fpm.log');
define('LOG_MYSQL',    '/var/log/mysql/slow.log');

// Auth
define('ADMIN_USER',   'staff');
define('ADMIN_HASH',   password_hash('yourpassword', PASSWORD_ARGON2ID));

// Stream settings
define('SSE_BUFFER_SIZE', 50);    // lines per flush
define('SSE_FLUSH_INTERVAL', 500); // milliseconds
```

-----

## nginx Configuration

```nginx
server {
    server_name logs.u-232.com;

    location / {
        proxy_pass http://127.0.0.1:9001;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;           # critical for SSE
        proxy_cache off;
        proxy_read_timeout 3600s;      # keep SSE connection alive
        chunked_transfer_encoding on;
    }
}
```

> ⚠️ `proxy_buffering off` is non-negotiable for SSE. nginx will buffer the stream and the browser receives nothing until the connection closes. Always disable it for the SSE endpoint.

-----

## Deployment

Ocelot-Eye runs as a PHP-FPM application behind nginx. No daemon management needed beyond PHP-FPM itself.

```bash
# Verify log file read permissions
sudo usermod -aG adm www-data  # or adjust as needed for your log paths

# Test SSE endpoint directly
curl -N https://logs.u-232.com/stream.php \
     -H "Cookie: session=your_session_token"
```

Live at **[logs.u-232.com](https://logs.u-232.com)**.

-----

## Security Notes

- Ocelot-Eye exposes real-time infrastructure data — **never** make it publicly accessible
- Argon2id hashes mean brute-forcing credentials is computationally expensive
- Consider IP allowlisting at nginx level for additional hardening
- Log file access is read-only — Ocelot-Eye cannot modify anything it monitors

-----

## Integration with U-232 V5.5

Ocelot-Eye is purpose-built to understand U-232 and Ocelot-U232 log formats. It knows what a SwarmPromoter event looks like. It knows the cheat detection flag format. It knows what Ocelot’s dynamic interval assignment entries mean.

It is not a generic log viewer. It is a tracker operations tool.

-----

## License

GPL v3 — consistent with the U-232 V5.5 ecosystem.

-----

*Part of the U-232 V5.5 “Chaos Edition” open-source tracker ecosystem.*  
*Inspired by glTail. Built for production. Running at [logs.u-232.com](https://logs.u-232.com).*
