CREATE DATABASE social_network;
USE social_network;

-- Tabla de Usuarios
CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50)
);

CREATE TABLE posts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    usuario_id INT,
    contenido TEXT,
    fecha_publicacion DATETIME,
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE TABLE comentarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    post_id INT,
    usuario_id INT,
    comentario TEXT,
    fecha_comentario DATETIME,
    FOREIGN KEY (post_id) REFERENCES posts(id),
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

-- Tabla de Seguidores (Relación muchos a muchos)
CREATE TABLE seguidores (
    seguidor_id INT NOT NULL,
    seguido_id INT NOT NULL,
    fecha_seguimiento TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (seguidor_id, seguido_id),
    FOREIGN KEY (seguidor_id) REFERENCES usuarios(id),
    FOREIGN KEY (seguido_id) REFERENCES usuarios(id)
);

-- Optimización
-- Índice para acelerar la carga del muro (feed) de un usuario
CREATE INDEX idx_posts_usuario_fecha ON posts(usuario_id, fecha_publicacion DESC);

-- Índice para búsquedas de comentarios por post
CREATE INDEX idx_comentarios_post ON comentarios(post_id);

-- Índice para verificar estados de cuenta rápidos
ALTER TABLE usuarios ADD COLUMN activo TINYINT(1) DEFAULT 1;
CREATE INDEX idx_usuarios_activo ON usuarios(activo);

-- Índice para acelerar la verificación de "Quién sigue a quién"
CREATE INDEX idx_seguidores_busqueda ON seguidores(seguido_id, seguidor_id);

-- Índice Full-Text para búsquedas de contenido
ALTER TABLE posts ADD FULLTEXT(idx_busqueda_contenido);

-- Índice para optimizar el reporte de engagement (visto más adelante)
CREATE INDEX idx_comentarios_usuario_fecha ON comentarios(usuario_id, fecha_comentario);

-- Transacciones: Publicación con comentario inicial
DELIMITER //

CREATE PROCEDURE publicar_con_nota(
    IN p_usuario_id INT,
    IN p_contenido TEXT,
    IN p_nota TEXT
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    START TRANSACTION;
        -- 1. Insertar el post
        INSERT INTO posts (usuario_id, contenido, fecha_publicacion) 
        VALUES (p_usuario_id, p_contenido, NOW());
        
        SET @last_post_id = LAST_INSERT_ID();

        -- 2. Insertar el comentario inicial
        INSERT INTO comentarios (post_id, usuario_id, comentario, fecha_comentario)
        VALUES (@last_post_id, p_usuario_id, p_nota, NOW());
    COMMIT;
END //

DELIMITER ;
-- consulta optimizada 
-- Consulta eficiente para obtener los últimos 10 posts de personas que sigo
SELECT p.id, p.contenido, p.fecha_publicacion, u.username
FROM posts p
JOIN usuarios u ON p.usuario_id = u.id
WHERE EXISTS (
    SELECT 1 FROM seguidores s 
    WHERE s.seguidor_id = 1 -- ID del usuario logueado
    AND s.seguido_id = p.usuario_id
)
ORDER BY p.fecha_publicacion DESC
LIMIT 10;

-- Reporte de Engagement con funciones de ventana
SELECT 
    u.username,
    COUNT(c.id) AS total_comentarios_recibidos,
    RANK() OVER (ORDER BY COUNT(c.id) DESC) as ranking_popularidad,
    SUM(COUNT(c.id)) OVER() as total_global_comentarios,
    ROUND(COUNT(c.id) * 100.0 / SUM(COUNT(c.id)) OVER(), 2) as porcentaje_impacto
FROM usuarios u
JOIN posts p ON u.id = p.usuario_id
JOIN comentarios c ON p.id = c.post_id
GROUP BY u.id, u.username;

-- Datos de prueba
-- 1. Insertar varios usuarios
INSERT INTO usuarios (username, activo) VALUES 
('juan_dev', 1),
('maria_tech', 1),
('pedro_sql', 1),
('ana_data', 1),
('luis_web', 0); -- Usuario inactivo para pruebas

-- 2. Insertar Posts (distribuidos en el tiempo)
INSERT INTO posts (usuario_id, contenido, fecha_publicacion) VALUES 
(1, 'Iniciando mi camino en SQL', '2023-10-01 10:00:00'),
(1, '¿Qué prefieren: JOIN o Subquery?', '2023-10-02 11:00:00'),
(2, 'Mi primer despliegue en la nube', '2023-10-01 12:00:00'),
(2, 'Python es increíble para datos', '2023-10-03 09:30:00'),
(3, 'Optimizando índices en MySQL', '2023-10-04 15:00:00'),
(4, 'Aprendiendo funciones de ventana hoy', '2023-10-05 08:00:00');

-- 3. Insertar Comentarios (para generar popularidad variable)
-- Primero aseguramos que los posts existan antes de comentar
-- Usamos (SELECT id FROM...) para no fallar si los IDs cambiaron

INSERT INTO comentarios (post_id, usuario_id, comentario, fecha_comentario) 
VALUES 
((SELECT id FROM posts WHERE contenido LIKE 'Iniciando%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'maria_tech'), 
 '¡Mucho éxito, Juan!', '2023-10-01 10:30:00'),
((SELECT id FROM posts WHERE contenido LIKE 'Iniciando%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'pedro_sql'), 
 'SQL es la base de todo.', '2023-10-01 11:00:00'),

-- Comentarios para el Post 2 (de juan_dev)
((SELECT id FROM posts WHERE contenido LIKE '%JOIN%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'ana_data'), 
 '¡Depende del caso de uso!', '2023-10-02 11:15:00'),

-- Comentario para el Post 3 (de maria_tech)
((SELECT id FROM posts WHERE contenido LIKE '%nube%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'juan_dev'), 
 '¡Felicidades Maria!', '2023-10-01 12:30:00'),

-- Comentarios para el Post 5 (de pedro_sql)
((SELECT id FROM posts WHERE contenido LIKE '%indices%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'juan_dev'), 
 'Buen post, muy técnico.', '2023-10-04 15:10:00'),

((SELECT id FROM posts WHERE contenido LIKE '%indices%' LIMIT 1), 
 (SELECT id FROM usuarios WHERE username = 'maria_tech'), 
 'Me sirvió mucho para mi DB.', '2023-10-04 15:20:00');

-- 4. Insertar Seguidores (Relaciones)
INSERT INTO seguidores (seguidor_id, seguido_id) VALUES 
(1, 2), -- Juan sigue a Maria
(1, 3), -- Juan sigue a Pedro
(2, 1), -- Maria sigue a Juan
(3, 1), -- Pedro sigue a Juan
(4, 1), -- Ana sigue a Juan
(4, 2); -- Ana sigue a Maria