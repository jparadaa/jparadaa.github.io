# Optimización de persistencia ADO y SQL Server en Harbour: Un enfoque de alto rendimiento

Al construir la capa de datos de mis aplicaciones web con Harbour y UHttpd2, identifiqué que el mayor cuello de botella no estaba en la complejidad de las consultas SQL, sino en algo más fundamental: el costo de establecer una conexión con SQL Server en cada request HTTP.

Esto no es una intuición — es algo que medí. Instrumenté cada función de acceso a datos y descubrí que, en peticiones que deberían resolverse en milisegundos, la mayor parte del tiempo la consumía la apertura de la conexión ADO antes de ejecutar cualquier consulta.

Establecer una sesión con SQL Server no es una operación trivial. Implica una negociación de protocolo TDS, autenticación de credenciales y asignación de recursos en el motor. Repetida en cada request de un servidor multi-hilo, esa latencia acumulada degrada el rendimiento del sistema de forma predecible y silenciosa.

---

## 1. Gestión de conexiones persistentes por hilo

La solución es directa una vez que se entiende el modelo de ejecución de UHttpd2: cada request es atendido por un hilo del pool del servidor, y ese hilo persiste entre requests. Esto abre la posibilidad de mantener recursos costosos — como una conexión ADO abierta — vinculados al ciclo de vida del hilo en lugar del ciclo de vida del request.

Para eso utilizo `THREAD STATIC`. Una variable declarada con esta directiva se inicializa una sola vez por hilo y conserva su valor entre llamadas sucesivas dentro del mismo hilo. Es el equivalente a una variable global, pero con alcance estrictamente local al hilo de ejecución — exactamente lo que necesitamos para una conexión a base de datos.

La lógica de `GetADOConnection()` aplica un patrón de *lazy initialization* con validación de estado: si la conexión del hilo ya existe y está abierta (`State != 0`), se devuelve inmediatamente sin negociar nada con el servidor. Solo en el primer request del hilo — o tras un error que invalide la conexión — se abre una nueva sesión. El resultado práctico: en un servidor con un pool de, digamos, 8 hilos, SQL Server recibe exactamente 8 conexiones durante la vida del proceso, sin importar cuántas miles de peticiones procese la aplicación.

`ResetADOConnection()` complementa este patrón: ante cualquier error en tiempo de ejecución, destruye el objeto `toConn` del hilo actual, forzando una reconexión limpia en el siguiente request. Sin este mecanismo, un hilo con una conexión zombie seguiría fallando indefinidamente.

```harbour
#include "files/include/ado.ch"

#xcommand TRY  => BEGIN SEQUENCE WITH {| oErr | Break( oErr ) }
#xcommand CATCH [<!oErr!>] => RECOVER [USING <oErr>] <-oErr->
#xcommand FINALLY => ALWAYS

THREAD STATIC toConn := NIL

FUNCTION GetADOConnection()
    LOCAL cConnString
    LOCAL oError

    IF ! hb_IsObject( toConn )
        toConn := Win_OleCreateObject("ADODB.Connection")
        IF ! hb_IsObject( toConn )
            QOut( '[GetADOConnection] ERROR: No se pudo crear objeto ADODB.Connection' )
            RETURN NIL
        ENDIF
    ENDIF

    IF toConn:State == 0
        cConnString := "Provider=" + AppCfg('sql','provider') + ;
                       ";Server="   + AppCfg('sql','server')   + ;
                       ";Database=" + AppCfg('sql','database') + ;
                       ";Uid="      + AppCfg('sql','user')     + ;
                       ";Pwd="      + AppCfg('sql','pass')     + ";"
        TRY
            toConn:Open( cConnString )
        CATCH oError
            QOut( '[GetADOConnection] ERROR al abrir conexión: ' + oError:Description )
            toConn := NIL
            RETURN NIL
        END
    ENDIF

RETURN toConn

FUNCTION ResetADOConnection()
    IF hb_IsObject( toConn )
        TRY
            IF toConn:State != 0
                toConn:Close()
            ENDIF
        CATCH
        END
    ENDIF
    toConn := NIL
RETURN NIL
```

---

## 2. Implementación de listado paginado

Con la conexión resuelta en `GetADOConnection()`, la función `ListDocuments()` se enfoca exclusivamente en la lógica de negocio. Utilizo `ADODB.Command` para invocar el stored procedure, lo que garantiza el tipado correcto de los parámetros y protege contra SQL Injection sin ninguna sanitización manual.

Un detalle deliberado: `ListDocuments()` no llama a `Close()` sobre la conexión al terminar. Esto es intencional — la conexión pertenece al hilo, no al request. Cerrarla aquí destruiría la ventaja que acabamos de construir, obligando a `GetADOConnection()` a renegociar con SQL Server en el siguiente request. Lo que sí se cierra explícitamente es el `Recordset`, que es un cursor de resultados con recursos propios en el servidor, y cuyo ciclo de vida sí está acotado al request actual.

La instrumentación de tiempos está deliberadamente incrustada en la función: permite comparar en los logs del servidor cuánto toma cada fase — validar la conexión, ejecutar el SP, iterar el RecordSet y serializar a JSON — lo que facilita identificar regresiones de rendimiento si el esquema o los datos cambian.

```harbour
FUNCTION ListDocuments()
    LOCAL hGet           := UGet()
    LOCAL nPage          := Val( hGet["page"] )
    LOCAL nSize          := Val( hGet["size"] )
    LOCAL hCredentials   := {=>}
    LOCAL cAgente
    LOCAL oConn
    LOCAL oError
    LOCAL oRs
    LOCAL oCmd
    LOCAL aRows          := {}
    LOCAL hResp          := {=>}
    LOCAL hErrResp       := {=>}
    LOCAL nTotalRows     := 0
    LOCAL lSuccess       := .F.
    LOCAL nT0, nT1, nT2, nT3, nT4, nT5

    IF ! USessionReady()
        URedirect('login')
        RETURN NIL
    ENDIF

    hCredentials := USession('credenciales')
    cAgente      := hCredentials['agente']

    nT0   := hb_MilliSeconds()
    oConn := GetADOConnection()

    IF oConn == NIL
        hErrResp['success'] := .F.
        hErrResp['data']    := {}
        hErrResp['error']   := .T.
        hErrResp['message'] := "Error crítico: No hay conexión a la base de datos"
        RETURN hb_jsonencode( hErrResp )
    ENDIF

    nT1 := hb_MilliSeconds()
    QOut( '[ListDocuments] conexion check/open: ' + LTrim( Str( nT1 - nT0, 8 ) ) + ' ms' )

    TRY
        oCmd := Win_OleCreateObject("ADODB.Command")

        WITH OBJECT oCmd
            :ActiveConnection := oConn
            :CommandText      := "dbo.usp_GetDocumentsPaged"
            :CommandType      := adCmdStoredProc
            :Parameters:Append( :CreateParameter("page",   adInteger, adParamInput, 0,  nPage  ) )
            :Parameters:Append( :CreateParameter("size",   adInteger, adParamInput, 0,  nSize  ) )
            :Parameters:Append( :CreateParameter("agente", adVarChar, adParamInput, 12, cAgente) )
        END WITH

        nT2 := hb_MilliSeconds()
        oRs  := oCmd:Execute()
        nT3  := hb_MilliSeconds()
        QOut( '[ListDocuments] sp execute:       ' + LTrim( Str( nT3 - nT2, 8 ) ) + ' ms' )

        WHILE ! oRs:Eof()
            nTotalRows := oRs:Fields('TotalFilas'):Value
            AAdd( aRows, { ;
                'id'            => oRs:Fields('idcotizacion'):Value,        ;
                'fecha'         => hb_TSToStr( oRs:Fields('fecha'):Value ), ;
                'total'         => oRs:Fields('total'):Value,               ;
                'estatus'       => oRs:Fields('estatus'):Value,             ;
                'codigoagente'  => oRs:Fields('codigoagente'):Value,        ;
                'agente'        => oRs:Fields('agente'):Value,              ;
                'codigocliente' => oRs:Fields('codigocliente'):Value,       ;
                'cliente'       => oRs:Fields('cliente'):Value,             ;
                'referencia'    => oRs:Fields('referencia'):Value,          ;
                'totalFilas'    => nTotalRows                               ;
            } )
            oRs:MoveNext()
        END WHILE

        nT4 := hb_MilliSeconds()
        QOut( '[ListDocuments] fetch rows:       ' + LTrim( Str( nT4 - nT3, 8 ) ) + ' ms  (' + LTrim( Str( Len( aRows ) ) ) + ' filas)' )

        hResp["last_page"] := Int( nTotalRows / nSize ) + IIF( nTotalRows % nSize == 0, 0, 1 )
        hResp['data']      := aRows

        nT5 := hb_MilliSeconds()
        QOut( '[ListDocuments] json encode:      ' + LTrim( Str( nT5 - nT4, 8 ) ) + ' ms' )
        QOut( '[ListDocuments] TOTAL:            ' + LTrim( Str( nT5 - nT0, 8 ) ) + ' ms' )

        lSuccess := .T.

    CATCH oError
        QOut( '[ListDocuments] Error: ' + oError:Description )
        ResetADOConnection()
        hErrResp['success'] := .F.
        hErrResp['data']    := {}
        hErrResp['error']   := .T.
        hErrResp['message'] := "Error en consulta: " + oError:Description

    FINALLY
        IF hb_IsObject( oRs )
            IF oRs:State != 0
                oRs:Close()
            ENDIF
            oRs := NIL
        ENDIF
        IF hb_IsObject( oCmd )
            oCmd := NIL
        ENDIF

    END

    IF ! lSuccess
        RETURN hb_jsonencode( hErrResp )
    ENDIF

RETURN hb_jsonencode( hResp )
```

---

## Justificación técnica: `TotalFilas` como columna vs. parámetro `OUTPUT`

Es una pregunta razonable: ¿por qué devolver el conteo total como una columna en el RecordSet y no como un parámetro `OUTPUT` del stored procedure? Hay tres razones técnicas concretas.

**Un solo flujo de datos.** Cuando ADO recibe un RecordSet, todos los datos — filas y metadatos — llegan en una sola transferencia de red. Los parámetros `OUTPUT` de SQL Server, en cambio, solo están disponibles *después* de que el cursor se ha procesado completamente y cerrado. En la práctica esto significa que ADO debe cerrar el RecordSet, hacer una segunda lectura del objeto `Command` para leer los parámetros, y solo entonces tiene acceso al total. Eso es latencia adicional que no existe cuando el dato viene en el mismo flujo.

**Costo marginal en SQL Server.** Calcular el total con `COUNT(*) OVER()` — una función de ventana — se ejecuta sobre el mismo conjunto de filas que ya está siendo procesado para la paginación. El motor no necesita un segundo escaneo: el conteo sale del mismo plan de ejecución. El costo computacional de incluirlo es prácticamente nulo comparado con el overhead de gestionar un parámetro de salida.

**Comportamiento consistente independiente del cursor.** Los parámetros `OUTPUT` pueden comportarse de forma diferente según el tipo de cursor que ADO negocie con el proveedor OLE DB — cursores de servidor vs. cliente tienen reglas distintas sobre cuándo los parámetros de salida son accesibles. Devolver el total como columna elimina esa variabilidad: el dato está disponible en el primer `Fields()` de la primera fila, sin importar la configuración del proveedor ni el tipo de cursor.

El resultado es una arquitectura de datos más predecible, con menos round-trips y sin dependencias en el comportamiento específico del driver ADO.

---

## Resultados reales de la implementación

Estos son los tiempos medidos en producción antes y después de implementar el patrón `THREAD STATIC` + `GetADOConnection()`:

|                          | Antes    | Después | Reducción |
|--------------------------|----------|---------|-----------|
| Primera llamada          | ~494 ms  | 50 ms   | 90%       |
| Segunda llamada (crucero)| ~461 ms  | 18 ms   | 96%       |

La primera llamada del día incluye el `Open` de la conexión (~14 ms) más la ejecución del SP con plan frío (~36 ms). A partir de la segunda llamada — con la conexión persistente ya establecida y el plan de ejecución en caché — el tiempo de crucero real es de 18 ms. El patrón elimina prácticamente por completo la latencia de conexión que dominaba cada request.

---

> **Nota:** Una pregunta natural al revisar este patrón es cuándo y cómo se cierran estas conexiones persistentes. La respuesta es simple: cuando finaliza el proceso del servidor UHttpd2, el sistema operativo libera todos los recursos — memoria, handles, sockets y conexiones COM — de forma inmediata y ordenada. SQL Server detecta el cierre del socket TCP y limpia las sesiones de su lado en cuestión de segundos. No se requiere ningún mecanismo adicional de shutdown para este tipo de arquitectura.
