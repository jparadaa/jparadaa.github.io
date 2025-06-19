# Primeros Pasos con TDolphin en Ubuntu WSL

Continuando con mis pruebas, ahora estoy explorando TDolphin. La razón principal por la que decidí probarlo es que con WDO no logré implementar un ejemplo en particular (al final incluiré una nota sobre cómo lo resolví con WDO; sí, finalmente lo conseguí).

Me descargué el código de la biblioteca desde:
[https://bitbucket.org/danielgarciagil/workspace/repositories/](https://bitbucket.org/danielgarciagil/workspace/repositories/)

En mi directorio home, creé una carpeta para TDolphin con el comando:

```bash
mkdir tdolphin
```

Luego, simplemente ejecuté:

```bash
hbmk2 tdolphin.hbp
```

A pesar de algunos warnings, se generó la biblioteca `libtdolphin.a` en el directorio `/tdolphin/lib`

Para que esté disponible, hay que copiarla al directorio de bibliotecas de Harbour (gracias, Riztan). Esto lo hice con:

```bash
cd lib
sudo cp libtdolphin.a /usr/local/lib/harbour/
```

Todo esto funciona porque previamente instalé MySQL y sus dependencias. Hablé de ese tema aquí:
[https://jparadaa.blogspot.com/2025/02/primeros-pasos-con-wdo-en-ubuntu-wsl.html](https://jparadaa.blogspot.com/2025/02/primeros-pasos-con-wdo-en-ubuntu-wsl.html)

Mi primera prueba consistió en verificar si podía conectar a MySQL con TDolphin. Para ello, utilicé el ejemplo `testcon.prg`. Primero, edité el archivo `connect.ini` para ingresar los datos de conexión a mi servidor:

```ini
[mysql]
host=localhost
user=harbour
psw=hb1234
flags=0
port=3306
dbname=dbharbour
```

Compilé el programa con:

```bash
hbmk2 testcon.prg
```

Y al ejecutarlo con `./testcon`, ¡funcionó perfecto en mi caso!.

Ahora, el siguiente paso fue probar directamente con procedimientos almacenados. ¿Por qué? En mi caso, como usuario de MSSQL, prácticamente todo lo hago con procedimientos almacenados (usando ADO). Personalmente, no me gustan las consultas SQL embebidas; esa es mi forma de trabajar. Por eso, tiene sentido para mí comprobar si puedo manejar procedimientos almacenados en MySQL con parámetros de entrada y salida. Aquí fue donde fallé al intentar obtener los parámetros de salida con WDO.

## Creando un procedimiento almacenado simple

```mysql
DELIMITER $$
CREATE PROCEDURE GetCustomerDetails(
    IN startid INT,
    IN endid INT,
    OUT row_count INT
)
BEGIN
    SELECT id, first, city 
    FROM customer 
    WHERE id BETWEEN startid AND endid;
    -- Obtener el número total de filas en el rango
    SELECT COUNT(*) INTO row_count 
    FROM customer 
    WHERE id BETWEEN startid AND endid;
END $$
DELIMITER ;
```

Este procedimiento recibe dos parámetros de entrada (ID inicial y final del cliente que quiero consultar) y devuelve un parámetro de salida con el número de filas retornadas. Aunque en el código Harbour podemos obtener esta información directamente, no es lo que busco.

## Creando el archivo de prueba test_proc_out.prg

```harbour
#include "tdolphin.ch"

PROCEDURE Main()
  LOCAL oServer, oQry, oQryCount
    LOCAL cText := "CALL GetCustomerDetails(1, 32, @count)"
    
    IF ( oServer := ConnectTo() ) == NIL
      RETURN
    ENDIF
    
    oQry := oServer:Query( cText )
    
    ? "Resultados de clientes:"
    WHILE !oQry:Eof()
      ? "ID:", oQry:FieldGet("id"), ;
        "Nombre:", oQry:FieldGet("first"), ;
        "Ciudad:", oQry:FieldGet("city")
      oQry:Skip(1)
    END
    oQry:End()
    
    oServer:NextResult()
    
    // Obtener el valor del parámetro de salida @count
    oQryCount := oServer:Query("SELECT @count AS count")
    IF oQryCount:LastRec() > 0
      ? hb_utf8tostr("Número de filas:"), oQryCount:FieldGet("count")
    ENDIF
    oQryCount:End()
    
RETURN

FUNCTION ConnectTo()
  LOCAL oServer
  oServer := TDolphinSrv():New( "localhost", "harbour", "hb1234", ;
                                  3306, , "dbharbour" )
RETURN oServer
```

Compilamos con:

```bash
hbmk2 test_proc_out.prg -run
```

Si todo sale bien, deberíamos obtener una salida similar a:

```
ID: 27 Nombre: Gordon   Ciudad: Brick
ID: 28 Nombre: Robert   Ciudad: Lancaster
ID: 29 Nombre: Aaron    Ciudad: Indianapolis
Número de filas: 31
```

Esto demuestra que el código funciona y recupera correctamente el parámetro de salida, que es justo lo que necesitaba.

Lo importante aquí es notar el uso de `oServer:NextResult()`. Sin esa línea, nos encontraríamos con un error que me tuvo ocupado un par de días:

```
Commands out of sync; you can't run this command now
```

La explicación es la siguiente:

En MySQL, los procedimientos almacenados que devuelven conjuntos de resultados y parámetros de salida mantienen la conexión en un estado "multi-resultado". Hasta que no "limpies" o avances explícitamente esos resultados, la conexión no está lista para aceptar una nueva consulta. En nuestro ejemplo, después de procesar el conjunto de resultados con `oQry` y cerrarlo con `oQry:End()`, el servidor aún tiene información pendiente relacionada con la ejecución del procedimiento (metadatos o el estado de los parámetros de salida). Sin `oServer:NextResult()`, la conexión no se "reinicia" por completo, y la siguiente consulta (`SELECT @count`) falla porque el servidor considera que no has terminado de manejar la respuesta anterior.

Por lo tanto, `oServer:NextResult()` le indica a TDolphin que hemos terminado con el primer conjunto de resultados, permitiendo avanzar al siguiente o limpiar cualquier estado pendiente. Esto asegura que la conexión esté lista para la siguiente consulta y evita errores.

## Nota final sobre WDO

Con WDO, precisamente recibía el error `Commands out of sync; you can't run this command now`. En este caso, WDO no tiene un método equivalente a `NextResult()`. Como solución, podemos implementar esta función en nuestro código de prueba (sin modificar la clase, por ahora):

```harbour
FUNCTION MySqlNextResult(pLib, hMySql)
RETURN hb_DynCall( { "mysql_next_result", pLib, hb_bitOr( HB_DYN_CTYPE_INT, hb_SysCallConv() ), hb_SysLong() }, hMySql )
```

¡Y listo! Así resolvemos el tema con WDO.

Aquí comparto mi experiencia y los pasos que seguí, que me funcionaron. Espero que estas notas sean útiles para alguien más.
