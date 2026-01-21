# e-commerce
Project Structure
ecommerce-api/
│
├── index.js
├── db.json
├── package.json
│
├── routes/
│   ├── products.routes.js
│   ├── orders.routes.js
│   └── analytics.routes.js
│
└── utils/
    └── db.js

📄 db.json
{
  "products": [],
  "orders": []
}

📄 package.json
{
  "name": "ecommerce-api",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}

📄 utils/db.js
import fs from "fs";

const DB_PATH = "./db.json";

export const readDB = () => {
  const data = fs.readFileSync(DB_PATH, "utf-8");
  return JSON.parse(data);
};

export const writeDB = (data) => {
  fs.writeFileSync(DB_PATH, JSON.stringify(data, null, 2));
};

📄 index.js
import express from "express";
import productsRouter from "./routes/products.routes.js";
import ordersRouter from "./routes/orders.routes.js";
import analyticsRouter from "./routes/analytics.routes.js";

const app = express();
app.use(express.json());

app.use("/products", productsRouter);
app.use("/orders", ordersRouter);
app.use("/analytics", analyticsRouter);

app.listen(3000, () => {
  console.log("Server running on port 3000");
});

📄 routes/products.routes.js
import express from "express";
import { readDB, writeDB } from "../utils/db.js";

const router = express.Router();

router.post("/", (req, res) => {
  const db = readDB();
  const newProduct = {
    id: Date.now(),
    ...req.body
  };

  db.products.push(newProduct);
  writeDB(db);

  res.status(201).json(newProduct);
});

router.get("/", (req, res) => {
  const db = readDB();
  res.json(db.products);
});

export default router;

📄 routes/orders.routes.js
import express from "express";
import { readDB, writeDB } from "../utils/db.js";

const router = express.Router();

/* 1️⃣ Create Order */
router.post("/", (req, res) => {
  const { productId, quantity } = req.body;
  const db = readDB();

  const product = db.products.find(p => p.id === productId);
  if (!product) return res.status(404).json({ message: "Product not found" });

  if (product.stock === 0 || quantity > product.stock) {
    return res.status(400).json({ message: "Insufficient stock" });
  }

  const totalAmount = product.price * quantity;

  const order = {
    id: Date.now(),
    productId,
    quantity,
    totalAmount,
    status: "placed",
    createdAt: new Date().toISOString().split("T")[0]
  };

  product.stock -= quantity;
  db.orders.push(order);

  writeDB(db);
  res.status(201).json(order);
});

/* 2️⃣ Get All Orders */
router.get("/", (req, res) => {
  const db = readDB();
  res.json(db.orders);
});

/* 3️⃣ Cancel Order */
router.delete("/:orderId", (req, res) => {
  const db = readDB();
  const order = db.orders.find(o => o.id == req.params.orderId);

  if (!order) return res.status(404).json({ message: "Order not found" });
  if (order.status === "cancelled")
    return res.status(400).json({ message: "Order already cancelled" });

  const today = new Date().toISOString().split("T")[0];
  if (order.createdAt !== today)
    return res.status(400).json({ message: "Cancellation window expired" });

  order.status = "cancelled";

  const product = db.products.find(p => p.id === order.productId);
  product.stock += order.quantity;

  writeDB(db);
  res.json({ message: "Order cancelled successfully" });
});

/* 4️⃣ Change Order Status */
router.patch("/change-status/:orderId", (req, res) => {
  const { status } = req.body;
  const db = readDB();
  const order = db.orders.find(o => o.id == req.params.orderId);

  if (!order) return res.status(404).json({ message: "Order not found" });
  if (["cancelled", "delivered"].includes(order.status))
    return res.status(400).json({ message: "Status cannot be changed" });

  const flow = ["placed", "shipped", "delivered"];
  const currentIndex = flow.indexOf(order.status);
  if (flow[currentIndex + 1] !== status)
    return res.status(400).json({ message: "Invalid status flow" });

  order.status = status;
  writeDB(db);

  res.json(order);
});

export default router;

📄 routes/analytics.routes.js
import express from "express";
import { readDB } from "../utils/db.js";

const router = express.Router();

/* 1️⃣ All Orders */
router.get("/allorders", (req, res) => {
  const db = readDB();
  const orders = [];
  db.orders.forEach(o => orders.push(o));
  res.json({ count: orders.length, orders });
});

/* 2️⃣ Cancelled Orders */
router.get("/cancelled-orders", (req, res) => {
  const db = readDB();
  const cancelled = db.orders.filter(o => o.status === "cancelled");
  res.json({ count: cancelled.length, orders: cancelled });
});

/* 3️⃣ Shipped Orders */
router.get("/shipped", (req, res) => {
  const db = readDB();
  const shipped = db.orders.filter(o => o.status === "shipped");
  res.json({ count: shipped.length, orders: shipped });
});

/* 4️⃣ Total Revenue by Product */
router.get("/total-revenue/:productId", (req, res) => {
  const db = readDB();
  const product = db.products.find(p => p.id == req.params.productId);
  if (!product) return res.status(404).json({ message: "Product not found" });

  const totalRevenue = db.orders
    .filter(o => o.productId == product.id && o.status !== "cancelled")
    .reduce((sum, o) => sum + o.quantity * product.price, 0);

  res.json({ productId: product.id, totalRevenue });
});

/* 5️⃣ Overall Revenue */
router.get("/alltotalrevenue", (req, res) => {
  const db = readDB();

  const totalRevenue = db.orders
    .filter(o => o.status !== "cancelled")
    .reduce((sum, o) => {
      const product = db.products.find(p => p.id === o.productId);
      return sum + o.quantity * product.price;
    }, 0);

  res.json({ totalRevenue });
});

export default router;
