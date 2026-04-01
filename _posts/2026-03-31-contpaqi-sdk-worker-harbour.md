# SDK CONTPAQi desde Harbour: worker process de 32 bits con cola persistente en SQL Server

Integrar el SDK de CONTPAQi Comercial desde una aplicación web plantea de entrada una restricción arquitectónica concreta: el servidor web corre como proceso de 64 bits, pero `MGWServicios.dll` es estrictamente de 32 bits. No hay forma de cargar ambos en el mismo proceso. La solución es separar la ejecución en dos procesos independientes — la pregunta es cómo comunicarlos de forma confiable.

Este artículo describe la arquitectura que implementé: un worker process de 32 bits dedicado exclusivamente al SDK, comunicado con la aplicación web mediante una cola persistente en SQL Server.

---

## El modelo de ejecución del SDK: COM y el hilo único

Antes de elegir el mecanismo de comunicación entre procesos, es necesario entender cómo espera ser usado el SDK. `MGWServicios.dll` sigue el modelo COM con apartamento de hilo único — lo que en términos prácticos significa que la inicialización, el login, la apertura de empresa y cada llamada posterior deben ocurrir en el mismo hilo físico de ejecución. No es una recomendación — es una restricción del modelo COM en Windows.

Mi primer intento comunicó los procesos vía WebSockets, recibiendo peticiones en un hilo y delegando la ejecución del SDK a otro. El resultado fue errores `RPC_E_WRONG_THREAD`, bloqueos y fallos cíclicos de la DLL. El diagnóstico es simple: COM no permite que un objeto creado en un hilo sea operado desde otro.

La solución, confirmada por colegas con experiencia directa en el SDK, es ejecutar todas las operaciones sobre el hilo principal del worker process — sin excepciones.

> **Nota sobre Harbour:** Harbour es un compilador moderno sobre xBase que genera ejecutables nativos para Windows. Para efectos de llamadas a DLL vía STDCALL, se comporta de forma equivalente a cualquier proceso C++ o C#. La restricción de hilo único aplica de la misma forma.

---

## SQL Server como cola de peticiones

Con el modelo de ejecución resuelto, la siguiente decisión es el canal de comunicación entre la aplicación web (64 bits) y el worker process (32 bits).

Descarté WebSockets por un motivo concreto: si el worker process falla o se reinicia, los mensajes en vuelo se pierden. Para operaciones que crean documentos en un ERP, la pérdida silenciosa de datos no es aceptable.

La alternativa es usar SQL Server directamente como cola persistente. El flujo es el siguiente:

1. La aplicación web recibe el request HTTP, inserta una fila en la tabla de cola y devuelve el identificador asignado al cliente.
2. El frontend hace polling asíncrono consultando el estado de ese identificador hasta recibir la confirmación de procesado.
3. El worker process interroga la cola periódicamente, ejecuta la operación contra CONTPAQi y actualiza el registro con el resultado.

La ventaja de este esquema es la durabilidad: si el worker process cae y se reinicia, los registros pendientes siguen ahí esperando. No hay mensajes perdidos.

---

## El bucle principal del worker process

El worker opera exclusivamente sobre su hilo principal para respetar la restricción de hilo único de COM. El bucle es directo:
```harbour
DO WHILE lContinue
    nKey := Inkey( 0.1 )
    IF nKey == 27
        lContinue := .F.
    ELSE
        IF ProcessNext()
            // procesó una instrucción, vuelve inmediatamente
        ELSE
            hb_idleSleep( 0.5 ) // sin trabajo, pausa de 500ms
        ENDIF
    ENDIF
ENDDO
```

Cuando hay trabajo, el ciclo es ágil. Cuando la cola está vacía, cede CPU con una pausa de 500ms antes de volver a consultar. `ProcessNext()` extrae el siguiente registro pendiente, ejecuta la operación SDK y escribe la respuesta — todo en el mismo hilo, sin excepciones.

---

## Integración con el SDK: inicialización y ejecución

### Arranque y cierre del worker

`IniciarSDK()` se ejecuta una sola vez al arrancar el worker. Carga la DLL con `hb_LibLoad()`, establece la sesión con el SDK y abre la empresa inicial. Si cualquier paso falla, el worker se detiene — no tiene sentido continuar sin una conexión válida al SDK. `TerminarSDK()` hace el camino inverso: cierra la empresa activa si la hay, termina la sesión y libera la DLL.
```harbour
FUNCTION IniciarSDK()
  LOCAL nError
  LOCAL cError := Space( 512 )

  hb_Cwd( SDK_DIR )
  ? "CWD: " + hb_Cwd()

  nConn := hb_LibLoad( SDK_DLL )
  IF Empty( nConn )
    ? "ERROR: No se pudo cargar DLL: " + SDK_DLL
    RETURN .F.
  ENDIF
  ? "OK: DLL cargada."

  hb_DynCall( { "fInicioSesionSDK", nConn, HB_DYN_CALLCONV_STDCALL }, SDK_USER, SDK_PASS )
  ? "OK: fInicioSesionSDK ejecutado."

  nError := hb_DynCall( { "fSetNombrePAQ", nConn, HB_DYN_CALLCONV_STDCALL }, SDK_PAQ )
  IF nError != 0
    cError := Space( 512 )
    hb_DynCall( { "fError", nConn, HB_DYN_CALLCONV_STDCALL }, nError, @cError, 512 )
    ? "ERROR fSetNombrePAQ(" + AllTrim( Str( nError ) ) + "): " + AllTrim( StrTran( cError, Chr(0), "" ) )
    hb_LibFree( nConn )
    nConn := 0
    RETURN .F.
  ENDIF
  ? "OK: fSetNombrePAQ."

  hb_DynCall( { "fInicioSesionSDKCONTPAQi", nConn, HB_DYN_CALLCONV_STDCALL }, SDK_USER, "" )
  ? "OK: fInicioSesionSDKCONTPAQi ejecutado."

  nError := hb_DynCall( { "fAbreEmpresa", nConn, HB_DYN_CALLCONV_STDCALL }, SDK_EMPRESA )
  IF nError != 0
    cError := Space( 512 )
    hb_DynCall( { "fError", nConn, HB_DYN_CALLCONV_STDCALL }, nError, @cError, 512 )
    ? "ERROR fAbreEmpresa(" + AllTrim( Str( nError ) ) + "): " + AllTrim( StrTran( cError, Chr(0), "" ) )
    hb_LibFree( nConn )
    nConn := 0
    RETURN .F.
  ENDIF
  cEmpresaAbierta := SDK_EMPRESA
  ? "OK: Empresa " + cEmpresaAbierta + " abierta correctamente durante el inicio."

RETURN .T.

// ----------------------------------------------------------------//
FUNCTION TerminarSDK()
  IF ! Empty( nConn )
    IF ! Empty( cEmpresaAbierta )
      hb_DynCall( { "fCierraEmpresa", nConn, HB_DYN_CALLCONV_STDCALL } )
      ? "OK: Empresa cerrada."
      cEmpresaAbierta := ""
    ENDIF
    hb_DynCall( { "fTerminaSDK", nConn, HB_DYN_CALLCONV_STDCALL } )
    hb_LibFree( nConn )
    nConn := NIL
    ? "OK: SDK finalizado."
  ENDIF
RETURN NIL
```

### La estructura de datos: el puente entre Harbour y el SDK

El SDK espera recibir sus parámetros como estructuras de C — tipos y tamaños fijos en memoria, sin ninguna abstracción de alto nivel. `SProducto()` construye esa estructura usando `cStructure`, la clase que David Field implementó específicamente para este propósito. Sin ella, pasar datos correctamente a `MGWServicios.dll` desde Harbour no habría sido posible.
```harbour
FUNCTION SProducto()
  LOCAL oProducto := cStructure()

  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoProducto",           31 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cNombreProducto",           61 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cDescripcionProducto",     256 )
  oProducto:ADDITEM( CTYPE_INT,    "cTipoProducto"                )
  oProducto:ADDITEM( CTYPE_CHAR,   "cFechaAltaProducto",        24 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cFechaBaja",                24 )
  oProducto:ADDITEM( CTYPE_INT,    "cStatusProducto"              )
  oProducto:ADDITEM( CTYPE_INT,    "cControlExistencia"           )
  oProducto:ADDITEM( CTYPE_INT,    "cMetodoCosteo"               )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoUnidadBase",          31 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoUnidadNoConvertible", 33 )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio1"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio2"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio3"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio4"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio5"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio6"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio7"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio8"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio9"                     )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cPrecio10"                    )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImpuesto1"                   )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImpuesto2"                   )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImpuesto3"                   )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cRetencion1"                  )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cRetencion2"                  )
  oProducto:ADDITEM( CTYPE_CHAR,   "cNombreCaracteristica1",     61 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cNombreCaracteristica2",     61 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cNombreCaracteristica3",     61 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion1",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion2",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion3",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion4",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion5",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cCodigoValorClasificacion6",  4 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cTextoExtra1",               51 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cTextoExtra2",               51 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cTextoExtra3",               51 )
  oProducto:ADDITEM( CTYPE_CHAR,   "cFechaExtra",                24 )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImporteExtra1"               )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImporteExtra2"               )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImporteExtra3"               )
  oProducto:ADDITEM( CTYPE_DOUBLE, "cImporteExtra4"               )

RETURN oProducto
```

### Ejecución de operaciones y gestión de empresa

`EjecutarSDK()` es el núcleo del worker: recibe la acción, verifica si la empresa solicitada ya está abierta y solo llama a `fAbreEmpresa()` cuando hay un cambio. El dispatch por acción vive en el `DO CASE` — cada nueva operación que el sistema necesite se agrega aquí como un `CASE` adicional.
```harbour
FUNCTION EjecutarSDK( cAccion, cEmpresa, hPayload )
  LOCAL nError
  LOCAL cError := Space( 512 )
  LOCAL hRes   := { "status" => "ERROR", "msg" => "" }
  LOCAL nId    := 0
  LOCAL oProducto
  LOCAL nTick

  IF cEmpresaAbierta != cEmpresa
    IF ! Empty( cEmpresaAbierta )
      hb_DynCall( { "fCierraEmpresa", nConn, HB_DYN_CALLCONV_STDCALL } )
      ? "   Empresa anterior cerrada: " + cEmpresaAbierta
    ENDIF

    nTick  := hb_MilliSeconds()
    nError := hb_DynCall( { "fAbreEmpresa", nConn, HB_DYN_CALLCONV_STDCALL }, cEmpresa )
    ? "   [TIMER] fAbreEmpresa: " + hb_ntos( hb_MilliSeconds() - nTick ) + "ms"

    IF nError != 0
      cError := Space( 512 )
      hb_DynCall( { "fError", nConn, HB_DYN_CALLCONV_STDCALL }, nError, @cError, 512 )
      hRes[ "msg" ] := "fAbreEmpresa(" + AllTrim( Str( nError ) ) + "): " + AllTrim( StrTran( cError, Chr(0), "" ) )
      ? "   [!] " + hRes[ "msg" ]
      cEmpresaAbierta := ""
      RETURN hb_jsonEncode( hRes )
    ENDIF

    cEmpresaAbierta := cEmpresa
    ? "   Empresa abierta: " + cEmpresa
  ELSE
    ? "   Empresa ya abierta, reutilizando."
  ENDIF

  DO CASE
    CASE cAccion == "ALTA_PRODUCTO"

      oProducto := SProducto()
      oProducto:cCodigoProducto      := hPayload[ "codigo" ]
      oProducto:cNombreProducto      := hPayload[ "nombre" ]
      oProducto:cDescripcionProducto := hPayload[ "nombre" ]
      oProducto:cFechaAltaProducto   := hb_DtoC( Date(), "mm/dd/yyyy" )
      oProducto:cCodigoUnidadBase    := "PIEZA"
      oProducto:cTipoProducto        := 1
      oProducto:cControlExistencia   := 1
      oProducto:cPrecio1             := hPayload[ "precio" ]

      nTick  := hb_MilliSeconds()
      ? "   fAltaProducto: " + hPayload[ "codigo" ]
      nError := hb_DynCall( { "fAltaProducto", nConn, HB_DYN_CALLCONV_STDCALL }, @nId, oProducto:AsStructure() )
      ? "   [TIMER] fAltaProducto: " + hb_ntos( hb_MilliSeconds() - nTick ) + "ms"

      IF nError == 0
        hRes[ "status" ] := "OK"
        hRes[ "msg"    ] := "Producto creado. ID=" + LTrim( Str( nId ) )
        ? "   [+] " + hRes[ "msg" ]
      ELSE
        cError := Space( 512 )
        hb_DynCall( { "fError", nConn, HB_DYN_CALLCONV_STDCALL }, nError, @cError, 512 )
        hRes[ "msg" ] := "fAltaProducto(" + AllTrim( Str( nError ) ) + "): " + AllTrim( StrTran( cError, Chr(0), "" ) )
        ? "   [-] " + hRes[ "msg" ]
      ENDIF

    OTHERWISE
      hRes[ "msg" ] := "Accion desconocida: " + cAccion
      ? "   [!] " + hRes[ "msg" ]

  ENDCASE

RETURN hb_jsonEncode( hRes )
```

---

## El cuello de botella: `fAbreEmpresa()`

Con la arquitectura funcionando correctamente, el perfil de rendimiento reveló el siguiente problema: `fAbreEmpresa()` es cara. En mis pruebas, el arranque en frío tarda entre 9 y 12 segundos. Incluso en caliente — con el motor ya inicializado — el tiempo ronda los 4 a 6 segundos por llamada. Invocarla en cada petición hacía el sistema inviable en la práctica.

La solución, como se puede ver en `EjecutarSDK()`, es retener el estado: el worker guarda qué empresa tiene abierta actualmente y solo llama a `fAbreEmpresa()` cuando la petición entrante corresponde a una empresa distinta. El complemento natural es ajustar la prioridad en la cola para que el worker prefiera procesar primero las peticiones dirigidas a la empresa actualmente abierta, minimizando los cambios de contexto.

El resultado medido en mis pruebas: arranque en frío entre 4 y 6 segundos para la primera operación sobre una empresa. Las operaciones subsecuentes sobre la misma empresa: milisegundos. Para las pruebas iniciales usé creación de productos por ser la operación más simple disponible; los tiempos con documentos deberían ser similares una vez implementada esa parte.

---

## Agradecimientos

**David Field** implementó y me proporcionó la clase Harbour que mapea las estructuras de datos de C que requiere el SDK. Sin esa pieza, la integración con `MGWServicios.dll` desde Harbour simplemente no habría sido posible.

**Gabriel Ríos** me orientó en los dos puntos críticos del diseño: que toda interacción con el SDK debe ocurrir en el mismo hilo, y que `fAbreEmpresa()` debe administrarse como estado persistente del worker en lugar de invocarse por cada petición. Ambas recomendaciones cambiaron el rumbo de la implementación. La segunda la adopté con cierta cautela — Gabriel es el experto y los números en mis pruebas lo respaldan, pero es una estrategia que hay que observar con cuidado en producción: cómo se comporta bajo carga real, si los tiempos de espera se mantienen razonables, si el worker se mantiene estable en operación prolongada. Por ahora funciona, y los resultados son prometedores.
