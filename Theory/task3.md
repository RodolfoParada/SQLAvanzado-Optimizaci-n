Task 3: GROUP BY y HAVING (6 minutos)
GROUP BY agrupa filas con valores idénticos, HAVING filtra grupos.

GROUP BY Básico
-- Ventas por categoría
SELECT c.nombre AS categoria,
       COUNT(p.id) AS productos,
       SUM(p.stock) AS stock_total
FROM categorias c
LEFT JOIN productos p ON c.id = p.categoria_id
GROUP BY c.id, c.nombre
ORDER BY stock_total DESC;

-- Pedidos por usuario
SELECT u.nombre,
       COUNT(p.id) AS total_pedidos,
       SUM(p.total) AS total_gastado,
       AVG(p.total) AS promedio_pedido
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id
GROUP BY u.id, u.nombre
HAVING total_pedidos > 0  -- Solo usuarios con pedidos
ORDER BY total_gastado DESC;
HAVING vs WHERE
-- ❌ Incorrecto: WHERE no puede usar funciones agregadas
SELECT categoria_id, AVG(precio) AS precio_promedio
FROM productos
WHERE AVG(precio) > 50  -- ERROR
GROUP BY categoria_id;

-- ✅ Correcto: HAVING filtra después del GROUP BY
SELECT categoria_id, AVG(precio) AS precio_promedio
FROM productos
GROUP BY categoria_id
HAVING AVG(precio) > 50;

-- Combinación WHERE + HAVING
SELECT c.nombre AS categoria,
       COUNT(p.id) AS total_productos,
       AVG(p.precio) AS precio_promedio
FROM categorias c
JOIN productos p ON c.id = p.categoria_id
WHERE p.activo = 1                    -- Filtro antes de agrupar
GROUP BY c.id, c.nombre
HAVING COUNT(p.id) >= 3                -- Filtro después de agrupar
  AND AVG(p.precio) BETWEEN 20 AND 200 -- Múltiples condiciones
ORDER BY precio_promedio DESC;
GROUP BY con ROLLUP (MySQL)
-- Subtotales con ROLLUP
SELECT
  COALESCE(c.nombre, 'TOTAL') AS categoria,
  YEAR(p.fecha_pedido) AS anio,
  SUM(dp.cantidad * dp.precio_unitario) AS ventas
FROM categorias c
JOIN productos pr ON c.id = pr.categoria_id
JOIN detalle_pedidos dp ON pr.id = dp.producto_id
JOIN pedidos p ON dp.pedido_id = p.id
WHERE p.fecha_pedido >= '2023-01-01'
GROUP BY c.nombre, YEAR(p.fecha_pedido) WITH ROLLUP
ORDER BY c.nombre, anio;