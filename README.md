# Modelo ER para Gestión de Farmacia

<img width="589" height="662" alt="Diagrama sin título drawio" src="https://github.com/user-attachments/assets/d29537cf-e87a-447d-86d4-acb0cf15da1a" />

---

## Entidades

### FAMILIA
- **PK:** `familia_id` (INT) 
- **Atributos:** `nombre` (VARCHAR). Ej: `"Antibióticos"`.  
- **Descripción:** Agrupa medicamentos según la indicación terapéutica o el grupo farmacológico (por ejemplo: analgésicos, antihipertensivos). Facilita la búsqueda de sustitutos y el filtrado por categoría.
- **Dominio:** `familia_id` ∈ ENTERO POSITIVO; `nombre` ∈ VARCHAR(100) no vacío.

---

### LABORATORIO
- **PK:** `lab_id` 
- **Atributos:** `nombre`, `telefono`, `fax`, `direccion_postal`, `persona_contacto`.  
  Ej: `{ nombre: "NaturaLab S.A.", telefono: "+34 912345678" }`.  
- **Descripción:** Representa el proveedor o fabricante externo que suministra medicamentos a la farmacia. También identifica al responsable del medicamento cuando no se fabrica internamente.  
- **Dominio:**  
  - `nombre`: cadena no vacía. Ej.: `"NaturaLab S.A."`.  
  - `telefono`/`fax`: cadenas con formato internacional opcional. Ej.: `"+34 912345678"`.  
  - `direccion_postal`: número del código postal (`38107`).  
  - `persona_contacto`: nombre del contacto comercial. Ej.: `"Laura Gómez"`.

---

### MEDICAMENTO
- **PK:** `med_id`  
- **Atributos:** `nombre`, `tipo`, `unidades_stock`, `unidades_vendidas`, `precio_unitario`, `requiere_receta`, `lab_id`, `familia_id`.  
- **Descripción:** Catálogo de fármacos disponibles en la farmacia: identifica la presentación, su stock, precio y si necesita receta. Sirve como fuente de referencia para ventas, compras y gestión de inventario.  
- **Dominio:**  
  - `nombre`: cadena no vacía. Ej: `"Ibuprofeno 400 mg"`.  
  - `tipo`: valor del conjunto permitido; evita valores libres si quieres normalizar. Ej: `"comprimido"`.
  - `unidades_stock`: entero ≥ 0. Ej: `120`.
  - `unidades_vendidas`: entero ≥ 0.  Ej: `430`.
  - `precio_unitario`: decimal con dos decimales (≥ 0). Ej: `3.20`.
  - `requiere_receta`: booleano. Ej: `TRUE`/`FALSE`.

---

### CLIENTE
- **PK:** `cliente_id`  
- **Atributos:** `nombre`, `telefono`, `email`, `banco`, `es_credito`.  
  Ej: `{ nombre: "María Pérez", es_credito: true }`.  
- **Descripción:** Registro de personas o entidades que compran en la farmacia. Se distingue entre clientes que pagan al contado y clientes que tienen condiciones de crédito (p. ej. pago a fin de mes).  
- **Dominio:**  
  - `nombre`: cadena no vacía.  
  - `telefono`, `email`: cadenas con formatos habituales.  
  - `banco`: cadena no vacía.
  - `IBAN`:  cadena no vacía.
  - `es_credito`: booleano.  

---

### VENTA
- **PK:** `venta_id` 
- **Atributos:** `cliente_id`, `fecha_compra`, `fecha_pago`, `total`.  
- **Descripción:** Representa una transacción de venta a un cliente en una fecha concreta; registra también la fecha de pago cuando aplique (clientes con crédito).  
- **Dominio:**  
  - `fecha_compra`: fecha de la operación. Ej.: `2025-09-15`.
  - `total`: cantidad numérica derivable.

---

## Relaciones
- **FAMILIA - PERTENECE - MEDICAMENTO**  
  => Una familia agrupa 1..N medicamentos. Un medicamento pertenece a 0..1 familia.

- **LABORATORIO - SUMINISTRA - MEDICAMENTO**  
  => Un laboratorio suministra muchos medicamentos. Un medicamento es suministrado por 0..1 laboratorio.

- **VENTA - LÍNEA VENTA - MEDICAMENTO**
  => Una venta tiene ≥1 medicamentos. Cada medicamento se puede encontrar en 0..n ventas.

- **CLIENTE - REALIZA - VENTA**  
  => Un cliente puede realizar 0..n ventas. Cada venta tiene exactamente 1 cliente.

---

## Restricciones semánticas propuestas
- `MEDICAMENTO.unidades_stock >= 0`. Impedir venta si `cantidad > unidades_stock`.  
- `MEDICAMENTO.lab_id` (NULL = fabricado internamente).  
- `VENTA.cliente_id` NOT NULL (cada venta debe tener cliente).

---
