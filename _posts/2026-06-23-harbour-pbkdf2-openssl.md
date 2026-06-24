# PBKDF2 en Harbour: hashing seguro de contraseñas con OpenSSL

Durante años, el esquema de autenticación de nuestra aplicación web en Harbour
funcionó de la manera más común: usuario y contraseña, donde la contraseña se
almacenaba como un hash SHA-256. En un sistema de uso interno parecía suficiente.

Pero conforme la aplicación fue creciendo — expuesta vía Cloudflare Tunnel, con
más usuarios, más módulos, más datos sensibles — empecé a incomodarme con esa
implementación. El problema no es SHA-256 en sí; el problema es usarlo directamente
como función de hashing de contraseñas sin salt y sin iteraciones. SHA-256 fue
diseñado para ser rápido. Una GPU moderna puede calcular miles de millones de
hashes por segundo. Si alguien obtiene tu tabla de usuarios — por un backup mal
resguardado, por acceso indebido a la BD, por cualquier razón — las contraseñas
quedan expuestas en minutos con un simple ataque de diccionario.

Lo que se necesita para almacenar contraseñas de forma correcta es una función
deliberadamente lenta, con salt única por usuario y con un número alto de
iteraciones. Eso es exactamente **PBKDF2** (Password-Based Key Derivation Function 2).

El reto: `hbssl`, el contrib oficial de OpenSSL para Harbour, no expone
`PKCS5_PBKDF2_HMAC` ni `RAND_bytes`. La solución fue escribir un wrapper en C
que compile directamente con el proyecto vía `hbmk2`.

## Por qué dos funciones

Necesitamos exponer exactamente dos cosas:

**`HB_CRYPTO_RANDBYTES( nLen )`** — Genera bytes aleatorios usando `RAND_bytes()`
de OpenSSL, que es un CSPRNG (generador pseudoaleatorio criptográficamente seguro).
El `hb_Random()` nativo de Harbour es un PRNG determinístico — predecible si se
conoce la semilla, inaceptable para salt criptográfica. Cada usuario necesita una
salt única y verdaderamente aleatoria.

**`HB_CRYPTO_PBKDF2( cPass, cSalt, nIter, nKeyLen )`** — Aplica PBKDF2-HMAC-SHA256
con el número de iteraciones que configures. Con 100,000 iteraciones, una operación
que SHA-256 resuelve en microsegundos tarda ~100ms — haciendo los ataques de
diccionario impracticables incluso si alguien obtiene la tabla completa de usuarios.

## El wrapper: hb_crypto.c

El archivo `.c` se agrega directamente al `.hbp` de compilación — `hbmk2`
lo compila junto con el resto de los fuentes.

En mi caso utilicé OpenSSL 1.1.x.

```c
/*
 * hb_crypto.c
 * Wrapper OpenSSL para Harbour x64/MSVC64
 *
 * Expone dos funciones a Harbour:
 *   HB_CRYPTO_RANDBYTES( nLen )                        -> cBytes (raw)
 *   HB_CRYPTO_PBKDF2( cPass, cSalt, nIter, nKeyLen )  -> cHashHex
 */

#include "hbapi.h"
#include "hbapierr.h"
#include <openssl/rand.h>
#include <openssl/evp.h>
#include <openssl/sha.h>
#include <string.h>
#include <stdlib.h>

/* ----------------------------------------------------------------
   HB_CRYPTO_RANDBYTES( nLen ) --> cBytes
   Genera nLen bytes aleatorios usando CSPRNG de OpenSSL.
   Retorna string raw — usar hb_StrToHex() en Harbour para hex.
   ---------------------------------------------------------------- */
HB_FUNC( HB_CRYPTO_RANDBYTES )
{
   int nLen = hb_parni( 1 );

   if( nLen <= 0 || nLen > 1024 )
   {
      hb_errRT_BASE( EG_ARG, 3012, "nLen debe ser entre 1 y 1024",
                     HB_ERR_FUNCNAME, 0 );
      return;
   }

   unsigned char * pBuf = ( unsigned char * ) hb_xgrab( nLen );

   if( RAND_bytes( pBuf, nLen ) != 1 )
   {
      hb_xfree( pBuf );
      hb_errRT_BASE( EG_ARG, 3013, "RAND_bytes fallo",
                     HB_ERR_FUNCNAME, 0 );
      return;
   }

   hb_retclen( ( char * ) pBuf, nLen );
   hb_xfree( pBuf );
}

/* ----------------------------------------------------------------
   HB_CRYPTO_PBKDF2( cPass, cSalt, nIter, nKeyLen ) --> cHashHex
   PBKDF2-HMAC-SHA256.
   cPass    : contraseña en texto plano
   cSalt    : salt raw (bytes, resultado de HB_CRYPTO_RANDBYTES)
   nIter    : iteraciones — recomendado >= 100000
   nKeyLen  : longitud del hash en bytes (32 = 256 bits)
   Retorna string hexadecimal en minúsculas.
   ---------------------------------------------------------------- */
HB_FUNC( HB_CRYPTO_PBKDF2 )
{
   const char * cPass    = hb_parcx( 1 );
   HB_SIZE      nPassLen = hb_parclen( 1 );
   const char * cSalt    = hb_parcx( 2 );
   HB_SIZE      nSaltLen = hb_parclen( 2 );
   int          nIter    = hb_parni( 3 );
   int          nKeyLen  = hb_parni( 4 );

   if( nPassLen == 0 || nSaltLen == 0 )
   {
      hb_errRT_BASE( EG_ARG, 3014, "cPass y cSalt no pueden ser vacios",
                     HB_ERR_FUNCNAME, 0 );
      return;
   }
   if( nIter   <= 0 ) nIter   = 100000;
   if( nKeyLen <= 0 ) nKeyLen = 32;
   if( nKeyLen > 512 )
   {
      hb_errRT_BASE( EG_ARG, 3015, "nKeyLen maximo 512",
                     HB_ERR_FUNCNAME, 0 );
      return;
   }

   unsigned char * pKey = ( unsigned char * ) hb_xgrab( nKeyLen );

   int iResult = PKCS5_PBKDF2_HMAC(
      cPass,   ( int ) nPassLen,
      ( const unsigned char * ) cSalt, ( int ) nSaltLen,
      nIter,
      EVP_sha256(),
      nKeyLen,
      pKey
   );

   if( iResult != 1 )
   {
      hb_xfree( pKey );
      hb_errRT_BASE( EG_ARG, 3016, "PKCS5_PBKDF2_HMAC fallo",
                     HB_ERR_FUNCNAME, 0 );
      return;
   }

   char * pHex = ( char * ) hb_xgrab( nKeyLen * 2 + 1 );
   for( int i = 0; i < nKeyLen; i++ )
      snprintf( pHex + i * 2, 3, "%02x", pKey[ i ] );

   hb_retclen( pHex, nKeyLen * 2 );
   hb_xfree( pKey );
   hb_xfree( pHex );
}
```

## Integración en el .hbp

```
hb_crypto.c
-I"C:\OpenSSL64\include"
-llibcrypto-1_1-x64
```

La ruta al include y el nombre de la lib dependen de tu instalación.
Ajusta según tu entorno.

## Uso en Harbour

```harbour
EXTERNAL HB_CRYPTO_RANDBYTES
EXTERNAL HB_CRYPTO_PBKDF2

// Al registrar o cambiar una contraseña:
LOCAL cSaltRaw := HB_CRYPTO_RANDBYTES( 32 )
LOCAL cSaltHex := hb_StrToHex( cSaltRaw )
LOCAL cHash    := HB_CRYPTO_PBKDF2( cPass, cSaltRaw, 100000, 32 )

// Al verificar login — recuperar salt de BD y recalcular:
LOCAL cSaltRaw := hb_HexToStr( cSaltHex )
LOCAL cHash    := HB_CRYPTO_PBKDF2( cPass, cSaltRaw, 100000, 32 )
IF cHash == cHashBD
   // credenciales válidas
ENDIF
```

`HB_CRYPTO_RANDBYTES` devuelve bytes raw. Se convierte a hex con `hb_StrToHex()`
para guardar en BD como `VARCHAR(64)`, y se reconvierte con `hb_HexToStr()` al
verificar. El hash PBKDF2 ya sale en hex — 64 caracteres para una clave de 32 bytes.

## Prueba de validación

Antes de integrar al proyecto principal compilé un ejecutable independiente
para validar el comportamiento:

```
=== TEST HB_CRYPTO ===
Salt length (debe ser 32)     :         32
Salt hex   (debe ser 64 chars):         64
Salt hex                      : 24AE37E68FA8415F823ED41EF2FD20261FCFC4CAED52CD75EB4E4B0DEE43EEB1
Hash1                         : ec5a9ef1c7fe959f6cf8c3f1ceab8f893cb484837aa91d4f06b592db0809bd60
Hash2                         : ec5a9ef1c7fe959f6cf8c3f1ceab8f893cb484837aa91d4f06b592db0809bd60
Son iguales (debe ser .T.)    : .T.
Hash3 (salt diferente)        : 689c10ad8dca1320ce5d363e520322129a07355308d5a5c6c6d50ca648ca8578
Hash1 != Hash3 (debe ser .T.) : .T.
Hash4 (pass diferente)        : fbe7180bccdcefbeaaa5b928c7df9a73b084069bdf35034aa076d981d14f242d
Hash1 != Hash4 (debe ser .T.) : .T.
=== FIN TEST ===
```

Mismo salt + misma contraseña = mismo hash. Salt diferente = hash diferente.
Contraseña diferente = hash diferente.

## Conclusión

PBKDF2-HMAC-SHA256 con salt única por usuario es una implementación legítima
y sólida para el almacenamiento de contraseñas — la misma que usan sistemas
de producción en todo el mundo. No es una solución de segunda categoría respecto
a bcrypt o Argon2; la diferencia es que estos últimos incorporan parámetros
de memoria adicionales que los hacen más resistentes a hardware muy especializado,
pero PBKDF2 sigue siendo una elección correcta y ampliamente adoptada.

En Harbour, donde el ecosistema de librerías criptográficas expuestas es limitado,
este wrapper de ~80 líneas de C resuelve el problema de raíz: salt criptográfica
real con `RAND_bytes` y hashing lento con PBKDF2.

Espero que le sea útil a alguien más en la comunidad.
