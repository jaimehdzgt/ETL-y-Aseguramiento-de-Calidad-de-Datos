# ETL-y-Aseguramiento-de-Calidad-de-Datos



# ETL & Data Quality for E‑commerce (Pedidos)

Proyecto completo para demostrar **buenas prácticas de ETL y aseguramiento de calidad de datos** en un caso de e‑commerce.
Incluye normalización de esquema, limpieza, validaciones de negocio, deduplicación, corrección geográfica, **KPIs** e **insights** con un reporte HTML estilizado.

> **Tecnologías:** Python · Pandas · NumPy · Matplotlib · Google Colab

---

## 🧭 Objetivo y Enunciado de Negocio

**Objetivo:** limpiar y estandarizar el registro de pedidos para habilitar análisis confiables de ventas, rentabilidad y logística.

**Enunciado de negocio:** _“Como equipo de e‑commerce queremos mejorar la rentabilidad y la experiencia de envío. Para ello necesitamos datos consistentes por pedido, producto, cliente y geografía; así podremos detectar fugas de margen, optimizar modalidades de envío y priorizar categorías y segmentos de clientes.”_

---

## 📂 Estructura del repositorio

```
.
├── etl_enhanced_colab.ipynb           # Notebook principal (listo para Google Colab)
├── etl_cleaning_notebook_fixed.ipynb  # Variante previa (también funcional)
├── fact_orders_clean.csv              # Salida con datos limpios (se genera al ejecutar)
├── quality_report.csv                 # Resumen de calidad (se genera al ejecutar)
├── README.md                          # Este archivo
└── /datasets/                         # (opcional local) lugar para .xlsx si trabajas fuera de Colab
```

---

## 🚀 Ejecución rápida (Google Colab)

1. Sube **`etl_enhanced_colab.ipynb`** a tu Colab o abre desde tu Drive.
2. En la sección **Setup & Paths**, monta tu Drive y usa este `DATA_PATH` (según tu estructura en Drive):

```python
from google.colab import drive
drive.mount('/content/drive')

DATA_PATH = "/content/drive/MyDrive/Colab Notebooks/datasets/etl_dirty_orders.xlsx"
SHEET_RAW     = "raw_orders"
SHEET_STATES  = "lookup_states"
SHEET_SUBCATS = "lookup_subcats"
```

3. Ejecuta todas las celdas (Ctrl/Cmd + F9).  
   Al final tendrás:
   - `fact_orders_clean.csv`
   - `quality_report.csv`
   - Un **reporte HTML** con KPIs e insights renderizado dentro del notebook.

> **Tip:** opcionalmente copia el Excel al runtime para lectura más rápida:
>
> ```python
> !cp "/content/drive/MyDrive/Colab Notebooks/datasets/etl_dirty_orders.xlsx" "/content/etl_dirty_orders.xlsx"
> DATA_PATH = "/content/etl_dirty_orders.xlsx"
> ```

---

## 🛠️ Pipeline de ETL (resumen)

1. **Normalización de columnas**
   - Convierte nombres a `snake_case`; unifica variantes (`total_usd`→`total`, `order_id*`→`order_id` principal).
2. **Remoción de filas no-dato**  
   - Elimina `SUBTOTAL`, `TOTAL GENERAL`, etc., incluso si aparecen en cualquier columna.
3. **Parseo y tipificación**
   - **Fechas**: ISO, MDY, DMY y seriales de Excel (sin warnings de pandas).
   - **Moneda**: soporta símbolos/formatos US/EU y textos como `incluido`.
   - **Porcentajes**: a fracción [0–1].
4. **Limpieza de texto**
   - Trim, compresión de espacios, `Title Case` en atributos clave; emails en minúsculas.
5. **Lookups / Diccionarios**
   - Unificación de **estados** y **subcategorías** mediante hojas `lookup_states` y `lookup_subcats`.
6. **Reglas de negocio**
   - `ship_date ≥ order_date`, `payment_date ≥ order_date`.
   - Recalculo de `total_calc` y flag de discrepancia vs `total`.
   - `qty ≥ 1` (flag en inválidos).
7. **Corrección geográfica**
   - Detección e intercambio `lat/long` si están invertidos (valores dentro de México).
8. **Deduplicación robusta**
   - Llave (`order_id`, `productid`, `order_date`) tolerando **columnas duplicadas** por nombre.
9. **Exportación**
   - `fact_orders_clean.csv` + `quality_report.csv`.
10. **Insights**
    - Gráficos rápidos (ventas/utilidad por categoría, distribución de descuentos).
    - Reporte HTML con KPIs, segmentos, geografía, categorías, logística y próximos pasos.

---

## 📊 KPIs y métricas calculadas

- **Órdenes totales**
- **Ventas totales (`total`/`total_calc`)**
- **Utilidad total (`profit`)** y **Margen global** = Profit / Ventas
- **Tasa de órdenes rentables** = % de pedidos con `profit > 0`
- **Top segmentos** por conteo y por utilidad
- **Top regiones/estados** por utilidad
- **Top categorías/subcategorías** por ventas y utilidad
- **Modalidades de envío** (o proxy) por frecuencia y utilidad
- **Lead time** (días): `ship_date − order_date` *(si existen columnas)*
- **Tasa de devoluciones** *(si existe `order_status`)*
- **Flags de calidad**: fechas fuera de secuencia, discrepancias de total, cantidad inválida, lat/lon invertidos

---

## 📑 Diccionario básico de datos (esperado)

- `order_id` — Identificador del pedido.
- `order_date`, `ship_date`, `payment_date` — Fechas clave del ciclo del pedido.
- `productid`, `category`, `sub_category` — Identificadores y taxonomía de producto.
- `qty`, `unit_price`, `discount_pct`, `tax`, `shipping` — Componentes de precio.
- `total` / `total_calc` — Total reportado y recalculado.
- `profit` — Utilidad.
- `segment`, `region`, `state`, `city` — Atributos de cliente/ubicación.
- `payment_method`, `order_priority`, `order_status` — Atributos operativos.
- `latitude`, `longitude` — Geolocalización (opcional).

> El notebook es tolerante a columnas faltantes: si alguna no existe, salta los cálculos dependientes.

---

## 🧪 Reproducibilidad & Calidad

- **Sin warnings** en parseo de fechas: no usa `infer_datetime_format`; emplea formatos explícitos y seriales de Excel.
- **Regex sin `SyntaxWarning`**: usa `r'\s+'` para compresión de espacios.
- **Deduplicación sin errores** con columnas repetidas (`order_id` duplicado por nombre).

---

## 🧷 Fragmentos útiles

**Quitar filas no‑dato en cualquier columna**

```python
tokens = {"SUBTOTAL", "TOTAL GENERAL", "TOTAL_GENERAL"}
mask_bad = df.astype(str).apply(lambda s: s.str.strip().str.upper().isin(tokens)).any(axis=1)
df = df[~mask_bad].copy()
```

**Deduplicación robusta con llaves repetidas por nombre**

```python
import pandas as pd

keys_wanted = ['order_id', 'productid', 'order_date']

def first_series_by_name(df, name):
    idxs = [i for i, c in enumerate(df.columns) if c == name]
    return df.iloc[:, idxs[0]] if idxs else None

aux_cols, keys_present = {}, []
for k in keys_wanted:
    s = first_series_by_name(df, k)
    if s is not None:
        aux_cols[k] = s; keys_present.append(k)

if keys_present:
    df_keys = pd.DataFrame(aux_cols, index=df.index)
    order_idx = df_keys.sort_values(by=keys_present, kind='mergesort').index
    dup_mask = df_keys.loc[order_idx].duplicated(subset=keys_present, keep='first')
    df = df.loc[order_idx][~dup_mask].copy()
```

---

## 🐛 Solución de problemas

- **`No se encontró DATA_PATH`** → Verifica la ruta del Excel o usa la copia al runtime (`/content/etl_dirty_orders.xlsx`).  
- **`column label 'order_id' is not unique`** → Usa la deduplicación robusta (sección anterior).  
- **Fechas con warnings** → Garantiza que estás usando la función `parse_date_series` del notebook mejorado.  
- **Gráficas vacías** → Revisa que existan las columnas usadas (el notebook crea “fallbacks” cuando faltan).

---

## 📜 Licencia

Este proyecto es de uso educativo y demostrativo. Úsalo como base para tus propios procesos de limpieza y análisis.

---

## 👋 Contacto / Autoría

Si necesitas adaptar el pipeline a tu esquema real (nombres de columnas, reglas de negocio, lookups adicionales, dashboards), abre un issue o comenta en el notebook. ¡Con gusto lo ajustamos!

