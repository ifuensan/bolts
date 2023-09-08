<!-- omit in toc -->
# BOLT #1: Protocolo Base

<!-- omit in toc -->
## Visión General

Este protocolo asume un mecanismo subyacente de transporte autenticado y ordenado que se encarga de enmarcar mensajes individuales. 
[BOLT #8](08-transport.md) especifica la capa de transporte canónico utilizada en Lightning, aunque puede ser reemplazado por cualquier transporte que cumpla con las garantías anteriores.

El puerto TCP predeterminado depende de la red utilizada. Las redes más comunes son:

* Bitcoin `mainet` con número de puerto 9735 o el correspondiente
* Bitcoin `testnet` con el número de puerto 19735 (`0x4D17`)
* Bitcoin `signet` con el número de puerto 39735 (`0xF87`).

El punto Unicode para LIGHTNING <sup>[1](#reference-1)</sup>, y el puerto convenido trantan de seguir la convención de Bitcoin Core.

Todos los campos de datos son `big-endian` sin signo a menos que se especifique lo contrario.
<!-- omit in toc -->
## Índice
- [Manejo y multiplexado de conexiones](#manejo-y-multiplexado-de-conexiones)
- [Formato del Mensaje Lightning](#formato-del-mensaje-lightning)
- [Formato Tipo-Longitud-Valor](#formato-tipo-longitud-valor)
- [Tipos Fundamentales](#tipos-fundamentales)
- [Mensajes de configuración](#mensajes-de-configuración)
  - [El Mensaje `init`](#el-mensaje-init)
  - [El Mensaje de `error` y `warning`](#el-mensaje-de-error-y-warning)
- [Mensajes de control](#mensajes-de-control)
  - [Los mensajes `ping` y `pong`](#los-mensajes-ping-y-pong)
- [Apéndice A: Vectores de Prueba BigSize](#apéndice-a-vectores-de-prueba-bigsize)
- [Apéndice B: Vectores de Test Tipo-Longitud-Valor](#apéndice-b-vectores-de-test-tipo-longitud-valor)
- [Apéndice C: Extensión de Mensaje](#apéndice-c-extensión-de-mensaje)
- [Agradecimientos](#agradecimientos)
- [Referencias](#referencias)
- [Autores](#autores)


## Manejo y multiplexado de conexiones
Las implementaciones DEBEN usar una sola conexión por par; los mensajes de canal (que incluyen un ID de canal) se multiplexan a través de esta única conexión.

## Formato del Mensaje Lightning
Después del descifrado, todos los mensajes Lightning tienen el formato:

1. `type`: campo `big-endian` 2-byte que hace referencia la tipo de mensaje
2. `payload`: una carga útil (`payload`) de longitud variable que comprende el resto del mensaje y que se ajusta a un formato que coincide con el `tipo`
3. `extension`: an optional [TLV stream](#formato-tipo-longitud-valor)

El campo `type` indica como intérpretar el campo `payload`
El formato para cada tipo individual está definido por una especificación en este repositorio.
El tipo sigue la regla _está bien ser impar_, por lo que los nodos PUEDEN enviar tipos _impares_ sin asegurarse de que el destinatario los entienda.

Los mensajes se agrupan de forma lógica en cinco grupos, ordenados por el bit más significativo que se establece:

  - Configuración & Control (tipos `0`-`31`): mensajes relacionados con la configuración de la conexión, el control, las funciones admitidas y los informes de errores (descritos a continuación)
  - Canal (tipos `32`-`127`): mensajes utilizados para configurar y echar abajo canales de micropagos (descrito en [BOLT #2](02-peer-protocol.md))
  - Compromiso (tipos `128`-`255`): mensajes relacionados con la actualización de la transacción de compromiso actual, que incluye agregar, revocar y liquidar HTLC, así como actualizar tarifas e intercambiar firmas (descrito en [BOLT #2](02-peer-protocol.md))
  - Enrutamiento (types `256`-`511`): mensajes que contienen anuncios de nodos y canales, así como cualquier exploración de ruta activa (descrito en [BOLT #7](07-routing-gossip.md))
  - Personalización (tipos `32768`-`65535`): mensajes experimentales y específicos de la aplicación

La capa de transporte requiere que el tamaño del mensaje se ajuste a un int (entero) sin firmar de 2 bytes; por lo tanto, el tamaño máximo posible es 65535 bytes.

Un nodo emisor:

  - NO DEBE enviar un mensaje tipificado como par que no se encuentre aquí, sin negociación previa.
  - NO DEBE enviar registros TLV tipificados pares en la "extensión" sin negociación previa.
  - que negocia una opción en esta espeficicación
    - DEBE incluir todos los campos anotados con esa opción.
  - Al definir mensajes personalizados:
    - DEBERÍA elegir un `tipo` aleatorio para evitar la colisión con otros tipos personalizados.
    - DEBERÍA elegir un `tipo` que no entre en conflicto con otros experimentos enumerados en [esta `issue`](https://github.com/lightningnetwork/lightning-rfc/issues/716).
    - DEBERÍA elegir identificadores de `tipo` impar cuando los nodos regulares debieran ignorar los datos adicionales.
    - DEBERÍA elegir identificadores de 'tipo' par cuando los nodos regulares debieran rechazar el mensaje y cerrar la conexión.

Un nodo receptor:

  - al recibir un mensaje _impar_, tipo desconocido:
    - DEBE ignorar el mensaje recibido.
  - al recibir un mensaje _par_, tipo desconocido:
    - DEBE cerrar la conexión.
    - PUEDEN fallar los canales.
  - al recibir un mensaje conocido con longitud insuficiente para el contenido:
    - DEBE cerrar la conexión.
    - PUEDEN fallar los canales.
  - al recibir un mensaje con una `extensión`:
    - PUEDE ignorar la `extensión`.
    - De lo contrario, si la `extensión` no es válida:
      - DEBE cerrar la conexión.
      - PUEDEN fallar los canales.

<!-- omit in toc -->
### Base Lógica

Por defecto, las claves públicas de `SHA2` y Bitcoin están codificadas como `big endian`, por lo que sería inusual usar un `endian` diferente para otros campos.

La longitud está limitada a 65535 bytes por la envoltura criptográfica, y los mensajes en el protocolo nunca superan esa longitud de todos modos.

La regla _está bien ser impar_ permite futuras extensiones opcionales sin negociación o codificación especial en los clientes. De manera similar, el campo _extensión_ permite una expansión futura al permitir que los remitentes incluyan datos TLV adicionales. Tenga en cuenta que un campo de _extensión_ solo se puede agregar cuando el mensaje `carga útil` aún no llena la longitud máxima de 65535 bytes.

Las implementaciones pueden preferir que los datos del mensaje estén alineados en un límite de 8 bytes (el requisito de alineación natural más grande de cualquier tipo aquí); sin embargo, agregar un relleno de 6 bytes después del campo de tipo se consideraba un desperdicio: la alineación se puede lograr descifrando el mensaje en un búfer con pre-relleno de 6 bytes.

## Formato Tipo-Longitud-Valor

A lo largo del protocolo, se utiliza un formato TLV (Tipo-Longitud-Valor) para permitir la adición compatible con versiones anteriores de nuevos campos a tipos de mensajes existentes.

Un registro TLV (`tlv_record`) representa un único campo, codificado de la siguiente manera:

* [`bigsize`: `type`]
* [`bigsize`: `length`]
* [`length`: `value`]

Un flujo TLV (`tlv_stream`) es una serie de registros `tlv_record` (posiblemente cero), representados como la concatenación de los registros `tlv_record` codificados. Cuando se utilizan para extender mensajes existentes, un `tlv_stream` se coloca típicamente después de todos los campos actualmente definidos.

El `type` se codifica utilizando el formato BigSize. Funciona como un identificador específico del mensaje de 64 bits para el `tlv_record`, determinando cómo se debe decodificar el contenido de `value`. Los identificadores de `type` por debajo de 2^16 están reservados para su uso en esta especificación. Los identificadores de `type` iguales o superiores a 2^16 están disponibles para registros personalizados. Cualquier registro no definido en esta especificación se considera un registro personalizado. Esto incluye mensajes experimentales y específicos de la aplicación.

La `length` se codifica utilizando el formato BigSize, indicando el tamaño de `value` en bytes.

El `value` depende completamente del `type` y debe codificarse o decodificarse de acuerdo con el formato específico del mensaje determinado por el `type`.

<!-- omit in toc -->
### Requisitos

Para el nodo emisor:
- DEBE ordenar los `tlv_record`s en un `tlv_stream` por `type` estrictamente creciente, por lo tanto, NO DEBE producir más de un único registro TLV con el mismo `type`.
- DEBE codificar mínimamente `type` y `length`.
- Al definir identificadores de registro personalizados `type`:
  - DEBERÍA elegir identificadores `type` aleatorios para evitar colisiones con otros tipos personalizados.
  - DEBERÍA elegir identificadores `type` impares cuando los nodos regulares deban ignorar los datos adicionales.
  - DEBERÍA elegir identificadores `type` pares cuando los nodos regulares deban rechazar el flujo TLV completo que contiene el registro personalizado.
- NO DEBERÍA usar codificaciones redundantes de longitud variable en un `tlv_record`.

Para el nodo receptor:
- Si no quedan bytes antes de analizar un `type`:
  - DEBE dejar de analizar el `tlv_stream`.
- Si un `type` o `length` no está codificado de forma mínima:
  - DEBE fallar al analizar el `tlv_stream`.
- Si los `type`s decodificados no son estrictamente crecientes (incluyendo situaciones en las que se encuentren dos o más ocurrencias del mismo `type`):
  - DEBE fallar al analizar el `tlv_stream`.
- Si `length` supera la cantidad de bytes restantes en el mensaje:
  - DEBE fallar al analizar el `tlv_stream`.
- Si `type` es conocido:
  - DEBE decodificar los próximos `length` bytes utilizando la codificación conocida para `type`.
  - Si `length` no es exactamente igual al requerido para la codificación conocida para `type`:
    - DEBE fallar al analizar el `tlv_stream`.
  - Si los campos de longitud variable dentro de la codificación conocida para `type` no son mínimos:
    - DEBE fallar al analizar el `tlv_stream`.
- De lo contrario, si `type` es desconocido:
  - Si `type` es par:
    - DEBE fallar al analizar el `tlv_stream`.
  - De lo contrario, si `type` es impar:
    - DEBE descartar los próximos `length` bytes.

<!-- omit in toc -->
### Racional

La principal ventaja de usar TLV es que un lector puede ignorar nuevos campos que no comprende, ya que cada campo tiene el tamaño exacto del elemento codificado. Sin TLV, incluso si un nodo no desea usar un campo en particular, el nodo se ve obligado a agregar lógica de análisis para ese campo para determinar el desplazamiento de los campos que siguen.

La estricta restricción de monotonía asegura que todos los `type` sean únicos y que puedan aparecer como máximo una vez. Campos que se asignan a objetos complejos, p. ej. los vectores, mapas o estructuras deben hacerlo definiendo la codificación de modo que el objeto se serialice dentro de un solo `tlv_record`. La restricción de unicidad, entre otras cosas, permite las siguientes optimizaciones:

 - el orden canónico se define independientemente de los `value`s codificados
 - el orden canónico se puede conocer en tiempo de compilación, en lugar de determinarse dinámicamente en el momento de la codificación.
 - verificar el orden canónico requiere menos estado y es menos costoso.
 - los campos de tamaño variable pueden reservar por adelantado su tamaño esperado, en lugar de agregar elementos secuencialmente e incurrir en una sobrecarga de duplicación y copia.

El uso de un `bigsize` para `type` y `length` permite ahorrar espacio para `type`s pequeños o `value`s cortos. Potencialmente, esto deja más espacio para los datos de la aplicación a través del cable o en una carga útil de cebolla.

Todos los `type`s deben aparecer en orden creciente para crear una codificación canónica de los `tlv_record`s subyacentes. Esto es crucial cuando se calculan las firmas sobre un `tlv_stream`, ya que garantiza que los verificadores puedan volver a calcular el mismo resumen del mensaje que el firmante. Tenga en cuenta que el orden canónico sobre el conjunto de campos se puede aplicar incluso si el verificador no comprende lo que contienen los campos.

Los escritores deben evitar el uso de codificaciones redundantes de longitud variable en un `tlv_record`, ya que esto da como resultado la codificación de la longitud dos veces y complica el cálculo de la longitud exterior. Como ejemplo, al escribir una matriz de bytes de longitud variable, el `value` debe contener solo los bytes sin procesar y renunciar a una longitud interna adicional, ya que el `tlv_record` ya lleva la cantidad de bytes que siguen. Por otro lado, si un `tlv_record` contiene múltiples elementos de longitud variable, esto no se consideraría redundante y es necesario para permitir que el receptor analice elementos individuales de `value`.

## Tipos Fundamentales

En las especificaciones del mensaje se hace referencia a varios tipos fundamentales:

* `byte`: 1 byte 8-bit
* `u16`: 2 bytes entero sin signo.
* `u32`: 4 bytes entero sin signo.
* `u64`: 8 bytes entero sin signo.

Dentro de los registros TLV que contienen un solo valor, se pueden omitir los ceros iniciales en los números enteros:

* `tu16`: entre 0 y 2 bytes entero sin signo
* `tu32`: entre 0 y 4 bytes entero sin signo
* `tu64`: entre 0 y 8 bytes entero sin signo

Cuando se usa para codificar montos, los campos anteriores DEBEN cumplir con el límite superior de 21 millones de BTC:

* las cantidades de satoshis DEBEN ser como máximo `0x000775f05a074000`
* las cantidades de mili-satoshi DEBEN ser como máximo `0x1d24b2dfac520000`

También se definen los siguientes tipos de conveniencia:

- `chain_hash`: un identificador de cadena de 32 bytes (consultar [BOLT #0](00-introduction.md#glosario-y-guía-de-la-terminología)).
- `channel_id`: un canal de identificación de 32 bytes (consultar [BOLT #2](02-peer-protocol.md#definición-de-channel_id)).
- `sha256`: un hash SHA2-256 de 32 bytes.
- `signature`: una firma de curva elíptica de Bitcoin de 64 bytes.
- `point`: un punto de curva elíptica de 33 bytes (codificación comprimida según el [estándar SEC 1](http://www.secg.org/sec1-v2.pdf#subsubsection.2.3.3)).
- `short_channel_id`: un valor de 8 bytes que identifica un canal (consultar [BOLT #7](07-routing-gossip.md#definición-del-short_channel_id)).
- `bigsize`: un entero sin signo de longitud variable similar a la codificación CompactSize de Bitcoin, pero en big-endian. Se describe en [BigSize](#apéndice-a-vectores-de-prueba-bigsize).

## Mensajes de configuración

### El Mensaje `init`

Una vez que se completa la autenticación, el primer mensaje revela las características admitidas o requeridas por este nodo, incluso si se trata de una reconexión.

[BOLT #9](09-features.md) especifica listas de características. Por lo general, cada característica se representa mediante 2 bits. El bit menos significativo se numera a 0, que es _par_, y el siguiente bit más significativo se numera a 1, que es _impar_. Por razones históricas, las características se dividen en máscaras de bits de características globales y locales.

El campo `features` DEBE estar rellenado con bytes que contengan 0s.

1. type: 16 (`init`)
2. data:
   * [`u16`:`gflen`]
   * [`gflen*byte`:`globalfeatures`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]
   * [`init_tlvs`:`tlvs`]

1. `tlv_stream`: `init_tlvs`
2. types:
    1. type: 1 (`networks`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 3 (`remote_addr`)
    2. data:
        * [`...*byte`:`data`]

El opcional `networks` indica las cadenas de bloques en las que el nodo está interesado.
El opcional `remote_addr` puede ser utilizado para sortear problemas de NAT.

<!-- omit in toc -->
#### Requisitos

Para el nodo emisor:
   - DEBE enviar `init` como el primer mensaje de Lightning para cualquier conexión.
   - DEBE establecer los bits de características según se define en [BOLT #9](09-features.md).
   - DEBE establecer cualquier bit de características indefinido a 0.
   - NO DEBERÍA establecer características mayores que 13 en `globalfeatures`.
   - DEBERÍA utilizar la longitud mínima requerida para representar el campo `features`.
   - DEBERÍA establecer `networks` en todas las cadenas para las que va a difundir o abrir canales.
   - DEBERÍA establecer `remote_addr` para reflejar la dirección IP remota (y el puerto) de una conexión entrante, si el nodo es el receptor y la conexión se realizó mediante IP.
   - Si establece `remote_addr`:
     - DEBE configurarlo como un descriptor de dirección válido (1 byte de tipo y datos) según se describe en [BOLT 7](07-routing-gossip.md#el-mensaje-node_announcement).
     - NO DEBERÍA establecer direcciones privadas como `remote_addr`.

Para el nodo receptor:
   - DEBE esperar para recibir `init` antes de enviar cualquier otro mensaje.
   - DEBE combinar (OR lógico) los dos mapas de bits de características en un mapa lógico único llamado `features`.
   - DEBE responder a los bits de características conocidos según se especifica en [BOLT #9](09-features.md).
   - al recibir bits de características desconocidos _impares_ que no sean cero:
     - DEBE ignorar el bit.
   - al recibir bits de características desconocidos _pares_ que no sean cero:
     - DEBE cerrar la conexión.
   - al recibir `networks` que no contenga cadenas comunes:
     - PUEDE cerrar la conexión.
   - si el vector de características no establece todas las dependencias conocidas y transitivas:
     - DEBE cerrar la conexión.
   - PUEDE usar `remote_addr` para actualizar su `node_announcement`.

<!-- omit in toc -->
#### Justificación

Anteriormente, había dos campos de bits de características aquí, pero para mantener la compatibilidad con versiones anteriores, ahora se han combinado en uno solo.

Esta semántica permite tanto cambios futuros incompatibles como cambios futuros compatibles con versiones anteriores. Por lo general, los bits deben asignarse en pares, para que las características opcionales puedan convertirse posteriormente en obligatorias.

Los nodos esperan recibir las características del otro para simplificar el diagnóstico de errores cuando las características son incompatibles.

Dado que todas las redes comparten el mismo puerto, pero la mayoría de las implementaciones solo admiten una sola red, el campo `networks` evita que los nodos crean erróneamente que recibirán actualizaciones sobre su red preferida o que pueden abrir canales en ella.

### El Mensaje de `error` y `warning`

Para simplificar el diagnóstico, a menudo es útil informar a un par que algo es incorrecto.

1. type: 17 (`error`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

1. type: 1 (`warning`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

<!-- omit in toc -->
#### Requisitos

Para el canal se utiliza `channel_id`, a menos que `channel_id` sea 0 (es decir, todos los bytes son 0), en cuyo caso se refiere a todos los canales.

El nodo de financiación:
   - para todos los mensajes de error enviados antes (y hasta) el mensaje `funding_created`:
     - DEBE usar `temporary_channel_id` en lugar de `channel_id`.

El nodo receptor de fondos:
   - para todos los mensajes de error enviados antes (y que no incluyen) el mensaje `funding_signed`:
     - DEBE usar `temporary_channel_id` en lugar de `channel_id`.

Un nodo emisor:
   - DEBERÍA enviar un `error` en caso de violaciones del protocolo o errores internos que hagan que los canales sean inutilizables o que dificulten la comunicación adicional.
   - DEBERÍA enviar un `error` con el `channel_id` desconocido en respuesta a mensajes de tipo `32`-`255` relacionados con canales desconocidos.
   - al enviar un `error`:
     - DEBE fallar el o los canales a los que se refiere el mensaje de error.
     - PUEDE establecer `channel_id` en todos los ceros para indicar todos los canales.
   - al enviar un `warning`:
     - PUEDE establecer `channel_id` en todos los ceros si la advertencia no está relacionada con un canal específico.
     - PUEDE cerrar la conexión después de enviar.
     - PUEDE enviar un campo `data` vacío.
     - cuando la falla fue causada por una verificación de firma inválida:
       - DEBERÍA incluir la transacción en crudo en respuesta a un mensaje `funding_created`, `funding_signed`, `closing_signed` o `commitment_signed`.

El nodo receptor:
   - al recibir un `error`:
     - si `channel_id` es todo cero:
       - DEBE fallar todos los canales con el nodo emisor.
     - de lo contrario:
       - DEBE fallar el canal al que se refiere `channel_id`, si ese canal está con el nodo emisor.
   - al recibir un `warning`:
     - DEBERÍA registrar el mensaje para un diagnóstico posterior.
     - PUEDE desconectarse.
     - PUEDE reconectarse después de un cierto retraso para volver a intentarlo.
     - PUEDE intentar un `shutdown` si está permitido en este momento.
   - si no existe un canal existente al que se refiere `channel_id`:
     - DEBE ignorar el mensaje.
   - si `data` no está compuesto únicamente por caracteres ASCII imprimibles (Para referencia: el conjunto de caracteres imprimibles incluye valores de byte del 32 al 126, inclusive):
     - NO DEBERÍA imprimir `data` tal cual.

<!-- omit in toc -->
#### Justificación

Existen errores irreparables que requieren la interrupción de conversaciones; si la conexión simplemente se cierra, el par puede intentar nuevamente la conexión. También es útil describir violaciones del protocolo para el diagnóstico, ya que esto indica que uno de los pares tiene un error.

Por otro lado, el uso excesivo de mensajes de error ha llevado a implementaciones a ignorarlos (para evitar una desconexión costosa del canal), por lo que se agregó el mensaje de "warning" para permitir cierto grado de reintentos o recuperación de errores espurios.

Puede ser prudente no distinguir errores en entornos de producción, para evitar que se filtre información, de ahí el campo opcional `data`.

## Mensajes de control

### Los mensajes `ping` y `pong`

Con el fin de permitir la existencia de conexiones TCP de larga duración, en ocasiones puede ser necesario que ambos extremos mantengan viva la conexión TCP a nivel de la aplicación. Estos mensajes también permiten la ofuscación de patrones de tráfico.

1. type: 18 (`ping`)
2. data:
    * [`u16`:`num_pong_bytes`]
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

El mensaje `pong` se envía siempre que se recibe un mensaje `ping`. Sirve como respuesta y también para mantener la conexión activa, al mismo tiempo que notifica explícitamente al otro extremo que el receptor sigue activo. Dentro del mensaje `ping` recibido, el remitente especificará la cantidad de bytes que se deben incluir en la carga de datos del mensaje `pong`.

1. type: 19 (`pong`)
2. data:
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

<!-- omit in toc -->
#### Requisitos

Un nodo que envía un mensaje `ping`:
   - DEBERÍA establecer `ignored` en 0s.
   - NO DEBE establecer `ignored` en datos sensibles como secretos o porciones de memoria inicializada.
   - Si no recibe un `pong` correspondiente:
     - PUEDE cerrar la conexión de red,
       - Y NO DEBE fallar los canales en este caso.

Un nodo que envía un mensaje `pong`:
   - DEBERÍA establecer `ignored` en 0s.
   - NO DEBE establecer `ignored` en datos sensibles como secretos o porciones de memoria inicializada.

Un nodo que recibe un mensaje `ping`:
   - Si `num_pong_bytes` es menor que 65532:
     - DEBE responder enviando un mensaje `pong`, con `byteslen` igual a `num_pong_bytes`.
   - De lo contrario (`num_pong_bytes` **no** es menor que 65532):
     - DEBE ignorar el `ping`.

Un nodo que recibe un mensaje `pong`:
   - Si `byteslen` no corresponde a ningún valor `num_pong_bytes` de `ping` que haya enviado:
     - PUEDE cerrar la conexión.

<!-- omit in toc -->
### Justificación

El mensaje más grande posible es de 65535 bytes; por lo tanto, el `byteslen` máximo razonable es de 65531, para tener en cuenta el campo de tipo (`pong`) y el propio `byteslen`. Esto permite un límite conveniente para `num_pong_bytes` para indicar que no se debe enviar ninguna respuesta.

Las conexiones entre nodos dentro de la red pueden ser de larga duración, ya que los canales de pago tienen una duración indefinida. Sin embargo, es probable que no se intercambie nuevos datos durante una parte significativa de la vida útil de una conexión. Además, en varias plataformas es posible que los clientes de Lightning se pongan en modo de suspensión sin previo aviso. Por lo tanto, se utiliza un mensaje `ping` distinto para sondear la conectividad del otro lado, así como para mantener activa la conexión establecida.

Además, la capacidad para que un remitente solicite que el receptor envíe una respuesta con un número específico de bytes permite a los nodos en la red crear tráfico sintético. Este tipo de tráfico se puede utilizar para defenderse parcialmente contra el análisis de paquetes y de sincronización de tiempos, ya que los nodos pueden simular los patrones de tráfico de intercambios típicos sin aplicar ninguna actualización real a sus respectivos canales.

Cuando se combina con el protocolo de enrutamiento de cebolla definido en [BOLT #4](04-onion-routing.md), el tráfico sintético cuidadosamente diseñado estadísticamente puede contribuir a fortalecer aún más la privacidad de los participantes en la red.

Se recomiendan precauciones limitadas contra inundaciones de `ping`, sin embargo, se otorga cierta flexibilidad debido a las demoras en la red. Tenga en cuenta que existen otros métodos de inundación de tráfico entrante (por ejemplo, enviar tipos de mensajes desconocidos _impares_ o llenar al máximo cada mensaje).

Finalmente, el uso periódico de mensajes `ping` sirve para promover rotaciones de claves frecuentes, como se especifica en [BOLT #8](08-transport.md).

## Apéndice A: Vectores de Prueba BigSize

Los siguientes vectores de prueba se pueden utilizar para verificar la corrección de una implementación de BigSize utilizada en el formato TLV. El formato es idéntico a la codificación CompactSize utilizada en Bitcoin, pero reemplaza la codificación de valores multibyte de little-endian con big-endian.

Los valores codificados con BigSize producirán una codificación de 1, 3, 5 o 9 bytes, dependiendo del tamaño del entero. La codificación es una función dividida en piezas que toma un valor `uint64` `x` y produce:
```
        uint8(x)                si x < 0xfd
        0xfd + be16(uint16(x))  si x < 0x10000
        0xfe + be32(uint32(x))  si x < 0x100000000
        0xff + be64(x)          en otros casos.
```
Aquí, `+` denota concatenación y `be16`, `be32` y `be64` producen una codificación big-endian de la entrada para enteros de 16, 32 y 64 bits, respectivamente.

Se dice que un valor está _minimamente codificado_ si no podría ser codificado usando menos bytes. Por ejemplo, una codificación de BigSize que ocupa 5 bytes pero cuyo valor es menor que 0x10000 no está codificada de manera mínima. Todos los valores decodificados con BigSize deben verificarse para asegurarse de que estén codificados de manera mínima.

<!-- omit in toc -->
### Pruebas de Decodificación BigSize

Lo siguiente es un ejemplo de cómo ejecutar las pruebas de decodificación de BigSize.
```golang
func testReadBigSize(t *testing.T, test bigSizeTest) {
        var buf [8]byte 
        r := bytes.NewReader(test.Bytes)
        val, err := tlv.ReadBigSize(r, &buf)
        if err != nil && err.Error() != test.ExpErr {
                t.Fatalf("expected decoding error: %v, got: %v",
                        test.ExpErr, err)
        }

        // Si esperamos un error de decodificación, no tiene sentido verificar el valor.
        if test.ExpErr != "" {
                return
        }

        if val != test.Value {
                t.Fatalf("expected value: %d, got %d", test.Value, val)
        }
}
```

Una implementación correcta debería pasar estas pruebas con estos vectores de prueba:
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    },
    {
        "name": "two byte not canonical",
        "value": 0,
        "bytes": "fd00fc",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "four byte not canonical",
        "value": 0,
        "bytes": "fe0000ffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "eight byte not canonical",
        "value": 0,
        "bytes": "ff00000000ffffffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "two byte short read",
        "value": 0,
        "bytes": "fd00",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte short read",
        "value": 0,
        "bytes": "feffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte short read",
        "value": 0,
        "bytes": "ffffffffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "one byte no read",
        "value": 0,
        "bytes": "",
        "exp_error": "EOF"
    },
    {
        "name": "two byte no read",
        "value": 0,
        "bytes": "fd",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte no read",
        "value": 0,
        "bytes": "fe",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte no read",
        "value": 0,
        "bytes": "ff",
        "exp_error": "unexpected EOF"
    }
]
```

<!-- omit in toc -->
### Pruebas de Programación BigSize

Lo siguiente es un ejemplo de cómo ejecutar las pruebas de programación de BigSize.
```golang
func testWriteBigSize(t *testing.T, test bigSizeTest) {
        var (
                w   bytes.Buffer
                buf [8]byte
        )
        err := tlv.WriteBigSize(&w, test.Value, &buf)
        if err != nil {
                t.Fatalf("unable to encode %d as bigsize: %v",
                        test.Value, err)
        }

        if bytes.Compare(w.Bytes(), test.Bytes) != 0 {
                t.Fatalf("expected bytes: %v, got %v",
                        test.Bytes, w.Bytes())
        }
}
```

Una correcta implementación debe pasar perfectamente los siguientes vectores de prueba:
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    }
]
```

## Apéndice B: Vectores de Test Tipo-Longitud-Valor

Las siguientes pruebas asumen que existen dos espacios de nombres TLV separados: n1 y n2.

El espacio de nombres n1 admite los siguientes tipos TLV:

1. `tlv_stream`: `n1`
2. types:
   1. type: 1 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 2 (`tlv2`)
   2. data:
     * [`short_channel_id`:`scid`]
   1. type: 3 (`tlv3`)
   2. data:
     * [`point`:`node_id`]
     * [`u64`:`amount_msat_1`]
     * [`u64`:`amount_msat_2`]
   1. type: 254 (`tlv4`)
   2. data:
     * [`u16`:`cltv_delta`]

El espacio de nombres n2 admite los siguientes tipos TLV:

1. `tlv_stream`: `n2`
2. types:
   1. type: 0 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 11 (`tlv2`)
   2. data:
     * [`tu32`:`cltv_expiry`]

<!-- omit in toc -->
### Fallos en la Decodificación TLV

Las siguientes secuencias TLV en cualquier espacio de nombres deberían desencadenar un fallo de decodificación:

1. Invalid stream: 0xfd
2. Reason: type truncated

1. Invalid stream: 0xfd01
2. Reason: type truncated

1. Invalid stream: 0xfd0001 00
2. Reason: not minimally encoded type

1. Invalid stream: 0xfd0101
2. Reason: missing length

1. Invalid stream: 0x0f fd
2. Reason: (length truncated)

1. Invalid stream: 0x0f fd26
2. Reason: (length truncated)

1. Invalid stream: 0x0f fd2602
2. Reason: missing value

1. Invalid stream: 0x0f fd0001 00
2. Reason: not minimally encoded length

1. Invalid stream: 0x0f fd0201 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
2. Reason: value truncated

Las siguientes secuencias TLV en cualquiera de los espacios de nombres, deberían desencadenar un fallo en la decodificación:

1. Invalid stream: 0x12 00
2. Reason: unknown even type.


1. Invalid stream: 0xfd0102 00
2. Reason: unknown even type.

1. Invalid stream: 0xfe01000002 00
2. Reason: unknown even type.

1. Invalid stream: 0xff0100000000000002 00
2. Reason: unknown even type.

Las siguientes secuencias TLV en el espacio de nombres `n1` deberían desencadenar un fallo en la decodificación:

1. Invalid stream: 0x01 09 ffffffffffffffffff
2. Reason: greater than encoding length for `n1`s `tlv1`.

1. Invalid stream: 0x01 01 00
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 02 0001
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 03 000100
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 04 00010000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 05 0001000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 06 000100000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 07 00010000000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x01 08 0001000000000000
2. Reason: encoding for `n1`s `tlv1`s `amount_msat` is not minimal

1. Invalid stream: 0x02 07 01010101010101
2. Reason: less than encoding length for `n1`s `tlv2`.

1. Invalid stream: 0x02 09 010101010101010101
2. Reason: greater than encoding length for `n1`s `tlv2`.

1. Invalid stream: 0x03 21 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 29 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 30 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb000000000000000100000000000001
2. Reason: less than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0x03 31 043da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. Reason: `n1`s `node_id` is not a valid point.

1. Invalid stream: 0x03 32 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001000000000000000001
2. Reason: greater than encoding length for `n1`s `tlv3`.

1. Invalid stream: 0xfd00fe 00
2. Reason: less than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0xfd00fe 01 01
2. Reason: less than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0xfd00fe 03 010101
2. Reason: greater than encoding length for `n1`s `tlv4`.

1. Invalid stream: 0x00 00
2. Reason: unknown even field for `n1`s namespace.

<!-- omit in toc -->
### Éxitos en la Decodificación TLV

Las siguientes secuencias TLV en cualquiera de los espacios de nombres deberían decodificarse correctamente y ser ignoradas:

1. Valid stream: 0x
2. Explanation: empty message

1. Valid stream: 0x21 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd0201 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd00fd 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfd00ff 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xfe02000001 00
2. Explanation: Unknown odd type.

1. Valid stream: 0xff0200000000000001 00
2. Explanation: Unknown odd type.

Las siguientes secuencias TLV en el espacio de nombres `n1` deberían decodificarse correctamente, con los valores proporcionados aquí:

1. Valid stream: 0x01 00
2. Values: `tlv1` `amount_msat`=0

1. Valid stream: 0x01 01 01
2. Values: `tlv1` `amount_msat`=1

1. Valid stream: 0x01 02 0100
2. Values: `tlv1` `amount_msat`=256

1. Valid stream: 0x01 03 010000
2. Values: `tlv1` `amount_msat`=65536

1. Valid stream: 0x01 04 01000000
2. Values: `tlv1` `amount_msat`=16777216

1. Valid stream: 0x01 05 0100000000
2. Values: `tlv1` `amount_msat`=4294967296

1. Valid stream: 0x01 06 010000000000
2. Values: `tlv1` `amount_msat`=1099511627776

1. Valid stream: 0x01 07 01000000000000
2. Values: `tlv1` `amount_msat`=281474976710656

1. Valid stream: 0x01 08 0100000000000000
2. Values: `tlv1` `amount_msat`=72057594037927936

1. Valid stream: 0x02 08 0000000000000226
2. Values: `tlv2` `scid`=0x0x550

1. Valid stream: 0x03 31 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. Values: `tlv3` `node_id`=023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb `amount_msat_1`=1 `amount_msat_2`=2

1. Valid stream: 0xfd00fe 02 0226
2. Values: `tlv4` `cltv_delta`=550

<!-- omit in toc -->
### Fallo en la Decodificación de Secuencia TLV

La adición de una secuencia inválida a una secuencia válida debería desencadenar un fallo en la decodificación.

La adición de una secuencia válida de un número más alto a una secuencia válida de un número más bajo no debería desencadenar un fallo en la decodificación.

Además, las siguientes secuencias TLV en el espacio de nombres `n1` deberían desencadenar un fallo en la decodificación:

1. Invalid stream: 0x02 08 0000000000000226 01 01 2a
2. Reason: valid TLV records but invalid ordering

1. Invalid stream: 0x02 08 0000000000000231 02 08 0000000000000451
2. Reason: duplicate TLV type

1. Invalid stream: 0x1f 00 0f 01 2a
2. Reason: valid (ignored) TLV records but invalid ordering

1. Invalid stream: 0x1f 00 1f 01 2a
2. Reason: duplicate TLV type (ignored)

La siguiente secuencia TLV en el espacio de nombres `n2` debería desencadenar un fallo en la decodificación:

1. Invalid stream: 0xffffffffffffffffff 00 00 00
2. Reason: valid TLV records but invalid ordering

## Apéndice C: Extensión de Mensaje

Esta sección contiene ejemplos de extensiones válidas e inválidas en el mensaje `init`. El mensaje `init` base (sin extensiones) para estos ejemplos es `0x001000000000` (todas las funciones desactivadas).

Los siguientes mensajes `init` son válidos:

- `0x001000000000`: no se proporciona ninguna extensión.
- `0x001000000000c9012acb0104`: la extensión contiene dos registros TLV desconocidos _impares_ (con tipos `0xc9` y `0xcb`).

Los siguientes mensajes `init` son inválidos:

- `0x00100000000001`: la extensión está presente pero truncada.
- `0x001000000000ca012a`: la extensión contiene registros TLV desconocidos _pares_ (asumiendo que el tipo TLV `0xca` es desconocido).
- `0x001000000000c90101c90102`: la secuencia TLV de la extensión es inválida (duplicación del tipo de registro TLV `0xc9`).

Tenga en cuenta que cuando los mensajes están firmados, la _extensión_ forma parte de los bytes firmados. Los nodos deben almacenar los bytes de _extensión_ incluso si no los comprenden para poder verificar correctamente las firmas.

## Agradecimientos

[ TODO: (roasbeef); fin ]

## Referencias

1. <a id="reference-1">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Autores

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
