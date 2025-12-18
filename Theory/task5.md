Task 5: Transacciones y ACID (4 minutos)
Transacciones garantizan que las operaciones sean atómicas y consistentes.

Conceptos ACID
-- ACID:
-- Atomicity: Todo o nada
-- Consistency: Estado consistente
-- Isolation: Transacciones independientes
-- Durability: Cambios permanentes

-- Transacción básica
START TRANSACTION;

INSERT INTO pedidos (usuario_id, fecha_pedido, total)
VALUES (1, NOW(), 150.00);

INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
VALUES (LAST_INSERT_ID(), 1, 1, 1299.99);

UPDATE productos SET stock = stock - 1 WHERE id = 1;

COMMIT;  -- Confirmar cambios

-- Transacción con manejo de errores
DELIMITER //

CREATE PROCEDURE crear_pedido_seguro(
  IN usuario_id INT,
  IN producto_id INT,
  IN cantidad INT
)
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;

  START TRANSACTION;

  -- Verificar stock
  IF (SELECT stock FROM productos WHERE id = producto_id) < cantidad THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuficiente';
  END IF;

  -- Crear pedido
  INSERT INTO pedidos (usuario_id, fecha_pedido, total)
  VALUES (usuario_id, NOW(), 0);

  SET @pedido_id = LAST_INSERT_ID();

  -- Agregar detalle
  INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
  SELECT @pedido_id, id, cantidad, precio
  FROM productos WHERE id = producto_id;

  -- Actualizar stock
  UPDATE productos SET stock = stock - cantidad WHERE id = producto_id;

  -- Calcular total
  UPDATE pedidos SET total = (
    SELECT SUM(cantidad * precio_unitario)
    FROM detalle_pedidos
    WHERE pedido_id = @pedido_id
  ) WHERE id = @pedido_id;

  COMMIT;
END //

DELIMITER ;

-- Uso del procedimiento
CALL crear_pedido_seguro(1, 2, 2);
Niveles de Aislamiento
-- Ver nivel de aislamiento actual
SELECT @@transaction_isolation;

-- Cambiar nivel de aislamiento
SET SESSION transaction_isolation = 'READ-COMMITTED';

-- Niveles:
-- READ UNCOMMITTED: Puede leer datos no confirmados
-- READ COMMITTED: Solo datos confirmados
-- REPEATABLE READ: Garantiza lecturas consistentes (MySQL default)
-- SERIALIZABLE: Máxima consistencia, menor concurrencia