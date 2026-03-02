**services/migration.js (Lógica de Carga Masiva)**  
Este script procesa el CSV e inserta en cascada respetando tus llaves foráneas.  
  
const fs = require('fs');  
const csv = require('csv-parser');  
const { pool } = require('../config/db');  
  
const migrateData = async (filePath) => {  
    const client = await pool.connect();  
    try {  
        await client.query('BEGIN');  
        const rows = [];  
  
        // Leer archivo  
        await new Promise((resolve) => {  
            fs.createReadStream(filePath)  
                .pipe(csv())  
                .on('data', (data) => rows.push(data))  
                .on('end', resolve);  
        });  
  
        for (const row of rows) {  
            // 1. Insertar Categoría  
            await client.query(  
                'INSERT INTO categories (name) VALUES ($1) ON CONFLICT (name) DO NOTHING',  
                [row.product_category]  
            );  
            const cat = await client.query('SELECT id FROM categories WHERE name = $1', [row.product_category]);  
  
            // 2. Insertar Proveedor  
            await client.query(  
                'INSERT INTO suppliers (supplier_name, supplier_email) VALUES ($1, $2) ON CONFLICT (supplier_name) DO NOTHING',  
                [row.supplier_name, row.supplier_email]  
            );  
            const supp = await client.query('SELECT id FROM suppliers WHERE supplier_name = $1', [row.supplier_name]);  
  
            // 3. Insertar Cliente  
            await client.query(  
                'INSERT INTO customers (customer_name, customer_email, customer_address, customer_phone) VALUES ($1, $2, $3, $4) ON CONFLICT (customer_email) DO NOTHING',  
                [row.customer_name, row.customer_email, row.customer_address, row.customer_phone]  
            );  
            const cust = await client.query('SELECT id FROM customers WHERE customer_email = $1', [row.customer_email]);  
  
            // 4. Insertar Producto  
            await client.query(  
                'INSERT INTO products (sku, name, unit_price, category_id, supplier_id) VALUES ($1, $2, $3, $4, $5) ON CONFLICT (sku) DO NOTHING',  
                [row.product_sku, row.product_name, row.unit_price, cat.rows[0].id, supp.rows[0].id]  
            );  
            const prod = await client.query('SELECT id FROM products WHERE sku = $1', [row.product_sku]);  
  
            // 5. Insertar Orden (Maestro)  
            await client.query(  
                'INSERT INTO orders (transaction_id, id_customer, created_at) VALUES ($1, $2, $3) ON CONFLICT (transaction_id) DO NOTHING',  
                [row.transaction_id, cust.rows[0].id, row.date]  
            );  
            const order = await client.query('SELECT id FROM orders WHERE transaction_id = $1', [row.transaction_id]);  
  
            // 6. Insertar Detalle de Orden  
            await client.query(  
                'INSERT INTO order_details (id_order, id_product, quantity, subtotal) VALUES ($1, $2, $3, $4)',  
                [order.rows[0].id, prod.rows[0].id, row.quantity, row.total_line_value]  
            );  
        }  
  
        await client.query('COMMIT');  
        console.log("Migración completada exitosamente.");  
    } catch (e) {  
        await client.query('ROLLBACK');  
        throw e;  
    } finally {  
        client.release();  
    }  
};  
  
module.exports = { migrateData };  
  
  
**2. Consultas de Business Intelligence (Fase 5)**  
Actualizadas para usar tus nombres de tabla (order_details, customers, etc.).  
  
// A. Análisis de Proveedores  
app.get('/api/reports/suppliers', async (req, res) => {  
    const query = `  
        SELECT s.supplier_name, SUM(od.quantity) as total_items, SUM(od.subtotal) as total_value  
        FROM suppliers s  
        JOIN products p ON s.id = p.supplier_id  
        JOIN order_details od ON p.id = od.id_product  
        GROUP BY s.id, s.supplier_name  
    `;  
    const result = await pool.query(query);  
    res.json(result.rows);  
});  
  
// B. Historial de Cliente  
app.get('/api/reports/customer/:email', async (req, res) => {  
    const query = `  
        SELECT o.transaction_id, o.created_at as date, p.name as product, od.quantity, od.subtotal  
        FROM customers c  
        JOIN orders o ON c.id = o.id_customer  
        JOIN order_details od ON o.id = od.id_order  
        JOIN products p ON od.id_product = p.id  
        WHERE c.customer_email = $1  
    `;  
    const result = await pool.query(query, [req.params.email]);  
    res.json(result.rows);  
});  
  
  
Variables de entorno   
PORT=3000  
DB_HOST=localhost  
DB_USER=root  
DB_PASS=tu_password  
DB_NAME=db_megastore_exam  
MONGO_URI=mongodb://localhost:27017/db_megastore_exam  
  
  
Conexiones   
  
**config/db.js (Conexiones)**  
  
const mysql = require('mysql2/promise');  
const mongoose = require('mongoose');  
require('dotenv').config();  
  
const pool = mysql.createPool({  
    host: process.env.DB_HOST,  
    user: process.env.DB_USER,  
    password: process.env.DB_PASS,  
    database: process.env.DB_NAME  
});  
  
const connectMongo = async () => {  
    await mongoose.connect(process.env.MONGO_URI);  
    console.log("MongoDB Connected");  
};  
  
module.exports = { pool, connectMongo };  
  
  
**models/AuditLog.js (Fase 2: NoSQL)**  
const mongoose = require('mongoose');  
  
const AuditLogSchema = new mongoose.Schema({  
    action: String,  
    entity: String,  
    entityId: String,  
    details: Object,  
    timestamp: { type: Date, default: Date.now }  
});  
  
module.exports = mongoose.model('AuditLog', AuditLogSchema);  
  
  
**services/migration.js (Fase 3: Migración Masiva)**  
Este script maneja la **idempotencia**: busca si el cliente/proveedor existe antes de crearlo.  
  
const fs = require('fs');  
const csv = require('csv-parser');  
const { pool } = require('../config/db');  
  
const migrateData = async (filePath) => {  
    const connection = await pool.getConnection();  
    try {  
        await connection.beginTransaction();  
        const results = [];  
  
        fs.createReadStream(filePath)  
            .pipe(csv())  
            .on('data', (data) => results.push(data))  
            .on('end', async () => {  
                for (const row of results) {  
                    // 1. Manejar Proveedor  
                    await connection.query(  
                        'INSERT IGNORE INTO suppliers (name, email) VALUES (?, ?)',  
                        [row.supplier_name, row.supplier_email]  
                    );  
                    const [supp] = await connection.query('SELECT id FROM suppliers WHERE name = ?', [row.supplier_name]);  
  
                    // 2. Manejar Cliente  
                    await connection.query(  
                        'INSERT IGNORE INTO customers (name, email, address, phone) VALUES (?, ?, ?, ?)',  
                        [row.customer_name, row.customer_email, row.customer_address, row.customer_phone]  
                    );  
                    const [cust] = await connection.query('SELECT id FROM customers WHERE email = ?', [row.customer_email]);  
  
                    // 3. Manejar Producto  
                    await connection.query(  
                        'INSERT IGNORE INTO products (sku, name, category, unit_price, supplier_id) VALUES (?, ?, ?, ?, ?)',  
                        [row.product_sku, row.product_name, row.product_category, row.unit_price, supp[0].id]  
                    );  
  
                    // 4. Manejar Orden  
                    await connection.query(  
                        'INSERT INTO orders (transaction_id, order_date, customer_id, product_sku, quantity, total_value) VALUES (?, ?, ?, ?, ?, ?)',  
                        [row.transaction_id, row.date, cust[0].id, row.product_sku, row.quantity, row.total_line_value]  
                    );  
                }  
                await connection.commit();  
                console.log("Migración exitosa");  
            });  
    } catch (error) {  
        await connection.rollback();  
        throw error;  
    } finally {  
        connection.release();  
    }  
};  
  
module.exports = { migrateData };  
  
  
**app.js (Fase 4 y 5: API y Consultas)**  
Aquí integramos el CRUD de Productos con Log de auditoría en Mongo y las consultas de BI.    
  
const express = require('express');  
const { pool, connectMongo } = require('./config/db');  
const AuditLog = require('./models/AuditLog');  
const { migrateData } = require('./services/migration');  
  
const app = express();  
app.use(express.json());  
  
// Migración  
app.post('/api/migrate', async (req, res) => {  
    await migrateData('./data.csv');  
    res.send("Migration started...");  
});  
  
// CRUD Productos - DELETE con Audit Log  
app.delete('/api/products/:sku', async (req, res) => {  
    const { sku } = req.params;  
    const [product] = await pool.query('SELECT * FROM products WHERE sku = ?', [sku]);  
      
    if (product.length > 0) {  
        await pool.query('DELETE FROM products WHERE sku = ?', [sku]);  
        [span_15](start_span)[span_16](start_span)// Guardar Log en MongoDB[span_15](end_span)[span_16](end_span)  
        await AuditLog.create({ action: 'DELETE', entity: 'Product', entityId: sku, details: product[0] });  
        return res.json({ message: "Product deleted and logged" });  
    }  
    res.status(404).send("Not found");  
});  
  
[span_17](start_span)[span_18](start_span)// Fase 5: Consultas BI[span_17](end_span)[span_18](end_span)  
app.get('/api/reports/suppliers', async (req, res) => {  
    const [rows] = await pool.query(`  
        SELECT s.name, SUM(o.quantity) as total_items, SUM(o.total_value) as inventory_value  
        FROM suppliers s  
        JOIN products p ON s.id = p.supplier_id  
        JOIN orders o ON p.sku = o.product_sku  
        GROUP BY s.id  
    `);  
    res.json(rows);  
});  
  
connectMongo();  
app.listen(3000, () => console.log("Server running on port 3000"));  
  
  
**Análisis de Proveedores (SQL)**  
**Requerimiento:** Saber qué proveedores han vendido más productos y el valor total del inventario asociado.    
-- SQL: Agrupación y suma de ventas por proveedor  
SELECT   
    s.name AS supplier_name,   
    SUM(o.quantity) AS total_items_sold,   
    SUM(o.total_value) AS total_inventory_value  
FROM suppliers s  
JOIN products p ON s.id = p.supplier_id  
JOIN orders o ON p.sku = o.product_sku  
GROUP BY s.id, s.name  
ORDER BY total_items_sold DESC;  
  
        res.status(500).json({ error: error.message });  
    }  
});  
  
  
**services/migration.js (Lógica de Carga Masiva)**  
Este script procesa el CSV e inserta en cascada respetando tus llaves foráneas.    
  
const fs = require('fs');  
const csv = require('csv-parser');  
const { pool } = require('../config/db');  
  
const migrateData = async (filePath) => {  
    const client = await pool.connect();  
    try {  
        await client.query('BEGIN');  
        const rows = [];  
  
        // Leer archivo  
        await new Promise((resolve) => {  
            fs.createReadStream(filePath)  
                .pipe(csv())  
                .on('data', (data) => rows.push(data))  
                .on('end', resolve);  
        });  
  
        for (const row of rows) {  
            // 1. Insertar Categoría  
            await client.query(  
                'INSERT INTO categories (name) VALUES ($1) ON CONFLICT (name) DO NOTHING',  
                [row.product_category]  
            );  
            const cat = await client.query('SELECT id FROM categories WHERE name = $1', [row.product_category]);  
  
            // 2. Insertar Proveedor  
            await client.query(  
                'INSERT INTO suppliers (supplier_name, supplier_email) VALUES ($1, $2) ON CONFLICT (supplier_name) DO NOTHING',  
                [row.supplier_name, row.supplier_email]  
            );  
            const supp = await client.query('SELECT id FROM suppliers WHERE supplier_name = $1', [row.supplier_name]);  
  
            // 3. Insertar Cliente  
            await client.query(  
                'INSERT INTO customers (customer_name, customer_email, customer_address, customer_phone) VALUES ($1, $2, $3, $4) ON CONFLICT (customer_email) DO NOTHING',  
                [row.customer_name, row.customer_email, row.customer_address, row.customer_phone]  
            );  
            const cust = await client.query('SELECT id FROM customers WHERE customer_email = $1', [row.customer_email]);  
  
            // 4. Insertar Producto  
            await client.query(  
                'INSERT INTO products (sku, name, unit_price, category_id, supplier_id) VALUES ($1, $2, $3, $4, $5) ON CONFLICT (sku) DO NOTHING',  
                [row.product_sku, row.product_name, row.unit_price, cat.rows[0].id, supp.rows[0].id]  
            );  
            const prod = await client.query('SELECT id FROM products WHERE sku = $1', [row.product_sku]);  
  
            // 5. Insertar Orden (Maestro)  
            await client.query(  
                'INSERT INTO orders (transaction_id, id_customer, created_at) VALUES ($1, $2, $3) ON CONFLICT (transaction_id) DO NOTHING',  
                [row.transaction_id, cust.rows[0].id, row.date]  
            );  
            const order = await client.query('SELECT id FROM orders WHERE transaction_id = $1', [row.transaction_id]);  
  
            // 6. Insertar Detalle de Orden  
            await client.query(  
                'INSERT INTO order_details (id_order, id_product, quantity, subtotal) VALUES ($1, $2, $3, $4)',  
                [order.rows[0].id, prod.rows[0].id, row.quantity, row.total_line_value]  
            );  
        }  
  
        await client.query('COMMIT');  
        console.log("Migración completada exitosamente.");  
    } catch (e) {  
        await client.query('ROLLBACK');  
        throw e;  
    } finally {  
        client.release();  
    }  
};  
  
module.exports = { migrateData };  
  
  
**Consultas de Business Intelligence (Fase 5)**  
Actualizadas para usar tus nombres de tabla (order_details, customers, etc.).  
  
// A. Análisis de Proveedores  
app.get('/api/reports/suppliers', async (req, res) => {  
    const query = `  
        SELECT s.supplier_name, SUM(od.quantity) as total_items, SUM(od.subtotal) as total_value  
        FROM suppliers s  
        JOIN products p ON s.id = p.supplier_id  
        JOIN order_details od ON p.id = od.id_product  
        GROUP BY s.id, s.supplier_name  
    `;  
    const result = await pool.query(query);  
    res.json(result.rows);  
});  
  
// B. Historial de Cliente  
app.get('/api/reports/customer/:email', async (req, res) => {  
    const query = `  
        SELECT o.transaction_id, o.created_at as date, p.name as product, od.quantity, od.subtotal  
        FROM customers c  
        JOIN orders o ON c.id = o.id_customer  
        JOIN order_details od ON o.id = od.id_order  
        JOIN products p ON od.id_product = p.id  
        WHERE c.customer_email = $1  
    `;  
    const result = await pool.query(query, [req.params.email]);  
    res.json(result.rows);  
});  
