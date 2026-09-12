const express = require("express");
const session = require("express-session");
const Database = require("better-sqlite3");
const path = require("path");

const app = express();
const db = new Database(path.join(__dirname, "data", "shop.db"));

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use(
  session({
    secret: process.env.SESSION_SECRET || "change-this-secret",
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      sameSite: "lax"
    }
  })
);

app.use(express.static(path.join(__dirname, "public")));

db.exec(`
CREATE TABLE IF NOT EXISTS products (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  description TEXT DEFAULT '',
  price INTEGER NOT NULL,
  stock INTEGER DEFAULT 0,
  image TEXT DEFAULT '/product-cover.png',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS payments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT NOT NULL,
  amount INTEGER NOT NULL,
  envelope TEXT NOT NULL,
  status TEXT DEFAULT 'pending',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT NOT NULL,
  product_id INTEGER NOT NULL,
  price INTEGER NOT NULL,
  status TEXT DEFAULT 'paid',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
`);

const count = db
  .prepare("SELECT COUNT(*) AS c FROM products")
  .get().c;

if (!count) {
  db.prepare(`
    INSERT INTO products
    (name, description, price, stock, image)
    VALUES (?, ?, ?, ?, ?)
  `).run(
    "ไก่ทำเงิน 100M+",
    "สินค้า Roblox สำหรับผู้เล่น",
    49,
    10,
    "/product-cover.png"
  );
}

const ADMIN_USER = process.env.ADMIN_USER || "admin";
const ADMIN_PASS = process.env.ADMIN_PASS || "change-me-now";

function adminOnly(req, res, next) {
  if (!req.session.admin) {
    return res.status(401).json({
      error: "ต้องเข้าสู่ระบบแอดมิน"
    });
  }

  next();
}

// =========================
// PRODUCTS
// =========================

app.get("/api/products", (req, res) => {
  const products = db
    .prepare("SELECT * FROM products ORDER BY id DESC")
    .all();

  res.json(products);
});

// =========================
// PAYMENT
// =========================

app.post("/api/payments", (req, res) => {
  const {
    username,
    amount,
    envelope
  } = req.body;

  if (
    !username ||
    !Number.isInteger(Number(amount)) ||
    Number(amount) <= 0 ||
    !envelope
  ) {
    return res.status(400).json({
      error: "ข้อมูลไม่ครบ"
    });
  }

  const info = db
    .prepare(`
      INSERT INTO payments
      (username, amount, envelope)
      VALUES (?, ?, ?)
    `)
    .run(
      username.trim(),
      Number(amount),
      envelope.trim()
    );

  res.json({
    ok: true,
    id: info.lastInsertRowid
  });
});

// =========================
// ORDERS
// =========================

app.post("/api/orders", (req, res) => {
  const {
    username,
    productId
  } = req.body;

  const product = db
    .prepare("SELECT * FROM products WHERE id = ?")
    .get(productId);

  if (!username || !product) {
    return res.status(400).json({
      error: "ไม่พบสินค้า"
    });
  }

  if (product.stock <= 0) {
    return res.status(400).json({
      error: "สินค้าหมด"
    });
  }

  const transaction = db.transaction(() => {

    db.prepare(`
      UPDATE products
      SET stock = stock - 1
      WHERE id = ?
    `).run(product.id);

    return db.prepare(`
      INSERT INTO orders
      (username, product_id, price)
      VALUES (?, ?, ?)
    `).run(
      username.trim(),
      product.id,
      product.price
    );
  });

  const info = transaction();

  res.json({
    ok: true,
    orderId: info.lastInsertRowid
  });
});

// =========================
// ADMIN LOGIN
// =========================

app.post("/api/admin/login", (req, res) => {
  const {
    username,
    password
  } = req.body;

  if (
    username === ADMIN_USER &&
    password === ADMIN_PASS
  ) {
    req.session.admin = true;

    return res.json({
      ok: true
    });
  }

  res.status(401).json({
    error: "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง"
  });
});

// =========================
// ADMIN LOGOUT
// =========================

app.post("/api/admin/logout", adminOnly, (req, res) => {
  req.session.destroy(() => {
    res.json({
      ok: true
    });
  });
});

// =========================
// ADMIN PAYMENTS
// =========================

app.get("/api/admin/payments", adminOnly, (req, res) => {
  const payments = db
    .prepare(`
      SELECT *
      FROM payments
      ORDER BY id DESC
    `)
    .all();

  res.json(payments);
});

// =========================
// ADMIN ORDERS
// =========================

app.get("/api/admin/orders", adminOnly, (req, res) => {
  const orders = db
    .prepare(`
      SELECT
        orders.*,
        products.name AS product_name
      FROM orders
      JOIN products
        ON products.id = orders.product_id
      ORDER BY orders.id DESC
    `)
    .all();

  res.json(orders);
});

// =========================
// APPROVE PAYMENT
// =========================

app.post(
  "/api/admin/payments/:id/approve",
  adminOnly,
  (req, res) => {

    db.prepare(`
      UPDATE payments
      SET status = 'approved'
      WHERE id = ?
    `).run(req.params.id);

    res.json({
      ok: true
    });
  }
);

// =========================
// ADD PRODUCT
// =========================

app.post("/api/admin/products", adminOnly, (req, res) => {

  const {
    name,
    description,
    price,
    stock,
    image
  } = req.body;

  if (
    !name ||
    !Number.isFinite(Number(price)) ||
    Number(price) < 0
  ) {
    return res.status(400).json({
      error: "ข้อมูลสินค้าไม่ถูกต้อง"
    });
  }

  const info = db.prepare(`
    INSERT INTO products
    (name, description, price, stock, image)
    VALUES (?, ?, ?, ?, ?)
  `).run(
    name.trim(),
    (description || "").trim(),
    Number(price),
    Math.max(0, Number(stock || 0)),
    image || "/product-cover.png"
  );

  res.json({
    ok: true,
    id: info.lastInsertRowid
  });
});

// =========================
// EDIT PRODUCT
// =========================

app.patch(
  "/api/admin/products/:id",
  adminOnly,
  (req, res) => {

    const {
      name,
      description,
      price,
      stock,
      image
    } = req.body;

    db.prepare(`
      UPDATE products
      SET
        name = ?,
        description = ?,
        price = ?,
        stock = ?,
        image = ?
      WHERE id = ?
    `).run(
      name,
      description || "",
      Number(price),
      Math.max(0, Number(stock)),
      image || "/product-cover.png",
      req.params.id
    );

    res.json({
      ok: true
    });
  }
);

// =========================
// DELETE PRODUCT
// =========================

app.delete(
  "/api/admin/products/:id",
  adminOnly,
  (req, res) => {

    db.prepare(`
      DELETE FROM products
      WHERE id = ?
    `).run(req.params.id);

    res.json({
      ok: true
    });
  }
);

// =========================
// ADMIN PAGE
// =========================

app.get("/admin", (req, res) => {
  res.sendFile(
    path.join(
      __dirname,
      "public",
      "admin.html"
    )
  );
});

// =========================
// START SERVER
// =========================

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(
    `Queenzy Roblox Shop running on port ${PORT}`
  );
});
