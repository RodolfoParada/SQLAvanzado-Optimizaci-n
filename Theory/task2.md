Task 2: Funciones Agregadas (6 minutos)
Funciones que operan sobre grupos de filas para calcular estadísticas.

Funciones Básicas
-- COUNT: Contar filas
SELECT COUNT(*) FROM productos;                    -- Total productos
SELECT COUNT(*) FROM productos WHERE activo = 1;   -- Productos activos
SELECT COUNT(DISTINCT categoria_id) FROM productos; -- Categorías únicas

-- SUM: Suma de valores
SELECT SUM(stock) FROM productos;                  -- Total unidades
SELECT SUM(precio * stock) FROM productos;         -- Valor total inventario

-- AVG: Promedio
SELECT AVG(precio) FROM productos;                 -- Precio promedio
SELECT AVG(precio) FROM productos WHERE categoria_id = 1; -- Precio promedio categoría

-- MIN/MAX: Valores extremos
SELECT MIN(precio) FROM productos;                 -- Precio mínimo
SELECT MAX(precio) FROM productos;                 -- Precio máximo
SELECT MIN(fecha_pedido), MAX(fecha_pedido) FROM pedidos; -- Rango de fechas
Funciones de Grupo Avanzadas
-- Estadísticas de ventas por mes
SELECT
  DATE_FORMAT(fecha_pedido, '%Y-%m') AS mes,
  COUNT(*) AS pedidos,
  SUM(total) AS ingresos,
  AVG(total) AS ticket_promedio,
  MIN(total) AS pedido_minimo,
  MAX(total) AS pedido_maximo
FROM pedidos
GROUP BY DATE_FORMAT(fecha_pedido, '%Y-%m')
ORDER BY mes;

-- Análisis de productos
SELECT
  categoria_id,
  COUNT(*) AS total_productos,
  AVG(precio) AS precio_promedio,
  MIN(precio) AS precio_minimo,
  MAX(precio) AS precio_maximo,
  SUM(stock) AS stock_total,
  AVG(stock) AS stock_promedio
FROM productos
WHERE activo = 1
GROUP BY categoria_id;