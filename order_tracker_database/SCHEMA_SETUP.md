# Order Tracker Database Schema (PostgreSQL)

This database schema was created via the `psql` CLI command in `db_connection.txt`, executing **one SQL statement at a time** (no `.sql` migration files), per container conventions.

Connection:
- `db_connection.txt` contains the canonical command:
  - `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions / Types

- Extension:
  - `pgcrypto` (used for `gen_random_uuid()`)

- Enums:
  - `user_role`: `customer`, `admin`
  - `order_status`: `created`, `processing`, `shipped`, `out_for_delivery`, `delivered`, `cancelled`, `returned`

## Tables

### `app_user`
Stores users for authentication and role-based access.

Columns:
- `id` uuid PK (default `gen_random_uuid()`)
- `email` text UNIQUE NOT NULL
- `password_hash` text NOT NULL
- `full_name` text NULL
- `role` user_role NOT NULL (default `customer`)
- `is_active` boolean NOT NULL (default `true`)
- `created_at` timestamptz NOT NULL (default `now()`)
- `updated_at` timestamptz NOT NULL (default `now()`)

Trigger:
- `trg_app_user_updated_at` updates `updated_at` on row updates.

### `customer_order`
Stores trackable orders.

Columns:
- `id` uuid PK (default `gen_random_uuid()`)
- `user_id` uuid NULL FK → `app_user(id)` ON DELETE SET NULL
- `order_number` text UNIQUE NOT NULL
- `title` text NULL
- `description` text NULL
- `current_status` order_status NOT NULL (default `created`)
- `eta` timestamptz NULL
- `created_at` timestamptz NOT NULL (default `now()`)
- `updated_at` timestamptz NOT NULL (default `now()`)

Indexes:
- `idx_customer_order_user_id` on `(user_id)`
- `idx_customer_order_status` on `(current_status)`

Trigger:
- `trg_customer_order_updated_at` updates `updated_at` on row updates.

### `order_status_event`
Append-only order status history/events.

Columns:
- `id` uuid PK (default `gen_random_uuid()`)
- `order_id` uuid NOT NULL FK → `customer_order(id)` ON DELETE CASCADE
- `status` order_status NOT NULL
- `note` text NULL
- `created_by` uuid NULL FK → `app_user(id)` ON DELETE SET NULL
- `created_at` timestamptz NOT NULL (default `now()`)

Indexes:
- `idx_order_status_event_order_id_created_at` on `(order_id, created_at DESC)`

### `notification_registration`
Stores notification endpoints/tokens per user (push/email/SMS/etc).

Columns:
- `id` uuid PK (default `gen_random_uuid()`)
- `user_id` uuid NOT NULL FK → `app_user(id)` ON DELETE CASCADE
- `channel` text NOT NULL
- `endpoint` text NOT NULL
- `p256dh` text NULL
- `auth` text NULL
- `device_info` text NULL
- `is_enabled` boolean NOT NULL (default `true`)
- `created_at` timestamptz NOT NULL (default `now()`)
- `updated_at` timestamptz NOT NULL (default `now()`)

Constraints:
- UNIQUE `(user_id, channel, endpoint)`

Indexes:
- `idx_notification_registration_user_enabled` on `(user_id, is_enabled)`

Trigger:
- `trg_notification_registration_updated_at` updates `updated_at` on row updates.

### `notification_preference`
Stores per-user notification preferences (one row per user).

Columns:
- `id` uuid PK (default `gen_random_uuid()`)
- `user_id` uuid NOT NULL FK → `app_user(id)` ON DELETE CASCADE
- `notify_on_status_change` boolean NOT NULL (default `true`)
- `notify_on_delivery` boolean NOT NULL (default `true`)
- `notify_on_exception` boolean NOT NULL (default `true`)
- `created_at` timestamptz NOT NULL (default `now()`)
- `updated_at` timestamptz NOT NULL (default `now()`)

Constraints:
- UNIQUE `(user_id)`

Trigger:
- `trg_notification_preference_updated_at` updates `updated_at` on row updates.

## Seed Data (minimal)

Inserted with `ON CONFLICT DO NOTHING`:

Users:
- `admin@demo.local` (role: `admin`, password_hash: `DEMO_ADMIN_HASH`)
- `customer@demo.local` (role: `customer`, password_hash: `DEMO_CUSTOMER_HASH`)

Order:
- `DEMO-1001` owned by `customer@demo.local`, status `processing`, ETA now + 2 days.

Order events for `DEMO-1001`:
- `created` (note: "Order created (seed).", created_by admin)
- `processing` (note: "Processing started (seed).", created_by admin)

Notification preferences:
- default row for both demo users.

## Notes
- The seed `password_hash` values are placeholders for smoke testing; production authentication should store real password hashes computed by the backend.
- All schema objects were created in the `public` schema.
