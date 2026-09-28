# Documentación de Conectividad y Transformación de Datos - Set de Ventas Legacy

## 1. Estrategia de Extracción de Datos (ETL)
Debido a restricciones de autenticación y bloqueos de seguridad perimetral en la URL pública original de Google Sheets, se procedió a realizar una descarga manual del archivo en su formato nativo de **Microsoft Excel (.xlsx)**. Esto garantizó la disponibilidad local del set de datos sin alterar de ninguna forma la estructura ni la integridad de los registros del sistema fuente original.

---

## 2. Criterio de Normalización y Arquitectura del Modelo
Para cumplir con las buenas prácticas de modelado de datos y optimizar el rendimiento, se identificó que el archivo original mezclaba entidades conceptuales distintas en una sola sábana de datos. Se normalizó la estructura separando la información en **dos tablas independientes** dentro de Power Query:

*   **`dim_clientes` (Tabla de Dimensión):** Almacena de forma exclusiva los atributos maestros y geográficos de los compradores (`ID_cliente`, `Nombre_cliente`, `Mail_cliente`, `Telefono_cliente`, `Ciudad_cliente`, `Provincia_cliente`, `Segmento_cliente`, `Estado_activo` y `Fecha_alta_cliente`). Su clave primaria es `ID_cliente`.
*   **`fact_ventas` (Tabla de Hechos):** Almacena los registros métricos y cuantitativos de las transacciones comerciales (`Codigo_operacion`, `Fecha_venta`, `ID_producto`, `Descripcion_producto`, `Rubro_producto`, `Cantidad`, `Precio_unitario`, `Descuento_porcentaje`, `Total_venta`, `Codigo_moneda` y `Canal_venta`). Mantiene la columna `ID_cliente` como clave foránea (Foreign Key) para permitir la relación lógica entre ambas tablas (relación de 1 a varios, 1:N).

---

## 3. Justificación Técnica de las Transformaciones en Power Query

### A. Renombrado de Columnas (Estandarización)
Se eliminaron todos los prefijos técnicos crudos y las abreviaciones confusas del sistema legacy (ejemplos: `PU_VTA`, `FLG_ACT`, `F_VTA`). Se renombraron bajo la nomenclatura **Pascal_Snake_Case** (ejemplo: `Precio_unitario`, `Estado_activo`, `Fecha_venta`). Esto asegura que el modelo de datos sea completamente legible y autoexplicativo para cualquier usuario de negocio o analista de la empresa.

### B. Corrección de Tipos de Datos
Se asignaron tipificaciones estrictas en las cabeceras de las columnas según su naturaleza matemática:
*   **Fechas (`Fecha_venta` y `Fecha_alta_cliente`):** Se transformaron de texto largo (DateTime de sistema) a tipo **Fecha (Date)**. Esto eliminó las horas residuales en cero (`00:00:00.000`) y es un paso mandatorio para habilitar las funciones DAX de Inteligencia de Tiempo (*Time Intelligence*).
*   **Montos Económicos (`Precio_unitario` y `Total_venta`):** Se configuraron como **Número decimal fijo (Currency)**. Esto mitiga el riesgo de errores de redondeo de punto flotante al realizar sumas acumulativas o promedios sobre millones de transacciones.
*   **Identificadores (`Codigo_operacion` e `ID_cliente`):** Se configuraron como **Texto (Text)**. Aunque contienen números, los IDs actúan como variables categóricas sobre las cuales jamás se realizarán operaciones aritméticas. Tratarlos como texto optimiza el indexado de las relaciones en Power BI.
*   **Métricas de Control (`Cantidad` y `Descuento_porcentaje`):** Se asignó **Número entero** a las cantidades (debido a la granularidad unitaria del negocio) y **Número decimal** a los descuentos para interpretar correctamente valores como `0.05` (5%).

### C. Tratamiento de Valores Nulos (`null`)
Al activar las barras de calidad de datos, se detectó que el 5% inferior del archivo original (a partir de la fila 982) correspondía a registros completamente vacíos (`null`) generados por un error en la exportación del sistema legacy. Para evitar la distorsión de promedios generales y totales en las visualizaciones, se aplicó un filtro de exclusión de vacíos en las columnas principales, purgando la tabla de filas muertas.

### D. Eliminación de Duplicados
Se aplicó la limpieza de registros repetidos utilizando un criterio diferenciado según la tabla:
*   En **`dim_clientes`**, se removieron duplicados sobre la columna `ID_cliente` para garantizar que cada comprador aparezca una única vez en la lista maestra.
*   En **`fact_ventas`**, se removieron duplicados únicamente sobre la columna `Codigo_operacion` para asegurar la unicidad de las facturas, permitiendo intencionalmente que los IDs de clientes se repitan, reflejando así la recurrencia real de compra de los usuarios.

## 4. Resolución de Conflictos de Integridad Referencial (Caché del Motor)
Durante el despliegue final del modelo en la interfaz de relaciones, el sistema arrojó una advertencia de restricción debido a la presencia de un registro con valor en blanco (`blank / null`) residual dentro de la columna clave primaria `ID_cliente` en la tabla `dim_clientes`. Esta colisión de integridad referencial impedía la asignación de una cardinalidad pura. Para solucionarlo de raíz, se regresó al Editor de Power Query y se aplicó un filtro estricto de exclusión de vacíos en dicha columna maestro. Esta purga definitiva eliminó la celda huérfana, destrabando el sistema y permitiendo consolidar con éxito la relación Uno a Varios (1:*) requerida para el modelo en estrella.
