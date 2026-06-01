# BI_CUBO_OLAP_TABULAR

Proyecto de **Inteligencia de Negocios** basado en el dataset **TheLook Ecommerce**, implementado con **SQL Server**, **Data Warehouse dimensional**, **SSAS Tabular**, **DAX avanzado**, **DAX Studio** y **Power BI**.

El objetivo del proyecto es construir un flujo BI completo: desde una base transaccional normalizada, pasando por un modelo dimensional tipo constelación, hasta un modelo tabular OLAP con medidas DAX, KPIs y pruebas comparativas de rendimiento.

---

## 📌 Descripción del proyecto

Este repositorio contiene el desarrollo de un modelo BI para analizar información de ventas del e-commerce **TheLook**.

El proyecto incluye:

- Base de datos transaccional en SQL Server.
- Modelo DWH dimensional tipo constelación.
- Proceso ETL desde la base transaccional hacia el DWH.
- Modelo SSAS Tabular desplegado en Analysis Services.
- Medidas DAX avanzadas.
- KPIs de negocio.
- Consultas SQL y DAX para comparar rendimiento.
- Conexión con Power BI Desktop y Power BI Service mediante On-premises Data Gateway.

---

## 🧱 Arquitectura BI

```text
CSV / Dataset TheLook Ecommerce
        ↓
Base Transaccional SQL Server
        ↓
Data Warehouse Dimensional
        ↓
SSAS Tabular Model
        ↓
DAX Studio / Power BI Desktop
        ↓
Power BI Service
```

---

## 🗃️ Bases de datos utilizadas

### Base transaccional

```sql
db_transc_thelook
```

Contiene las tablas normalizadas desde los archivos CSV originales.

Tablas principales:

```text
distribution_centers
products
inventory_items
users
user_addresses
orders
order_items
sessions
events
```

Tablas catálogo:

```text
product_categories
product_brands
product_departments
genders
countries
states
cities
traffic_sources
order_statuses
order_item_statuses
browsers
event_types
event_locations
```

---

### Base DWH dimensional

```sql
db_dwh_thelook
```

Modelo dimensional tipo **constelación de hechos**, compuesto por varias tablas fact y dimensiones compartidas.

---

## ⭐ Modelo dimensional

### Tablas de hechos

```text
fact_sales
fact_orders
fact_inventory
fact_events
```

### Dimensiones

```text
dim_date
dim_product
dim_user
dim_traffic_source
dim_order_status
dim_order_item_status
dim_browser
dim_event_type
dim_event_location
dim_session
```

---

## 🧩 Modelo Tabular SSAS

El modelo tabular fue implementado en una instancia local de **SQL Server Analysis Services Tabular**.

Servidor utilizado:

```text
NIK-RAMIREZ-BAU\TABULAR
```

Base tabular desplegada:

```text
fact_sales_cubo
```

Tablas cargadas en el modelo tabular de ventas:

```text
fact_sales
dim_date
dim_product
dim_user
dim_order_status
dim_order_item_status
```

---

## 📊 Medidas DAX principales

### Ventas

```DAX
Total Sales :=
SUM(fact_sales[sale_price])
```

```DAX
Total Gross Margin :=
SUM(fact_sales[gross_margin])
```

```DAX
Total Quantity :=
SUM(fact_sales[quantity])
```

```DAX
Total Orders :=
DISTINCTCOUNT(fact_sales[order_id])
```

```DAX
Margin % :=
DIVIDE(
    [Total Gross Margin],
    [Total Sales],
    0
)
```

---

## ⏱️ Medidas DAX de inteligencia de tiempo

```DAX
Total Sales LY :=
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR(dim_date[full_date])
)
```

```DAX
Sales YOY Growth % :=
DIVIDE(
    [Total Sales] - [Total Sales LY],
    [Total Sales LY],
    0
)
```

```DAX
Total Sales YTD :=
TOTALYTD(
    [Total Sales],
    dim_date[full_date]
)
```

```DAX
Total Sales Rolling 3 Months :=
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        dim_date[full_date],
        MAX(dim_date[full_date]),
        -3,
        MONTH
    )
)
```

---

## 🚦 KPIs implementados

El modelo tabular incluye KPIs para evaluar el desempeño del negocio.

KPIs principales:

```text
Total Orders
Margin %
Total Sales YTD
Sales YOY Growth %
Total Sales Rolling 3 Months
```

Ejemplo de meta para crecimiento anual:

```DAX
Sales YOY Growth Target :=
0.05
```

Ejemplo de estado KPI:

```DAX
Sales YOY Growth Status :=
SWITCH(
    TRUE(),
    [Sales YOY Growth %] >= 0.05, 1,
    [Sales YOY Growth %] >= 0, 0,
    -1
)
```

Interpretación:

```text
1  = Verde
0  = Amarillo
-1 = Rojo
```

---

## 🧪 Comparación de rendimiento

Se realizaron pruebas comparando consultas directas al DWH en SQL Server contra consultas equivalentes en SSAS Tabular usando DAX Studio.

### Consulta SQL directa al DWH

```sql
USE db_dwh_thelook;
GO

SET STATISTICS TIME ON;
SET STATISTICS IO ON;
GO

SELECT
    dd.year_number,
    dp.department_name,
    SUM(fs.sale_price) AS total_sales,
    SUM(fs.gross_margin) AS total_margin,
    SUM(fs.quantity) AS total_quantity,
    COUNT(DISTINCT fs.order_id) AS total_orders,
    CASE 
        WHEN SUM(fs.sale_price) = 0 THEN 0
        ELSE SUM(fs.gross_margin) / SUM(fs.sale_price)
    END AS margin_percent
FROM dbo.fact_sales AS fs
INNER JOIN dbo.dim_product AS dp
    ON fs.product_key = dp.product_key
INNER JOIN dbo.dim_date AS dd
    ON fs.order_created_date_key = dd.date_key
WHERE dd.year_number = 2021
GROUP BY
    dd.year_number,
    dp.department_name
ORDER BY
    total_sales DESC;
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

### Consulta equivalente en DAX Studio

```DAX
EVALUATE
SUMMARIZECOLUMNS(
    dim_date[year_number],
    dim_product[department_name],
    TREATAS({2021}, dim_date[year_number]),

    "total_sales", [Total Sales],
    "total_margin", [Total Gross Margin],
    "total_quantity", [Total Quantity],
    "total_orders", [Total Orders],
    "margin_percent", [Margin %]
)
ORDER BY
    [total_sales] DESC
```

---

## 📈 Resultado de rendimiento observado

Ejemplo de prueba realizada:

| Motor | Tiempo aproximado | Detalle |
|---|---:|---|
| SQL Server DWH | 246 ms | Consulta SQL con JOIN y GROUP BY |
| SSAS Tabular / DAX Studio | 10 ms | Consulta DAX con Server Timings |

En DAX Studio se observaron métricas como:

```text
Total: 10 ms
Formula Engine: 7 ms
Storage Engine: 3 ms
SE Queries: 2
SE Cache: 100%
```

Conclusión:

> Para la consulta analítica de ventas por departamento en 2021, el modelo SSAS Tabular respondió más rápido que la consulta SQL directa al Datamart, aprovechando almacenamiento columnar en memoria, compresión y caché del motor tabular.

---

## 📁 Estructura del repositorio

```text
BI_CUBO_OLAP_TABULAR/
│
├── fact_sales_cubo/
│   └── Proyecto SSAS Tabular
│
├── fact_sales_cubo.sln
├── README.md
├── LICENSE.txt
├── .gitignore
└── .gitattributes
```

Estructura recomendada adicional:

```text
BI_CUBO_OLAP_TABULAR/
│
├── SQL/
│   ├── 01_create_db_transaccional.sql
│   ├── 02_bulk_insert_transaccional.sql
│   ├── 03_create_dwh_dimensional.sql
│   ├── 04_etl_transaccional_to_dwh.sql
│   └── 05_consultas_rendimiento.sql
│
├── DAX/
│   ├── medidas_dax.md
│   └── consultas_dax_studio.dax
│
├── PowerBI/
│   └── dashboard_thelook.pbix
│
├── docs/
│   ├── modelo_transaccional.png
│   ├── modelo_dwh.png
│   ├── modelo_tabular.png
│   └── evidencia_rendimiento.png
```

---

## ⚙️ Herramientas utilizadas

```text
SQL Server 2025 Developer
SQL Server Management Studio
SQL Server Analysis Services Tabular
Visual Studio / SSDT
DAX Studio
Power BI Desktop
Power BI Service
On-premises Data Gateway
GitHub
```

---

## 🌐 Power BI Service y Gateway

Para conectar Power BI Service con el modelo SSAS Tabular local, se configuró:

```text
On-premises Data Gateway
```

Gateway utilizado:

```text
Gateway_NIK_RAMIREZ
```

Servidor Analysis Services:

```text
NIK-RAMIREZ-BAU\TABULAR
```

Base tabular:

```text
fact_sales_cubo
```

---

## 📌 Evidencias esperadas

Para documentar el proyecto se recomienda incluir capturas de:

- Modelo transaccional en SQL Server.
- Modelo DWH dimensional.
- Proyecto SSAS Tabular en Visual Studio.
- Medidas DAX creadas.
- KPIs configurados.
- DAX Studio con Server Timings.
- Dashboard en Power BI Desktop.
- Publicación en Power BI Service.
- Gateway en estado Online.

---

## 🚀 Cómo ejecutar el proyecto

1. Restaurar o crear la base transaccional `db_transc_thelook`.
2. Cargar los CSV transformados con `BULK INSERT`.
3. Crear la base DWH `db_dwh_thelook`.
4. Ejecutar el ETL desde transaccional hacia DWH.
5. Abrir el proyecto tabular en Visual Studio.
6. Configurar el servidor de despliegue:

```text
NIK-RAMIREZ-BAU\TABULAR
```

7. Implementar y procesar el modelo tabular.
8. Validar medidas en DAX Studio.
9. Conectar Power BI Desktop en modo Live Connection.
10. Publicar el dashboard en Power BI Service.
11. Configurar On-premises Data Gateway.

---

## 📚 Objetivo académico

Este proyecto demuestra la construcción de una solución BI completa, integrando:

- Modelado transaccional.
- Modelado dimensional.
- Procesos ETL.
- Modelos semánticos tabulares.
- Medidas DAX avanzadas.
- KPIs.
- Análisis de rendimiento.
- Visualización y publicación en Power BI Service.

---

## 👤 Autor

Proyecto desarrollado por:

```text
Nik-920
```

Repositorio:

```text
BI_CUBO_OLAP_TABULAR
```

---

## 📄 Licencia

Este proyecto se distribuye bajo licencia MIT.
