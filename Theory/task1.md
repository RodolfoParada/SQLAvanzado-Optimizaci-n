Task 1: Subconsultas y Consultas Anidadas (8 minutos)
Subconsultas permiten consultas dentro de otras consultas para mayor flexibilidad.

Subconsultas en WHERE
-- Encontrar productos con precio mayor al promedio
SELECT nombre, precio
FROM productos
WHERE precio > (SELECT AVG(precio) FROM productos);

-- Usuarios que han hecho pedidos
SELECT nombre, email
FROM usuarios
WHERE id IN (
  SELECT DISTINCT usuario_id
  FROM pedidos
);

-- Productos no vendidos
SELECT nombre
FROM productos
WHERE id NOT IN (
  SELECT DISTINCT producto_id
  FROM detalle_pedidos
);

-- Empleados con salario mayor que el de su departamento
SELECT e.nombre, e.salario, e.departamento
FROM empleados e
WHERE e.salario > (
  SELECT AVG(salario)
  FROM empleados
  WHERE departamento = e.departamento
);
Subconsultas en FROM (Tablas Derivadas)
-- Ventas por categoría
SELECT categoria, total_ventas
FROM (
  SELECT c.nombre AS categoria,
         SUM(dp.cantidad * dp.precio_unitario) AS total_ventas
  FROM categorias c
  LEFT JOIN productos p ON c.id = p.categoria_id
  LEFT JOIN detalle_pedidos dp ON p.id = dp.producto_id
  GROUP BY c.id, c.nombre
) AS ventas_categoria
ORDER BY total_ventas DESC;

-- Ranking de productos por ventas
SELECT nombre, ventas_totales,
       RANK() OVER (ORDER BY ventas_totales DESC) AS ranking
FROM (
  SELECT p.nombre,
         COALESCE(SUM(dp.cantidad), 0) AS ventas_totales
  FROM productos p
  LEFT JOIN detalle_pedidos dp ON p.id = dp.producto_id
  GROUP BY p.id, p.nombre
) AS productos_ranking;
Subconsultas en SELECT
-- Información completa de pedidos
SELECT
  p.id,
  u.nombre AS cliente,
  p.fecha_pedido,
  p.total,
  (
    SELECT COUNT(*)
    FROM detalle_pedidos dp
    WHERE dp.pedido_id = p.id
  ) AS total_productos,
  (
    SELECT GROUP_CONCAT(pr.nombre)
    FROM detalle_pedidos dp
    JOIN productos pr ON dp.producto_id = pr.id
    WHERE dp.pedido_id = p.id
  ) AS productos
FROM pedidos p
JOIN usuarios u ON p.usuario_id = u.id;
Subconsultas Correlacionadas
-- Último pedido de cada usuario
SELECT u.nombre,
       p.fecha_pedido,
       p.total
FROM usuarios u
JOIN pedidos p ON u.id = p.usuario_id
WHERE p.fecha_pedido = (
  SELECT MAX(fecha_pedido)
  FROM pedidos
  WHERE usuario_id = u.id
);

-- Productos con ventas por encima del promedio de su categoría
SELECT p.nombre, p.categoria_id, ventas.total_vendido
FROM productos p
JOIN (
  SELECT dp.producto_id, SUM(dp.cantidad) AS total_vendido
  FROM detalle_pedidos dp
  GROUP BY dp.producto_id
) ventas ON p.id = ventas.producto_id
WHERE ventas.total_vendido > (
  SELECT AVG(cat_ventas.total_vendido)
  FROM (
    SELECT SUM(dp2.cantidad) AS total_vendido
    FROM productos p2
    JOIN detalle_pedidos dp2 ON p2.id = dp2.producto_id
    WHERE p2.categoria_id = p.categoria_id
    GROUP BY dp2.producto_id
  ) cat_ventas
);