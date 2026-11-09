# Administracion-RendimientoSQL
# Tutorial: Administración del Rendimiento en SQL (Optimización de Consultas Lentas)

Este repositorio contiene la guía paso a paso y los scripts para reproducir una prueba de concepto sobre la **Administración del Rendimiento en SQL**, analizando el comportamiento de un plan de ejecución ante un volumen masivo de datos (1,000,000 de registros) y la optimización mediante índices compuestos en MySQL.

---

## 1. Introducción
La administración del rendimiento (*Performance Tuning*) busca optimizar el uso de recursos (CPU, Memoria, E/S de disco) y reducir el tiempo de respuesta de las consultas en bases de datos relacionales. 

En este tutorial demostramos cómo una consulta sobre un millón de registros sin optimizar genera un escaneo completo de tabla (*Full Table Scan*), y cómo la aplicación de un índice compuesto (*B-Tree*) reduce drásticamente el tiempo de ejecución.

---

## 2. Entorno y Herramientas Utilizadas
* **Servidor de Base de Datos:** MySQL / MariaDB (XAMPP).
* **Gestor Web:** phpMyAdmin.
* **Lenguaje:** SQL.

---

## 3. Guía Paso a Paso

### Paso 3.1: Creación de la Base de Datos y Tabla
Creación del esquema inicial sin índices (únicamente la clave primaria auto-incremental):

```sql
CREATE DATABASE optimizacion_db;
USE optimizacion_db;

CREATE TABLE ventas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    cliente_id INT NOT NULL,
    fecha_venta DATETIME NOT NULL,
    monto DECIMAL(10, 2) NOT NULL,
    estado VARCHAR(20) NOT NULL,
    codigo_sucursal VARCHAR(10) NOT NULL
);
```
### Paso 3.2: Generación Masiva de Datos (1,000,000 Registros)
Para simular un volumen real y masivo, creamos un procedimiento almacenado que inserta 1,000,000 de filas en lotes (commits cada 10,000 registros):
```sql
DELIMITER //

CREATE PROCEDURE generar_ventas_masivas()
BEGIN
    DECLARE i INT DEFAULT 1;
    SET autocommit = 0;
    
    WHILE i <= 1000000 DO
        INSERT INTO ventas (cliente_id, fecha_venta, monto, estado, codigo_sucursal)
        VALUES (
            FLOOR(1 + (RAND() * 50000)),
            DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 730) DAY),
            ROUND(10 + (RAND() * 5000), 2),
            ELT(FLOOR(1 + (RAND() * 3)), 'Completado', 'Pendiente', 'Cancelado'),
            CONCAT('SUC-', LPAD(FLOOR(1 + (RAND() * 100)), 3, '0'))
        );
        
        IF i % 10000 = 0 THEN
            COMMIT;
        END IF;
        
        SET i = i + 1;
    END WHILE;
    
    COMMIT;
    SET autocommit = 1;
END //

DELIMITER ;

-- Ejecución
CALL generar_ventas_masivas();
```
<img width="1650" height="822" alt="Captura de pantalla 2026-11-08 225509" src="https://github.com/user-attachments/assets/7e7c3f3e-a73c-4fa1-bdc5-09b9d4ef6fb5" />

### Paso 3.3: Ejecución de la Consulta Lenta (Sin Optimización)
Ejecutamos la consulta compleja de agregación sobre el rango de fechas y estado:
```sql
EXPLAIN SELECT 
    codigo_sucursal, 
    COUNT(*) AS total_ventas, 
    SUM(monto) AS monto_total
FROM ventas
WHERE fecha_venta BETWEEN '2025-01-01' AND '2025-06-30'
  AND estado = 'Completado'
GROUP BY codigo_sucursal
ORDER BY monto_total DESC;
```
<img width="829" height="322" alt="Captura de pantalla 2026-11-08 225640" src="https://github.com/user-attachments/assets/4dfa2ff5-a207-48f7-b0b5-63f9fecc348d" />

Resultado de Análisis (Plan de Ejecución):
Tipo de búsqueda: type = ALL (Full Table Scan).

Filas examinadas: 996,816 filas.

Tiempo de ejecución real: 8.81 segundos.

### Paso 3.4: Estrategia de Optimización (Creación de Índice)
Creación de un índice compuesto sobre las columnas de filtrado (fecha_venta, estado):
```sql
CREATE INDEX idx_ventas_fecha_estado 
ON ventas (fecha_venta, estado);
```
### Paso 3.5: Ejecución de la Consulta Optimizada
Volvemos a ejecutar la consulta haciendo uso del índice creado:
```sql
EXPLAIN SELECT 
    codigo_sucursal, 
    COUNT(*) AS total_ventas, 
    SUM(monto) AS monto_total
FROM ventas FORCE INDEX (idx_ventas_fecha_estado)
WHERE fecha_venta BETWEEN '2025-01-01' AND '2025-06-30'
  AND estado = 'Completado'
GROUP BY codigo_sucursal
ORDER BY monto_total DESC;
```
<img width="1017" height="250" alt="Captura de pantalla 2026-11-08 230423" src="https://github.com/user-attachments/assets/98cfe9bd-f626-4e4c-bcfc-0a59daf3ef2b" />

Resultado de Análisis (Plan de Ejecución Optimizado):
Tipo de búsqueda: type = range.

Llave utilizada: key = idx_ventas_fecha_estado.

Reducción de lectura: Inspección acotada a través del árbol del índice.

## 4. Resultados y Conclusión
Rendimiento: Al sustituir la lectura secuencial de casi un millón de filas por una estructura indexada, eliminamos el costo de E/S en disco y reducimos la sobrecarga de CPU en la base de datos.

Conclusión: La optimización en SQL no consiste únicamente en escribir la sintaxis correcta, sino en analizar la forma en que el motor ejecuta la consulta (EXPLAIN) y diseñar estructuras de acceso a datos (como índices compuestos) que aceleren la recuperación de información a gran escala.

