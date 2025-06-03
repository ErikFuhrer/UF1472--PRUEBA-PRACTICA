# 📊 Vistas SQL en Northwind PostgreSQL

Este documento explica las vistas SQL generadas para análisis de ventas, clientes, productos y proveedores dentro del proyecto Northwind adaptado a PostgreSQL.

---

## 🗓️ `vw_ventas_mes`

**Descripción:** Agrupa las ventas por mes y muestra el total vendido.

```sql
SELECT TO_CHAR(DATE_TRUNC('month', o.order_date), 'MM-YYYY') AS mes,
       ROUND(SUM((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) AS ventas_totales
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
WHERE o.order_date IS NOT NULL
GROUP BY DATE_TRUNC('month', o.order_date)
ORDER BY DATE_TRUNC('month', o.order_date);
```

## 📅 vw_ventas_diarias

**Descripción:** Muestra ventas diarias de los últimos 30 días.

Como la base de datos es histórica (última fecha: `1998-05-06`), este filtro está fijado. Si agregas datos recientes, puedes cambiar el WHERE por:

```sql
WHERE o.order_date >= (CURRENT_DATE - INTERVAL '30 days')
```

```sql
CREATE VIEW vw_ventas_diarias as
select o.order_date as fecha, round(sum((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as ventas
from orders o
join order_details od on o.order_id = od.order_id
where o.order_date >=('1998-05-06'::date -'30 days'::interval)
group by o.order_date
order by o.order_date desc
```

## 👨‍💼 vw_ventas_empleado

**Descripción:** Total de ventas y número de órdenes realizadas por cada empleado.

```sql
create view vw_ventas_empleado as
select concat(e.first_name,' ', e.last_name) as nombre, count(distinct o.order_id) as numero_de_ventas, round(sum((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as ganancias
from orders o
join order_details od on o.order_id = od.order_id
join employees e on o.employee_id = e.employee_id
group by e.first_name, e.last_name
order by nombre;
```

## 🛒 vw_top_productos

**Descripción:** Ranking de productos según el número total de unidades vendidas.

```sql
create view vw_top_productos as
select p.product_name as Producto, sum(od.quantity) as "Numero de veces comprado"
from order_details od
join products p on od.product_id = p.product_id
join orders o on od.order_id = o.order_id
group by p.product_name
order by 2 desc;
```

## 👥 vw_top_clientes

**Descripción:** Clientes que más han gastado, con detalle de sus pedidos y productos comprados.

```sql
create or replace view vw_top_clientes as
select c.company_name,
round(sum((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as "Suma total gastada",
count(o.order_id) as "Numero de pedidos",
count(distinct od.product_id) as "Numero de productos comprados"
from customers c
join orders o on c.customer_id = o.customer_id
join order_details od on o.order_id = od.order_id
group by 1
order by 2 desc;
```

## 🧾 vw_ventas_categoria

**Descripción:** Total de ventas agrupadas por categoría de producto.

```sql
create view vw_ventas_categoria as
select c.category_name, round(sum((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as ventas
from products p
join order_details od on p.product_id = od.product_id
join categories c on p.category_id = c.category_id
group by 1
order by 2 desc;
```

## 🌍 vw_ventas_pais

**Descripción:** Ventas totales agrupadas por país del cliente.

```sql
create view vw_ventas_pais as
select c.country, round(sum((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as ventas
from orders o
join order_details od on o.order_id = od.order_id
join customers c on o.customer_id = o.customer_id
group by 1
order by 2 desc;
```

## 🏙️ vw_ordenes_ciudad

**Descripción:** Muestra la cantidad de órdenes y total de ventas por ciudad.

```sql
create view vw_ordenes_ciudad as
SELECT
c.city,
COUNT(DISTINCT o.order_id) AS numero_de_ordenes,
ROUND(SUM((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) AS ventas_totales
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_details od ON o.order_id = od.order_id
GROUP BY c.city
ORDER BY ventas_totales DESC;
```

## 📉 vw_stock_bajo

**Descripción:** Lista de productos cuyo stock actual está en o por debajo del mínimo.

```sql
create view vw_stock_bajo as
select product_name, units_in_stock
from products
where units_in_stock <= reorder_level and discontinued = 0
order by 2;
```

## 💰 vw_precios_categoria

**Descripción:** Muestra el precio promedio, mínimo y máximo de productos por categoría.

```sql
create view vw_precios_categoria as
select c.category_name,
round(avg(p.unit_price)::numeric,2) as promedio,
round(min(p.unit_price)::numeric, 2) as precio_minimo,
round(max(p.unit_price)::numeric, 2) as precio_maximo
from products p
join categories c on c.category_id = p.category_id
where p.discontinued = 0
group by 1
order by 2;
```

## 🔍 vw_analisis_clientes

**Descripción:** Analiza el comportamiento de compra de cada cliente: total gastado, frecuencia, ubicación y tiempo desde su última compra.

```sql
create view vw_analisis_clientes as
select  c.company_name as "nombre de la compañia",
        ROUND(SUM((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) as "total gastado",
        count(o.order_id) as "numero de pedidos",
        ROUND(SUM((od.unit_price * od.quantity * (1 - od.discount))::numeric), 2) / count(o.order_id) as "promedio por pedido",
        min(o.order_date) as "fecha primer pedido",
        max(o.order_date) as "fecha ultimo pedido",
        current_date - max(o.order_date) as "dias desde el ultimo pedido",
        c.country as pais,
        c.city as ciudad
from customers c
join orders o on o.customer_id = c.customer_id
join order_details od on od.order_id = o.order_id
group by 1, 8, 9
order by 2 desc;
```

## 📦 vw_ordenes_detalladas

**Descripción:** Vista detallada de cada orden: cliente, empleado, total, estado de envío y número de productos.

```sql
create view vw_ordenes_detalladas as
select  o.order_id as "id de orden",
        o.order_date as "fecha de orden",
        c.company_name as "nombre de cliente",
        c.country as "pais",
        c.city as "ciudad",
        concat(e.first_name, ' ' , e.last_name) as "empleado",
        round(sum((od.unit_price * od.quantity) * (1 - od.discount))::numeric, 2) as "total gastado",
        count(od.product_id) as "numero de productos",
        case
            when o.shipped_date is null then 'Pendiente'
            else 'Enviado'
        end as estado
from orders o
join customers c on o.customer_id = c.customer_id
join employees e on o.employee_id = e.employee_id
join order_details od on o.order_id = od.order_id
group by o.order_id, o.order_date, c.company_name, c.country, c.city, e.first_name, e.last_name
order by 9;
```

## 🏭 vw_performance_proveedores

**Descripción:** Evalúa a los proveedores según número de productos, unidades vendidas y ventas generadas.

```sql
create view vw_performance_proveedores as
select  s.company_name as "nombre de la compañia",
        s.country as "pais",
        s.city as "ciudad",
        count(p.product_id) as "cantidad de productos que ofrece",
        sum(od.quantity) as "total de unidades vendidas de sus productos",
        round(sum((od.unit_price * od.quantity) * (1 - od.discount))::numeric, 2) as "total generado",
        max(o.order_date) as "fecha de la ultima venta de uno de sus productos"
from suppliers s
join products p on s.supplier_id = p.supplier_id
left join order_details od on p.product_id = od.product_id
left join orders o on od.order_id = o.order_id
group by s.company_name, s.country, s.city
order by "total generado" desc nulls last;
```

## 📁 Recomendaciones de uso

Estas vistas están diseñadas como base para:

- **Dashboards de BI** (Power BI, Metabase, Superset…)
- **Consultas analíticas en SQL**
- **Reportes de ventas y rendimiento**

**Puedes consultarlas directamente así:**

```sql
SELECT * FROM vw_ventas_mes;
SELECT * FROM vw_top_clientes LIMIT 10;
SELECT * FROM vw_stock_bajo;
```
