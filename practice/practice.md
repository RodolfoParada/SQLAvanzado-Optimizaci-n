- Sistema avanzado de análisis de ventas
USE ecommerce;

-- Crear índices para optimización
CREATE INDEX idx_pedidos_fecha ON pedidos(fecha_pedido);
CREATE INDEX idx_pedidos_usuario ON pedidos(usuario_id);
CREATE INDEX idx_detalle_pedido ON detalle_pedidos(pedido_id, producto_id);
CREATE INDEX idx_productos_categoria ON productos(categoria_id);
CREATE INDEX idx_productos_activo ON productos(activo);

-- 1. ANÁLISIS DE VENTAS POR PERÍODO
DELIMITER //

CREATE PROCEDURE analisis_ventas_periodo(
  IN fecha_inicio DATE,
  IN fecha_fin DATE
)
BEGIN
  SELECT
    'Resumen del Período' AS tipo,
    COUNT(DISTINCT p.id) AS pedidos_totales,
    COUNT(DISTINCT p.usuario_id) AS clientes_unicos,
    SUM(p.total) AS ingresos_totales,
    AVG(p.total) AS ticket_promedio,
    SUM(dp.cantidad) AS productos_vendidos
  FROM pedidos p
  JOIN detalle_pedidos dp ON p.id = dp.pedido_id
  WHERE p.fecha_pedido BETWEEN fecha_inicio AND fecha_fin
    AND p.estado = 'completado';
END //

DELIMITER ;

-- Uso
CALL analisis_ventas_periodo('2024-01-01', '2024-12-31');

-- 2. PRODUCTOS MÁS VENDIDOS CON SUBCONSULTAS
SELECT
  p.nombre,
  p.precio,
  ventas.total_vendido,
  ventas.ingresos_generados,
  CASE
    WHEN ventas.total_vendido > (SELECT AVG(total_vendido) FROM (
      SELECT SUM(dp.cantidad) AS total_vendido
      FROM detalle_pedidos dp
      GROUP BY dp.producto_id
    ) avg_ventas) THEN 'Alto'
    WHEN ventas.total_vendido > (SELECT AVG(total_vendido) * 0.5 FROM (
      SELECT SUM(dp.cantidad) AS total_vendido
      FROM detalle_pedidos dp
      GROUP BY dp.producto_id
    ) avg_ventas) THEN 'Medio'
    ELSE 'Bajo'
  END AS rendimiento
FROM productos p
JOIN (
  SELECT
    dp.producto_id,
    SUM(dp.cantidad) AS total_vendido,
    SUM(dp.cantidad * dp.precio_unitario) AS ingresos_generados
  FROM detalle_pedidos dp
  JOIN pedidos ped ON dp.pedido_id = ped.id
  WHERE ped.estado = 'completado'
  GROUP BY dp.producto_id
) ventas ON p.id = ventas.producto_id
ORDER BY ventas.total_vendido DESC;

-- 3. ANÁLISIS DE CLIENTES CON FUNCIONES AGREGADAS
SELECT
  CASE
    WHEN total_gastado > 500 THEN 'VIP'
    WHEN total_gastado > 200 THEN 'Premium'
    WHEN total_gastado > 50 THEN 'Regular'
    ELSE 'Nuevo'
  END AS segmento_cliente,
  COUNT(*) AS cantidad_clientes,
  AVG(total_gastado) AS gasto_promedio_segmento,
  MIN(total_gastado) AS gasto_minimo,
  MAX(total_gastado) AS gasto_maximo,
  SUM(total_gastado) AS ingresos_segmento
FROM (
  SELECT
    u.id,
    COALESCE(SUM(p.total), 0) AS total_gastado
  FROM usuarios u
  LEFT JOIN pedidos p ON u.id = p.usuario_id AND p.estado = 'completado'
  GROUP BY u.id
) cliente_gastos
GROUP BY
  CASE
    WHEN total_gastado > 500 THEN 'VIP'
    WHEN total_gastado > 200 THEN 'Premium'
    WHEN total_gastado > 50 THEN 'Regular'
    ELSE 'Nuevo'
  END
ORDER BY ingresos_segmento DESC;

-- 4. REPORTE DE INVENTARIO CON HAVING
SELECT
  c.nombre AS categoria,
  COUNT(p.id) AS productos_activos,
  SUM(p.stock) AS stock_total,
  AVG(p.precio) AS precio_promedio,
  MIN(p.precio) AS precio_minimo,
  MAX(p.precio) AS precio_maximo,
  SUM(p.precio * p.stock) AS valor_inventario
FROM categorias c
JOIN productos p ON c.id = p.categoria_id
WHERE p.activo = 1
GROUP BY c.id, c.nombre
HAVING SUM(p.stock) > 10  -- Solo categorías con stock significativo
  AND COUNT(p.id) >= 2     -- Al menos 2 productos
ORDER BY valor_inventario DESC;

-- 5. ANÁLISIS DE TENDENCIAS CON VENTANAS
SELECT
  DATE_FORMAT(fecha_pedido, '%Y-%m') AS mes,
  COUNT(*) AS pedidos_mes,
  SUM(total) AS ingresos_mes,
  AVG(SUM(total)) OVER (
    ORDER BY DATE_FORMAT(fecha_pedido, '%Y-%m')
    ROWS 2 PRECEDING
  ) AS promedio_movil_3_meses,
  SUM(total) - LAG(SUM(total)) OVER (
    ORDER BY DATE_FORMAT(fecha_pedido, '%Y-%m')
  ) AS diferencia_mes_anterior
FROM pedidos
WHERE estado = 'completado'
  AND fecha_pedido >= '2024-01-01'
GROUP BY DATE_FORMAT(fecha_pedido, '%Y-%m')
ORDER BY mes;

-- 6. SISTEMA DE RECOMENDACIONES
SELECT
  p.nombre AS producto,
  GROUP_CONCAT(DISTINCT c2.nombre) AS categorias_compradas_juntas
FROM productos p
JOIN detalle_pedidos dp1 ON p.id = dp1.producto_id
JOIN detalle_pedidos dp2 ON dp1.pedido_id = dp2.pedido_id AND dp1.producto_id != dp2.producto_id
JOIN productos c2 ON dp2.producto_id = c2.id
WHERE p.id = 1  -- Recomendaciones para el producto con ID 1
GROUP BY p.id, p.nombre
ORDER BY COUNT(*) DESC
LIMIT 5;

-- 7. TRANSACCIÓN PARA PROCESAR PEDIDO COMPLETO
DELIMITER //

CREATE PROCEDURE procesar_pedido_completo(
  IN usuario_id INT,
  IN productos_json JSON
)
BEGIN
  DECLARE pedido_id INT;
  DECLARE producto_id INT;
  DECLARE cantidad INT;
  DECLARE precio_actual DECIMAL(10,2);
  DECLARE i INT DEFAULT 0;
  DECLARE num_productos INT;

  -- Handler para errores
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;

  START TRANSACTION;

  -- Crear pedido
  INSERT INTO pedidos (usuario_id, fecha_pedido, estado, total)
  VALUES (usuario_id, NOW(), 'procesando', 0.00);

  SET pedido_id = LAST_INSERT_ID();
  SET num_productos = JSON_LENGTH(productos_json);

  -- Procesar cada producto
  WHILE i < num_productos DO
    SET producto_id = JSON_EXTRACT(productos_json, CONCAT('$[', i, '].id'));
    SET cantidad = JSON_EXTRACT(productos_json, CONCAT('$[', i, '].cantidad'));

    -- Verificar stock disponible
    SELECT precio INTO precio_actual
    FROM productos
    WHERE id = producto_id AND activo = 1 FOR UPDATE;

    IF precio_actual IS NULL THEN
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Producto no disponible';
    END IF;

    IF (SELECT stock FROM productos WHERE id = producto_id) < cantidad THEN
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuficiente';
    END IF;

    -- Insertar detalle del pedido
    INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
    VALUES (pedido_id, producto_id, cantidad, precio_actual);

    -- Actualizar stock
    UPDATE productos SET stock = stock - cantidad WHERE id = producto_id;

    SET i = i + 1;
  END WHILE;

  -- Calcular total del pedido
  UPDATE pedidos SET
    total = (
      SELECT SUM(cantidad * precio_unitario)
      FROM detalle_pedidos
      WHERE pedido_id = pedidos.id
    ),
    estado = 'confirmado'
  WHERE id = pedido_id;

  COMMIT;

  -- Retornar información del pedido
  SELECT
    p.id,
    p.fecha_pedido,
    p.total,
    p.estado,
    JSON_ARRAYAGG(
      JSON_OBJECT(
        'producto_id', dp.producto_id,
        'nombre', pr.nombre,
        'cantidad', dp.cantidad,
        'precio_unitario', dp.precio_unitario,
        'subtotal', dp.cantidad * dp.precio_unitario
      )
    ) AS productos
  FROM pedidos p
  JOIN detalle_pedidos dp ON p.id = dp.pedido_id
  JOIN productos pr ON dp.producto_id = pr.id
  WHERE p.id = pedido_id
  GROUP BY p.id, p.fecha_pedido, p.total, p.estado;

END //

DELIMITER ;

-- 8. OPTIMIZACIÓN FINAL: EXPLICAR CONSULTAS LENTAS
EXPLAIN SELECT
  u.nombre,
  COUNT(p.id) AS pedidos_totales,
  SUM(p.total) AS total_gastado
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id
WHERE u.activo = 1
GROUP BY u.id, u.nombre
HAVING COUNT(p.id) > 0
ORDER BY total_gastado DESC;

-- Crear índices recomendados
CREATE INDEX idx_usuarios_activo ON usuarios(activo);
CREATE INDEX idx_pedidos_usuario_total ON pedidos(usuario_id, total);