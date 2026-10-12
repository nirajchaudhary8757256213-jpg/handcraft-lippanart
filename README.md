PRAGMA foreign_keys = ON;
CREATE TABLE IF NOT EXISTS users (
 id TEXT PRIMARY KEY, email TEXT NOT NULL COLLATE NOCASE UNIQUE, name TEXT NOT NULL,
 phone TEXT, password_hash TEXT NOT NULL, role TEXT NOT NULL DEFAULT 'customer' CHECK(role IN ('customer','admin')),
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS sessions (
 id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
 token_hash TEXT NOT NULL UNIQUE, expires_at TEXT NOT NULL, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS sessions_expiry ON sessions(expires_at);
CREATE TABLE IF NOT EXISTS categories (
 id TEXT PRIMARY KEY, name TEXT NOT NULL, slug TEXT NOT NULL UNIQUE, description TEXT NOT NULL DEFAULT '', image TEXT NOT NULL DEFAULT '', sort_order INTEGER NOT NULL DEFAULT 0, active INTEGER NOT NULL DEFAULT 1
);
CREATE TABLE IF NOT EXISTS products (
 id TEXT PRIMARY KEY, sku TEXT NOT NULL UNIQUE, name TEXT NOT NULL, slug TEXT NOT NULL UNIQUE,
 description TEXT NOT NULL, price_paise INTEGER NOT NULL CHECK(price_paise >= 0), compare_price_paise INTEGER,
 category TEXT NOT NULL, image TEXT NOT NULL, images_json TEXT NOT NULL DEFAULT '[]',
 dimensions TEXT NOT NULL DEFAULT '', materials TEXT NOT NULL DEFAULT '', stock_quantity INTEGER,
 availability TEXT NOT NULL DEFAULT 'made_to_order' CHECK(availability IN ('in_stock','made_to_order','out_of_stock')),
 featured INTEGER NOT NULL DEFAULT 0, is_new INTEGER NOT NULL DEFAULT 0, active INTEGER NOT NULL DEFAULT 1,
 customization_json TEXT NOT NULL DEFAULT '[]', production_info TEXT NOT NULL DEFAULT '',
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS products_category_active ON products(category,active);
CREATE INDEX IF NOT EXISTS products_active_created ON products(active,created_at DESC);
CREATE TABLE IF NOT EXISTS cart_items (
 cart_key TEXT NOT NULL, product_id TEXT NOT NULL REFERENCES products(id), customization TEXT NOT NULL DEFAULT '',
 quantity INTEGER NOT NULL CHECK(quantity > 0), updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
 PRIMARY KEY(cart_key,product_id,customization)
);
CREATE TABLE IF NOT EXISTS addresses (
 id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE, label TEXT NOT NULL DEFAULT 'Home',
 name TEXT NOT NULL, phone TEXT NOT NULL, line1 TEXT NOT NULL, line2 TEXT NOT NULL DEFAULT '', city TEXT NOT NULL,
 state TEXT NOT NULL, pincode TEXT NOT NULL, country TEXT NOT NULL DEFAULT 'India', is_default INTEGER NOT NULL DEFAULT 0,
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS addresses_user ON addresses(user_id);
CREATE TABLE IF NOT EXISTS coupons (
 id TEXT PRIMARY KEY, code TEXT NOT NULL UNIQUE COLLATE NOCASE, discount_type TEXT NOT NULL CHECK(discount_type IN ('percent','fixed')),
 amount INTEGER NOT NULL CHECK(amount > 0), min_order_paise INTEGER NOT NULL DEFAULT 0, max_uses INTEGER,
 uses INTEGER NOT NULL DEFAULT 0, expires_at TEXT, active INTEGER NOT NULL DEFAULT 1, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS orders (
 id TEXT PRIMARY KEY, order_number TEXT NOT NULL UNIQUE, user_id TEXT REFERENCES users(id) ON DELETE SET NULL, cart_key TEXT NOT NULL,
 guest_email TEXT, customer_name TEXT NOT NULL, phone TEXT NOT NULL, address_line1 TEXT NOT NULL, address_line2 TEXT NOT NULL DEFAULT '',
 city TEXT NOT NULL, state TEXT NOT NULL, pincode TEXT NOT NULL, country TEXT NOT NULL DEFAULT 'India', notes TEXT NOT NULL DEFAULT '',
 status TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending','payment_confirmed','processing','shipped','out_for_delivery','delivered','cancelled','refund_processing','refunded','pending_payment')),
 payment_status TEXT NOT NULL DEFAULT 'pending' CHECK(payment_status IN ('pending','paid','failed','not_required','refunded')),
 payment_method TEXT NOT NULL, subtotal_paise INTEGER NOT NULL, shipping_paise INTEGER NOT NULL,
 discount_paise INTEGER NOT NULL DEFAULT 0, total_paise INTEGER NOT NULL, coupon_code TEXT,
 tracking_carrier TEXT, tracking_number TEXT, inventory_released_at TEXT,
 cancellation_requested_at TEXT,
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS orders_user_created ON orders(user_id,created_at DESC);
CREATE INDEX IF NOT EXISTS orders_status_created ON orders(status,created_at DESC);
CREATE TABLE IF NOT EXISTS order_items (
 id TEXT PRIMARY KEY, order_id TEXT NOT NULL REFERENCES orders(id) ON DELETE CASCADE, product_id TEXT,
 sku TEXT NOT NULL, product_name TEXT NOT NULL, unit_price_paise INTEGER NOT NULL, quantity INTEGER NOT NULL,
 customization TEXT NOT NULL DEFAULT ''
);
CREATE INDEX IF NOT EXISTS order_items_order ON order_items(order_id);
CREATE TABLE IF NOT EXISTS payments (
 id TEXT PRIMARY KEY, order_id TEXT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
 provider TEXT NOT NULL, provider_order_id TEXT UNIQUE, provider_payment_id TEXT UNIQUE,
 amount_paise INTEGER NOT NULL, currency TEXT NOT NULL DEFAULT 'INR', status TEXT NOT NULL DEFAULT 'created',
 signature TEXT, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS webhook_events (
 event_id TEXT PRIMARY KEY, received_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS custom_enquiries (
 id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, phone TEXT NOT NULL, product_type TEXT NOT NULL,
 dimensions TEXT NOT NULL DEFAULT '', colors TEXT NOT NULL DEFAULT '', description TEXT NOT NULL,
 budget_paise INTEGER, reference_file TEXT, status TEXT NOT NULL DEFAULT 'new' CHECK(status IN ('new','quoted','in_progress','converted','closed')),
 quote_paise INTEGER, admin_notes TEXT NOT NULL DEFAULT '', created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS enquiries_status_created ON custom_enquiries(status,created_at DESC);
CREATE TABLE IF NOT EXISTS newsletter_subscribers (
 id TEXT PRIMARY KEY, email TEXT NOT NULL UNIQUE COLLATE NOCASE, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, active INTEGER NOT NULL DEFAULT 1
);
CREATE TABLE IF NOT EXISTS reviews (
 id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
 product_id TEXT NOT NULL REFERENCES products(id) ON DELETE CASCADE, order_id TEXT REFERENCES orders(id) ON DELETE SET NULL,
 rating INTEGER NOT NULL CHECK(rating BETWEEN 1 AND 5), body TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending','approved','rejected')),
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, UNIQUE(user_id,product_id,order_id)
);
CREATE INDEX IF NOT EXISTS reviews_public ON reviews(product_id,status,created_at DESC);
CREATE TABLE IF NOT EXISTS password_resets (
 id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
 token_hash TEXT NOT NULL UNIQUE, expires_at TEXT NOT NULL, used_at TEXT
);
CREATE TABLE IF NOT EXISTS checkout_idempotency (
 key_hash TEXT PRIMARY KEY, request_hash TEXT NOT NULL, order_id TEXT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS store_settings (
 key TEXT PRIMARY KEY, value TEXT NOT NULL, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS cart_state (
 cart_key TEXT PRIMARY KEY, coupon_code TEXT, updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS wishlist_items (
 user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE, product_id TEXT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
 created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY(user_id,product_id)
);
CREATE INDEX IF NOT EXISTS wishlist_user ON wishlist_items(user_id,created_at DESC);
