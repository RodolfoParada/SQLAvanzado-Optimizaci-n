Task 4: Índices y Optimización (6 minutos)
Índices mejoran el rendimiento de consultas pero afectan las operaciones de escritura.

Tipos de Índices
-- Índice en columna única
CREATE INDEX idx_productos_precio ON productos(precio);
CREATE INDEX idx_usuarios_email ON usuarios(email);

-- Índice compuesto
CREATE INDEX idx_pedidos_usuario_fecha ON pedidos(usuario_id, fecha_pedido);

-- Índice único
CREATE UNIQUE INDEX idx_productos_codigo ON productos(codigo);

-- Índice de texto completo
CREATE FULLTEXT INDEX idx_productos_descripcion ON productos(descripcion);

-- Ver índices existentes
SHOW INDEX FROM productos;
Optimización de Consultas
-- ❌ Consulta no optimizada
SELECT * FROM productos WHERE precio > 100 AND categoria_id = 1;
-- Necesita: INDEX(precio), INDEX(categoria_id), o mejor INDEX(categoria_id, precio)

-- ✅ Consulta optimizada
CREATE INDEX idx_productos_cat_precio ON productos(categoria_id, precio);

-- Análisis de consultas lentas
EXPLAIN SELECT * FROM productos WHERE precio > 100 AND categoria_id = 1;

-- Optimización con índices compuestos
CREATE INDEX idx_pedidos_fecha_usuario ON pedidos(fecha_pedido DESC, usuario_id);

-- Para consultas de rango
CREATE INDEX idx_productos_precio_categoria ON productos(precio, categoria_id);

-- Para búsquedas de texto
SELECT * FROM productos
WHERE MATCH(descripcion) AGAINST('laptop gaming' IN NATURAL LANGUAGE MODE);