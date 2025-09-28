# Modelo ER para Gestión de Farmacia — Documentación

Este documento describe el modelo Entidad-Relación (ER) corregido y explicado para el enunciado:  
> *La gestión de una farmacia requiere poder llevar control de los medicamentos existentes, así como de los que se van sirviendo...* (ver especificación completa al final).

Incluye: descripción de entidades, atributos (dominio y ejemplos), relaciones con cardinalidades (min..max), restricciones semánticas recomendadas, diagrama ASCII (Chen corregido), DDL SQL mínimo (Postgres/MySQL estilo) y ejemplos de datos.

---

## Índice
1. Resumen conceptual
2. Entidades (descripción detallada)
3. Relaciones (cardinalidades y explicaciones)
4. Restricciones semánticas (cómo implementarlas)
5. Diagrama ASCII (Chen)
6. DDL SQL mínimo
7. Triggers/Reglas útiles (ejemplos)
8. Ejemplos de datos
9. Lista de verificación

---

## 1. Resumen conceptual (rápido)
- Entidades principales: **MEDICAMENTO, LABORATORIO, FAMILIA, CLIENTE, CUENTA_BANCARIA, VENTA, LINEA_VENTA**.  
- `LINEA_VENTA` es la **entidad asociativa** entre `VENTA` y `MEDICAMENTO` (captura `cantidad` y `precio_unitario` histórico).  
- Cardinalidades clave:
  - `FAMILIA 1 — 0..N MEDICAMENTO`
  - `LABORATORIO 1 — 0..N MEDICAMENTO` (`med.lab_id` nullable → fabricado internamente)
  - `CLIENTE 1 — 0..N VENTA`
  - `VENTA 1 — 1..N LINEA_VENTA`
  - `MEDICAMENTO 1 — 0..N LINEA_VENTA`

---

## 2. Entidades (detalle: PK, atributos, dominio y ejemplos)

### FAMILIA (Entidad fuerte)
- **PK:** `familia_id` (INT autoincrement)
- **Atributos:**
  - `nombre` (VARCHAR). Ej: `"Antibióticos"`.
- **Descripción:** categoría terapéutica que agrupa medicamentos.

### LABORATORIO (Entidad fuerte)
- **PK:** `lab_id`
- **Atributos:**
  - `nombre` (VARCHAR). Ej: `"NaturaLab S.A."`
  - `telefono` (VARCHAR) — posible multivaluado. Ej: `"+34 912345678"`.
  - `fax` (VARCHAR). Ej: `"+34 912111222"`.
  - `direccion_postal` (compuesto: calle, ciudad, cp). Ej: `"C/ Mayor 5, Madrid, 28013"`.
  - `persona_contacto` (VARCHAR). Ej: `"Dra. López"`.
- **Nota:** Un medicamento puede no tener laboratorio si la farmacia lo fabrica.

### MEDICAMENTO (Entidad fuerte)
- **PK:** `med_id`
- **Atributos:**
  - `nombre` (VARCHAR). Ej: `"Amoxicilina 500 mg"`.
  - `tipo` (VARCHAR). Ej: `"comprimido"`, `"jarabe"`.
  - `unidades_stock` (INT ≥ 0). Ej: `120`.
  - `unidades_vendidas` (INT ≥ 0) — **derivado** de `LINEA_VENTA` (opcionalmente persistido).
  - `precio_unitario` (DECIMAL). Ej: `6.50`.
  - `requiere_receta` (BOOLEAN). Ej: `TRUE`.
  - `lab_id` (FK → LABORATORIO, NULLABLE).
  - `familia_id` (FK → FAMILIA, NULLABLE).
- **Observación:** `unidades_vendidas` puede calcularse con `SUM(LINEA_VENTA.cantidad)` por `med_id`. Si se almacena, mantenerla con triggers.

### CLIENTE (Entidad fuerte)
- **PK:** `cliente_id`
- **Atributos:**
  - `nombre` (VARCHAR). Ej: `"María Pérez"`.
  - `telefono` (VARCHAR) — podría normalizarse a tabla `CLIENTE_TELEFONO`.
  - `email` (VARCHAR)
  - `direccion` (compuesto).
  - `es_credito` (BOOLEAN). Indica pago mensual/condición de crédito.
- **Regla:** datos bancarios no se guardan directamente aquí para evitar atributos condicionales.

### CUENTA_BANCARIA (Entidad fuerte, recomendada)
- **PK:** `cuenta_id`
- **Atributos:**
  - `cliente_id` (FK → CLIENTE)
  - `banco` (VARCHAR). Ej: `"Banco Ejemplo"`.
  - `iban` (VARCHAR). Ej: `"ES79..."`.
  - `titular` (VARCHAR).
- **Descripción:** contiene las cuentas asociadas a clientes (0..N por cliente). Si `es_credito = TRUE`, debería existir al menos una.

### VENTA (Entidad fuerte)
- **PK:** `venta_id`
- **Atributos:**
  - `cliente_id` (FK → CLIENTE, NOT NULL).
  - `fecha_compra` (DATE). Ej: `"2025-09-15"`.
  - `fecha_pago` (DATE, NULLABLE) — para clientes con crédito.
  - `total` (DECIMAL) — derivable: `SUM(cantidad * precio_unitario)` desde `LINEA_VENTA`.
- **Uso:** cabecera de operación de venta.

### LINEA_VENTA (Entidad asociativa / débil)
- **PK:** compuesta `(venta_id, med_id)` o `linea_id` (surrogate) si se prefiere.
- **Atributos:**
  - `venta_id` (FK → VENTA, NOT NULL)
  - `med_id` (FK → MEDICAMENTO, NOT NULL)
  - `cantidad` (INT > 0). Ej: `2`.
  - `precio_unitario` (DECIMAL) — precio histórico.
- **Semántica:** cada fila vincula **exactamente 1** VENTA con **exactamente 1** MEDICAMENTO.

---

## 3. Relaciones (detalladas con cardinalidades min..max)

> Notación: `EntidadA (min..max) — RELACIÓN — (min..max) EntidadB`

1. **FAMILIA — PERTENECE — MEDICAMENTO**  
   - `FAMILIA (1..1) — PERTENECE — (0..N) MEDICAMENTO`  
   - *Lectura:* una familia puede agrupar cero o más medicamentos; cada medicamento pertenece a cero o una familia (en versión simplificada).

2. **LABORATORIO — SUMINISTRA — MEDICAMENTO**  
   - `LABORATORIO (1..1) — SUMINISTRA — (0..N) MEDICAMENTO`  
   - *Lectura:* un laboratorio puede suministrar muchos medicamentos; un medicamento puede no tener laboratorio (NULL) o tener uno.

3. **VENTA — CONTIENE — MEDICAMENTO** (implementada con `LINEA_VENTA`)  
   - En práctica relacional:
     - `VENTA (1..1) — 1..N LINEA_VENTA`
     - `MEDICAMENTO (1..1) — 0..N LINEA_VENTA`  
   - *Lectura:* una venta tiene 1..N líneas; cada línea pertenece a 1 venta; un medicamento puede aparecer en 0..N líneas.

4. **CLIENTE — REALIZA — VENTA**  
   - `CLIENTE (1..1) — REALIZA — (0..N) VENTA`  
   - *Lectura:* un cliente puede realizar 0..N ventas; cada venta es realizada por exactamente 1 cliente.

5. **CLIENTE — POSEE_CUENTA — CUENTA_BANCARIA**  
   - `CLIENTE (1..1) — POSEE_CUENTA — (0..N) CUENTA_BANCARIA`  
   - *Lectura:* un cliente puede tener varias cuentas bancarias.

---

## 4. Restricciones semánticas (reglas de negocio y cómo implementarlas)

1. **`lab_id` nullable → fabricación interna**  
   - Implementación: FK `med.lab_id` referenciando `lab_id` y permitiendo `NULL`.  
   - Alternativa: añadir `fabricado_interno BOOLEAN` y un CHECK:  
     ```sql
     CHECK( (fabricado_interno = TRUE AND lab_id IS NULL) OR (fabricado_interno = FALSE AND lab_id IS NOT NULL) )
     ```

2. **Datos bancarios de clientes con crédito**  
   - Regla: si `CLIENTE.es_credito = TRUE` debería existir al menos una `CUENTA_BANCARIA` para ese cliente.  
   - Implementación: forzar desde la capa aplicación o mediante TRIGGER que en inserción/actualización verifique existencia en `CUENTA_BANCARIA`.

3. **`unidades_stock >= 0`**  
   - Implementación: `CHECK(unidades_stock >= 0)` y lógica que impida vender más unidades que el stock disponible (trigger o control en la aplicación).

4. **`fecha_pago >= fecha_compra` si no es NULL**  
   - Implementación: `CHECK (fecha_pago IS NULL OR fecha_pago >= fecha_compra)`.

5. **Precio histórico**  
   - Regla: `LINEA_VENTA.precio_unitario` guarda el precio al momento de la venta (no depende de `MEDICAMENTO.precio_unitario` futuro).

6. **Unidades vendidas consistentes**  
   - Opción 1: no persistir `unidades_vendidas` (calcular en consulta).  
   - Opción 2: persistir y mantener con triggers en inserts/updates/deletes sobre `LINEA_VENTA`.

7. **Receta para medicamentos con receta**  
   - Si `MEDICAMENTO.requiere_receta = TRUE`, exigir `RECETA` asociada y que `LINEA_VENTA` referencie `receta_id`. Aplicable si la farmacia necesita auditar recetas.

8. **Integridad referencial**  
   - Implementar FKs entre: `MEDICAMENTO.lab_id` → `LABORATORIO.lab_id`, `MEDICAMENTO.familia_id` → `FAMILIA.familia_id`, `VENTA.cliente_id` → `CLIENTE.cliente_id`, `LINEA_VENTA.venta_id` → `VENTA`, `LINEA_VENTA.med_id` → `MEDICAMENTO`.