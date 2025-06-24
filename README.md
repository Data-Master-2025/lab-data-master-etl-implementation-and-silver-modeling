![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | Implementación de ETL y Modelado Silver en Snowflake

## 🧭 Objetivo

Aplicar un pipeline completo ETL en Snowflake para mover datos desde Bronze a Silver con buenas prácticas, diseño de modelo Silver en SqlDBM, optimización de cargas incrementales y estandarización conforme a las normativas SEAT.

## Requisitos

* Haz un ***fork*** de este repositorio.
* Clona este repositorio.

## Entrega

- Haz Commit y Push
- Crea un Pull Request (PR)
- Copia el enlace a tu PR (con tu solución) y pégalo en el campo de entrega del portal del estudiante – solo así se considerará entregado el lab

## 🧱 1. Crear consulta para mover datos de Bronze a Silver

**Origen:** `DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_COMPLEX_WIDE`  
**Destino:** `DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_CLEAN_WIDE`

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_CLEAN_WIDE AS
SELECT
  ID_ORDER,
  TRY_TO_DATE(DTE_ORDER, 'YYYY-MM-DD') AS DTE_ORDER,
  ID_CUSTOMER,
  CASE WHEN ID_PART != 'ERROR' THEN ID_PART ELSE NULL END AS ID_PART,
  CASE 
    WHEN IS_NUMBER(QTY_ORDERED) AND TRY_TO_NUMBER(QTY_ORDERED) > 0 THEN TRY_TO_NUMBER(QTY_ORDERED)
    ELSE NULL 
  END AS QTY_ORDERED,
  TRY_TO_NUMBER(AMT_TOTAL) AS AMT_TOTAL,
  NULLIF(UPPER(REF_PAYMENT_METHOD), '') AS REF_PAYMENT_METHOD,
  TRY_TO_DATE(DTE_DELIVERY_EST, 'YYYY-MM-DD') AS DTE_DELIVERY_EST,
  DES_ORDER_NOTE,
  NULLIF(CAT_ORDER_TYPE, '') AS CAT_ORDER_TYPE,
  NULLIF(TYP_CUSTOMER_SEGMENT, '') AS TYP_CUSTOMER_SEGMENT,
  CASE WHEN REF_DISCOUNT_CODE IN ('SUMMER20', 'BLACKFRIDAY') THEN REF_DISCOUNT_CODE ELSE NULL END AS REF_DISCOUNT_CODE,
  CASE WHEN IS_NUMBER(QTY_UNITS_BACKORDER) THEN TRY_TO_NUMBER(QTY_UNITS_BACKORDER) ELSE NULL END AS QTY_UNITS_BACKORDER,
  TRY_TO_TIMESTAMP(TST_INSERTION) AS TST_INSERTION,
  AUD_USR_INSERT,
  CURRENT_TIMESTAMP() AS AUD_TST_LOAD
FROM DEV_BRONZE_DB.B_LEGACY_ERP.B_ORDERS_RAW_COMPLEX_WIDE;
```

## 🚀 2. Optimización de cargas incrementales

Simular incremental usando `WHERE` con timestamp de última ejecución:

```sql
-- Supongamos que cargamos sólo pedidos nuevos
INSERT INTO DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_CLEAN_WIDE
SELECT * FROM (
  -- mismo SELECT que arriba
) WHERE TRY_TO_TIMESTAMP(TST_INSERTION) > (SELECT MAX(TST_INSERTION) FROM DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_CLEAN_WIDE);
```

## ⚙️ 3. Optimización de consultas en Snowflake

- Uso de `TRY_` evita abortos por errores.
- `IS_NUMBER`, `NULLIF`, y `CASE` aplican reglas condicionales eficientes.
- Crear `CLUSTER BY` para mejorar consultas por columnas como `ID_CUSTOMER`, `DTE_ORDER`.

```sql
CREATE OR REPLACE TABLE DEV_SILVER_DB.S_SUPPLY_CHAIN.S_ORDERS_CLEAN_WIDE CLUSTER BY (ID_CUSTOMER, DTE_ORDER)
AS SELECT ...
```

## 🧩 4. Modelado Silver en SqlDBM

### ✅ Pasos a seguir:

1. Crear proyecto nuevo en SqlDBM: `S_SUPPLY_CHAIN`
2. Usar prefijo `S_` para tablas y nombres físicos con guión bajo (`S_ORDERS_CLEAN_WIDE`)
3. Agrupar por dominio → SUPPLY_CHAIN
4. Establecer claves primarias y relaciones
5. Aplicar nomenclatura oficial:
  - `ID_` para claves
  - `CAT_` para categorías
  - `TYP_` para tipos
  - `REF_` para referencias
  - `AUD_` para auditoría
  - `DTE_`, `TST_`, `QTY_`, `AMT_` según tipo

## 📐 5. Documentar la tabla Silver

| Campo | Tipo | Descripción |
| --- | --- | --- |
| ID_ORDER | STRING | Identificador único del pedido |
| DTE_ORDER | DATE | Fecha del pedido |
| ID_CUSTOMER | STRING | Identificador del cliente |
| ID_PART | STRING | Pieza solicitada |
| QTY_ORDERED | NUMBER | Unidades pedidas |
| AMT_TOTAL | NUMBER(12,2) | Importe total en EUR |
| REF_PAYMENT_METHOD | STRING | Código del método de pago |
| DTE_DELIVERY_EST | DATE | Fecha estimada de entrega |
| ... | ... | ... (ver tabla completa en código) |

## 🧠 Ejercicio

1. Cargar el dataset en SqlDBM
2. Identificar campos candidatos a FK (e.g. `REF_PAYMENT_METHOD`, `TYP_CUSTOMER_SEGMENT`)
3. Crear una tabla referencial y establecer relaciones
4. Validar que cumple 3FN y normativas SEAT
5. Exportar `CREATE TABLE` y comparar con versión automatizada en Snowflake

## Entregables

Dentro de tu repositorio forkeado, asegúrate de incluir los siguientes archivos:

* `silver.sql` – Script con la transformación de Bronze a Silver (`S_ORDERS_CLEAN_WIDE`)
* `incremental_load.sql` – Script con la lógica de carga incremental basada en `TST_INSERTION`
* `clustered_table.sql` – Script opcional con la creación de tabla optimizada usando `CLUSTER BY`
* `sqlDBM_export.sql` – Export del modelo Silver creado en SqlDBM (con claves, relaciones y nomenclatura aplicada)
* `lab-notes.md` – Documento explicativo que incluya:

  * Descripción de las reglas de limpieza y validación aplicadas en Silver
  * Explicación del modelo diseñado en SqlDBM (dominios, relaciones, convenciones SEAT)
  * Qué campos fueron documentados como FK o referenciales
  * Respuestas al ejercicio de modelado
* *(Opcional)* Capturas de pantalla del proyecto en SqlDBM

## ✅ Conclusión

En este lab has implementado un flujo real de carga ETL optimizada desde Bronze a Silver, respetando nomenclatura SEAT, modelo por dominios y gobernanza documental. SqlDBM te ha permitido formalizar tu modelo físico de datos con estandarización.

- 📁 Dataset: `B_ORDERS_RAW_COMPLEX_WIDE.csv`  
- 📂 Esquema Silver: `S_SUPPLY_CHAIN`  
- 📐 Herramienta de modelado: SqlDBM