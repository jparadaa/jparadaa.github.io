# Gráficas profesionales en Harbour con LunaSVG: sin Docker, sin servicios externos

Hace unas semanas publiqué una entrada donde compartí cómo generar gráficas profesionales en
reportes PDF usando [QuickChart y Docker](https://jparadaa.github.io/2026/03/05/harbour-quickchart-pdf.html).
La solución funciona muy bien, pero empecé a cuestionarme algo: **levantar un
contenedor Docker completo únicamente para renderizar una imagen se me hace excesivo**.

Docker levanta su propio namespace, filesystem, red virtual y proceso supervisor. Todo eso solo
para convertir texto SVG a PNG. Empecé a buscar alternativas y encontré **LunaSVG**: una librería
C++ ligera, bajo licencia MIT, que renderiza SVG a PNG directamente en proceso. Sin red, sin
contenedor, sin latencia. Exactamente lo que necesitaba.

> **Licencia:** LunaSVG está bajo licencia **MIT**, lo que permite su uso en proyectos privados
> y comerciales, incluyendo enlace estático dentro del ejecutable, sin ninguna restricción.

## La comparación honesta

Antes de entrar al tema, quiero ser directo sobre las ventajas y desventajas reales de cada
enfoque, porque no es una decisión obvia.

**QuickChart + Docker:**
- ✅ Se describen los datos en JSON y QuickChart hace todo el trabajo visual
- ✅ Gráficas muy complejas sin escribir una sola coordenada
- ❌ Requiere Docker corriendo en producción
- ❌ Latencia de red, proceso HTTP, contenedor con runtime de Node.js
- ❌ Infraestructura adicional que monitorear y mantener

**LunaSVG + Harbour:**
- ✅ Cero dependencias en producción — todo queda dentro del `.exe`
- ✅ Renderizado en milisegundos, sin red, sin procesos externos
- ✅ Control total del resultado visual
- ❌ Hay que calcular coordenadas, posiciones y escalas manualmente
- ❌ Más líneas de código por gráfica

Lo que puede parecer una desventaja importante — más código — en la práctica es manejable.
Cada tipo de gráfica se encapsula en su propia función dentro de una mini-librería, y una vez
que está lista, consumirla desde el reporte es una sola llamada. El mantenimiento no es distinto
al de cualquier otra función utilitaria en Harbour.

La conclusión: **si ya se tiene Docker en producción por otras razones, QuickChart es
perfectamente válido**. Pero si Docker existe únicamente para el renderizado de gráficas, el
precio es demasiado alto. LunaSVG gana en ese escenario.

## Obtener y compilar LunaSVG
```bash
git clone https://github.com/sammycage/lunasvg.git
cd lunasvg
git submodule update --init --recursive
```

El `submodule update` es importante — LunaSVG depende de **plutovg** como submódulo. Sin ese
comando la carpeta `plutovg/` queda vacía y cmake falla.

### Compilación estática con MSVC

Para que funcione con Harbour/MSVC64 se necesita compilar como librería estática con el runtime
**MT** (multi-threaded estático), que es el que usa Harbour.

> ⚠️ **Nota:** El proceso de compilación puede comportarse diferente dependiendo de la versión
> de CMake y Visual Studio en cada equipo. En mi caso, lo que funcionó fue usar un toolchain
> file que fuerza el runtime MT. Lo comparto como referencia — puede que en tu equipo compile
> sin necesitarlo.

Crear el archivo `mt_toolchain.cmake` en la raíz del repositorio:
```cmake
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded" CACHE STRING "" FORCE)
foreach(flag_var
    CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO
    CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
  if(${flag_var} MATCHES "/MD")
    string(REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
    set(${flag_var} "${${flag_var}}" CACHE STRING "" FORCE)
  endif()
endforeach()
```

Desde el **x64 Native Tools Command Prompt** de Visual Studio:
```bat
cmake -B build_static -S . ^
  -DBUILD_SHARED_LIBS=OFF ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_TOOLCHAIN_FILE=mt_toolchain.cmake ^
  -G "NMake Makefiles"

cmake --build build_static
```

Para verificar que quedó con el runtime correcto:
```bat
dumpbin /directives build_static\lunasvg.lib | findstr "DEFAULTLIB"
```

Debe aparecer `LIBCMT`. Si aparece `MSVCRT` habrá errores al linkear con Harbour.

## La arquitectura de la solución

La solución se estructura en una mini-librería `aura_charts.prg` con funciones públicas
`AURA_Chart*` que generan el SVG internamente y producen el PNG via LunaSVG. El programa
simplemente llama la función pasando un hash de opciones y recibe el archivo listo para
insertar en el reporte PDF.

El código completo está disponible en:

👉 **[github.com/jparadaa/harbour-lunasvg-wrapper](https://github.com/jparadaa/harbour-lunasvg-wrapper)**

## El resultado

![Gráfica de barras agrupadas generada con LunaSVG](/img/lunasvg-wrapper-harbour/reporte_ventas.png)

## Conclusión

Empecé buscando una alternativa más ligera a Docker y terminé construyendo una mini-librería de
gráficas para Harbour. No estaba en el plan, pero el resultado vale la pena.

Lo que más satisface de esta arquitectura es su simplicidad en producción: un solo `.exe`, sin
dependencias que instalar, sin servicios que monitorear, sin contenedores que levantar. LunaSVG
se compila una vez y queda embebida para siempre.

¿Es más código que QuickChart? Sí. ¿Vale la pena? Para mi caso, definitivamente sí.

¡Hasta la próxima!
