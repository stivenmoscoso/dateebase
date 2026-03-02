project/  
 ├── src/  
 ├── scripts/  
 │     └── run-migration.js  
 ├── data/  
 │     └── raw_data.xlsx  
 ├── config/  
 │     └── postgres.js  
  
  
Instralar depemdencisd   
  
npm install pg xlsx dotenv  
  
Conexión posgrets  
  
import pkg from "pg";  
import dotenv from "dotenv";  
  
dotenv.config();  
  
const { Pool } = pkg;  
  
export const pool = new Pool({  
  host: process.env.DB_HOST,  
  user: process.env.DB_USER,  
  password: process.env.DB_PASSWORD,  
  database: process.env.DB_NAME,  
  port: process.env.DB_PORT,  
});  
  
  
  
Scrip de migración.   
  
import xlsx from "xlsx";  
import { pool } from "../config/postgres.js";  
  
const workbook = xlsx.readFile("./data/raw_data.xlsx");  
const sheet = workbook.Sheets[workbook.SheetNames[0]];  
const data = xlsx.utils.sheet_to_json(sheet);  
  
async function migrate() {  
  const client = await pool.connect();  
  
  try {  
    await client.query("BEGIN");  
  
    for (const row of data) {  
      const {  
        "ID Transacción": transactionId,  
        Fecha: date,  
        "Nombre Cliente": customerName,  
        "Email Cliente": customerEmail,  
        Dirección: address,  
        "Categoría Producto": categoryName,  
        SKU: sku,  
        "Nombre Producto": productName,  
        "Precio Unitario": unitPrice,  
        Cantidad: quantity,  
        "Nombre Proveedor": supplierName,  
        "Contacto Proveedor": supplierContact,  
      } = row;  
  
      // 1️⃣ Customer  
      let customerRes = await client.query(  
        `INSERT INTO customers (name, email, address)  
         VALUES ($1,$2,$3)  
         ON CONFLICT (email) DO UPDATE SET name=EXCLUDED.name  
         RETURNING id`,  
        [customerName, customerEmail, address]  
      );  
  
      const customerId = customerRes.rows[0].id;  
  
      // 2️⃣ Supplier  
      let supplierRes = await client.query(  
        `INSERT INTO suppliers (name, contact)  
         VALUES ($1,$2)  
         ON CONFLICT (name) DO UPDATE SET contact=EXCLUDED.contact  
         RETURNING id`,  
        [supplierName, supplierContact]  
      );  
  
      const supplierId = supplierRes.rows[0].id;  
  
      // 3️⃣ Category  
      let categoryRes = await client.query(  
        `INSERT INTO categories (name)  
         VALUES ($1)  
         ON CONFLICT (name) DO NOTHING  
         RETURNING id`,  
        [categoryName]  
      );  
  
      let categoryId;  
      if (categoryRes.rows.length > 0) {  
        categoryId = categoryRes.rows[0].id;  
      } else {  
        const existing = await client.query(  
          "SELECT id FROM categories WHERE name=$1",  
          [categoryName]  
        );  
        categoryId = existing.rows[0].id;  
      }  
  
      // 4️⃣ Product  
      let productRes = await client.query(  
        `INSERT INTO products (sku, name, unit_price, category_id, supplier_id)  
         VALUES ($1,$2,$3,$4,$5)  
         ON CONFLICT (sku) DO UPDATE SET name=EXCLUDED.name  
         RETURNING id`,  
        [sku, productName, unitPrice, categoryId, supplierId]  
      );  
  
      const productId = productRes.rows[0].id;  
  
      // 5️⃣ Order  
      let orderRes = await client.query(  
        `INSERT INTO orders (transaction_id, order_date, customer_id)  
         VALUES ($1,$2,$3)  
         ON CONFLICT (transaction_id) DO NOTHING  
         RETURNING id`,  
        [transactionId, date, customerId]  
      );  
  
      let orderId;  
      if (orderRes.rows.length > 0) {  
        orderId = orderRes.rows[0].id;  
      } else {  
        const existing = await client.query(  
          "SELECT id FROM orders WHERE transaction_id=$1",  
          [transactionId]  
        );  
        orderId = existing.rows[0].id;  
      }  
  
      // 6️⃣ Order Details  
      const subtotal = quantity * unitPrice;  
  
      await client.query(  
        `INSERT INTO order_details (order_id, product_id, quantity, subtotal)  
         VALUES ($1,$2,$3,$4)`,  
        [orderId, productId, quantity, subtotal]  
      );  
    }  
  
    await client.query("COMMIT");  
    console.log("Migration completed successfully ✅");  
  } catch (error) {  
    await client.query("ROLLBACK");  
    console.error("Migration failed ❌", error);  
  } finally {  
    client.release();  
  }  
}  
  
migrate();  
  
Ejecutar migración   
  
node scripts/run-migration.js  
  
Crud   
src/  
 ├── config/  
 │     ├── postgres.js  
 │     └── mongo.js  
 ├── controllers/  
 │     └── product.controller.js  
 ├── routes/  
 │     └── product.routes.js  
 ├── services/  
 │     └── product.service.js  
 ├── models/  
 │     └── auditLog.model.js  
 ├── app.js  
 └── server.js  
  
npm install express mongoose pg dotenv cors  
  
import mongoose from "mongoose";  
import dotenv from "dotenv";  
  
dotenv.config();  
  
export const connectMongo = async () => {  
  try {  
    await mongoose.connect(process.env.MONGO_URI);  
    console.log("MongoDB connected ✅");  
  } catch (error) {  
    console.error("MongoDB connection error ❌", error);  
    process.exit(1);  
  }  
};  
  
  
Modelo   
  
import mongoose from "mongoose";  
  
const auditLogSchema = new mongoose.Schema({  
  entity: { type: String, required: true },  
  entityId: { type: String, required: true },  
  action: { type: String, required: true },  
  deletedData: { type: Object },  
  deletedAt: { type: Date, default: Date.now }  
});  
  
export const AuditLog = mongoose.model("AuditLog", auditLogSchema);  
  
  
Lógica de negocio   
  
import { pool } from "../config/postgres.js";  
import { AuditLog } from "../models/auditLog.model.js";  
  
export const getAllProducts = async () => {  
  const result = await pool.query("SELECT * FROM products");  
  return result.rows;  
};  
  
export const getProductById = async (id) => {  
  const result = await pool.query(  
    "SELECT * FROM products WHERE id=$1",  
    [id]  
  );  
  return result.rows[0];  
};  
  
export const createProduct = async (data) => {  
  const { sku, name, unit_price, category_id, supplier_id } = data;  
  
  const result = await pool.query(  
    `INSERT INTO products (sku, name, unit_price, category_id, supplier_id)  
     VALUES ($1,$2,$3,$4,$5)  
     RETURNING *`,  
    [sku, name, unit_price, category_id, supplier_id]  
  );  
  
  return result.rows[0];  
};  
  
export const updateProduct = async (id, data) => {  
  const { name, unit_price } = data;  
  
  const result = await pool.query(  
    `UPDATE products  
     SET name=$1, unit_price=$2  
     WHERE id=$3  
     RETURNING *`,  
    [name, unit_price, id]  
  );  
  
  return result.rows[0];  
};  
  
export const deleteProduct = async (id) => {  
  const product = await pool.query(  
    "SELECT * FROM products WHERE id=$1",  
    [id]  
  );  
  
  if (product.rows.length === 0) {  
    return null;  
  }  
  
  // Guardar log en Mongo  
  await AuditLog.create({  
    entity: "Product",  
    entityId: id.toString(),  
    action: "DELETE",  
    deletedData: product.rows[0]  
  });  
  
  // Eliminar en SQL  
  await pool.query("DELETE FROM products WHERE id=$1", [id]);  
  
  return product.rows[0];  
};  
  
Control del producto   
  
import * as productService from "../services/product.service.js";  
  
export const getProducts = async (req, res) => {  
  try {  
    const products = await productService.getAllProducts();  
    res.status(200).json(products);  
  } catch (error) {  
    res.status(500).json({ message: "Server error" });  
  }  
};  
  
export const getProduct = async (req, res) => {  
  try {  
    const product = await productService.getProductById(req.params.id);  
    if (!product) return res.status(404).json({ message: "Not found" });  
    res.json(product);  
  } catch (error) {  
    res.status(500).json({ message: "Server error" });  
  }  
};  
  
export const createProduct = async (req, res) => {  
  try {  
    const product = await productService.createProduct(req.body);  
    res.status(201).json(product);  
  } catch (error) {  
    res.status(400).json({ message: "Bad request" });  
  }  
};  
  
export const updateProduct = async (req, res) => {  
  try {  
    const product = await productService.updateProduct(  
      req.params.id,  
      req.body  
    );  
    res.json(product);  
  } catch (error) {  
    res.status(400).json({ message: "Update error" });  
  }  
};  
  
export const deleteProduct = async (req, res) => {  
  try {  
    const deleted = await productService.deleteProduct(req.params.id);  
    if (!deleted) return res.status(404).json({ message: "Not found" });  
    res.json({ message: "Deleted successfully", deleted });  
  } catch (error) {  
    res.status(500).json({ message: "Delete error" });  
  }  
};  
  
Router   
  
import express from "express";  
import * as controller from "../controllers/product.controller.js";  
  
const router = express.Router();  
  
router.get("/", controller.getProducts);  
router.get("/:id", controller.getProduct);  
router.post("/", controller.createProduct);  
router.put("/:id", controller.updateProduct);  
router.delete("/:id", controller.deleteProduct);  
  
export default router;  
  
  
App   
  
import express from "express";  
import cors from "cors";  
import productRoutes from "./routes/product.routes.js";  
  
export const app = express();  
  
app.use(cors());  
app.use(express.json());  
  
app.use("/api/products", productRoutes);  
  
Server   
  
import dotenv from "dotenv";  
import { app } from "./app.js";  
import { pool } from "./config/postgres.js";  
import { connectMongo } from "./config/mongo.js";  
  
dotenv.config();  
  
const startServer = async () => {  
  try {  
    await pool.query("SELECT 1");  
    console.log("PostgreSQL connected ✅");  
  
    await connectMongo();  
  
    app.listen(process.env.PORT, () => {  
      console.log(`Server running on port ${process.env.PORT}`);  
    });  
  } catch (error) {  
    console.error("Server failed to start ❌", error);  
  }  
};  
  
startServer();  
  
  
Postran   
  
  
[http://localhost:3000/api/products](http://localhost:3000/api/products)  
  
db.auditlogs.find().pretty()  
  
  
Sql   
  
SELECT   
    s.id,  
    s.name,  
    SUM(od.quantity) AS total_items_sold,  
    SUM(od.subtotal) AS total_revenue  
FROM suppliers s  
JOIN products p ON p.supplier_id = s.id  
JOIN order_details od ON od.product_id = p.id  
GROUP BY s.id, s.name  
ORDER BY total_items_sold DESC;  
  
  
Service   
  
import { pool } from "../config/postgres.js";  
  
export const getSupplierAnalysis = async () => {  
  const result = await pool.query(`  
    SELECT   
      s.id,  
      s.name,  
      SUM(od.quantity) AS total_items_sold,  
      SUM(od.subtotal) AS total_revenue  
    FROM suppliers s  
    JOIN products p ON p.supplier_id = s.id  
    JOIN order_details od ON od.product_id = p.id  
    GROUP BY s.id, s.name  
    ORDER BY total_items_sold DESC  
  `);  
  
  return result.rows;  
};  
  
Historial del cliente   
  
SELECT   
    o.transaction_id,  
    o.order_date,  
    p.name AS product_name,  
    od.quantity,  
    od.subtotal  
FROM customers c  
JOIN orders o ON o.customer_id = c.id  
JOIN order_details od ON od.order_id = o.id  
JOIN products p ON p.id = od.product_id  
WHERE c.email = $1  
ORDER BY o.order_date DESC;  
  
export const getCustomerHistory = async (email) => {  
  const result = await pool.query(`  
    SELECT   
      o.transaction_id,  
      o.order_date,  
      p.name AS product_name,  
      od.quantity,  
      od.subtotal  
    FROM customers c  
    JOIN orders o ON o.customer_id = c.id  
    JOIN order_details od ON od.order_id = o.id  
    JOIN products p ON p.id = od.product_id  
    WHERE c.email = $1  
    ORDER BY o.order_date DESC  
  `, [email]);  
  
  return result.rows;  
};  
  
Top categoría   
  
SELECT   
    p.id,  
    p.name,  
    SUM(od.quantity) AS total_units_sold,  
    SUM(od.subtotal) AS total_revenue  
FROM categories c  
JOIN products p ON p.category_id = c.id  
JOIN order_details od ON od.product_id = p.id  
WHERE c.name = $1  
GROUP BY p.id, p.name  
ORDER BY total_revenue DESC;  
  
export const getTopProductsByCategory = async (categoryName) => {  
  const result = await pool.query(`  
    SELECT   
      p.id,  
      p.name,  
      SUM(od.quantity) AS total_units_sold,  
      SUM(od.subtotal) AS total_revenue  
    FROM categories c  
    JOIN products p ON p.category_id = c.id  
    JOIN order_details od ON od.product_id = p.id  
    WHERE c.name = $1  
    GROUP BY p.id, p.name  
    ORDER BY total_revenue DESC  
  `, [categoryName]);  
  
  return result.rows;  
};  
  
  
Contróller  
  
import * as biService from "../services/bi.service.js";  
  
export const supplierAnalysis = async (req, res) => {  
  try {  
    const data = await biService.getSupplierAnalysis();  
    res.json(data);  
  } catch (error) {  
    res.status(500).json({ message: "Error fetching supplier analysis" });  
  }  
};  
  
export const customerHistory = async (req, res) => {  
  try {  
    const data = await biService.getCustomerHistory(req.params.email);  
    res.json(data);  
  } catch (error) {  
    res.status(500).json({ message: "Error fetching customer history" });  
  }  
};  
  
export const topProductsByCategory = async (req, res) => {  
  try {  
    const data = await biService.getTopProductsByCategory(req.params.category);  
    res.json(data);  
  } catch (error) {  
    res.status(500).json({ message: "Error fetching top products" });  
  }  
};  
  
Reuter   
  
import express from "express";  
import * as controller from "../controllers/bi.controller.js";  
  
const router = express.Router();  
  
router.get("/suppliers-analysis", controller.supplierAnalysis);  
router.get("/customer-history/:email", controller.customerHistory);  
router.get("/top-products/:category", controller.topProductsByCategory);  
  
export default router;  
  
Agregar app   
  
import biRoutes from "./routes/bi.routes.js";  
  
app.use("/api/bi", biRoutes);  
  
  
Rearme   
  
**MegaStore Global – Data Modernization Project**  
  
📌** Overview**  
  
MegaStore Global previously managed all inventory, sales, suppliers, and customers using a single Excel file. Due to data growth and inconsistencies (duplicate customers, incorrect prices, unreliable stock tracking), the system became inefficient and unreliable.  
  
This project migrates the legacy flat file system into a modern, scalable, and persistent architecture using:  
	•	**PostgreSQL** (Relational Database)  
	•	**MongoDB** (NoSQL Database)  
	•	**MongoDB** (NoSQL Database)  
	•	**Node.js + Express.js** (REST API)  
  
The system supports data normalization, mass migration, business intelligence queries, and audit logging.  
  
⸻  
  
🏗** Architecture**  
  
🐘** PostgreSQL (Relational Engine)**  
  
Used for structured and transactional data:  
	•	Customers  
	•	Suppliers  
	•	Categories  
	•	Products  
	•	Orders  
	•	Order Details  
  
**Why SQL?**  
	•	Strong relational integrity (FK, PK, UNIQUE)  
	•	ACID compliance  
	•	Required JOIN operations for Business Intelligence  
	•	Enforced 3rd Normal Form (3NF)  
  
⸻  
  
🍃** MongoDB (NoSQL Engine)**  
  
Used for:  
	•	Audit logs (Product deletions)  
  
**Why MongoDB?**  
	•	Flexible schema  
	•	Optimized for logging and event storage  
	•	Does not impact transactional consistency  
	•	Suitable for write-heavy operations  
  
Embedding is used for deletedData because audit logs require preserving the full deleted record snapshot.  
  
⸻  
  
🧠** Database Design Justification**  
  
**SQL Normalization Process**  
  
**1NF**  
	•	All attributes are atomic.  
	•	No repeating groups.  
  
**2NF**  
	•	Order details separated from orders.  
	•	Removed partial dependencies.  
  
**3NF**  
	•	Suppliers separated from Products.  
	•	Categories separated from Products.  
	•	Customers separated from Orders.  
	•	No transitive dependencies.  
  
All tables depend strictly on their primary key.  
  
⸻  
  
🗂** Relational Model**  
  
**Tables**  
	•	customers  
	•	suppliers  
	•	categories  
	•	products  
	•	orders  
	•	order_details  
  
**Key Relationships**  
	•	Customers (1) → (N) Orders  
	•	Suppliers (1) → (N) Products  
	•	Categories (1) → (N) Products  
	•	Orders (1) → (N) Order_Details  
	•	Products (1) → (N) Order_Details  
  
All constraints include:  
	•	PRIMARY KEY  
	•	FOREIGN KEY  
	•	UNIQUE  
	•	NOT NULL  
	•	CHECK constraints  
