# Documentación del proyecto: dashboard comercial de Andes Capital Real Estate

Proceso completo de construcción del dashboard en Power BI Desktop. El resumen y las capturas están en el [README](README.md).

## Datos

**hecho_ventas_propiedades**
Tabla de hechos con las transacciones de venta (precio, cliente, propiedad, canal, fecha). Contiene 8.500 ventas, una por fila, identificadas por `id_venta`. Se conecta con las dimensiones mediante `id_cliente`, `id_propiedad` y `fecha_venta`. Durante la preparación se corrigió `porcentaje_comision`, que se leía como entero por la configuración regional, y se definieron `precio_venta` y `monto_comision` como decimales.

**dim_clientes**
Información y segmentación de clientes. Contiene 3.500 clientes únicos (`id_cliente`) con su segmento de comprador, país y ciudad. Fue necesario promover la primera fila como encabezado.

**dim_propiedades**
Características de las propiedades (tipo, tamaño, ubicación, etc.). Contiene 8.000 propiedades únicas (`id_propiedad`) con tipo, ciudad, barrio, habitaciones, tamaño, precio publicado y categoría. Se definió `precio_publicado` como decimal.

**dim_fecha**
Tabla calendario creada con DAX (`CALENDAR` y `ADDCOLUMNS`) a partir de la fecha mínima y máxima de `fecha_venta`. Incluye las columnas Date, Año, Mes, Mes Numero y Año-Mes; se marcó como tabla de fechas y el mes se ordena por su número.

**Validación de calidad:** en las tres tablas de origen todas las columnas tienen 100 % de valores válidos, sin errores ni vacíos, y las claves de cada tabla no tienen duplicados.

## Preguntas que responde el dashboard

**Desempeño general**

- ¿Cuál es el **ingreso total** generado por la empresa?
- ¿Cuántas **propiedades se han vendido**?
- ¿Cuál es el **precio promedio de venta**?
- ¿Cuál es la **comisión total** generada por la empresa?

**Análisis comercial**

- ¿Qué **tipo de propiedad genera más ingresos**?
- ¿Qué **segmento de clientes compra más propiedades**?
- ¿Qué canal de venta **genera mayor ingresos**?

**Análisis temporal**

- ¿Cómo evolucionan las **ventas en el tiempo**?
- ¿El negocio está **creciendo año contra año**?
- ¿Cómo va el desempeño **acumulado del año actual (YTD)**?

**Cohortes de clientes**

- ¿Los clientes **vuelven a comprar** después de su primera compra?
- ¿Qué **cohortes de clientes generan más compras/ingresos con el tiempo**?

## 1. Limpieza de datos

**1.1. Tipos de datos**
- `fecha_venta` se definió como tipo **Fecha**.
- Campos numéricos:
  - Decimales: `precio_venta`, `monto_comision`, `porcentaje_comision` y `precio_publicado`.
  - Enteros: `habitaciones` y `tamano_m2`.
  - Claves y campos categóricos como texto.
- `porcentaje_comision` se cargaba como entero (por ejemplo, 473 en lugar de 0,0473) porque el archivo usa punto decimal y la configuración regional lo interpretaba como separador de miles. Se corrigió leyendo la columna como texto y convirtiéndola a decimal con configuración regional Inglés (Estados Unidos). En el modelo se le aplicó formato **porcentaje** (4,73 %).
- Validación: `monto_comision` coincide aproximadamente con `precio_venta × porcentaje_comision`.
- En `dim_clientes` fue necesario usar la primera fila como encabezado.

**1.2. Valores nulos**
- Con el perfil de columnas sobre todo el conjunto de datos, las tres tablas muestran 100 % de valores válidos, 0 % de errores y 0 % de vacíos en todas las columnas.
- Al no existir nulos, no fue necesario eliminar ni imputar registros; se conservan todas las filas.

**1.3. Duplicados**
- `dim_clientes`: 3.500 registros y 3.500 valores distintos en `id_cliente`.
- `dim_propiedades`: 8.000 registros y 8.000 valores distintos en `id_propiedad`.
- Adicionalmente, `hecho_ventas_propiedades` tiene 8.500 registros y 8.500 valores distintos en `id_venta`.
- No existen duplicados en las claves primarias, por lo que las relaciones uno a muchos con la tabla de hechos se pueden construir correctamente.

## 2. Tabla calendario

Se creó la tabla calendario `dim_fecha` en Power BI como tabla calculada con DAX:

    dim_fecha =
    ADDCOLUMNS(
        CALENDAR( MIN(hecho_ventas_propiedades[fecha_venta]), MAX(hecho_ventas_propiedades[fecha_venta]) ),
        "Año", YEAR([Date]),
        "Mes", FORMAT([Date], "MMMM"),
        "Mes Numero", MONTH([Date]),
        "Año-Mes", FORMAT([Date], "YYYY-MM")
    )

- `CALENDAR` genera una fila por cada día entre la fecha mínima y la fecha máxima de `fecha_venta`, por lo que el rango se ajusta automáticamente si cambian los datos.
- `ADDCOLUMNS` agrega las columnas Año, Mes (nombre), Mes Numero y Año-Mes (formato YYYY-MM).

Ajustes realizados:
- La tabla se marcó como **tabla de fechas** usando la columna `Date`, lo que habilita las funciones de inteligencia de tiempo.
- La columna `Mes` se ordenó por `Mes Numero` para que los meses aparezcan en orden cronológico y no alfabético.
- `Date` se definió como tipo Fecha con formato corto, y `Año` y `Mes Numero` como "No resumir" para que no se sumen en los visuales.

## 3. Modelado de datos (esquema estrella)

Se construyó un modelo en **esquema estrella** en la vista de modelo de Power BI, con `hecho_ventas_propiedades` como tabla central y tres dimensiones a su alrededor.

| Dimensión (lado 1) | Tabla de hechos (lado *) | Cardinalidad | Dirección del filtro | Estado |
|---|---|---|---|---|
| `dim_clientes[id_cliente]` | `hecho_ventas_propiedades[id_cliente]` | 1:* | Única (single) | Activa |
| `dim_propiedades[id_propiedad]` | `hecho_ventas_propiedades[id_propiedad]` | 1:* | Única (single) | Activa |
| `dim_fecha[Date]` | `hecho_ventas_propiedades[fecha_venta]` | 1:* | Única (single) | Activa |

Decisiones de modelado:
- Las relaciones se basan en las claves primarias de cada dimensión, validadas sin duplicados en la sección 1, lo que permite la cardinalidad uno a muchos.
- La dirección de filtro única hace que el filtro viaje desde cada dimensión hacia la tabla de hechos, evitando rutas ambiguas.
- No existen relaciones entre dimensiones.
- Se revisaron las relaciones detectadas automáticamente para asegurar que ninguna se basara en columnas repetidas entre tablas, como `ciudad` o `tipo_propiedad`.
- Se ocultaron en la vista de informe las claves técnicas de la tabla de hechos (`id_cliente`, `id_propiedad`, `fecha_venta`) y las columnas repetidas en las dimensiones (`ciudad`, `tipo_propiedad`), para que los filtros y segmentaciones se tomen siempre de las tablas de dimensión.

## 4. Medidas DAX

Todas las medidas se crearon en DAX dentro de la tabla `hecho_ventas_propiedades`.

**4.1 Medidas base**

| Medida | Fórmula | Propósito |
|---|---|---|
| Ingreso Total | `SUM(hecho_ventas_propiedades[precio_venta])` | Valor total de las propiedades vendidas |
| Cantidad de Ventas | `COUNTROWS(hecho_ventas_propiedades)` | Número de transacciones (una fila por venta) |
| Ticket Promedio | `DIVIDE([Ingreso Total], [Cantidad de Ventas])` | Ingreso promedio por venta |
| Comisión Total | `SUM(hecho_ventas_propiedades[monto_comision])` | Ingreso real de la inmobiliaria por comisiones |

**4.2 Medidas con contexto de filtro (participación %)**

| Medida | Fórmula |
|---|---|
| % Ingresos Tipo Propiedad | `DIVIDE([Ingreso Total], CALCULATE([Ingreso Total], ALL(dim_propiedades[tipo_propiedad])))` |
| % Ingresos Canal Venta | `DIVIDE([Ingreso Total], CALCULATE([Ingreso Total], ALL(hecho_ventas_propiedades[canal_venta])))` |
| % Ingresos Segmento Cliente | `DIVIDE([Ingreso Total], CALCULATE([Ingreso Total], ALL(dim_clientes[segmento_comprador])))` |

- El numerador conserva el filtro de la categoría que se está analizando.
- En el denominador, `CALCULATE` modifica el contexto y `ALL` elimina únicamente el filtro de esa dimensión, de modo que representa el total de todas las categorías. Los demás filtros (por ejemplo, el año) se mantienen.
- Se usarán como tooltip o etiqueta adicional en los gráficos.

**4.3 Inteligencia de tiempo**

| Medida | Fórmula | Propósito |
|---|---|---|
| Ventas YTD | `TOTALYTD([Ingreso Total], dim_fecha[Date])` | Acumulado desde el inicio del año |
| Ventas MTD | `TOTALMTD([Ingreso Total], dim_fecha[Date])` | Acumulado desde el inicio del mes |
| Ventas Año Anterior | `CALCULATE([Ingreso Total], SAMEPERIODLASTYEAR(dim_fecha[Date]))` | Ingreso del mismo periodo del año anterior |
| Crecimiento YoY % | `DIVIDE([Ingreso Total] - [Ventas Año Anterior], [Ventas Año Anterior])` | Variación porcentual año contra año |

Todas usan `dim_fecha[Date]`, tabla marcada como tabla de fechas en la sección 2.

**4.4 Columnas calculadas para cohortes**

| Columna | Fórmula | Propósito |
|---|---|---|
| Primera Compra | `CALCULATE(MIN(hecho_ventas_propiedades[fecha_venta]), ALLEXCEPT(hecho_ventas_propiedades, hecho_ventas_propiedades[id_cliente]))` | Fecha de la primera compra de cada cliente |
| Mes Cohorte | `FORMAT(hecho_ventas_propiedades[Primera Compra], "YYYY-MM")` | Mes de adquisición del cliente |
| Mes Venta | `FORMAT(hecho_ventas_propiedades[fecha_venta], "YYYY-MM")` | Mes en que ocurre cada transacción |

- Se crearon como columnas calculadas porque deben evaluarse fila por fila y quedar fijas para cada venta.
- `ALLEXCEPT` hace que, para cada venta, se consideren todas las compras del mismo cliente y se tome la fecha mínima.
- En la matriz de cohortes, las filas serán el Mes Cohorte, las columnas el Mes Venta y los valores la Cantidad de Ventas o el Ingreso Total. Los valores posteriores al mes de cohorte indican clientes que volvieron a comprar.

**Formato aplicado**
- Porcentaje con 1 decimal: medidas de participación y Crecimiento YoY %.
- Separador de miles sin decimales: montos y cantidades.

## 5. Estructura del reporte

El dashboard se construyó en Power BI con tres páginas. En todas se mantiene la misma regla de color: azul (`#1F4E79`) para el dato principal, gris para el año de comparación y rojo/verde solo para señales de desempeño. Cada visual tiene un título que comunica el hallazgo y un subtítulo que agrega contexto.

**5.1 Overview Ejecutivo**
- **Segmentador de Año** (selección única, 2024 por defecto), necesario para que el crecimiento YoY compare un año contra el anterior.
- **KPIs principales:** Ingreso Total, Cantidad de Ventas, Ticket Promedio y Comisión Total, más Crecimiento YoY % con color condicional (verde si crece, rojo si cae).
- **Tendencia de ventas:** gráfico de líneas de ingresos por mes, con una línea por año (2023 en gris y 2024 en azul). Se desactivó la interacción con el segmentador para que siempre se vean ambos años.
- **Ingresos por ciudad:** gráfico de barras por ciudad de la propiedad, comparando 2023 y 2024.
- **Crecimiento YoY:** se muestra en la tarjeta KPI y se refuerza visualmente en la comparación por mes y por ciudad.

**5.2 Análisis Comercial**
- **Ingreso por tipo de propiedad:** gráfico de barras.
- **Ingreso por canal de venta:** gráfico de columnas.
- **Ingreso por segmento de cliente:** gráfico de barras, con la cantidad de ventas en el tooltip.
- **Tooltips de participación:** cada gráfico muestra su medida de participación (% Ingresos Tipo Propiedad, % Ingresos Canal Venta, % Ingresos Segmento Cliente).
- **Tabla con formato condicional:** tipo de propiedad, Ingreso Total, Cantidad de Ventas, Ticket Promedio, Comisión Total y % del ingreso. Se aplicó un semáforo (rojo, amarillo y verde) en ingreso, cantidad de ventas y ticket promedio para resaltar el mejor y el peor desempeño.

**5.3 Análisis de Cohortes**
- **Matriz de cohortes:** filas = Mes Cohorte (mes de primera compra), columnas = Mes Venta y valores = Cantidad de Ventas.
- Se aplicó un mapa de calor (degradado de blanco a azul) para identificar visualmente las cohortes más activas.
- La diagonal representa las ventas del mes de entrada de cada cohorte; los valores a la derecha muestran a los clientes que vuelven a comprar en meses posteriores.
- Esta página no tiene segmentador de año, para poder observar la recompra a lo largo de todo el periodo.

## 6. Resumen ejecutivo

### Hallazgos clave
- El ingreso total del periodo 2023–2024 fue de **6.012,5 millones**, generado por **8.500 ventas**. En 2024 el ingreso creció **11,1 %** frente a 2023 (de 2.847,7 a 3.164,8 millones).
- El tipo de propiedad que genera mayor ingreso es **Casa** (2.240,5 millones; 37 % del total). Sin embargo, la participación está equilibrada: Comercial aporta 32 % y Departamento 31 %.
- **Departamento** concentra el volumen (5.105 ventas, 60 % del total) con el ticket más bajo, mientras **Comercial** vende pocas unidades (1.071) con el ticket más alto (más de 1,7 millones por venta).
- La ciudad con mayor ingreso es **Ciudad de México** (3.242 millones frente a 2.770 de Bogotá). Ambas crecen a un ritmo similar en 2024: +10,8 % y +11,5 %, respectivamente.
- El canal con mayor ingreso es el **Corredor**, que genera cerca del **73 %** del total (4.380 millones); el canal directo aporta el 27 %.
- Las ventas son **estacionales**: los picos se concentran en marzo–abril y septiembre–noviembre, y los meses más bajos son enero–febrero y junio–agosto.

### Métricas principales (periodo 2023–2024)
- **Ingreso Total:** 6.012,5 millones (2024: 3.164,8 millones)
- **Cantidad de Ventas:** 8.500 (2024: 4.455)
- **Ticket Promedio:** 707 mil (2024: 710 mil)
- **Comisión Total:** 200,6 millones (2024: 105,9 millones), equivalente a cerca del 3,3 % del ingreso
- **Crecimiento YoY 2024:** +11,1 %

### Insights accionables
- El segmento **Primera vez** concentra la mayor parte del ingreso (cerca del 63 %). **Inversionista** es el segmento que más crece en 2024 (+19 %), mientras **Alto patrimonio** se mantiene estancado (+0,5 %).
- Los clientes sí vuelven a comprar: cerca del **61 %** de las ventas ocurre después del mes de primera compra. Las cohortes de **marzo y abril de 2023** muestran la mayor recurrencia, con más de 1.000 ventas acumuladas cada una.
- En 2024, cerca del **82 %** de las ventas provino de clientes que ya habían comprado antes, mientras la captación de nuevos clientes cayó con fuerza (las cohortes pasan de 150–370 ventas iniciales en 2023 a 26–129 en 2024).
- Las ventas muestran un **crecimiento de 11,1 % YoY**, impulsado sobre todo por la temporada baja: entre mayo y agosto 2024 supera claramente a 2023.
- Por tipo de propiedad, **Comercial** es el que más crece (+18 %), seguido de Departamento (+12 %); Casa crece poco (+5 %) pese a ser el de mayor ingreso.

### Recomendaciones estratégicas
- **Priorizar la comercialización de propiedades comerciales**, que combinan el ticket más alto con el mayor crecimiento, y mantener el volumen de departamentos como base estable del negocio.
- **Fortalecer el canal Corredor**, que genera tres cuartas partes del ingreso, y evaluar incentivos para aumentar la participación del canal directo, que no paga intermediación.
- **Reactivar la captación de nuevos clientes**, ya que el crecimiento actual depende de la recompra de clientes antiguos y las cohortes de 2024 son mucho más pequeñas.
- **Implementar estrategias de retención y recompra** para las cohortes recientes, replicando lo que funcionó con las cohortes de inicios de 2023.
- **Desarrollar una oferta para el segmento Inversionista**, el de mayor crecimiento, y revisar la propuesta de valor para Alto patrimonio, que no crece.
- **Planificar campañas antes de las temporadas altas** (marzo–abril y septiembre–noviembre) y acciones para sostener la venta en enero–febrero y junio–agosto.
