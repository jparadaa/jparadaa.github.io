# Gráficas profesionales en PDF con Harbour y QuickChart

---

Hace unos días me enfrenté a un reto que, aunque suena sencillo, no lo fue tanto: generar un reporte gerencial en PDF con gráficas profesionales, todo desde Harbour.

El reporte debía mostrar ventas comparativas por canal, participación porcentual y volumen por línea de producto. Nada fuera de lo ordinario para cualquier herramienta moderna de BI — pero en Harbour no existe una forma nativa de generar gráficas con el nivel visual que hoy esperan los directivos.

## El problema

Harbour es poderoso para todo lo que tiene que ver con lógica de negocio, acceso a datos y generación de documentos. Pero gráficas profesionales — con colores, leyendas, etiquetas, escalas adaptivas — no es su fuerte nativo.

Entonces recordé **QuickChart**.

## La solución: QuickChart + Chart.js v2

[QuickChart](https://quickchart.io) es un servidor open source que recibe un JSON con la configuración de una gráfica Chart.js y devuelve una imagen PNG lista para usar. Lo mejor: puedes instalarlo localmente con Docker en un solo comando.

```bash
docker run -d -p 3000:3000 ianw/quickchart
```

Desde ese momento tienes un servidor local en `http://localhost:3000` que acepta peticiones GET con el parámetro `c` conteniendo el JSON de la gráfica y devuelve un PNG. Exactamente lo que necesitaba.

La idea completa es:

1. Construir el JSON de la gráfica en Harbour usando hashes (`{=>}`)
2. Serializar con `hb_jsonEncode()`
3. Encodar la URL con una función RFC 3986
4. Hacer un GET con `libcurl` y guardar el PNG resultante
5. Insertar el PNG en el PDF con `tpdfclass` (wrapper de libharu)

## El truco del placeholder JavaScript

Chart.js permite definir funciones JavaScript para formatear las etiquetas de los ejes. El problema es que `hb_jsonEncode()` no puede serializar funciones JS — las convierte a strings con comillas y Chart.js no las ejecuta.

La solución es elegante — y el crédito es de **Riztan**, quien hace unos 5 años ya había resuelto este problema con este mismo enfoque: poner un placeholder string en el hash, serializar normalmente, y después reemplazar el placeholder (incluyendo sus comillas dobles) por la función JS real usando `StrTran()`.

```harbour
// En el hash — placeholder como string normal
hYAxis["ticks"] := { "callback" => "##JS_Y##" }

// Después de hb_jsonEncode — reemplazar incluyendo las comillas dobles
cChartJs := StrTran( cChartJs, '"##JS_Y##"', ;
   "function(v){if(v>=1000)return '$'+(v/1000).toFixed(0)+' mil';return '$'+v;}" )
```

El JSON resultante tiene la función JS sin comillas y QuickChart la ejecuta correctamente.

## Proporciones PNG → PDF

Un detalle importante: si el PNG de QuickChart es de `900×450` píxeles (ratio 2:1) y en el PDF lo insertas con dimensiones que no respetan ese ratio, la imagen se distorsiona.

`tpdfclass` usa el método `CmSayBitmap(fila, columna, archivo, ANCHO, ALTO)`. La clave es respetar siempre el ratio original:

```harbour
// PNG 900x450 → ratio 2:1 → en PDF: 16cm ancho, 8cm alto
oPdf:CmSayBitmap( 2.8, 0.8, cImgBarras, 16.0, 8.0 )

// PNG 450x450 → ratio 1:1 → en PDF: 8cm ancho, 8cm alto  
oPdf:CmSayBitmap( 2.8, 17.5, cImgDona, 8.0, 8.0 )
```

## Importante: versión de Chart.js

La imagen Docker `ianw/quickchart` instala Chart.js **v2**, no v3. La sintaxis es diferente en algunos puntos clave:

| Elemento | Chart.js v2 | Chart.js v3 |
|---|---|---|
| Ejes | `yAxes: [{}]` | `scales: { y: {} }` |
| Tamaño fuente | `fontSize: 12` | `font: { size: 12 }` |
| Dona hueco | `cutoutPercentage: 50` | `cutout: "50%"` |

Si mezclas sintaxis v2 y v3 obtendrás un PNG con mensaje de error en lugar de la gráfica. Aprende de mis iteraciones. 😄

## El resultado

Una gráfica de barras con línea comparativa y una dona de participación, ambas en PNG de alta resolución, insertadas en un PDF carta horizontal generado completamente desde Harbour.

Todo el proceso — consulta a base de datos, generación de gráficas, construcción del PDF — en menos de 3 segundos.

## Código fuente

El ejemplo completo con datos ficticios está disponible en GitHub:

👉 **[github.com/jparadaa/harbour-quickchart-pdf](https://github.com/jparadaa/harbour-quickchart-pdf)**

Incluye las dos gráficas, la función de URL encode RFC 3986 y el PDF de una página completo y funcional.

---

Si estás en la comunidad Harbour y alguna vez necesitaste gráficas profesionales en tus reportes, espero que esto te ahorre las horas que yo invertí. Cualquier pregunta o mejora, los comentarios están abiertos.

¡Hasta la próxima!
