<!-- omit in toc -->
# BOLT #8: Transporte cifrado y autenticado

Todas las comunicaciones entre los nodos Lightning se cifran para brindar confidencialidad a todas las transcripciones entre los nodos y se autentican para evitar interferencias maliciosas. Cada nodo tiene un identificador conocido a largo plazo que es una clave pública en la curva `secp256k1` de Bitcoin. Esta clave pública a largo plazo se usa dentro del protocolo para establecer una conexión encriptada y autenticada con pares, y también para autenticar cualquier información anunciada en nombre de un nodo.

<!-- omit in toc -->
# Índice
- [Cryptographic Messaging Overview](#cryptographic-messaging-overview)
  - [Authenticated Key Agreement Handshake](#authenticated-key-agreement-handshake)
  - [Handshake Versioning](#handshake-versioning)
  - [Noise Protocol Instantiation](#noise-protocol-instantiation)
- [Authenticated Key Exchange Handshake Specification](#authenticated-key-exchange-handshake-specification)
  - [Handshake State](#handshake-state)
  - [Handshake State Initialization](#handshake-state-initialization)
  - [Handshake Exchange](#handshake-exchange)
- [Lightning Message Specification](#lightning-message-specification)
  - [Encrypting and Sending Messages](#encrypting-and-sending-messages)
  - [Receiving and Decrypting Messages](#receiving-and-decrypting-messages)
- [Lightning Message Key Rotation](#lightning-message-key-rotation)
- [Security Considerations](#security-considerations)
- [Appendix A: Transport Test Vectors](#appendix-a-transport-test-vectors)
  - [Initiator Tests](#initiator-tests)
  - [Responder Tests](#responder-tests)
  - [Message Encryption Tests](#message-encryption-tests)
- [Acknowledgments](#acknowledgments)
- [References](#references)
- [Authors](#authors)

## Descripción general de la mensajería criptográfica

Antes de enviar cualquier mensaje Lightning, los nodos DEBEN iniciar primero el estado de sesión criptográfica que se utiliza para cifrar y autenticar todos los mensajes enviados entre nodos. La inicialización de este estado de sesión criptográfica es completamente distinta de cualquier encabezado o convención de mensaje de protocolo interno.

La transcripción entre dos nodos se separa en dos segmentos distintos:

1. Antes de cualquier transferencia de datos real, ambos nodos participan en un protocolo de enlace de acuerdo de clave autenticado, que se basa en el marco de protocolo de ruido<sup>[2](#reference-2)</sup>.
2. Si el protocolo de enlace inicial es exitoso, los nodos ingresan a la fase de intercambio de mensajes Lightning. En la fase de intercambio de mensajes Lightning, todos los mensajes son textos cifrados de cifrado autenticado con datos asociados (AEAD).

### `handshake` de acuerdo de clave autenticado

El `handshake` elegido para el intercambio de claves autenticado es `Noise_XK`. Como mensaje previo, el iniciador debe conocer la clave pública de identidad del respondedor. Esto proporciona un grado de ocultación de identidad para el respondedor, ya que su clave pública estática _nunca_ se transmite durante el `handshake`. En cambio, la autenticación se logra implícitamente a través de una serie de operaciones Elliptic-Curve Diffie-Hellman (ECDH) seguidas de una verificación de MAC.

El acuerdo de clave autenticado (`Noise_XK`) se realiza en tres pasos distintos (actos). Durante cada acto del `handshake` ocurre lo siguiente: algún material de clave (posiblemente cifrado) se envía a la otra parte; se realiza un ECDH, basado exactamente en qué acto se está ejecutando, y el resultado se mezcla con el conjunto actual de claves de cifrado (`ck` la clave de encadenamiento y `k` la clave de cifrado); y se envía una carga útil o `payload` AEAD con un texto cifrado de longitud cero. Como esta carga útil no tiene longitud, solo se envía un MAC. La mezcla de salidas de ECDH en un resumen de hash forma un `handshake` TripleDH incremental.

Usando el lenguaje del `Noise Protocol`, `e` y `s` (ambas claves públicas, siendo `e` la clave efímera y `s` la clave estática, que en nuestro caso suele ser `nodeid`) indican una clave posiblemente encriptada material, y `es`, `ee` y `se` indican cada uno una operación ECDH entre dos claves. El `handshake` se presenta de la siguiente manera:
```
    Noise_XK(s, rs):
       <- s
       ...
       -> e, es
       <- e, ee
       -> s, se
```
Todos los datos de protocolo de enlace enviados a través del cable, incluido el material de clave, se procesan gradualmente en un 'resumen de protocolo de enlace' de toda la sesión, `h`. Tenga en cuenta que el estado de protocolo de enlace `h` nunca se transmite durante el protocolo de enlace; en su lugar, se utiliza el resumen como datos asociados dentro de los mensajes AEAD de longitud cero.

La autenticación de cada mensaje enviado garantiza que un hombre en el medio (`man-in-the-middle` MITM) no haya modificado ni reemplazado ninguno de los datos enviados como parte de un protocolo de enlace, ya que MACcheck fallaría en el otro lado si fuera así.
 
Una verificación exitosa de MAC por parte del receptor indica implícitamente que toda la autenticación ha sido exitosa hasta ese momento. Si alguna vez falla una verificación de MAC durante el proceso de negociación, la conexión se terminará de inmediato.

### Versión de protocolo de enlace

Cada mensaje enviado durante el protocolo de enlace inicial comienza con un solo byte inicial, que indica la versión utilizada para el protocolo de enlace actual. Una versión de 0 indica que no es necesario ningún cambio, mientras que una versión distinta de cero indica que el cliente se ha desviado del protocolo especificado originalmente en este documento.

Los clientes DEBEN rechazar los intentos de protocolo de enlace iniciados con una versión desconocida.

### Creación de instancias de protocolo de ruido

Las instancias concretas del protocolo de ruido requieren la definición de tres objetos criptográficos abstractos: la función hash, la curva elíptica y el esquema de cifrado AEAD. Para Lightning, se elige `SHA-256` como función hash, `secp256k1` como curva elíptica y `ChaChaPoly-1305` como construcción AEAD.

La composición de `ChaCha20` y `Poly1305` que se utilizan DEBE cumplir con `RFC 8439`<sup>[1](#reference-1)</sup>.

El nombre de protocolo oficial para la variante Lightning de Noise es `Noise_XK_secp256k1_ChaChaPoly_SHA256`. La representación de cadena ASCII de este valor se convierte en un resumen que se utiliza para inicializar el estado inicial del protocolo de enlace. Si los nombres de protocolo de dos puntos finales difieren, el proceso de reconocimiento falla inmediatamente.

## Especificación de reconocimiento de intercambio de claves autenticado

El apretón de manos procede en tres actos, tomando 1.5 viajes de ida y vuelta. Cada protocolo de enlace es una carga útil de tamaño _fijo_ sin ningún encabezado ni metadatos adicionales adjuntos. El tamaño exacto de cada acto es el siguiente:

   * **Primer Acto**: 50 bytes
   * **Sundo Acto**: 50 bytes
   * **Tercer Acto**: 66 bytes

### Estado del `Handshake'

A lo largo del proceso de apretón de manos, cada lado mantiene estas variables:

 * `ck`: la **clave de encadenamiento**. Este valor es el hash acumulado de todas las salidas ECDH anteriores. Al final del protocolo de enlace, se usa `ck` para derivar las claves de cifrado para los mensajes Lightning.

 * `h`: el **hash del `handshake`**. Este valor es el hash acumulado de _todos_ los datos de protocolo de enlace que se han enviado y recibido hasta el momento durante el proceso de protocolo de enlace.

 * `temp_k1`, `temp_k2`, `temp_k3`: las **claves intermedias**. Estos se utilizan para cifrar y descifrar las cargas útiles de AEAD de longitud cero al final de cada mensaje de protocolo de enlace.

 * `e`: el **par de claves efímeras** de una parte. Para cada sesión, un nodo DEBE generar una nueva clave efímera con una fuerte aleatoriedad criptográfica.

 * `s`: el **par de claves estáticas** de una parte (`ls` para local, `rs` para remoto).

También se hará referencia a las siguientes funciones:

  * `ECDH(k, rk)`: realiza una operación Diffie-Hellman de curva elíptica usando `k`, que es una clave privada `secp256k1` válida, y `rk`, que es una clave pública válida
      * El valor devuelto es el SHA256 del formato comprimido del punto generado.

  * `HKDF(salt,ikm)`: una función definida en `RFC 5869`<sup>[3](#reference-3)</sup>, evaluada con un campo `info` de longitud cero
     * Todas las invocaciones de `HKDF` devuelven implícitamente 64 bytes de aleatoriedad criptográfica utilizando el componente de extracción y expansión de `HKDF`.

  * `encryptWithAD(k, n, ad, plaintext)`: genera `encrypt(k, n, ad, plaintext)`
     * Donde `encrypt` es una evaluación de `ChaCha20-Poly1305` (variante IETF) con los argumentos pasados, con nonce `n` codificado como 32 bits a cero, seguido de un valor *little-endian* de 64 bits. Nota: esto sigue la convención del `Noise Protocol`, en lugar de nuestro endian normal.

  * `decryptWithAD(k, n, ad, ciphertext)`: genera `decrypt(k, n, ad, ciphertext)`
     * Donde `decrypt` es una evaluación de `ChaCha20-Poly1305` (variante IETF) con los argumentos pasados, con nonce `n` codificado como 32 bits cero, seguido de un valor *little-endian* de 64 bits.

  * `generateKey()`: genera y devuelve un nuevo par de claves `secp256k1`
     * Donde el objeto devuelto por `generateKey` tiene dos atributos:
         * `.pub`, que devuelve un objeto abstracto que representa la clave pública
         * `.priv`, que representa la clave privada utilizada para generar la clave pública
     * Donde el objeto también tiene un solo método:
         * `.serializeCompressed()`

  * `a || b` denota la concatenación de dos cadenas de bytes `a` y `b`

### Inicialización del estado del `Handshake`

Antes del comienzo del Primer Acto, ambos lados inicializan su estado por sesión de la siguiente manera:

 1. `h = SHA-256 (protocolName)`
    * donde `protocolName = 'Noise_XK_secp256k1_ChaChaPoly_SHA256'` codificado como una cadena ASCII

 2. `ck = h`

 3. `h = SHA-256(h || prologue)`
    * donde `prologue` es la cadena ASCII: `lightning`

Como paso final, ambas partes mezclan la clave pública del respondedor en el resumen del apretón de manos:

 * El nodo iniciador mezcla la clave pública estática del nodo que responde serializada en el formato comprimido de Bitcoin:
   * `h = SHA-256(h || rs.pub.serializeCompressed())`

 * El nodo que responde se mezcla en su clave pública estática local serializada en el formato comprimido de Bitcoin:
   * `h = SHA-256(h || ls.pub.serializeCompressed())`

### Intercambio del `Handshake`

<!-- omit in toc -->
#### Primer Acto

```
    -> e, es
```

El Primer Acto se envía del iniciador al respondedor. Durante el Primer Acto, el iniciador intenta satisfacer un desafío implícito del respondedor. Para completar este desafío, el iniciador debe conocer la clave pública estática del respondedor.

El mensaje de reconocimiento tiene _exactamente_ 50 bytes: 1 byte para la versión de reconocimiento, 33 bytes para la clave pública efímera comprimida del iniciador y 16 bytes para la etiqueta `poly1305`.

**Acciones de Quien Envía:**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * La clave efímera recién generada se acumula en el resumen de protocolo de enlace en ejecución.
3. `es = ECDH(e.priv, rs)`
     * El iniciador realiza un ECDH entre su clave efímera recién generada y la clave pública estática del nodo remoto.
4. `ck, temp_k1 = HKDF(ck, es)`
     * Se genera una nueva clave de cifrado temporal, que se utiliza para generar la MAC de autenticación.
5. `c = encryptWithAD(temp_k1, 0, h, zero)`
     * Donde `zero` es un texto plano de longitud cero.
6. `h = SHA-256(h || c)`
     * Finalmente, el texto cifrado generado se acumula en el resumen del protocolo de enlace de autenticación.
7. Envía `m = 0 || e.pub.serializeCompressed() || c` al respondedor a través del búfer de la red.

**Acciones del Receptor:**

1. Lea _exactamente_ 50 bytes del búfer de red.
2. Analice el mensaje leído (`m`) en `v`, `re` y `c`:
    * donde `v` es el _primer_ byte de `m`, `re` son los siguientes 33 bytes de `m` y `c` son los últimos 16 bytes de `m`.
    * Los bytes sin procesar de la clave pública efímera (`re`) de la parte remota deben deserializarse en un punto de la curva usando coordenadas afines codificadas por el formato compuesto serializado de la clave.
3. Si `v` es una versión de `handshake` no reconocida, entonces el respondedor DEBE abortar el intento de conexión.
4. `h = SHA-256(h || re.serializeCompressed())`
    * El respondedor acumula la clave efímera del iniciador en el resumen del `handshake` de autenticación.
5. `es = ECDH(s.priv, re)`
    * El respondedor realiza un ECDH entre su clave privada estática y la clave pública efímera del iniciador.
6. `ck, temp_k1 = HKDF(ck, es)`
    * Se genera una nueva clave de cifrado temporal, que en breve se utilizará para verificar la MAC de autenticación.
7. `p = descifrar con AD (temp_k1, 0, h, c)`
    * Si la verificación de MAC en esta operación falla, entonces el iniciador _no_ conoce la clave pública estática del respondedor. Si este es el caso, entonces el respondedor DEBE terminar la conexión sin más mensajes.
8. `h = SHA-256(h || c)`
     * El texto cifrado recibido se mezcla en el resumen del `handshake`. Este paso sirve para garantizar que un MITM no haya modificado la carga útil.

<!-- omit in toc -->
#### Segundo Acto

```
   <- e, ee
```

El Segundo Acto se envía desde el respondedor al iniciador. El Segundo Acto _solo_ tendrá lugar si el Primer Acto fue exitoso. El Primer Acto tuvo éxito si el respondedor pudo descifrar correctamente y verificar el MAC de la etiqueta enviada al final del primer acto.

El `handshake` es _exactamente_ 50 bytes: 1 byte para la versión de `handshake`, 33 bytes para la clave pública efímera comprimida del respondedor y 16 bytes para la etiqueta `poly1305`.

**Acciones de Quien Envía:**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * The newly generated ephemeral key is accumulated into the running handshake digest.
3. `ee = ECDH(e.priv, re)`
     * where `re` is the ephemeral key of the initiator, which was received during Act One
4. `ck, temp_k2 = HKDF(ck, ee)`
     * A new temporary encryption key is generated, which is used to generate the authenticating MAC.
5. `c = encryptWithAD(temp_k2, 0, h, zero)`
     * where `zero` is a zero-length plaintext
6. `h = SHA-256(h || c)`
     * Finally, the generated ciphertext is accumulated into the authenticating handshake digest.
7. Send `m = 0 || e.pub.serializeCompressed() || c` to the initiator over the network buffer.

**Acciones del Receptor:**

1. Lee _exactamente_ 50 bytes del búfer de red.
2. Analiza el mensaje leído (`m`) en `v`, `re` y `c`:
    * donde `v` es el _primer_ byte de `m`, `re` son los siguientes 33 bytes de `m` y `c` son los últimos 16 bytes de `m`.
3. Si `v` es una versión de `handshake` no reconocida, entonces quien responde DEBE abortar el intento de conexión.
4. `h = SHA-256(h || re.serializeCompressed())`
5. `ee = ECDH(e.priv, re)`
    * donde `re` es la clave pública efímera de quien responde
    * Los bytes sin procesar de la clave pública efímera (`re`) de la parte remota deben deserializarse en un punto de la curva usando coordenadas afines codificadas por el formato compuesto serializado de la clave.
6. `ck, temp_k2 = HKDF(ck, ee)`
     * Se genera una nueva clave de cifrado temporal, que se utiliza para generar la MAC de autenticación.
7. `p = descifrar con AD (temp_k2, 0, h, c)`
    * Si la verificación de MAC en esta operación falla, entonces el iniciador DEBE terminar la conexión sin más mensajes.
8. `h = SHA-256(h || c)`
     * El texto cifrado recibido se mezcla en el resumen del `handshake`. Este paso sirve para garantizar que un MITM no haya modificado la carga útil.

<!-- omit in toc -->
#### Tercer Acto

```
   -> s, se
```

El Tercer Acto es la fase final del acuerdo de clave autenticada descrito en esta sección. Este acto se envía desde el iniciador al respondedor como un paso final. El tercer acto se ejecuta _si y sólo si_ el segundo acto tuvo éxito.
Durante el tercer acto, el iniciador transporta su clave pública estática al respondedor encriptada con _strong_ forward secrecy, utilizando la clave secreta acumulada derivada de 'HKDF' en este punto del `handshake`.

El protocolo de enlace es _exactamente_ 66 bytes: 1 byte para la versión del `handshake`, 33 bytes para la clave pública estática cifrada con el cifrado de flujo `ChaCha20`, 16 bytes para la etiqueta de la clave pública cifrada generada a través de la construcción AEAD y 16 bytes para una clave final etiqueta de autenticación.

**Acciones de Quien Envía:**

1. `c = encryptWithAD(temp_k2, 1, h, s.pub.serializeCompressed())`
    * donde `s` es la clave pública estática del iniciador
2. `h = SHA-256(h || c)`
3. `se = ECDH(s.priv, re)`
    * donde `re` es la clave pública efímera de quien responde
4. `ck, temp_k3 = HKDF(ck, se)`
    * El secreto compartido intermedio final se mezcla con la clave de encadenamiento en ejecución.
5. `t = cifrarConAD(temp_k3, 0, h, zero)`
     * donde `zero` es un texto sin formato de longitud cero
6. `sk, rk = HKDF(ck, zero)`
     * donde 'zero' es un texto sin formato de longitud cero, 'sk' es la clave que utilizará el iniciador para cifrar los mensajes para quien responde, y 'rk` es la clave que utilizará el iniciador para descifrar los mensajes enviados por quien responde
     * Se generan las claves de cifrado finales, que se utilizarán para enviar y recibir mensajes durante la duración de la sesión.
7. `rn = 0, sn = 0`
     * Los nonces de envío y recepción se inicializan a 0.
8. Enviar `m = 0 || do || t` sobre el búfer de la red.


**Acciones del Receptor:**

1. Lea _exactamente_ 66 bytes del búfer de red.
2. Analice el mensaje leído (`m`) en `v`, `c` y `t`:
    * donde `v` es el _primer_ byte de `m`, `c` son los siguientes 49 bytes de `m` y `t` son los últimos 16 bytes de `m`
3. Si `v` es una versión de `handshake` no reconocida, entonces quien responde DEBE abortar el intento de conexión.
4. `rs = decryptWithAD(temp_k2, 1, h, c)`
     * En este punto, quien responde ha recuperado la clave pública estática del iniciador.
     * Si la verificación de MAC en esta operación falla, entonces quien responde DEBE terminar la conexión sin más mensajes.
5. `h = SHA-256(h || c)`
6. `se = ECDH(e.priv, rs)`
     * donde `e` es la clave efímera original de quien responde
7. `ck, temp_k3 = HKDF(ck, se)`
8. `p = descifrar con AD (temp_k3, 0, h, t)`
     * Si la verificación de MAC en esta operación falla, entonces quien responde DEBE terminar la conexión sin más mensajes.
9. `rk, sk = HKDF(ck, cero)`
     * donde `cero` es un texto sin formato de longitud cero,
       `rk` es la clave que utilizará quien responde para descifrar los mensajes enviados por el iniciador,
       y `sk` es la clave que utilizará quien responde para cifrar mensajes para el iniciador
     * Se generan las claves de cifrado finales, que se utilizarán para enviar y recibir mensajes durante la duración de la sesión.
10. `rn = 0, sn = 0`
     * Los nonces de envío y recepción se inicializan a 0.

## Especificación de mensaje Lightning

Al finalizar el Tercer Acto, ambas partes obtuvieron las claves de cifrado, que se utilizarán para cifrar y descifrar mensajes durante el resto de la sesión.

Los mensajes reales del protocolo Lightning están encapsulados dentro de los textos cifrados de AEAD.
Cada mensaje tiene el prefijo de otro texto cifrado AEAD, que codifica la longitud total del siguiente mensaje Lightning (sin incluir su MAC).

El tamaño *máximo* de _cualquier_ mensaje Lightning NO DEBE exceder `65535` bytes. Un tamaño máximo de `65535` simplifica las pruebas, facilita la gestión de la memoria y ayuda a mitigar los ataques de agotamiento de la memoria (`memory-exhaustion attacks`).

Para dificultar el análisis del tráfico, el prefijo de longitud de todos los mensajes Lightning cifrados también está encriptado. Además, se agrega una etiqueta `Poly-1305` de 16 bytes al prefijo de longitud cifrada para garantizar que la longitud del paquete no se haya modificado durante el vuelo y también para evitar la creación de un oráculo de descifrado.

La estructura de los paquetes en el cable se parece a la siguiente:

```
+---------------------------------------
|Longitud del mensaje cifrado de 2 bytes|
+---------------------------------------
|     MAC de 16 bytes de longitud       |
|         del mensaje cifrado           |
+---------------------------------------
|                                       |
|                                       |
|           Mensaje Lightning           |
|               cifrado                 |
|                                       |
+---------------------------------------
|         MAC de 16 bytes del           |
|          mensaje Lightning            |
+---------------------------------------
```

La longitud del mensaje prefijado se codifica como un entero big-endian de 2 bytes, para una longitud máxima total del paquete de `2 + 16 + 65535 + 16` = `65569` bytes.

### Cifrado y envío de mensajes

Para cifrar y enviar un mensaje Lightning (`m`) al flujo de red, dada una clave de envío (`sk`) y un nonce (`sn`), se completan los siguientes pasos:

1. Sea `l = len(m)`.
    * donde `len` obtiene la longitud en bytes del mensaje Lightning
2. Serialice `l` en 2 bytes codificados como un entero big-endian.
3. Cifre `l` (usando `ChaChaPoly-1305`, `sn` y `sk`), para obtener `lc`
    (18 bytes)
    * El nonce `sn` está codificado como un número little-endian de 96 bits. Como el nonce decodificado es de 64 bits, el nonce de 96 bits se codifica como: 32 bits de 0 iniciales seguidos de un valor de 64 bits.
        * El nonce `sn` DEBE incrementarse después de este paso.
    * Un segmento de bytes de longitud cero debe pasarse como AD (datos asociados).
4. Finalmente, cifre el mensaje en sí (`m`) utilizando el mismo procedimiento que usó para cifrar el prefijo de longitud. Que el texto cifrado encriptado se conozca como `c`.
    * El nonce `sn` DEBE incrementarse después de este paso.
5. Enviar `lc || c` sobre el búfer de red.

### Recibir y descifrar mensajes

Para descifrar el _siguiente_ mensaje en el flujo de red, se completan los siguientes pasos:

1. Lea _exactamente_ 18 bytes del búfer de red.
2. Deje que el prefijo de longitud cifrada se conozca como `lc`.
3. Descifre `lc` (utilizando `ChaCha20-Poly1305`, `rn` y `rk`) para obtener el tamaño del paquete cifrado `l`.
    * Un segmento de bytes de longitud cero debe pasarse como AD (datos asociados).
    * El nonce `rn` DEBE incrementarse después de este paso.
4. Lea _exactamente_ `l+16` bytes del búfer de red y deje que los bytes se conozcan como `c`.
5. Descifrar `c` (utilizando `ChaCha20-Poly1305`, `rn` y `rk`) para obtener el paquete de texto sin formato `p` descifrado.
    * El nonce `rn` DEBE incrementarse después de este paso.

## Rotación de clave de mensaje Lightning

Cambiar las claves regularmente y olvidar las claves anteriores es útil para evitar el descifrado de mensajes antiguos, en el caso de una fuga de claves posterior (es decir, secreto hacia atrás).

La rotación de claves se realiza para _cada_ clave (`sk` y `rk`) _individualmente_. Una clave debe rotarse después de que una parte encripte o desencripte 1000 veces con ella (es decir, cada 500 mensajes).
Esto se puede contabilizar adecuadamente rotando la clave una vez que el nonce dedicado a ella exceda 1000.

La rotación de claves para una clave `k` se realiza de acuerdo con los siguientes pasos:

1. Sea `ck` la clave de encadenamiento obtenida al final del tercer acto.
2. `ck', k' = HKDF(ck, k)`
3. Restablezca el nonce para la clave a `n = 0`.
4. 'k = k'`
5. 'ck = ck'`

# Consideraciones de Seguridad

Se recomienda enfáticamente que las bibliotecas validadas, de uso común y existentes se utilicen para el cifrado y el descifrado, a fin de evitar los muchos posibles errores de implementación.

# Apéndice A: Vectores de prueba de transporte

Para hacer un `handshake` de prueba repetible, lo siguiente especifica qué devolverá `generateKey()` (es decir, el valor de `e.priv`) para cada lado. Tenga en cuenta que esto es una violación de la especificación, que requiere aleatoriedad.

## Pruebas del iniciador

El iniciador DEBERÍA producir la salida dada cuando se alimenta esta entrada.
Los comentarios reflejan estados internos, con fines de depuración.

```
    name: transport-initiator successful handshake
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    # Act One
    # h=0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    # ss=0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    # HKDF(0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3)
    # ck,temp_k1=0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    # encryptWithAD(0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f, 0x000000000000000000000000, 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c, <empty>)
    # c=0df6086551151f58b8afe6c195782c6a
    # h=0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # re=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # h=0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    # ss=0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    # HKDF(0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47)
    # ck,temp_k2=0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000000, 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf, 0x6e2470b93aac583c9ef6eafca3f730ae)
    # h=0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    # Act Three
    # encryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000100000000000000, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa)
    # c=0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822
    # h=0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    # ss=0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    # HKDF(0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766)
    # ck,temp_k3=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    # encryptWithAD(0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520, 0x000000000000000000000000, 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660, <empty>)
    # t=0x8dc68b1c466263b47fdf31e560e139ba
    output: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    # HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,zero)
    output: sk,rk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name: transport-initiator act2 short read test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730
    output: ERROR (ACT2_READ_FAILED)

    name: transport-initiator act2 bad version test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0102466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    output: ERROR (ACT2_BAD_VERSION 1)

    name: transport-initiator act2 bad key serialization test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0004466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    output: ERROR (ACT2_BAD_PUBKEY)

    name: transport-initiator act2 bad MAC test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730af
    output: ERROR (ACT2_BAD_TAG)
```

## Pruebas de respuesta

El respondedor DEBE producir la salida dada cuando recibe esta entrada.

```
    name: transport-responder successful handshake
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # re=0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    # h=0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    # ss=0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    # HKDF(0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3)
    # ck,temp_k1=0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    # decryptWithAD(0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f, 0x000000000000000000000000, 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c, 0x0df6086551151f58b8afe6c195782c6a)
    # h=0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    # Act Two
    # e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27 e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    # h=0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    # ss=0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    # HKDF(0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47)
    # ck,temp_k2=0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    # encryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000000, 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf, <empty>)
    # c=0x6e2470b93aac583c9ef6eafca3f730ae
    # h=0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000100000000000000, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822)
    # rs=0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    # h=0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    # ss=0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    # HKDF(0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766)
    # ck,temp_k3=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    # decryptWithAD(0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520, 0x000000000000000000000000, 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660, 0x8dc68b1c466263b47fdf31e560e139ba)
    # HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,zero)
    output: rk,sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name: transport-responder act1 short read test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c
    output: ERROR (ACT1_READ_FAILED)

    name: transport-responder act1 bad version test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x01036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    output: ERROR (ACT1_BAD_VERSION)

    name: transport-responder act1 bad key serialization test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00046360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    output: ERROR (ACT1_BAD_PUBKEY)

    name: transport-responder act1 bad MAC test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6b
    output: ERROR (ACT1_BAD_TAG)

    name: transport-responder act3 bad version test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x01b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    output: ERROR (ACT3_BAD_VERSION 1)

    name: transport-responder act3 short read test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139
    output: ERROR (ACT3_READ_FAILED)

    name: transport-responder act3 bad MAC for ciphertext test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00c9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    output: ERROR (ACT3_BAD_CIPHERTEXT)

    name: transport-responder act3 bad rs test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00bfe3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa2235536ad09a8ee351870c2bb7f78b754a26c6cef79a98d25139c856d7efd252c2ae73c
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000001, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0xd7fedc211450dd9602b41081c9bd05328b8bf8c0238880f7b7cb8a34bb6d8354081e8d4b81887fae47a74fe8aab3008653)
    # rs=0x044f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    output: ERROR (ACT3_BAD_PUBKEY)

    name: transport-responder act3 bad MAC test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139bb
    output: ERROR (ACT3_BAD_TAG)
```

## Pruebas de cifrado de mensajes

En esta prueba, el iniciador envía mensajes de longitud 5 que contienen "hola" 1001 veces. Solo se muestran seis salidas de ejemplo, por brevedad y para probar dos rotaciones clave:

	name: transport-message test
    ck=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01
	sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9
	rk=0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442
    # encrypt l: cleartext=0x0005, AD=NULL, sn=0x000000000000000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802
    # encrypt m: cleartext=0x68656c6c6f, AD=NULL, sn=0x000000000100000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
	output 0: 0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
    # encrypt l: cleartext=0x0005, AD=NULL, sn=0x000000000200000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x72887022101f0b6753e0c7de21657d35a4cb
    # encrypt m: cleartext=0x68656c6c6f, AD=NULL, sn=0x000000000300000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
	output 1: 0x72887022101f0b6753e0c7de21657d35a4cb2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
    # 0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 = HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01, 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9)
    # 0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 = HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01, 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9)
    output 500: 0x178cb9d7387190fa34db9c2d50027d21793c9bc2d40b1e14dcf30ebeeeb220f48364f7a4c68bf8
    output 501: 0x1b186c57d44eb6de4c057c49940d79bb838a145cb528d6e8fd26dbe50a60ca2c104b56b60e45bd
    # 0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06, 0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd = HKDF(0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8)
    # 0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06, 0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd = HKDF(0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8)
    output 1000: 0x4a2f3cc3b5e78ddb83dcb426d9863d9d9a723b0337c89dd0b005d89f8d3c05c52b76b29b740f09
    output 1001: 0x2ecd8c8a5629d0d02ab457a0fdd0f7b90a192cd46be5ecb6ca570bfc5e268338b1a16cf4ef2d36

# Agradecimientos

TODO(roasbeef); fin

# Referencias
1. <a id="reference-1">https://tools.ietf.org/html/rfc8439</a>
2. <a id="reference-2">http://noiseprotocol.org/noise.html</a>
3. <a id="reference-3">https://tools.ietf.org/html/rfc5869</a>

# Autores

FIXME

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
