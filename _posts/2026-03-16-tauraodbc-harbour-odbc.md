# Conectando Harbour a SQL Server en Linux con ODBC nativo

En todos mis proyectos Harbour la conexión a SQL Server siempre ha sido
mediante ADO — `win_OleCreateObject("ADODB.Connection")` de `hbwin`. Funciona
bien pero tiene una limitación clara: es exclusivo de Windows.

Uno de los proyectos que tengo a mediano plazo es intentar migrar mi
aplicación en producción a Linux. Una de las razones es el costo — los
servidores Linux en general son más económicos que sus equivalentes Windows.
El problema es que el framework UHttpd2 con el que trabajo sí puede compilarse
y ejecutarse en Linux, pero me quedaba sin solución para la capa de datos.

Así que construimos **TAuraODBC**: un wrapper sobre `libhbodbc`, el contrib
oficial de Harbour, que expone una API orientada a objetos similar a ADO y
funciona igual en Windows y Linux.

> Todo lo documentado aquí fue probado sobre Ubuntu 24.04. Los comandos
> de instalación y configuración me funcionaron en esa versión — en otras
> distribuciones o versiones puede haber diferencias.

---

## Lo que hace

La clase envuelve las funciones C de `libhbodbc` y expone una API que
cualquier desarrollador Harbour reconoce inmediatamente:
```harbour
oDb := TAuraODBC():New( cConnStr )

IF oDb:Connect()
   IF oDb:Execute( "SELECT sku, nombre FROM productos WHERE categoria = ?", ;
                   { "Filtros" } )
      DO WHILE oDb:Fetch()
         ? oDb:FieldByName( "sku" ), oDb:FieldByName( "nombre" )
      ENDDO
      oDb:Close()
   ENDIF
   oDb:Disconnect()
ENDIF
```

Eso es todo lo que necesitas para una consulta con parámetros seguros
desde Linux conectando a SQL Server.

---

## Los problemas que encontré

### El driver ODBC en Ubuntu 24.04

Microsoft no publica paquetes nativos para Ubuntu 24.04 todavía.
El repositorio que funciona es el de Ubuntu 22.04. Hay que agregarlo
manualmente:
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft-prod.gpg] \
https://packages.microsoft.com/ubuntu/22.04/prod jammy main" | \
sudo tee /etc/apt/sources.list.d/mssql-release.list

sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install -y msodbcsql17 unixodbc unixodbc-dev
```

### Conectando desde WSL a SQL Server en Windows

Como tengo SQL Server en una instalación de Windows, lo que hice fue
usar esa misma instalación desde WSL. Desde WSL no se llega a SQL Server
por `localhost` sino por la IP del host Windows, que se obtiene con:
```bash
ip route show default | awk '{print $3}'
```

La connection string queda así:
```harbour
#define CONN_STR "Driver={ODBC Driver 17 for SQL Server};" + ;
                 "Server=IP_SERVIDOR\INSTANCIA;"           + ;
                 "Database=MiBase;"                        + ;
                 "Uid=usuario;"                            + ;
                 "Pwd=password;"
```

Ten en cuenta que esta IP puede cambiar con cada reinicio de Windows.

### El `?` dentro de literales SQL

Al implementar la sustitución de parámetros encontré un caso edge que
no es obvio: si el SQL contiene un `?` dentro de un literal de string
— por ejemplo `'¿Qué pasó?'` — el parser ingenuo lo confundiría con
un marcador de parámetro.

La solución fue hacer un parser carácter por carácter que rastrea si
estás dentro de un literal de string, incluyendo el caso de comillas
escapadas `''` dentro del literal:
```harbour
DO WHILE i <= Len( cSQL )
   cChar := SubStr( cSQL, i, 1 )
   cNext := SubStr( cSQL, i + 1, 1 )
   IF cChar == "'"
      IF lInString .AND. cNext == "'"  // comilla escapada: ''
         cSQLFinal += "''"
         i += 2
         LOOP
      ELSE
         lInString := ! lInString
         cSQLFinal += cChar
      ENDIF
   ELSEIF cChar == "?" .AND. ! lInString
      cSQLFinal += ::_Escape( aParams[ nParamIdx ] )
      nParamIdx++
   ELSE
      cSQLFinal += cChar
   ENDIF
   i++
ENDDO
```

### Memory leak con SQLFreeStmt

La primera versión del método `Close()` dejaba el handle del statement
sin liberar correctamente. La corrección fue llamar `SQLFreeStmt` con
el flag `SQL_CLOSE` antes de nulificar el handle:
```harbour
METHOD Close() CLASS TAuraODBC
   IF ::hStmt != NIL
      SQLFreeStmt( ::hStmt, 1 )
   ENDIF
   ::hStmt := NIL
   ...
RETURN NIL
```

---

## Resultados de las pruebas

Todas las pruebas pasan sobre WSL conectando a SQL Server 2022 Express
en Windows:
```
============================================
 RESULTADO FINAL
  OK  :         29
  FAIL:          0
============================================
```

Los números de rendimiento con esa configuración:

| Prueba | Resultado |
|---|---|
| 1000 SELECTs consecutivos | 1975ms / 1.9ms promedio |
| 500 INSERTs en transacción | 1209ms / 2.4ms por INSERT |

En un servidor Linux real conectado directamente a SQL Server los tiempos
serían menores.

---

## Sobre el desarrollo

TAuraODBC fue desarrollada íntegramente con ayuda de Claude (Anthropic)
como par de programación — mi trabajo fue iterar, definir los requerimientos,
probar en producción y tomar decisiones de diseño. Una vez lista, pasé el
código por revisión con Gemini Pro (Google) que detectó varios problemas
concretos: el memory leak con `SQLFreeStmt`, el caso edge del parser de
strings con literales `''` y algunas simplificaciones en el escape.

Creo que este flujo — usar IA para construir, usar otra IA para revisar,
y tener un humano que entiende el problema y valida todo — es bastante
sólido para proyectos internos.

---

El repo está en [github.com/jparadaa/harbour-auraodbc](https://github.com/jparadaa/harbour-auraodbc).

> **Nota:** El prefijo *Aura* en TAuraODBC hace referencia a mi asistente
> interno de IA — **AURA** (Agente de Utilidad y Respuesta Avanzada).
