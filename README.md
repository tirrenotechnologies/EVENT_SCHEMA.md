# tirreno event schema

This schema holds everything [tirreno](https://www.tirreno.com/) learns from the events of entities (or users) in your application. Your app reports what each entity does, such as logins, page views, searches and edits, and each report becomes an event. tirreno files the details of every event into separate tables: who did it, from which IP address, on which page, on which device and in which session.

Details that many entities share, such as an IP address, a page or a browser, are stored only once and reused, each with counters showing how often it is seen and by how many accounts. Details that belong to one entity, such as its devices, email addresses and phone numbers, are stored per account. From all this, tirreno calculates a risk score for each account.

## The event record and how the tables fit together

Below are all 17 columns of the event record (`event`) with their PostgreSQL types. Each letter leads to the explanation of that column; an arrow shows the table the column links to.

```
   Column       Type          Meaning
a) id           bigint        Unique event ID, filled in automatically
b) key          smallint      API key the event was sent with -> API keys [table: dshb_api]
c) account      bigint        Who did it -> Accounts [table: event_account]
d) ip           bigint        IP address it came from -> IP addresses [table: event_ip]
e) url          bigint        Page it happened on -> Pages [table: event_url]
f) device       bigint        Browser and language it came from -> Devices [table: event_device]
g) time         timestamp(3)  When it happened, in UTC
h) query        bigint        Query string of the URL -> Query strings [table: event_url_query] *
i) traceid      varchar(36)   Trace ID of the sensor request, for debugging *
j) referer      bigint        Page the visitor came from -> Referrers [table: event_referer] *
k) type         smallint      What kind of event it was -> Event types [table: event_type]
l) email        integer       Email address sent with the event -> Email addresses [table: event_email] *
m) phone        integer       Phone number sent with the event -> Phone numbers [table: event_phone] *
n) http_code    smallint      HTTP status code *
o) http_method  smallint      How the page was requested -> HTTP methods [table: event_http_method] *
p) session_id   bigint        Visit it belongs to -> Sessions [table: event_session]
q) payload      bigint        Extra structured data, such as a search -> Event payloads [table: event_payload] *

* optional: the column can be NULL
```

### Example event

A real row from a test install: a `page_search` event with every optional column filled. Columns that link to another table store that row's ID; the last column shows the linked value.

```
   Column       Value                                 Refers to
a) id           35
b) key          1                                     API key (tracking ID not shown)
c) account      34                                    userid user-42
d) ip           1                                     203.0.113.10
e) url          3                                     /search
f) device       34                                    Chrome 128.0, Windows, en-US,en;q=0.9
g) time         2026-10-09 09:50:15.000
h) query        35                                    ?q=invoices
i) traceid      e0875f5f-c4e2-4a8f-b927-a7b8a8d4ddf7
j) referer      1                                     https://www.example.org/start
k) type         4                                     page_search
l) email        34                                    jane.doe@example.com
m) phone        35                                    +12025550142
n) http_code    200
o) http_method  1                                     GET
p) session_id   34
q) payload      35                                    {"field_id":"q","value":"invoices","field_name":null}
```

## The event columns explained

### a) `id` (bigint)

Identifies the event and is filled in automatically when the event is stored.

### b) `key` (smallint) -> API keys [table: `dshb_api`]

API keys, also called tracking IDs. Each belongs to the console operator who created it and carries its own settings: the review and auto-blacklist thresholds, data retention and the enrichment token.

Columns:

- `id` (integer): API key ID. The `key` column of the other tables holds this number.
- `key` (text): Tracking ID that trackers send in the `Api-Key` header.
- `quote` (integer): Not in use.
- `creator` (bigint): Operator who created and owns the key. Points to Operators [table: `dshb_operators`].
- `created_at` (timestamp): Creation time.
- `skip_enriching_attributes` (jsonb): JSON array of attribute types never sent for enrichment (`ip`, `email`, `domain`, `phone`).
- `retention_policy` (smallint): Data retention period in weeks (Settings, Data retention).
- `skip_blacklist_sync` (boolean): Not in use.
- `token` (varchar, optional): Enrichment API token; enrichment is skipped while it is `NULL`.
- `last_call_reached` (boolean, optional): Whether the latest enrichment call reached the enrichment service.
- `blacklist_threshold` (integer): Auto-blacklisting: accounts scoring at or below this value are blacklisted; `-1` = off.
- `review_queue_threshold` (integer): Accounts scoring at or below this value enter the review queue.

### c) `account` (bigint) -> Accounts [table: `event_account`]

One row per entity (or user) of your application. tirreno creates it the first time a new user ID arrives and updates it with every event. Besides names and activity counters, it holds the trust score (99 looks clean, 0 is high risk) and the operator's decision: blacklisted, whitelisted or undecided. Devices, email addresses, phone numbers, sessions, scores, the review-queue entry and the field change history all belong to an account and are deleted with it.

Columns:

- `id` (bigint): Account ID.
- `userid` (varchar(100)): The application's user ID, sent as `userName` (max 100 characters).
- `created` (timestamp): `userCreated` when sent, else when the first event was received.
- `key` (integer, optional): API key this row belongs to.
- `lastip` (inet, optional): IP address of the latest event.
- `lastseen` (timestamp): Time of the latest event.
- `fullname` (text, optional): Full name (`fullName`).
- `is_important` (boolean): Whether the account is on the watchlist.
- `firstname` (varchar(100), optional): First name (`firstName`).
- `middlename` (varchar(100), optional): Middle name.
- `lastname` (varchar(100), optional): Last name (`lastName`).
- `total_visit` (integer): Number of events.
- `total_country` (integer): Number of distinct countries.
- `total_ip` (integer): Number of distinct IP addresses.
- `total_device` (integer): Number of distinct devices.
- `score_updated_at` (timestamp, optional): When the score was last computed; `NULL` until the account is first scored.
- `score` (integer, optional): Trust score from 0 to 99: 99 is clean, lower is riskier.
- `score_details` (jsonb, optional): JSON array of matched rules with their weights (`uid`, `score`).
- `lastemail` (integer, optional): Latest email address. Points to Email addresses [table: `event_email`], not checked by the database.
- `lastphone` (integer, optional): Latest phone number. Points to Phone numbers [table: `event_phone`], not checked by the database.
- `total_shared_ip` (integer): Number of the account's IP addresses also used by other accounts.
- `total_shared_phone` (integer): Number of the account's phone numbers also used by other accounts.
- `reviewed` (boolean): Whether an operator has reviewed the account.
- `fraud` (boolean, optional): Review decision: `true` = blacklisted, `false` = whitelisted, `NULL` = none.
- `latest_decision` (timestamp, optional): Date of the latest review decision.
- `score_recalculate` (boolean): Requests a score recalculation; cleared after scoring.
- `session_id` (bigint, optional): Current session. A new one starts after 30 minutes without activity or once a session has run 4 hours. Points to Sessions [table: `event_session`], not checked by the database.
- `updated` (timestamp): When the counters were last recalculated.
- `added_to_review` (timestamp, optional): When the account entered the review queue.

### d) `ip` (bigint) -> IP addresses [table: `event_ip`]

Every IP address seen, stored once per API key and shared by all accounts that used it, with counters. With enrichment turned on, it also shows the country and internet provider, and whether the address is a VPN, Tor, a datacenter or on a spam list.

Columns:

- `id` (bigint): IP ID.
- `ip` (inet): IP address.
- `key` (smallint): API key this row belongs to.
- `country` (smallint): Country; `0` when unknown. Points to Country list [table: `countries`], not checked by the database.
- `cidr` (text, optional): Network block the address belongs to (from enrichment).
- `data_center` (boolean, optional): Whether the address belongs to a hosting or datacenter network (from enrichment).
- `tor` (boolean, optional): Whether the address is assigned to the Tor network (from enrichment).
- `vpn` (boolean, optional): Whether the address belongs to a VPN (from enrichment).
- `checked` (boolean): Whether enrichment has been done for this value.
- `relay` (boolean, optional): Whether the address belongs to iCloud Private Relay (from enrichment).
- `lastseen` (timestamp): Latest event from this address.
- `created` (timestamp): First seen.
- `updated` (timestamp): When the counters were last recalculated.
- `lastcheck` (timestamp, optional): Not used in v0.10.1.
- `total_visit` (integer): Number of events from this address.
- `blocklist` (boolean, optional): Whether the address is on a spam list (from enrichment).
- `isp` (bigint, optional): Internet service provider. Points to Internet providers [table: `event_isp`].
- `shared` (smallint): Number of accounts that used this address.
- `domains_count` (json, optional): JSON array of domains hosted on the address (from enrichment, shown as "Domains hosting").
- `fraud_detected` (boolean): Whether the address was already blacklisted when the event arrived.
- `hash` (text, optional): SHA-256 of the address.
- `alert_list` (boolean, optional): Whether the address is on the global alert list (from enrichment).
- `starlink` (boolean, optional): Whether the address belongs to the Starlink network (from enrichment).

### e) `url` (bigint) -> Pages [table: `event_url`]

Each page (URL path) of your application that appeared in events, with its title, latest HTTP status and counters.

Columns:

- `id` (bigint): URL ID.
- `key` (smallint): API key this row belongs to.
- `url` (text, optional): URL path.
- `lastseen` (timestamp): Latest event on this URL.
- `created` (timestamp): First seen.
- `updated` (timestamp): When the counters were last recalculated.
- `title` (text, optional): Page title (`pageTitle`).
- `total_visit` (integer): Number of events.
- `total_ip` (integer): Number of distinct IP addresses.
- `total_device` (integer): Number of distinct devices.
- `total_account` (integer): Number of distinct accounts.
- `total_country` (integer): Number of distinct countries.
- `http_code` (smallint, optional): HTTP status code of the latest event.
- `total_edit` (integer): Number of edit events.

### f) `device` (bigint) -> Devices [table: `event_device`]

A device is one browser (user agent) together with its language setting, as used by one account. The user agent itself is stored once in User agents and shared.

Columns:

- `id` (bigint): Device ID.
- `account_id` (bigint): Owning account. Points to Accounts [table: `event_account`].
- `key` (smallint): API key this row belongs to.
- `created` (timestamp): First seen.
- `lastseen` (timestamp): Latest event from this device.
- `updated` (timestamp): When the counters were last recalculated.
- `user_agent` (bigint): Parsed user agent. Points to User agents [table: `event_ua_parsed`].
- `lang` (text, optional): Browser language as sent (`browserLanguage`), such as `en-US,en;q=0.9`.
- `total_visit` (integer): Number of events.

### g) `time` (timestamp(3))

When the event happened, as sent by your app in `eventTime`, stored in UTC with millisecond precision. Always filled.

### h) `query` (bigint) -> Query strings [table: `event_url_query`]

Query strings seen on pages (the part of the URL after the question mark). Each belongs to one page.

Columns:

- `id` (bigint): Query ID.
- `key` (smallint): API key this row belongs to.
- `url` (integer): URL the query string belongs to. Points to Pages [table: `event_url`].
- `query` (text, optional): Query string.
- `lastseen` (timestamp): Latest event with this query string.
- `created` (timestamp): First seen.

### i) `traceid` (varchar(36))

Value of the `X-Request-Id` header of the sensor request, when one is sent; useful when debugging. Optional.

### j) `referer` (bigint) -> Referrers [table: `event_referer`]

Referring URLs that events came from.

Columns:

- `id` (bigint): Referer ID.
- `key` (smallint): API key this row belongs to.
- `referer` (text, optional): Referer URL.
- `lastseen` (timestamp): Latest event with this referer.
- `created` (timestamp): First seen.

### k) `type` (smallint) -> Event types [table: `event_type`]

The 13 kinds of event, from page view to field edit. Events store these IDs.

Columns:

- `id` (smallint): Type ID.
- `value` (text): Type code, sent as `eventType`.
- `name` (text): Display name.

Values:

- 1 = `page_view` (Page View)
- 2 = `page_edit` (Page Edit)
- 3 = `page_delete` (Page Delete)
- 4 = `page_search` (Page Search)
- 5 = `account_login` (Login)
- 6 = `account_logout` (Logout)
- 7 = `account_login_fail` (Login Fail)
- 8 = `account_registration` (Registration)
- 9 = `account_email_change` (Email Change)
- 10 = `account_password_change` (Password Change)
- 11 = `account_edit` (Account Edit)
- 12 = `page_error` (Page Error)
- 13 = `field_edit` (Field Edit)

### l) `email` (integer) -> Email addresses [table: `event_email`]

Each email address used by an account. The domain part is stored separately in Email domains. Most other details come from the optional enrichment service and stay empty without it.

Columns:

- `id` (bigint): Email ID.
- `account_id` (bigint): Owning account. Points to Accounts [table: `event_account`].
- `email` (email domain over citext, optional): Email address, case-insensitive.
- `lastseen` (timestamp): Latest event with this address.
- `created` (timestamp): First seen.
- `key` (integer): API key this row belongs to.
- `checked` (boolean, optional): Whether enrichment has been done for this value.
- `data_breach` (boolean, optional): Whether the address appears in known data breaches (from enrichment).
- `profiles` (integer, optional): Number of online profiles linked to the address (from enrichment).
- `blockemails` (boolean, optional): Whether the address is on a spam list (from enrichment).
- `domain_contact_email` (boolean, optional): Whether the address is a contact address of its domain (from enrichment).
- `domain` (bigint, optional): Domain of the address. Points to Email domains [table: `event_domain`].
- `fraud_detected` (boolean): Whether the address was already blacklisted when the event arrived.
- `hash` (text, optional): SHA-256 of the address.
- `alert_list` (boolean, optional): Whether the address is on the global alert list (from enrichment).
- `data_breaches` (integer, optional): Number of data breaches involving the address (from enrichment).
- `earliest_breach` (text, optional): Date of the earliest known breach (from enrichment).

### m) `phone` (integer) -> Phone numbers [table: `event_phone`]

Each phone number used by an account, with validation and carrier details. Most details come from the optional enrichment service.

Columns:

- `id` (bigint): Phone ID.
- `account_id` (bigint): Owning account. Points to Accounts [table: `event_account`].
- `key` (integer): API key this row belongs to.
- `phone_number` (varchar(19)): Phone number as sent (max 19 characters).
- `calling_country_code` (integer, optional): International calling code, such as 41 (from enrichment).
- `national_format` (varchar, optional): Number in national format (from enrichment).
- `country_code` (smallint, optional): Country; `0` when unknown. Points to Country list [table: `countries`], not checked by the database.
- `validation_errors` (json, optional): JSON with validation errors for the number.
- `mobile_country_code` (smallint, optional): Mobile country code (MCC).
- `mobile_network_code` (smallint, optional): Mobile network code (MNC).
- `carrier_name` (varchar(128), optional): Telecom carrier (from enrichment).
- `type` (varchar(32), optional): Line type (from enrichment).
- `lastseen` (timestamp): Latest event with this number.
- `created` (timestamp): First seen.
- `updated` (timestamp): Last update time.
- `checked` (boolean, optional): Whether enrichment has been done for this value.
- `shared` (smallint): Number of accounts that used this number.
- `fraud_detected` (boolean): Whether the number was already blacklisted when the event arrived.
- `hash` (text, optional): SHA-256 of the number.
- `alert_list` (boolean, optional): Whether the number is on the global alert list (from enrichment).
- `profiles` (integer, optional): Number of online profiles linked to the number (from enrichment).
- `iso_country_code` (varchar(8), optional): ISO country code of the number (from enrichment).
- `invalid` (boolean, optional): Whether the number's format is invalid.

### n) `http_code` (smallint)

The HTTP status code of the request, as sent by your app in `httpCode`, such as 200 or 404. A code of 400 or more makes the event a page error (see k). Optional.

### o) `http_method` (smallint) -> HTTP methods [table: `event_http_method`]

The 11 HTTP methods, such as GET and POST. Events store these IDs.

Columns:

- `id` (smallint): Method ID.
- `value` (text): Lower-case method, such as `post`.
- `name` (text): Display name, such as `POST`.

Values:

- 1 = `get` (GET)
- 2 = `post` (POST)
- 3 = `head` (HEAD)
- 4 = `put` (PUT)
- 5 = `delete` (DELETE)
- 6 = `patch` (PATCH)
- 7 = `trace` (TRACE)
- 8 = `connect` (CONNECT)
- 9 = `options` (OPTIONS)
- 10 = `link` (LINK)
- 11 = `unlink` (UNLINK)

### p) `session_id` (bigint) -> Sessions [table: `event_session`]

A session is one visit: a run of events from one account. A new session starts after 30 minutes without activity, or once a session has lasted 4 hours. Each event points to its session.

Columns:

- `id` (bigint): Session ID.
- `key` (smallint): API key this row belongs to.
- `account_id` (bigint): Owning account. Points to Accounts [table: `event_account`].
- `total_visit` (integer): Number of events in the session.
- `total_device` (integer): Number of distinct devices.
- `total_ip` (integer): Number of distinct IP addresses.
- `total_country` (integer): Number of distinct countries.
- `lastseen` (timestamp(3)): Latest event in the session.
- `created` (timestamp(3)): First event in the session.
- `updated` (timestamp): When the counters were last recalculated.

### q) `payload` (bigint) -> Event payloads [table: `event_payload`]

Extra structured data that came with an event, such as the text of a search. The event points to its payload.

Columns:

- `id` (bigint): Payload ID.
- `key` (smallint): API key this row belongs to.
- `created` (timestamp): Time received.
- `payload` (json, optional): Payload as JSON, such as `{"field_id": "q", "value": "invoices"}`.

## Things to know

- All times are stored in UTC.
- Types are PostgreSQL types as defined in the database: `varchar` is `character varying`, `timestamp` is `timestamp without time zone`, `timestamptz` is `timestamp with time zone`, `citext` is case-insensitive text, `inet` holds an IP address, and `json` and `jsonb` hold JSON.
- Counters (the `total_*` columns and the `shared` counts) are recalculated after events arrive, so they can lag slightly behind the newest events.
- Details marked "from enrichment" are only filled in when an enrichment API token is set; otherwise they stay empty.
- `lastseen` always holds the latest activity and never moves backwards. `updated` says when a row's counters were last recalculated.
- `checked` means enrichment has been done for the value, `hash` is a SHA-256 fingerprint of it, `fraud_detected` means it was already blacklisted when it arrived, and `alert_list` means it is on tirreno's global alert list.
- Scores go from 99 (looks clean) down to 0. After a fresh install every account scores 99, because every rule's weight starts at 0 until you set weights in the console.
- To delete a user, use tirreno's deletion queue: it removes the account together with its events. Deleting an account also removes its devices, email addresses, phone numbers, sessions, scores, review-queue entry and field change history.
- "Not checked by the database" means a column holds another table's ID but the database doesn't verify it.
- Columns marked optional can be empty. Columns the database fills in by itself, such as IDs and creation times, are not marked.

---

This document covers the event record and the 13 tables its columns point to. Five more tables are only named in the column descriptions and not described here: `dshb_operators`, `event_isp`, `countries`, `event_ua_parsed` and `event_domain`.

---
## Resources

| Resource | URL |
|----------|-----|
| Live Demo | [play.tirreno.com](https://play.tirreno.com) (admin/tirreno) |
| Resource center | [tirreno.com/bat](https://www.tirreno.com/bat/) |
| Developers Guide | [github.com/tirrenotechnologies/DEVELOPMENT.md](https://github.com/tirrenotechnologies/DEVELOPMENT.md) |
| Administrator guide | [github.com/tirrenotechnologies/ADMIN.md](https://github.com/tirrenotechnologies/ADMIN.md) |
| Operator guide | [github.com/tirrenotechnologies/OPERATOR.md](https://github.com/tirrenotechnologies/OPERATOR.md) |
| Event DB schema | [github.com/tirrenotechnologies/EVENT_SCHEMA.md](https://github.com/tirrenotechnologies/EVENT_SCHEMA.md) |
| API reference | [github.com/tirrenotechnologies/API.md](https://github.com/tirrenotechnologies/API.md) |
| GitHub | [github.com/tirrenotechnologies/tirreno](https://github.com/tirrenotechnologies/tirreno) |
| GitLab Mirror | [gitlab.com/tirreno/tirreno](https://gitlab.com/tirreno/tirreno) |
| Docker Hub | [hub.docker.com/r/tirreno/tirreno](https://hub.docker.com/r/tirreno/tirreno) |
| Packagist | [packagist.org/packages/tirreno/tirreno](https://packagist.org/packages/tirreno/tirreno) |
| PHP Tracker | [github.com/tirrenotechnologies/tirreno-php-tracker](https://github.com/tirrenotechnologies/tirreno-php-tracker) |
| Python Tracker | [github.com/tirrenotechnologies/tirreno-python-tracker](https://github.com/tirrenotechnologies/tirreno-python-tracker) |
| Node.js Tracker | [github.com/tirrenotechnologies/tirreno-nodejs-tracker](https://github.com/tirrenotechnologies/tirreno-nodejs-tracker) |
| WordPress Tracker | [github.com/tirrenotechnologies/tirreno-wordpress-tracker](https://github.com/tirrenotechnologies/tirreno-wordpress-tracker) |
| Community Chat | [chat.tirreno.com](https://chat.tirreno.com) |

---

## Found a mistake?

If you have found a mistake in the documentation, no matter how large or small, please let us know by [creating a new issue](https://github.com/tirrenotechnologies/tirreno/issues) in the tirreno repository.

---

## License

tirreno and this documentation are licensed under the **GNU Affero General Public License v3 (AGPL-3.0)**.

The name "tirreno" is a registered trademark of tirreno technologies sàrl.

---

*tirreno Copyright (C) 2026 tirreno technologies sàrl, Vaud, Switzerland.*
