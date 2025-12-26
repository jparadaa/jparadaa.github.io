# Compresión de Imágenes con Harbour y libGD: Optimizando para AWS S3

## El contexto: Un nuevo desafío

Después de implementar exitosamente [AWS S3 en Harbour con Signature V4](https://jparadaa.github.io/2025/11/26/harbour-aws-s3.html), surgió un problema que no había anticipado: **las fotografías de alta resolución estaban consumiendo demasiado espacio y ancho de banda**.

En mi proyecto, los técnicos toman fotografías desde sus dispositivos móviles como evidencia de los servicios realizados. Las cámaras modernas generan imágenes de 4000-6000 píxeles y 3-8 MB por fotografía.

## El problema real con números

Un caso real del proyecto:
```
Foto original de campo:
- Archivo: foto1.jpg
- Dimensiones: 4902 x 6290 píxeles
- Tamaño: 4,104 KB (4.1 MB)

Foto optimizada:
- Dimensiones: 841 x 1080 píxeles
- Tamaño: 113 KB
- Reducción: 97.3%
```

**Antes de la compresión:**
- 50 fotos/día × 4 MB = 200 MB/día
- ~6 GB/mes solo en fotos
- Subidas lentas desde campo (1-2 min por foto en 4G)
- Costos S3 crecientes exponencialmente

**Después de la compresión:**
- 50 fotos/día × 120 KB = 6 MB/día
- ~180 MB/mes total
- Subidas rápidas (5-10 segundos por foto)
- **Ahorro del 97%** en almacenamiento y transferencias

Para evidencias fotográficas donde solo necesitamos **demostrar que el servicio se realizó**, una foto de 1920x1080 (Full HD) es más que suficiente para identificar el lugar, mostrar el trabajo realizado y servir como evidencia legal.

## La solución: Comprimir antes de subir

El workflow es directo:

1. Usuario toma foto desde su dispositivo (4-8 MB)
2. Servidor Harbour recibe la imagen
3. Compresor la optimiza automáticamente a 1920x1080 px
4. Se sube a S3 la versión optimizada (100-200 KB)

## ¿Por qué implementar un wrapper propio?

Harbour tiene la contrib **hbgd**, pero en mi caso por alguna razón no pude hacer funcionar ningún test. Es decir, compilaba perfectamente el código y se generaba la lib, pero al hacer pruebas no recibía errores de compilación ni errores en ejecución, pero simplemente no hacía nada el código.

Decidí no invertir más tiempo en investigar qué sucedía y con la ayuda de la IA implementé mi propio wrapper de funciones muy concretas para obtener la funcionalidad que requiero.

## El salvador: vcpkg

Para compilar libGD y todas sus dependencias, utilicé **vcpkg**, el gestor de paquetes de Microsoft para C/C++:
```batch
vcpkg install libgd:x64-windows-static
```

**Un solo comando, 15 minutos, todo funciona.** vcpkg descarga y compila automáticamente:

- libGD y todas sus dependencias (libpng, libjpeg, freetype, fontconfig...)
- Genera librerías estáticas listas para MSVC 64-bit
- Configura los paths correctamente

No más búsqueda de DLLs, configuración manual de paths, o dependencias rotas.

## El workflow completo

La integración perfecta entre compresión y S3:

1. Usuario toma foto en la app móvil
2. Foto se envía al servidor Harbour
3. Servidor comprime automáticamente
4. Imagen optimizada se sube a S3
5. URL de S3 se guarda en base de datos
6. App muestra la imagen optimizada

Todo transparente para el usuario, con ahorros masivos en costos y performance.

## Casos de uso

Esta implementación es útil para:

- **Evidencias fotográficas** - Servicios, inspecciones, auditorías
- **Documentos escaneados** - Facturas, contratos, recibos
- **Archivos de usuarios** - Perfiles, avatares, documentación
- **Galerías de productos** - E-commerce, catálogos
- **Backups de imágenes** - Reducir espacio de respaldo
- **Cualquier app con fotos** - Donde el tamaño importe

## Código abierto

He liberado todo el código bajo licencia MIT para que la comunidad Harbour pueda usarlo, modificarlo y mejorarlo libremente.

📦 **Repositorio**: [github.com/jparadaa/harbour-image-compressor](https://github.com/jparadaa/harbour-image-compressor)

El repositorio incluye:
- Wrapper completo de libGD
- Función de compresión lista para usar
- Configuración de compilación con vcpkg
- Documentación de instalación paso a paso

## Reflexión final

La combinación de [AWS S3 para almacenamiento](https://github.com/jparadaa/harbour-aws-s3) y compresión de imágenes con libGD, todo integrado nativamente en Harbour, crea una solución empresarial completa, económica y escalable.

Este proyecto puede ser útil para cualquier desarrollador que trabaje con Harbour y necesite manipular imágenes, ahorrándole días de trabajo y frustración.
