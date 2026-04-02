# Firmando JWT con RS256 en Harbour: El poder de hb_jsonDecode y hbssl

Hace unos días me topé con la necesidad de integrar uno de nuestros sistemas en Harbour con la API de **Google Sheets**. El reto no es menor: Google exige autenticación mediante **OAuth 2.0 para Cuentas de Servicio**, lo que implica generar un **JWT (JSON Web Token)** firmado con el algoritmo **RS256** (RSA con SHA-256).

Revisando mis notas y el histórico en el grupo de usuarios de Harbour, me di cuenta de que ya pasaron poco más de dos años desde mi primer intento. En aquel entonces desistí; no tenía un requerimiento de negocio urgente, eran simples pruebas de concepto y, honestamente, la comunidad no aportó mucha claridad sobre cómo manejar los punteros de `hbssl` para firmas asimétricas. **Hoy, como ya todos sabemos, con la ayuda de la IA como par de programación, lo tenemos sin tanto complejo.**

## El Escenario Técnico

Para que Google nos entregue un `access_token`, debemos enviarle un JWT que incluya los *claims* de la cuenta de servicio y, lo más importante, una firma generada con nuestra `private_key`. 

### 1. La importancia de hb_jsonDecode

A diferencia de mis intentos anteriores donde trataba de "parsear" el archivo JSON de Google con funciones de cadena como `At()` y `SubStr()`, la forma correcta y profesional es usar **Hashes**.

```harbour
   LOCAL hJSON := {=>}
   cJSON := hb_MemoRead( "service_account.json" )
   
   // Decodificación robusta por referencia
   hb_jsonDecode( cJSON, @hJSON )

   IF hb_HHasKey( hJSON, "private_key" )
      cKey := hJSON[ "private_key" ]
      // Google envía \n literales, hay que convertirlos a saltos reales
      cKey := StrTran( cKey, '\n', hb_bChar( 10 ) )
   ENDIF
```

El uso de `@hJSON` permite que la función pueble nuestra variable local de forma eficiente, dándonos acceso inmediato a la clave privada sin importar el formato del archivo.

### 2. La Firma RS256 con hbssl

El corazón de la solución es la librería `hbssl`. Para que funcione en entornos modernos (OpenSSL 1.1 o 3.0) y en 64 bits, la inicialización del contexto debe ser precisa.

### Ejemplo Completo de Implementación
A continuación, les comparto el código fuente íntegro para generar el token. Solo necesitan sustituir las constantes de email y el nombre del archivo JSON por los suyos. En la documentación oficial de Google Cloud encontrarán abundantes guías sobre cómo crear una "Service Account" y descargar su archivo de llaves privadas en formato JSON.

Este ejemplo utiliza el flujo completo: desde la lectura del archivo físico hasta la generación del JWT final listo para ser usado en un POST hacia los servidores de Google.

```harbour
#require "hbssl"

#define CLIENT_EMAIL  "tu-cuenta-servicio@tu-proyecto.iam.gserviceaccount.com"
#define SCOPE         "https://www.googleapis.com/auth/spreadsheets"

REQUEST __HBEXTERN__HBSSL__

FUNCTION Main()
   LOCAL cJWT
   LOCAL cJSON
   LOCAL cKey
   LOCAL hJSON := {=>}

   // Cargamos el archivo JSON descargado de Google Cloud Console
   cJSON := hb_MemoRead( "tu-archivo-credenciales.json" )

   IF Empty( cJSON )
      ? "ERROR: No se encontró el archivo JSON"
      RETURN NIL
   ENDIF

   // Convertimos el JSON a un Hash de Harbour de forma segura por referencia
   hb_jsonDecode( cJSON, @hJSON )

   // Verificamos si la clave existe en el Hash
   IF ! hb_HHasKey( hJSON, "private_key" )
      ? "ERROR: El JSON no contiene la entrada 'private_key'"
      RETURN NIL
   ENDIF

   // Extraemos la clave directamente por su nombre
   cKey := hJSON[ "private_key" ]
   
   // Google entrega la clave con '\n' literales, los convertimos a saltos de línea reales
   cKey := StrTran( cKey, '\n', hb_bChar( 10 ) )

   ? "Generando JWT RS256 con hbssl nativo..."
   cJWT := GenerarJWT_Nativo( CLIENT_EMAIL, cKey, SCOPE )

   IF Empty( cJWT )
      ? "ERROR: No se pudo generar el JWT"
      RETURN NIL
   ENDIF

   ? "JWT generado OK, longitud:", Len( cJWT )
   ? "Copia este token para probarlo en Postman:"
   ? cJWT

RETURN NIL

FUNCTION GenerarJWT_Nativo( cEmail, cPemKey, cScope )
   LOCAL nNow, nExp
   LOCAL cHeader, cPayload
   LOCAL cB64Header, cB64Payload, cSigningInput
   LOCAL cSignature, cJWT
   LOCAL bio, pKey, ctx
   LOCAL nRet

   nNow := GETUNIXTIME()
   nExp := nNow + 3600 // Token válido por 1 hora

   // 1. Definir Header y Payload
   cHeader  := '{"alg":"RS256","typ":"JWT"}'
   cPayload := '{"iss":"' + cEmail + '",' + ;
                '"scope":"' + cScope + '",' + ;
                '"aud":"https://oauth2.googleapis.com/token",' + ;
                '"iat":' + LTrim(Str(nNow)) + ',' + ;
                '"exp":' + LTrim(Str(nExp)) + '}'

   // 2. Codificar en Base64Url
   cB64Header    := Base64UrlEncode( cHeader )
   cB64Payload   := Base64UrlEncode( cPayload )
   cSigningInput := cB64Header + "." + cB64Payload

   // 3. Preparar OpenSSL para la firma
   SSL_INIT()
   OpenSSL_add_all_algorithms()
   ERR_load_PEM_strings()

   bio  := BIO_new_mem_buf( cPemKey )
   pKey := PEM_READ_PRIVATEKEY( bio, "" )

   IF Empty( pKey )
      ? "ERROR: No se pudo parsear la clave privada PEM."
      RETURN ""
   ENDIF

   // 4. Firmar usando EVP API (SHA256)
   ctx  := EVP_MD_CTX_new()
   nRet := EVP_SignInit_ex( ctx, "SHA256" )

   IF nRet != 1
      ? "ERROR: EVP_SignInit_ex falló"
      RETURN ""
   ENDIF

   nRet := EVP_SignUpdate( ctx, cSigningInput )

   IF nRet != 1
      ? "ERROR: EVP_SignUpdate falló"
      RETURN ""
   ENDIF

   cSignature := ""
   // Pasamos cSignature por referencia para que OpenSSL la pueble
   nRet := EVP_SignFinal( ctx, @cSignature, pKey )

   IF nRet != 1 .OR. Empty( cSignature )
      ? "ERROR: EVP_SignFinal falló"
      RETURN ""
   ENDIF

   // 5. Construir el JWT Final
   cJWT := cSigningInput + "." + Base64UrlEncode( cSignature )

RETURN cJWT

FUNCTION Base64UrlEncode( cData )
   LOCAL cB64 := hb_base64Encode( cData )
   // Conversión estándar de Base64 a Base64Url
   cB64 := StrTran( cB64, "+", "-" )
   cB64 := StrTran( cB64, "/", "_" )
   cB64 := StrTran( cB64, "=", "" )
   cB64 := StrTran( cB64, hb_bChar( 13 ), "" )
   cB64 := StrTran( cB64, hb_bChar( 10 ), "" )
RETURN cB64

#pragma BEGINDUMP
#include <time.h>
#include <hbapi.h>

HB_FUNC( GETUNIXTIME )
{
   hb_retnl( (long) time( NULL ) );
}
#pragma ENDDUMP
```
### Configuración del Proyecto (.hbp)
Para compilar este ejemplo, es fundamental enlazar correctamente las librerías de OpenSSL. Dependiendo de tu entorno (en mi caso MSVC64), deberás asegurarte de tener las DLLs correspondientes en tu PATH o junto al ejecutable.

Aquí les comparto mi archivo de proyecto .hbp

```harbour
# jwt_google_test.hbp
-ojwt_google_test

# Archivo fuente
jwt_google_test.prg

# Dependencias de Harbour
hbwin.hbc
hbssl.hbc

# Enlace directo con librerías de OpenSSL
-lhbssl
-lhbssls

-llibcrypto-1_1-x64
-llibssl-1_1-x64
```

### 3. Obtención del Timestamp (Unix Epoch)

Google requiere el tiempo en formato Unix (segundos desde 1970). Aunque se puede calcular en Harbour nativo, prefiero la precisión de una pequeña función en C vía `#pragma BEGINDUMP` para asegurar compatibilidad total con UTC:

```harbour
#pragma BEGINDUMP
#include <time.h>
#include <hbapi.h>

HB_FUNC( GETUNIXTIME )
{
   hb_retnl( (long) time( NULL ) );
}
#pragma ENDDUMP
```

## Conclusión

El resultado de este proceso es un JWT que, al ser enviado vía `POST` a Google, nos devuelve un `access_token` listo para usarse en cabeceras `Authorization: Bearer`.

Lo que hace dos años parecía un muro infranqueable por falta de documentación clara en la comunidad, hoy se resuelve con unas pocas líneas de código bien estructuradas. La clave está en no pelearse con los tipos de datos: usar **Hashes** para el JSON y confiar en el **EVP API** de OpenSSL para la criptografía.

Espero que esta aportación le ayude a alguien más que esté intentando modernizar sus procesos en Harbour.
