# Generar archivos PDF utilizando UT y PDFMake

## Un poco de historia

Lo crean o no, nunca tuve necesidad de generar archivos PDF. Siempre generé archivos Excel, primero mediante OLE/COM y después con el wrapper de la librería xlsxwriter que mi amigo Riztan desarrolló e implementó:

🔗 [hbxlsxwriter - GitHub](https://github.com/riztan/hbxlsxwriter)

Pero debido a un nuevo proyecto que estoy desarrollando, finalmente ese día ha llegado. Sí, irremediablemente tengo que "aprender" a generar archivos PDF con Harbour.

## Primer intento: tpdf con Haru

Así que he intentado con la clase `tpdfclass.prg`, basada en la contrib Haru de Harbour. Pero para mí me resulta, la verdad, muy compleja cuando quieres hacer diseños un poco más profesionales. Es decir, **es completamente posible**, pero en mi opinión requiere muchas líneas de código para lograr resultados aceptables.

## Segundo intento: Excel → PDF con COM

Luego tuve una "brillante idea": si yo soy experto en generar archivos Excel, ¿por qué no generar el archivo Excel con la librería `hbxlsxwriter.lib` y luego mediante COM convertir el Excel en PDF?

Bien, realicé las pruebas y claro que técnicamente **es posible**, pero esto demora entre **5-8 segundos**, lo cual es una eternidad en temas de UX (experiencia de usuario).

## La solución: PDFMake en el frontend

Así que seguí investigando y me encontré que con JavaScript, una de las librerías más utilizadas para generar archivos PDF en el frontend es **pdfMake**. 

Mi duda inicial era: ¿es posible desde el backend proporcionarle los datos al frontend y luego diseñar el formato que yo requiero en el navegador?

Después de leer un poco de documentación por aquí:

🔗 [PDFMake en acción - DEV Community](https://dev.to/malcode/pdfmake-en-accion-como-disenar-tickets-en-reactjs-con-pdfmake-43ck)

Y un poco de IA por acá, finalmente ha sido **un éxito la implementación**. 

## Resultados

Honestamente creo que funciona de lujo y es **extremadamente rápido** (< 1 segundo), y puedes tener diseños realmente complejos y profesionales.

### Ventajas de esta solución:

✅ **Velocidad**: Generación instantánea (< 1 segundo)  
✅ **Diseño profesional**: Tablas complejas, estilos, imágenes, múltiples páginas  
✅ **Backend ligero**: Solo preparas y envías datos JSON  
✅ **Sin dependencias**: No requiere software adicional instalado  
✅ **Mantenible**: Código JavaScript limpio y fácil de modificar  

## El proyecto

Les comparto el proyecto completo en UT que pueden ejecutar y ver lo realmente fácil que es generar PDF en el frontend.

🔗 **[Ver proyecto en GitHub](https://github.com/jparadaa/ut_pdfmake)**

### ¿Para quién es esta solución?

Esta alternativa **no es para todos**. Es para aquellos que, como yo, no se quieren liar con la complejidad de generar PDF directamente con Harbour, y prefieren aprovechar las herramientas modernas del navegador.
