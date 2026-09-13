# Normalización de datos con PostgreSQL — Migración Clientes/Pedidos

## Contexto

Este proyecto documenta el proceso real de migrar una planilla de Excel de un emprendimiento de indumentaria motorsport (miles de filas, con cliente y pedido mezclados en una sola fila) hacia un modelo relacional en PostgreSQL.

El objetivo final es doble:
1. Tener una base de datos limpia y normalizada como fundación de un futuro sistema de gestión.
2. Servir como laboratorio de aprendizaje práctico de SQL — diseño de esquemas, integridad referencial, limpieza de datos y migración por etapas — en el marco de mi formación como Analista de Sistemas (Técnico Jr. en redes Cisco, CCNA 1/2/3, curso de DevOps).

> **Nota sobre los datos:** por tratarse de información real de clientes (nombres, teléfonos), este repositorio no incluye la base de datos ni los CSV de origen. Todo el código (estructura de tablas, consultas, lógica de limpieza) es publicable porque no contiene datos personales — son instrucciones, no información.

---

## El problema de partida

La planilla original tenía una sola hoja donde cada fila mezclaba datos de **cliente** (nombre, teléfono, ciudad) con datos de **pedido** (producto, talle, fecha, estado). Esto generaba:

- Redundancia: el mismo cliente repetido en decenas de filas, uno por cada compra.
- Inconsistencia: el mismo dato (ej. teléfono) escrito de formas distintas según quién cargó la fila.
- Imposibilidad de responder preguntas simples como *"¿cuánto compró tal cliente en total?"* sin revisar la planilla entera a mano.

## Decisión de diseño: separar en dos entidades

Se aplicó una regla simple para decidir a qué tabla pertenece cada columna:

> Si el dato **puede cambiar entre dos pedidos del mismo cliente**, pertenece a `pedidos`. Si describe la identidad de la persona, pertenece a `clientes`.

Con esa regla se llegó al siguiente esquema:

```sql
CREATE TABLE clientes (
    id_cliente SERIAL PRIMARY KEY,
    nombre TEXT,
    telefono VARCHAR(20),
    medio_contacto VARCHAR(30),
    provincia_ciudad VARCHAR(50),
    es_revendedor BOOLEAN
);

CREATE TABLE pedidos (
    id_pedido SERIAL PRIMARY KEY,
    id_cliente INT REFERENCES clientes(id_cliente),
    producto VARCHAR(50),
    modelo VARCHAR(50),
    talle VARCHAR(10),
    estado VARCHAR(20),
    fecha_pedido DATE,
    fecha_entrega DATE,
    forma_pago VARCHAR(30),
    medio_envio VARCHAR(30),
    canal_venta VARCHAR(30),
    usuario_plataforma VARCHAR(50),
    valor_unitario NUMERIC(10,2),
    cantidad INT,
    total NUMERIC(10,2) GENERATED ALWAYS AS (valor_unitario * cantidad) STORED
);
```

**Decisiones puntuales que valieron la pena documentar:**

- `NUMERIC` en vez de `FLOAT` para valores monetarios, para evitar errores de redondeo en cálculos de dinero.
- `VARCHAR` en vez de `INT` para teléfono: preserva ceros a la izquierda y formatos no numéricos.
- `total` como columna generada (`GENERATED ALWAYS AS ... STORED`) en vez de cargada a mano, para que nunca quede desincronizada de `valor_unitario × cantidad`.
- `canal_venta` vive en `pedidos`, no en `clientes`: un mismo cliente puede comprar por distintos canales en distintas compras, y se necesita conservar ese historial para análisis de origen de ventas.
- `es_revendedor` se agregó como campo booleano en `clientes` tras descartar el campo original "vendedor" de la planilla, que en realidad identificaba a un integrante del equipo interno que atendía la venta, no si el cliente revendía.

## Estrategia de migración: tabla de staging

Antes de cargar los datos limpios en `clientes` y `pedidos`, cada bloque de filas del Excel se exporta a CSV y se importa a una **tabla temporal** (staging), con todas las columnas en `VARCHAR` para no rechazar ningún dato crudo por formato:

```sql
CREATE TABLE temp_clientes_bloque1 (
    nombre_cliente VARCHAR(100),
    comentario VARCHAR(100),
    enviado VARCHAR(50),
    usuario VARCHAR(100),
    telefono VARCHAR(50),
    talle VARCHAR(20),
    consulta_por VARCHAR(150),
    vendedor VARCHAR(50),
    envio VARCHAR(100)
);
```

Recién desde ahí se limpia y se migra hacia las tablas definitivas.

## Limpieza de datos

Casos reales encontrados y su resolución:

**Nombres vacíos con usuario de MercadoLibre disponible** — en ventas por ML, el nombre real de la persona no siempre estaba cargado, pero sí su usuario de la plataforma. Se decidió usar el usuario como identificador en esos casos, para no perder trazabilidad del cliente:

```sql
UPDATE temp_clientes_bloque1
SET nombre_cliente = usuario
WHERE nombre_cliente IS NULL;
```

**Teléfonos vacíos** — se estandarizaron a `'S/D'` en vez de dejarlos en `NULL`, como marca explícita de "dato no disponible":

```sql
UPDATE temp_clientes_bloque1
SET telefono = 'S/D'
WHERE telefono IS NULL;
```

**Error real: valor demasiado largo para `VARCHAR(20)`**

```
ERROR: el valor es demasiado largo para el tipo character varying(20)
SQL state: 22001
```

Se investigó con una consulta antes de asumir la causa:

```sql
SELECT telefono, LENGTH(telefono) AS largo
FROM temp_clientes_bloque1
ORDER BY largo DESC;
```

La causa fue una celda con texto adicional pegado al número (ej. una aclaración de a quién pertenecía ese teléfono). Se corrigió puntualmente la fila afectada en vez de simplemente ensanchar la columna, preservando el dato real:

```sql
UPDATE temp_clientes_bloque1
SET telefono = 'nuevo_valor'
WHERE telefono = 'valor_viejo_exacto';
```

## Evitar duplicar clientes entre bloques y entre pedidos del mismo cliente

Dentro de un mismo bloque, un cliente puede aparecer varias veces (una fila por cada producto comprado). Se usó `DISTINCT` para insertarlo una sola vez:

```sql
INSERT INTO clientes (nombre, telefono, medio_contacto, provincia_ciudad, es_revendedor)
SELECT DISTINCT nombre_cliente, telefono, NULL, NULL, FALSE
FROM temp_clientes_bloque1
WHERE nombre_cliente NOT IN (SELECT nombre FROM clientes);
```

El `WHERE nombre_cliente NOT IN (...)` es necesario al migrar bloques sucesivos: sin él, un cliente que ya compró en un bloque anterior se insertaría de nuevo con un `id_cliente` distinto, rompiendo el historial. Antes de ejecutar el `INSERT`, se verificó el comportamiento esperado con consultas de solo lectura:

```sql
-- nombres que ya existen (se van a descartar)
SELECT DISTINCT nombre_cliente FROM temp_clientes_bloque1
WHERE nombre_cliente IN (SELECT nombre FROM clientes);

-- nombres nuevos (se van a insertar)
SELECT DISTINCT nombre_cliente FROM temp_clientes_bloque1
WHERE nombre_cliente NOT IN (SELECT nombre FROM clientes);
```

Cuando la suma de ambos resultados no coincidía con el total de filas del bloque, se usó `GROUP BY` para confirmar que la diferencia correspondía a nombres repetidos dentro del propio bloque:

```sql
SELECT nombre_cliente, COUNT(*) AS veces
FROM temp_clientes_bloque1
GROUP BY nombre_cliente
ORDER BY veces DESC;
```

## Resolver la clave foránea con JOIN

La tabla de staging no contiene `id_cliente` (solo el nombre como texto), pero `pedidos` lo necesita como clave foránea. Se resolvió cruzando ambas tablas por nombre:

```sql
INSERT INTO pedidos (id_cliente, producto, modelo, talle, usuario_plataforma, estado,
    fecha_pedido, fecha_entrega, forma_pago, medio_envio, canal_venta, valor_unitario, cantidad)
SELECT clientes.id_cliente, consulta_por, consulta_por, talle, usuario,
    NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL
FROM temp_clientes_bloque1
JOIN clientes ON clientes.nombre = temp_clientes_bloque1.nombre_cliente;
```

`producto` y `modelo` se cargan ambos con el mismo texto de origen (`consulta_por`), ya que en la planilla original ambos datos venían mezclados en una sola columna sin separador consistente. Fue una decisión consciente: duplicar el dato en ambas columnas en vez de perder información, dejando la separación fina para una etapa posterior si se vuelve necesaria.

## Checklist de migración por bloque

Proceso repetible aplicado a cada bloque de filas de la planilla original:

1. Exportar el bloque a CSV desde Excel.
2. Crear o vaciar la tabla de staging.
3. Importar el CSV (con encabezado activado).
4. `UPDATE` para completar `nombre_cliente` con `usuario` donde falte.
5. `UPDATE` para completar `telefono` con `'S/D'` donde falte.
6. Revisar con `SELECT *` si aparecen casos de formato irregular.
7. Verificar con `SELECT ... WHERE ... IN/NOT IN` cuántos nombres son nuevos antes de insertar.
8. `INSERT INTO clientes ... WHERE nombre_cliente NOT IN (...)`.
9. `INSERT INTO pedidos ... JOIN clientes ON ...`.

## Estado actual

- Esquema de `clientes` y `pedidos` diseñado, creado y validado con datos reales.
- Proceso de migración por bloques probado y documentado, con más de 500 filas migradas exitosamente.

## Próximos pasos

- Diseñar desde cero el módulo de control de stock (tabla de inventario), como proyecto independiente y ya definido a nivel de negocio.
- Diseñar el módulo de costos: `insumos`, `productos`, `despiece`, `proveedores`, `insumo_proveedor` — un caso de relación muchos-a-muchos resuelto con tablas intermedias, para que el costo de cada producto se recalcule automáticamente al actualizar el precio de un insumo.
- Construir una aplicación de escritorio en C# (.NET) que consuma esta base de datos, para que el uso diario no requiera conocimientos de SQL.

---

*Este repositorio forma parte de un laboratorio de práctica más amplio, que incluye simulación de infraestructura de red multi-sucursal en EVE-NG con monitoreo Zabbix, en el marco de mi formación como Analista de Sistemas.*
