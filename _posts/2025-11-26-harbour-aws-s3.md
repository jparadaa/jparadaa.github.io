# Implementando AWS S3 en Harbour con Signature V4

## El contexto: Un problema real

Actualmente estoy trabajando en un proyecto que incluye un módulo web donde los usuarios necesitan tomar fotografías desde sus dispositivos. Esto planteó un desafío importante: ¿dónde almacenar todas esas imágenes de forma económica, escalable y confiable?

Después de evaluar opciones, AWS S3 apareció como la solución ideal: extremadamente económica (centavos por GB), altamente escalable, y con disponibilidad del 99.99%. El problema era que no existía una implementación completa y funcional para Harbour que soportara el estándar actual de AWS (Signature Version 4).

## El camino difícil

Este tipo de implementaciones nadie las "regala" o aporta fácilmente en la comunidad Harbour. Es trabajo complejo que requiere entender protocolos de autenticación criptográfica, especificaciones técnicas de AWS, y las particularidades de hbcurl.

Estuve **3 días completos** intentando hacer funcionar la integración con Harbour y fracasé repetidamente. Las firmas no coincidían, las peticiones eran rechazadas con errores crípticos, y cada intento parecía llevarme a otro callejón sin salida.

La frustración llegó al punto de considerar descartar Harbour para esta parte del proyecto. Hice pruebas con **PyOLE** (Python + OLE Automation) y fueron exitosas casi de inmediato. Python tiene un SDK oficial de AWS, así que la implementación fue directa y funcionó a la primera. Parecía la ruta fácil.

## Perseverancia y aprendizaje

Pero no me di por vencido con Harbour. Sabía que era posible, solo necesitaba encontrar el enfoque correcto.

La IA (Claude) ayudó muchísimo, pero **no fue suficiente por sí sola**. Generaba código que parecía correcto pero fallaba de formas sutiles. El verdadero avance vino cuando:

1. **Volví a los foros** - Busqué ejemplos reales de hbcurl en Harbour Users y Harbour Developers
2. **Leí la documentación oficial** - AWS Signature V4 especificación completa, ejemplos paso a paso
3. **Debuggeé sistemáticamente** - Comparé cada paso de mi implementación con ejemplos funcionales
4. **Guié a la IA correctamente** - Con mejor contexto técnico, las soluciones fueron más acertadas

Después de **casi una semana** de intentos, frustraciones, y aprendizaje, finalmente lo conseguí. Una implementación completa, probada, y funcional de AWS S3 para Harbour.

## ¿Qué es AWS Signature V4?

AWS Signature Version 4 es el método de autenticación actual que usa Amazon Web Services para verificar que las peticiones a sus servicios son legítimas. Es más seguro que versiones anteriores y **obligatorio** para regiones nuevas de AWS.

El proceso implica:
1. Crear un "canonical request" (petición normalizada)
2. Generar un "string to sign" con timestamp y scope
3. Derivar una signing key mediante 4 HMACs encadenados
4. Firmar la petición con esta key
5. Generar una URL presignada con tiempo de expiración

Todo esto debe hacerse **exactamente** según la especificación de AWS, o las peticiones serán rechazadas.

## Funcionalidades implementadas

Esta implementación incluye:

- ✅ **Subir archivos** desde memoria o disco
- ✅ **Descargar archivos** a memoria o disco
- ✅ **Eliminar objetos** de S3
- ✅ **Verificar existencia** usando método HEAD (optimizado)
- ✅ **Generar URLs presignadas** con expiración configurable
- ✅ **Soporte completo AWS Signature V4**

## Detalles técnicos de implementación

### El desafío de AWS Signature V4

La parte más compleja fue implementar correctamente la firma de AWS. La especificación requiere:

1. **Timestamp en formato ISO8601 UTC**
```harbour
   cDateTimeISO := "20251126T145034Z"
```

2. **Canonical Request** - Normalización exacta de la petición
```harbour
   cCanReq := cMethod + "\n" + ;
              CanonicalURI + "\n" + ;
              CanonicalQueryString + "\n" + ;
              CanonicalHeaders + "\n" + ;
              SignedHeaders + "\n" + ;
              PayloadHash
```

3. **Signing Key Derivation** - 4 HMACs encadenados
```harbour
   cSignKey := HMAC_SHA256(date, "AWS4" + secret)
   cSignKey := HMAC_SHA256(region, cSignKey)
   cSignKey := HMAC_SHA256("s3", cSignKey)
   cSignKey := HMAC_SHA256("aws4_request", cSignKey)
```

4. **Firma final** de la petición completa

### Harbour y soporte criptográfico nativo

Una de las ventajas de trabajar con Harbour es que incluye **nativamente** las funciones criptográficas necesarias para AWS Signature V4:

- `HB_SHA256()` - Hash SHA-256
- `HB_HMAC_SHA256()` - HMAC con SHA-256
- `hb_HexToStr()` / `hb_StrToHex()` - Conversión hexadecimal

El único desafío fue entender que `HB_HMAC_SHA256()` retorna el resultado en formato hexadecimal, por lo que hay que convertirlo a binario con `hb_HexToStr()` antes de usarlo como key en el siguiente HMAC de la cadena:
```harbour
// Primera derivación
cSignKey := hb_HexToStr( HB_HMAC_SHA256( cDateISO, "AWS4" + cSecretKey ) )

// Encadenamos 3 HMACs más, cada uno usando el resultado binario del anterior
cSignKey := hb_HexToStr( HB_HMAC_SHA256( cRegion, cSignKey ) )
cSignKey := hb_HexToStr( HB_HMAC_SHA256( cService, cSignKey ) )
cSignKey := hb_HexToStr( HB_HMAC_SHA256( "aws4_request", cSignKey ) )

// Firma final
cSignature := Lower( HB_HMAC_SHA256( cStringToSign, cSignKey ) )
```

### URLs Presignadas

Una ventaja de este enfoque es que generamos URLs presignadas que pueden usarse directamente desde cualquier cliente HTTP:
```harbour
cUrl := AWS_S3_GeneratePresignedUrl(cBucket, "archivo.pdf", ;
                                    cAccessKey, cSecretKey, ;
                                    cRegion, 3600, "GET")

// Esta URL funciona durante 1 hora (3600 segundos)
// Puede usarse desde un navegador, curl, wget, etc.
```

## Pruebas realizadas

La implementación fue probada exhaustivamente con un programa de test completo que valida:

1. ✅ Subida de archivos desde memoria
2. ✅ Descarga de archivos a memoria y verificación del contenido
3. ✅ Subida de archivos desde disco
4. ✅ Descarga de archivos directo a disco local
5. ✅ Eliminación de objetos en S3
6. ✅ Verificación de que los objetos fueron eliminados correctamente
7. ✅ Limpieza de archivos temporales locales

El test completo ejecuta en menos de 10 segundos y valida toda la funcionalidad end-to-end.

## Casos de uso reales

Esta implementación es útil para:

- **Backups automáticos** - Subir respaldos de bases de datos a S3
- **Almacenamiento de documentos** - Facturas, reportes, PDFs
- **Archivos de usuarios** - Subida de archivos desde aplicaciones
- **Logs y auditoría** - Almacenar logs en la nube
- **Distribución de archivos** - Generar URLs temporales de descarga
- **Integración con otros sistemas** - S3 como intermediario

## Requisitos

- Harbour con **hbcurl** compilado
- Cuenta AWS (capa gratuita incluye 5GB por 12 meses)
- Credenciales IAM con permisos S3

## Seguridad

Algunas consideraciones importantes:

- Las credenciales **nunca** se envían en las URLs
- Todas las conexiones usan **HTTPS**
- Las URLs presignadas **expiran** automáticamente
- Se usa el estándar **AWS Signature V4** (el más seguro)

## Código abierto

He liberado todo el código bajo licencia **MIT** para que la comunidad Harbour pueda usarlo, modificarlo y mejorarlo libremente.

📦 **Repositorio:** [github.com/jparadaa/harbour-aws-s3](https://github.com/jparadaa/harbour-aws-s3)

## Reflexión sobre el desarrollo

Este proyecto fue un ejercicio interesante de:

1. **Leer especificaciones técnicas** - AWS Signature V4 es complejo pero bien documentado
2. **Debuggear criptografía** - Un byte mal calculado y toda la firma falla
3. **Trabajar con hbcurl** - Aprender a usar correctamente sus funciones nativas
4. **Testing exhaustivo** - Probar cada función individualmente y en conjunto

Lo más valioso fue entender que Harbour ya tiene todas las herramientas necesarias para implementar protocolos modernos de autenticación, y que vale la pena investigar en la documentación y comunidad antes de tomar atajos que pueden no funcionar correctamente.

## Conclusión

Harbour sigue siendo una herramienta poderosa para desarrollo empresarial, y con implementaciones como esta, podemos integrar servicios modernos de cloud computing en nuestras aplicaciones sin depender de herramientas externas.

Si usas Harbour y necesitas almacenamiento en la nube, esta implementación te puede ahorrar días de trabajo.

El código está disponible, probado, y listo para usar. Espero que le sea útil a la comunidad.
