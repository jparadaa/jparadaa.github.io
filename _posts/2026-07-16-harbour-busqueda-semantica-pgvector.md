# Búsqueda semántica en Harbour: pgvector, embeddings locales y una fusión con RRF

Hace unas semanas me hice una pregunta simple: **¿puedo hacer búsqueda semántica desde Harbour, sin depender de ningún servicio en la nube?** No tenía experiencia previa con embeddings ni con bases de datos vectoriales — todo lo que cuento aquí lo aprendí armando el experimento pieza por pieza. Comparto el recorrido completo, incluyendo los tropiezos, porque creo que esa parte es la más útil.

> **Spoiler honesto:** esto terminó siendo un experimento de aprendizaje, no una solución lista para producción. Al final explico por qué, y qué le falta para llegar ahí.

Voy a admitir algo desde el inicio: llevo años acostumbrado a resolver búsquedas con SQL tradicional — `WHERE`, `LIKE`, índices, todo un terreno donde entiendo exactamente qué está pasando y por qué. Meterme a este tema me costó trabajo precisamente por eso. La búsqueda semántica no compara texto, compara *significado* representado como números, y varias veces durante este experimento algo no me hacía sentido simplemente porque estaba tratando de razonarlo con la lógica de SQL. Una vez que acepté que es un espacio de trabajo distinto, con sus propias reglas, empezó a acomodarse.

## El punto de partida

Ya tenía en mi stack una clase `TOLlama` (la base es de Antonio Linares), que uso para hablar con modelos locales vía Ollama usando `hbcurl`. La modifiqué para agregar una clase específica para generar **embeddings** — la representación numérica de un texto que permite buscar por significado, no por coincidencia exacta de palabras.

Y del lado de la base de datos, Harbour tiene un contrib nativo para PostgreSQL: **`hbpgsql`**, con la clase `TPQServer`. Del lado de PostgreSQL, existe la extensión **pgvector**, que agrega un tipo de dato `vector` y operadores de distancia (`<=>` para distancia coseno) — es lo que permite guardar esos embeddings en una columna y ordenar resultados por "qué tan parecido es el significado", en vez de por igualdad exacta.

## Generando embeddings con modelos locales

Con Ollama ya instalado, extendí el patrón de `TOLlama` en una clase dedicada, `TOLlamaEmbed`, que pega al endpoint de embeddings en vez del de chat:

```harbour
CLASS TOLlamaEmbed
   DATA cModel, cResponse, cUrl, hCurl
   METHOD New( cModel )
   METHOD GetEmbedding( cPrompt )
   METHOD End()
ENDCLASS

METHOD GetEmbedding( cPrompt ) CLASS TOLlamaEmbed
   // POST a http://localhost:11434/api/embeddings
   // Decodifica la respuesta y regresa el vector como
   // string "[0.123,-0.456,...]", listo para un ::vector de pgvector
   ...
RETURN cVecStr
```

Con eso, el flujo básico es: tomar un texto, generar su vector, guardarlo en una columna `vector(N)`, y comparar por distancia coseno con el operador `<=>` en vez de `=` o `LIKE`.

## Primer hallazgo: SQL tradicional y embeddings no compiten en la misma cancha

Armé una tabla de prueba con descripciones de productos de mi giro (filtros, aceites, refacciones), a propósito con vocabulario variado — algunas filas dicen "diésel" explícito, otras lo implican sin decirlo. La primera comparación fue simple:

```sql
SELECT codigo, descripcion FROM productos
WHERE descripcion ILIKE '%filtro%aceite%diesel%';
```

Cero resultados, aunque el producto correcto existiera con otras palabras. Ese es el límite conocido de la búsqueda por texto: exige que el usuario adivine el vocabulario exacto del catálogo.

Con el embedding de la misma consulta y `ORDER BY embedding <=> vector_busqueda`, sí aparecían resultados relevantes. Punto para los embeddings — pero no fue el final de la historia.

![Demostración de búsqueda semántica en Harbour](/img/busqueda-semantica-harbour/demo-busqueda-semantica.gif)

## Segundo hallazgo: no todos los modelos de embeddings son iguales

Probé tres modelos, todos gratuitos y corriendo en local vía Ollama:

- `nomic-embed-text` (768 dimensiones)
- `mxbai-embed-large` (1024 dimensiones)
- `bge-m3` (1024 dimensiones, multilingüe — importante, porque los dos anteriores están entrenados mayoritariamente en inglés y mis datos están en español)

Con consultas simples ("cambiar el aceite del carro"), los tres modelos daban prácticamente el mismo ranking. Pero con matices finos (distinguir "diésel" de "gasolina" entre productos que comparten mucho vocabulario), el modelo chico se equivocaba de forma consistente, colando resultados irrelevantes por encima de otros que sí aplicaban. `bge-m3`, al ser multilingüe y de mayor tamaño, ordenaba mejor en esos casos.

La lección real aquí no es "el modelo grande siempre gana" — es que **el modelo importa cuando la distinción es sutil, y no tanto cuando es obvia**. Y si tus datos no están en inglés, un modelo entrenado específicamente para tu idioma vale más que uno más grande pero monolingüe.

## Tercer hallazgo: la pieza que faltaba era Reciprocal Rank Fusion (RRF)

Mientras exploraba esto, encontré una técnica bien establecida en sistemas de búsqueda de producción: **RRF**, que combina dos rankings distintos (léxico y semántico) en uno solo, usando solo la *posición* de cada resultado en cada lista, sin necesidad de normalizar escalas incompatibles (un score de relevancia de texto y una distancia coseno no se pueden promediar directamente, están en escalas totalmente distintas).

Para el lado de texto usé el buscador de texto completo nativo de PostgreSQL, que conviene explicar porque no es evidente si nunca lo has usado:

- `to_tsvector('spanish', descripcion)` convierte un texto en una lista normalizada de palabras clave (quita acentos, reduce plurales y conjugaciones a su raíz — "filtros" y "filtro" cuentan como lo mismo).
- `to_tsquery('spanish', 'palabra1 | palabra2')` arma una consulta de búsqueda sobre esa lista; el `|` significa "cualquiera de estas palabras" (existe también `plainto_tsquery`, que arma la consulta directo desde una frase, pero conecta las palabras como si todas fueran obligatorias — eso me dio problemas, lo cuento más abajo).
- `ts_rank(...)` calcula qué tan relevante es cada fila respecto a la consulta, dándote un número para ordenar — el equivalente, del lado de texto, a lo que la distancia coseno hace del lado semántico.

Con eso, la fusión en Harbour queda así:

```harbour
function FusionarRRF( aResultadosSql, aResultadosVector, nK )
   local hScores := { => }
   local aFinal := {}
   local i, cCodigo

   hb_default( @nK, 60 )

   for i := 1 to Len( aResultadosSql )
      cCodigo := aResultadosSql[ i ][ "codigo" ]
      ...
      hScores[ cCodigo ][ "score" ] += 1 / ( nK + i )
   next

   for i := 1 to Len( aResultadosVector )
      ...
      hScores[ cCodigo ][ "score" ] += 1 / ( nK + i )
   next

   // ordenar por score descendente
ASort( aFinal,,, {| a, b | a[ "score" ] > b[ "score" ] } )
return aFinal
```

Esto sí cambió la calidad del resultado de forma notoria: cuando la búsqueda de texto completo y la búsqueda semántica coincidían en un resultado, ese resultado subía con fuerza al top del ranking — y en la práctica, el top 5-7 de mis pruebas quedaba consistentemente limpio.

## Los tropiezos que vale la pena contar

Ninguna implementación queda bien a la primera, y documentar los errores es más útil que fingir que no los hubo:

- **`plainto_tsquery` conecta las palabras como obligatorias**: mi primera versión de la búsqueda de texto usaba esa función directo sobre la frase completa del usuario, lo cual casi siempre daba 0 resultados con consultas de varias palabras (exige que todas aparezcan juntas). La corrección fue construir el `to_tsquery` a mano, separando la frase en palabras y conectándolas con `|` en vez de asumir que todas son obligatorias.
- **Fallas silenciosas**: si la consulta SQL fallaba (por ejemplo, por un carácter que rompía la sintaxis), el código simplemente seguía adelante y fusionaba "la nada" con los resultados del vector, sin avisar a nadie. Un log de errores a archivo resolvió la visibilidad.
- **Un bug real con parámetros largos**: al mover las consultas a queries parametrizadas (para evitar construir SQL por concatenación) noté que el parámetro del vector — una cadena de miles de caracteres con 1024 números — producía resultados de similitud incorrectos (todo en 0.000000, incluso pasando el filtro de umbral mínimo). La concatenación directa del vector en el SQL, en cambio, funcionó correctamente. Quedó pendiente diagnosticar la causa exacta; por ahora esa consulta específica usa concatenación (con el vector generado internamente, no con entrada de usuario cruda), mientras el resto de las rutas sí usan parámetros reales.
- **Un umbral de similitud frágil**: la distancia coseno siempre te da "el resultado más cercano", aunque nada sea realmente relevante — si buscas algo sin ninguna relación con el catálogo, el sistema igual te va a regresar los 10 "menos lejanos", que pueden no significar nada útil. Para evitarlo, agregué un **umbral de similitud**: un valor mínimo de parecido por debajo del cual un resultado se descarta directamente, en vez de forzar siempre un top 10. Lo calibré con un puñado de consultas de prueba — lo cual es honesto decir que es un punto de partida razonable, no un número validado con rigor.
- **Rutas especiales**: agregué detección de patrones tipo código de producto (`F045`), para que una búsqueda exacta de SKU no dependa de la fusión semántica/léxica y suba siempre primero.

## ¿Está esto listo para producción? No, y aquí está por qué

Quiero ser directo sobre esto, porque es fácil que un experimento así se lea como "receta lista para copiar":

- El umbral de similitud está calibrado con un puñado de pruebas manuales, no con un conjunto de evaluación etiquetado por alguien que conozca el dominio real de los datos.
- El bug de parámetros largos con el vector no está resuelto de raíz, solo evadido con concatenación directa en ese punto específico.
- Nunca lo probé contra datos reales de un ERP — todo el catálogo de prueba fue generado por mí para tener suficiente volumen y variedad.
- No hay reranking (una segunda capa que evalúa con más precisión los candidatos ya filtrados), que es lo que usan los sistemas de producción serios cuando la precisión importa de verdad.
- La conexión a Postgres se abre por request, sin pool — aceptable para una prueba, no para volumen real de tráfico concurrente.

El estándar real de la industria para esto — calibración con conjuntos de evaluación, reranking, manejo de errores exhaustivo, infraestructura de conexión robusta — es notablemente más complejo que lo que armé aquí en un par de tardes. Y está bien que así sea: este fue un ejercicio para entender los conceptos de raíz, no un producto terminado.

## Conclusión

Lo que más me deja este experimento no es el código en sí, sino tener ya un mapa mental claro de las piezas: qué hace un embedding, por qué SQL tradicional y búsqueda semántica no son sustitutos entre sí, por qué el modelo de embeddings importa, y por qué fusionar ambos enfoques (con algo como RRF) da mejores resultados que confiar en uno solo. Me costó salir de mi forma habitual de pensar las búsquedas, pero Harbour, con `hbpgsql` y `hbcurl`, resultó ser una base perfectamente viable para explorar todo esto sin salir de mi stack de siempre.
