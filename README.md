# Modelo ER para Gestión de Farmacia — Resumen

## Entidades (PK, atributos — dominio y ejemplo)
### FAMILIA
- **PK:** `familia_id` (INT)  
- **Atributos:** `nombre` (VARCHAR). Ej: `"Antibióticos"`.

### LABORATORIO
- **PK:** `lab_id` (INT)  
- **Atributos:** `nombre`, `telefono`, `fax`, `direccion_postal` (compuesto), `persona_contacto`.  
  Ej: `{ nombre: "NaturaLab S.A.", telefono: "+34 912345678" }`.

### MEDICAMENTO
- **PK:** `med_id` (INT)  
- **Atributos:**  
  - `nombre` (VARCHAR). Ej: `"Ibuprofeno 400 mg"`  
  - `tipo` (VARCHAR: jarabe|comprimido|pomada). Ej: `"comprimido"`  
  - `unidades_stock` (INT ≥ 0). Ej: `120`  
  - `unidades_vendidas` (INT ≥ 0) — **derivado** de LINEA_VENTA o persistido con triggers. Ej: `430`  
  - `precio_unitario` (DECIMAL ≥ 0). Ej: `3.20`  
  - `requiere_receta` (BOOLEAN). Ej: `TRUE`/`FALSE`  
  - `lab_id` (FK → LABORATORIO, **NULLABLE**; NULL = fabricado internamente)  
  - `familia_id` (FK → FAMILIA, NULLABLE)

### CLIENTE
- **PK:** `cliente_id` (INT)  
- **Atributos:** `nombre`, `telefono`, `email`, `direccion` (compuesto), `es_credito` (BOOLEAN).  
  Ej: `{ nombre: "María Pérez", es_credito: true }`.

### CUENTA_BANCARIA (recomendada)
- **PK:** `cuenta_id` (INT)  
- **Atributos:** `cliente_id` (FK), `banco`, `iban`, `titular`.  
  Uso: evita atributos condicionales en CLIENTE.

### VENTA
- **PK:** `venta_id` (INT)  
- **Atributos:** `cliente_id` (FK NOT NULL), `fecha_compra` (DATE), `fecha_pago` (DATE NULLABLE), `total` (DECIMAL, derivable).

### LINEA_VENTA (entidad asociativa)
- **PK:** `(venta_id, med_id)` (compuesta) o `linea_id` (surrogate)  
- **Atributos:** `venta_id` (FK), `med_id` (FK), `cantidad` (INT > 0), `precio_unitario` (DECIMAL ≥ 0).  
  Semántica: cada fila liga **1 VENTA** con **1 MEDICAMENTO** y almacena cantidad y precio histórico.

---

## Relaciones (nombre, cardinalidad min..max y explicación)
- **FAMILIA — PERTENECE — MEDICAMENTO**  
  `FAMILIA (1..1) — PERTENECE — (0..N) MEDICAMENTO`  
  => Una familia agrupa 0..N medicamentos. Un medicamento pertenece a 0..1 familia (simplificado).

- **LABORATORIO — SUMINISTRA — MEDICAMENTO**  
  `LABORATORIO (1..1) — SUMINISTRA — (0..N) MEDICAMENTO`  
  => Un laboratorio suministra muchos medicamentos; `med.lab_id = NULL` indica fabricación propia.

- **VENTA — CONTIENE — MEDICAMENTO** (implementada por LINEA_VENTA)  
  `VENTA (1..1) — 1..N LINEA_VENTA`  
  `MEDICAMENTO (1..1) — 0..N LINEA_VENTA`  
  => Una venta tiene ≥1 líneas; cada línea pertenece a 1 venta y referencia 1 medicamento.

- **CLIENTE — REALIZA — VENTA**  
  `CLIENTE (1..1) — REALIZA — (0..N) VENTA`  
  => Un cliente puede realizar 0..N ventas; cada venta tiene exactamente 1 cliente.

- **CLIENTE — POSEE_CUENTA — CUENTA_BANCARIA**  
  `CLIENTE (1..1) — POSEE_CUENTA — (0..N) CUENTA_BANCARIA`  
  => Si `es_credito = true` debería existir ≥1 cuenta para el cliente (forzar por trigger o lógica de aplicación).

---

## Restricciones semánticas propuestas (esenciales)
- `MEDICAMENTO.unidades_stock >= 0`. Impedir venta si `cantidad > unidades_stock`.  
- `MEDICAMENTO.unidades_vendidas = SUM(LINEA_VENTA.cantidad)` (derivado) — si se persiste, mantener con triggers.  
- `MEDICAMENTO.lab_id` **NULLABLE** (NULL = fabricado internamente).  
- `VENTA.cliente_id` NOT NULL (cada venta debe tener cliente).  
- `CHECK(fecha_pago IS NULL OR fecha_pago >= fecha_compra)`.  
- `LINEA_VENTA.precio_unitario` guarda precio histórico (no depender de MEDICAMENTO.precio_unitario).  
- Si `MEDICAMENTO.requiere_receta = TRUE` y se requiere auditar la receta, añadir entidad `RECETA` y relacionarla con `LINEA_VENTA`.  
- Para clientes con crédito, controlar existencia de `CUENTA_BANCARIA` mediante trigger o validación en la aplicación.

---
