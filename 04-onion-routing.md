<!-- omit in toc -->
# BOLT #4: Protocolo Onion Routing

<!-- omit in toc -->
## Overview

Este documento describe la construcción de un paquete enrutado cebolla que se utiliza para enrutar un pago desde un _nodo de origen_ a un _nodo final_. El paquete se enruta a través de una serie de nodos intermedios, llamados _hops_ o _saltos_.

El esquema de enrutado se basa en la contrucción [Sphinx][sphinx] y se extiende con la carga útil (`payload`) por salto.

Los nodos intermedios que reenvían el mensaje pueden verificar la integridad del paquete y pueden saber a qué nodo deben reenviar el paquete. No pueden saber qué otros nodos, además de su predecesor o sucesor, son parte de la ruta del paquete; tampoco pueden conocer la longitud del recorrido ni su posición dentro del mismo. El paquete se ofusca en cada salto para garantizar que un atacante a nivel de red no pueda asociar paquetes que pertenecen a la misma ruta (es decir, los paquetes que pertenecen a la misma ruta no comparten ninguna información de correlación). Tenga en cuenta que esto no excluye la posibilidad de asociación de paquetes por parte de un atacante a través del análisis de tráfico.

La ruta la construye el nodo origen, que conoce las claves públicas de cada nodo intermedio y del nodo final. Conocer la clave pública de cada nodo permite que el nodo de origen cree un secreto compartido (usando ECDH) para cada nodo intermedio y para el nodo final. Luego, el secreto compartido se usa para generar un _flujo pseudoaleatorio_ de bytes (que se usa para ofuscar el paquete) y una cantidad de _claves_ (que se usan para cifrar la carga útil y calcular los HMAC). Los HMAC se utilizan a su vez para garantizar la integridad del paquete en cada salto.

Cada salto a lo largo de la ruta solo ve una clave efímera para el nodo de origen, con el fin de ocultar la identidad del remitente. La clave efímera queda oculta por cada salto intermedio antes de pasar al siguiente, lo que hace que las cebollas no se puedan vincular a lo largo de la ruta.

Esta especificación describe la _versión 0_ del formato de paquete y el mecanismo de enrutamiento.

Un nodo:
  - al recibir un paquete de versión superior a la que implementa:
    - DEBE informar una falla de ruta al nodo de origen.
    - DEBE descartar el paquete.

<!-- omit in toc -->
# Índice

- [Convenciones](#convenciones)
- [Generación de claves](#generación-de-claves)
- [Flujo de Bytes Pseudoaleatorio](#flujo-de-bytes-pseudoaleatorio)
- [Estructura del Paquete](#estructura-del-paquete)
    - [`payload` format](#payload-format)
    - [Pagos Multiparte Básicos](#pagos-multiparte-básicos)
    - [Ocultar la ruta](#ocultar-la-ruta)
- [Aceptar y Reenviar un Pago](#aceptar-y-reenviar-un-pago)
  - [Reenvío No Estricto](#reenvío-no-estricto)
  - [Carga útil para el último nodo](#carga-útil-para-el-último-nodo)
- [Secreto compartido](#secreto-compartido)
- [Claves efímeras cegadoras](#claves-efímeras-cegadoras)
- [Construcción de paquetes](#construcción-de-paquetes)
- [Reenvío de paquetes](#reenvío-de-paquetes)
- [Generación del relleno](#generación-del-relleno)
- [Devolviendo errores](#devolviendo-errores)
  - [Mensajes de error](#mensajes-de-error)
  - [Recibir códigos de error](#recibir-códigos-de-error)
- [Test Vector](#test-vector)
  - [Returning Errors](#returning-errors)
- [Referencias](#referencias)
- [Authors](#authors)

# Convenciones

Hay una serie de convenciones a los que se adhiere a lo largo de este documento:

 - HMAC: la verificación de la integridad del paquete se basa en el código de autenticación de mensajes Keyed-Hash, según lo define el [estándar FIPS 198][fips198]/[RFC 2104][RFC2104], y utiliza un algoritmo hash `SHA256`.
 - Curva eliptica: para todos los cálculos que involucran curvas elípticas, se usa la curva de Bitcoin, como se especifica en [`secp256k1`][sec2]
 - Stream Pseudo-aleatorio: [`ChaCha20`][rfc8439] se utiliza para generar un flujo de bytes pseudoaleatorios. Para su generación, se utiliza un nonce nulo fijo de 96 bits (`0x0000000000000000000000000`), junto con una clave derivada de un secreto compartido y con un flujo de bytes `0x00` del tamaño de salida deseado como mensaje.
 - Los términos _nodo origen_ y _nodo final_ se refieren al remitente del paquete inicial y al receptor del paquete final, respectivamente.
 - Los términos _hop_ y _node_ a veces se usan indistintamente, pero un _hop_ generalmente se refiere a un nodo intermedio en la ruta en lugar de un nodo final.
        _nodo origen_ --> _hop_ --> ... --> _hop_ --> _nodo final_
 - El término _nodo de procesamiento_ se refiere al nodo específico a lo largo de la ruta que actualmente está procesando el paquete reenviado.
 - El término _peers_ se refiere solo a saltos que son vecinos directos (en la red superpuesta): más específicamente, _peers que envían_ reenvían paquetes a _peers que reciben_.
 - Cada salto en la ruta tiene una longitud variable `hop_payload`.
    - La longitud variable `hop_payload` tiene el prefijo `bigsize` que codifica la longitud en bytes, excluyendo el prefijo y el HMAC final.

# Generación de claves

Una serie de claves de encriptación y verificación se derivan del secreto compartido:

 - _rho_: se usa como clave al generar el flujo de bytes pseudoaleatorio que se usa para ofuscar la información por salto
 - _mu_: utilizado durante la generación de HMAC
 - _um_: utilizado durante el informe de errores
 - _pad_: se usa para generar bytes de relleno aleatorios para el paquete de encabezado de mezcla inicial

La función de generación de claves toma un tipo de clave (_rho_=`0x72686F`, _mu_=`0x6d75`, _um_=`0x756d` o _pad_=`0x706164`) y un secreto de 32 bytes como entradas y devuelve una clave de 32 bytes .

Las claves se generan calculando un HMAC (con `SHA256` como algoritmo hash) utilizando el tipo de clave adecuado (es decir, _rho_, _mu_, _um_ o _pad_) como clave HMAC y el secreto compartido de 32 bytes como mensaje. El HMAC resultante se devuelve como clave.

Tenga en cuenta que el tipo de clave no incluye un byte de terminación `0x00` de estilo C, p. la longitud del tipo de clave _rho_ es de 3 bytes, no de 4.

# Flujo de Bytes Pseudoaleatorio

El flujo de bytes pseudoaleatorios se usa para ofuscar el paquete en cada salto de la ruta, de modo que cada salto solo pueda recuperar la dirección y el HMAC del siguiente salto. El flujo de bytes pseudoaleatorio se genera mediante el cifrado (usando `ChaCha20`) de un flujo de bytes `0x00`, de la longitud requerida, que se inicializa con una clave derivada del secreto compartido y un cero nonce de 96 bits (` 0x000000000000000000000000`).

El uso de un nonce fijo es seguro, ya que las claves nunca se reutilizan.

# Estructura del Paquete

El paquete está formado por cuatro secciones:

 - un byte de `version`
 - 33 bytes comprimidos con `secp256k1` `public_key`, utilizado durante la generación del secreto compartido.
 - un `hop_payloads` de 1300 bytes que consta de múltiples cargas útiles `hop_payload` de longitud variable
 - un `hmac` de 32 bytes, utilizados para verificar la integridad del paquete.

El formato de red del paquete consta de secciones individuales serializadas en un flujo de bytes contiguo y luego transferidas al destinatario del paquete. Debido al tamaño fijo del paquete, no es necesario que tenga como prefijo su longitud cuando se transfiere a través de una conexión.

La estrcutura generla del paquete es la siguiente:

1. type: `onion_packet`
2. data:
   * [`byte`:`version`]
   * [`point`:`public_key`]
   * [`1300*byte`:`hop_payloads`]
   * [`32*byte`:`hmac`]

Para esta especificación (_versió 0_), `version` tiene un valor constante de `0x00`.

El campo `hop_payloads` es una estructura que contiene información de enrutamiento ofuscada y el HMAC asociado.

Tiene una longitud de 1300 bytes y tiene la siguiente estructura:

1. type: `hop_payloads`
2. data:
   * [`bigsize`:`length`]
   * [`length*byte`:`payload`]
   * [`32*byte`:`hmac`]
   * ...
   * `filler`

Donde, `length`, `payload`, y `hmac` se repiten para cada salto; y donde, `filler` consiste en un relleno ofuscado generado de forma determinista, como se detalla en [Generación de relleno](#filler-generation).
Además, `hop_payloads` se ofusca de forma incremental en cada salto.

Usando el campo `payload`, el nodo de origen puede especificar la ruta y la estructura de los HTLC reenviados en cada salto.
Como la `payload` está protegida por el HMAC de todo el paquete, la información que contiene se autentica por completo con cada relación de pares entre el remitente HTLC (nodo de origen) y cada salto en la ruta.

Con esta autenticación de extremo a extremo, cada salto puede cotejar los parámetros HTLC con los valores especificados de la `payload` y asegurarse de que el par de envío no haya reenviado un HTLC mal diseñado.

Dado que ningún valor TLV de `payload` puede ser inferior a 2 bytes, los valores de `length` de 0 y 1 están reservados. (`0` indica un formato heredado que ya no es compatible, y `1` está reservado para uso futuro).

### `payload` format

This is formatted according to the Type-Length-Value format defined in [BOLT #1](01-messaging.md#formato-tipo-longitud-valor).

1. `tlv_stream`: `payload`
2. types:
    1. type: 2 (`amt_to_forward`)
    2. data:
        * [`tu64`:`amt_to_forward`]
    1. type: 4 (`outgoing_cltv_value`)
    2. data:
        * [`tu32`:`outgoing_cltv_value`]
    1. type: 6 (`short_channel_id`)
    2. data:
        * [`short_channel_id`:`short_channel_id`]
    1. type: 8 (`payment_data`)
    2. data:
        * [`32*byte`:`payment_secret`]
        * [`tu64`:`total_msat`]
    1. type: 10 (`encrypted_recipient_data`)
    2. data:
        * [`...*byte`:`encrypted_data`]
    1. type: 12 (`current_blinding_point`)
    2. data:
        * [`point`:`blinding`]
    1. type: 16 (`payment_metadata`)
    2. data:
        * [`...*byte`:`payment_metadata`]
    1. type: 18 (`total_amount_msat`)
    2. data:
        * [`tu64`:`total_msat`]

`short_channel_id` es la ID del canal saliente utilizado para enrutar el mensaje; el par receptor debe operar el otro extremo de este canal.

`amt_to_forward` es la cantidad, en milisatoshis, para reenviar al siguiente par receptor especificado en la información de enrutamiento, o para el destino final.

Para los nodos no finales, esto incluye la _tarifa_ calculada del nodo de origen para el par receptor, calculada de acuerdo con el esquema de tarifas anunciadas del par receptor (como se describe en [BOLT #7](07-routing-gossip.md#htlc-fees)) .

`outgoing_cltv_value` es el valor CLTV que debe tener el HTLC _outgoing_ que transporta el paquete. La inclusión de este campo permite un salto para autenticar la información especificada por el nodo de origen y los parámetros del HTLC reenviado, y garantizar que el nodo de origen esté utilizando el valor `cltv_expiry_delta` actual.

Si los valores no se corresponden, esto indica que un nodo de reenvío ha manipulado los valores HTLC previstos o que el nodo de origen tiene un valor `cltv_expiry_delta` obsoleto.

Los requisitos aseguran la consistencia en la respuesta a un `outgoing_cltv_value` inesperado, sea el nodo final o no, para evitar filtrar su posición en la ruta.

<!-- omit in toc -->
### Requisitos

El creador de `encrypted_recipient_data` (generalmente, el destinatario del pago):

  - DEBE crear `encrypted_data_tlv` para cada nodo en la ruta ciega (incluido él mismo).
  - DEBE incluir `encrypted_data_tlv.short_channel_id` y `encrypted_data_tlv.payment_relay` para cada nodo no final.
  - DEBE configurar `encrypted_data_tlv.payment_constraints` para cada nodo no final:
    - `max_cltv_expiry` a la altura de bloque más grande a la que se permite usar la ruta, comenzando
    desde el nodo final y añadiendo `encrypted_data_tlv.payment_relay.cltv_expiry_delta` en cada salto.
    - `htlc_minimum_msat` al mayor valor mínimo de HTLC que permitan los nodos.
  - Si establece `encrypted_data_tlv.allowed_features`:
    - DEBE establecerlo en una matriz vacía.
  - DEBE calcular las tarifas totales y cltv delta de la ruta de la siguiente manera y comunicarlas al remitente:
    - `total_fee_base_msat(n+1) = (fee_base_msat(n+1) * 1000000 + total_fee_base_msat(n) * (1000000 + fee_proportional_millionths(n+1)) + 1000000 - 1) / 1000000`
    - `total_fee_proportional_millionths(n+1) = ((total_fee_proportional_millionths(n) + fee_proportional_millionths(n+1)) * 1000000 + total_fee_proportional_millionths(n) * fee_proportional_millionths(n+1) + 1000000 - 1) / 1000000`
  - DEBE crear `encrypted_recipient_data` a partir de `encrypted_data_tlv` como se requiere en [Route Blinding](#route-blinding).

El escritor de `tlv_payload`:

  - Para cada nodo dentro de una ruta ciega:
    - DEBE incluir los `encrypted_recipient_data` proporcionados por el destinatario
    - Para el primer nodo de la ruta ciega:
      - DEBE incluir el `blinding_point` proporcionado por el destinatario en `current_blinding_point`
    - Si es el nodo final:
      - DEBE incluir `amt_to_forward`, `outgoing_cltv_value` y `total_amount_msat`.
    - NO DEBE incluir ningún otro campo tlv.
  - Para cada nodo fuera de una ruta ciega:
    - DEBE incluir `amt_to_forward` y `outgoing_cltv_value`.
    - Para cada nodo no final:
      - DEBE incluir `short_channel_id`
      - NO DEBE incluir `payment_data`
    - Para el nodo final:
      - NO DEBE incluir `short_channel_id`
      - si el destinatario proporcionó `payment_secret`:
        - DEBE incluir `payment_data`
        - DEBE establecer `payment_secret` en el proporcionado
        - DEBE establecer `total_msat` en la cantidad total que enviará
      - si el destinatario proporcionó `payment_metadata`:
        - DEBE incluir `payment_metadata` con cada HTLC
        - No DEBE aplicar ningún límite al tamaño de `payment_metadata` excepto los límites implícitos en el tamaño de cebolla fijo

El lector:

  - Si `encrypted_recipient_data` está presente:
    - Si se establece `blinding_point` en el `update_add_htlc` entrante:
      - DEBE devolver un error si `current_blinding_point` está presente.
      - DEBE usar ese `blinding_point` como el punto ciego para el descifrado.
    - De lo contrario:
      - DEBE devolver un error si `current_blinding_point` no está presente.
      - DEBE usar ese `current_blinding_point` como el punto ciego para el descifrado.
      - DEBERÍA agregar un retraso aleatorio antes de devolver errores.
    - DEBE devolver un error si `encrypted_recipient_data` no se descifra usando el
      punto ciego como se describe en [Route Blinding](#route-blinding).
    - Si `payment_constraints` está presente:
      - DEBE devolver un error si:
        - el vencimiento es mayor que `encrypted_recipient_data.payment_constraints.max_cltv_expiry`.
        - la cantidad está por debajo de `encrypted_recipient_data.payment_constraints.htlc_minimum_msat`.
    - Si falta `características_permitidas`:
      - DEBE procesar el mensaje como si estuviera presente y contuviera una matriz vacía.
    - DEBE devolver un error si:
      - `encrypted_recipient_data.allowed_features.features` contiene un bit de función desconocido (incluso si es impar).
      - el pago utiliza una característica no incluida en `encrypted_recipient_data.allowed_features.features`.
    - Si no es el nodo final:
      - DEBE devolver un error si la carga útil contiene otros campos tlv además de `encrypted_recipient_data` y `current_blinding_point`.
      - DEBE devolver un error si `encrypted_recipient_data` no contiene `short_channel_id` o `next_node_id`.
      - DEBE devolver un error si `encrypted_recipient_data` no contiene `payment_relay`.
      - DEBE usar valores de `encrypted_recipient_data.payment_relay` para calcular `amt_to_forward` y `outgoing_cltv_value` de la siguiente manera:
        - `amt_to_forward = ((amount_msat - fee_base_msat) * 1000000 + 1000000 + fee_proportional_millionths - 1) / (1000000 + fee_proportional_millionths)`
        - `outgoing_cltv_value = cltv_expiry - pago_relay.cltv_expiry_delta`
    - Si es el nodo final:
      - DEBE devolver un error si la carga útil contiene otros campos tlv que `encrypted_recipient_data`, `current_blinding_point`, `amt_to_forward`, `outgoing_cltv_value` y `total_amount_msat`.
      - DEBE devolver un error si `amt_to_forward`, `outgoing_cltv_value` o `total_amount_msat` no están presentes.
      - DEBE devolver un error si `amt_to_forward` está por debajo de lo que espera para el pago.
      - DEBE devolver un error si `cltv_expiry` entrante < `outgoing_cltv_value`.
      - DEBE devolver un error si `cltv_expiry` entrante < `current_block_height` + `min_final_cltv_expiry_delta`.
  - En caso contrario (no forma parte de una ruta ciega):
    - DEBE devolver un error si `blinding_point` está configurado en `update_add_htlc` entrante o `current_blinding_point` está presente.
    - DEBE devolver un error si `amt_to_forward` o `outgoing_cltv_value` no están presentes.
    - si no es el nodo final:
      - DEBE devolver un error si:
        - `short_channel_id` no está presente,
        - no puede reenviar el HTLC al par indicado por el canal `short_channel_id`.
        - `amount_msat` entrante - `fee` < `amt_to_forward` (donde `fee` es la tarifa anunciada como se describe en [BOLT #7](07-routing-gossip.md#htlc-fees))
        - `cltv_expiry` - `cltv_expiry_delta` < `outgoing_cltv_value`
  - Si es el nodo final:
    - DEBE tratar `total_msat` como si fuera igual a `amt_to_forward` si no está presente.
    - DEBE devolver un error si:
      - entrante `amount_msat` < `amt_to_forward`.
      - `cltv_expiry` entrante < `outgoing_cltv_value`.
      - entrante `cltv_expiry` < `current_block_height` + `min_final_cltv_expiry_delta`.

Se especifican requisitos adicionales [aquí](#pagos-multiparte-básicos) para pagos de varias partes y [aquí](#route-blinding) para pagos ocultos.

### Pagos Multiparte Básicos

Un HTLC puede ser parte de un pago 'multiparte' más grande: tales pagos atómicos 'base' de rutas múltiples utilizarán el mismo `payment_hash` para todas las rutas.

Tenga en cuenta que `amt_to_forward` es la cantidad para este HTLC únicamente: un campo `total_msat` que contiene un valor mayor es una promesa del remitente final de que el resto del pago seguirá en los HTLC sucesivos; llamamos a estos HTLC sobresalientes que tienen la misma preimagen, un 'conjunto HTLC'.

Tenga en cuenta que hay dos campos tlv distintos que se pueden usar para transmitir `total_msat`. El último, `total_amount_msat`, se introdujo con rutas ciegas para las que `payment_secret` no tiene sentido.

`metadatos_de_pago` debe incluirse en cada parte de pago, de modo que los detalles de pago no válidos puedan detectarse lo antes posible.

<!-- omit in toc -->
#### Requisitos

El escritor:
  - si la factura ofrece la función `basic_mpp`:
    - PUEDE enviar más de un HTLC para pagar la factura.
    - DEBE usar el mismo `payment_hash` en todos los HTLC del conjunto.
    - DEBE enviar todos los pagos aproximadamente al mismo tiempo.
    - DEBERÍA tratar de utilizar diversas rutas al destinatario para cada HTLC.
    - DEBERÍA volver a intentar y/o volver a dividir los HTLC que fallan.
    - si la factura especifica un `amount`:
       - DEBE establecer `total_msat` en al menos esa `amount`, y menos
         que o igual al doble de la `amount`.
    - de lo contrario:
      - DEBE establecer `total_msat` en la cantidad que desea pagar.
    - DEBE asegurarse de que el `amt_to_forward` total del conjunto HTLC que llega
      en el beneficiario es igual o mayor que `total_msat`.
    - NO DEBE enviar otro HTLC si el `amt_to_forward` total del conjunto HTLC
      ya es mayor o igual a `total_msat`.
    - DEBE incluir `payment_secret`.
  - de lo contrario:
    - DEBE establecer `total_msat` igual a `amt_to_forward`.

El nodo final:
  - DEBE fallar el HTLC si lo dictan los requisitos en [Mensajes de error] (#mensajes de error)
    - Nota: 'cantidad pagada' especificada allí está el campo `total_msat`.
  - si no es compatible con `basic_mpp`:
    - DEBE fallar el HTLC si `total_msat` no es exactamente igual a `amt_to_forward`.
  - de lo contrario, si es compatible con `basic_mpp`:
    - DEBE agregarlo al conjunto HTLC correspondiente a ese `pago_hash`.
    - DEBERÍA fallar todo el conjunto HTLC si `total_msat` no es el mismo para
      todos los HTLC del conjunto.
    - si el `amt_to_forward` total de este conjunto HTLC es igual o mayor
      que `total_msat`:
      - DEBE cumplir con todos los HTLC en el conjunto HTLC
    - de lo contrario, si el `amt_to_forward` total de este conjunto HTLC es menor que
      `total_msat`:
      - NO DEBE cumplir con ningún HTLC en el conjunto HTLC
      - DEBEN fallar todos los HTLC en el conjunto HTLC después de un tiempo de espera razonable.
        - DEBE esperar al menos 60 segundos después del HTLC inicial.
        - DEBE usar `mpp_timeout` para el mensaje de error.
      - DEBE requerir `payment_secret` para todos los HTLC en el conjunto.
    - si cumple con cualquier HTLC en el conjunto HTLC:
       - DEBE cumplir con todo el conjunto HTLC.

<!-- omit in toc -->
#### Racional

Si `basic_mpp` está presente, provoca un retraso para permitir que se combinen otros pagos parciales. El monto total debe ser suficiente para el pago deseado, al igual que para los pagos únicos. Pero esto debe estar razonablemente acotado para evitar una denegación de servicio.

Debido a que las facturas no necesariamente especifican un monto, y debido a que los pagadores pueden agregar ruido al monto final, el monto total debe enviarse explícitamente. Los requisitos permiten superarlo ligeramente, ya que simplifica añadir ruido al importe al fraccionar, así como escenarios en los que los remitentes son realmente independientes (amigos dividiendo una cuenta, por ejemplo).

Debido a que un nodo puede necesitar pagar más de la cantidad deseada (debido al valor `htlc_minimum_msat` de los canales en la ruta deseada), los nodos pueden pagar más que el `total_msat` que especificaron. De lo contrario, los nodos estarían limitados en cuanto a las rutas que pueden tomar al volver a intentar pagos a lo largo de rutas específicas. Sin embargo, ningún HTLC individual puede ser por menos de la diferencia entre el total pagado y `total_msat`.

La restricción de enviar un HTLC una vez que el conjunto supera el total acordado impide que se libere la preimagen antes de que hayan llegado todos los pagos parciales: eso permitiría a cualquier nodo intermedio reclamar de inmediato los pagos parciales pendientes.

Una implementación puede optar por no cumplir con un conjunto HTLC que de otro modo cumple con el criterio de cantidad (por ejemplo, alguna otra falla o tiempo de espera de la factura), sin embargo, si solo cumpliera con algunos de ellos, los nodos intermediarios podrían simplemente reclamar los restantes.

### Ocultar la ruta

Los nodos que reciben paquetes cebolla pueden ocultar su identidad a los remitentes 'cegando' una cantidad arbitraria de saltos al final de una ruta cebolla.

Cuando se usa el enmascaramiento de rutas, los nodos encuentran una ruta hacia ellos mismos desde un 'nodo de introducción' y un 'punto de enmascaramiento' inicial dado. Luego usan ECDH con cada nodo en esa ruta para crear una ID de nodo 'ciego' y un blob encriptado (`encrypted_data`) para cada uno de los nodos ciegos.

Comunican esta ruta ciega y los blobs cifrados al remitente.
El remitente encuentra una ruta al nodo de introducción y la amplía con la ruta ciega proporcionada por el destinatario. El remitente incluye los blobs cifrados en las cargas útiles de cebolla correspondientes: permiten que los nodos en la parte oculta de la ruta 'desbloqueen' el siguiente nodo y reenvíen correctamente el paquete.

Tenga en cuenta que el remitente tiene dos formas de llegar al punto de introducción: una es crear un pago normal (no cegado) y colocar el punto de cegamiento inicial en `current_blinding_point` junto con el
`encrypted_data` en la carga útil de cebolla para el punto de introducción para iniciar la ruta ciega. La segunda forma es crear una ruta ciega al punto de introducción, establezca `next_blinding_override` dentro del
`encrypted_data_tlv` en el salto anterior al punto de introducción al punto ciego inicial, y enviarlo al nodo de introducción.

Los `datos_cifrados` son un flujo TLV, cifrado para un nodo ciego dado, que puede contener los siguientes campos TLV:

1. `tlv_stream`: `encrypted_data_tlv`
2. types:
    1. type: 1 (`padding`)
    2. data:
        * [`...*byte`:`padding`]
    1. type: 2 (`short_channel_id`)
    2. data:
        * [`short_channel_id`:`short_channel_id`]
    1. type: 4 (`next_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 6 (`path_id`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 8 (`next_blinding_override`)
    2. data:
        * [`point`:`blinding`]
    1. type: 10 (`payment_relay`)
    2. data:
        * [`u16`:`cltv_expiry_delta`]
        * [`u32`:`fee_proportional_millionths`]
        * [`tu32`:`fee_base_msat`]
    1. type: 12 (`payment_constraints`)
    2. data:
        * [`u32`:`max_cltv_expiry`]
        * [`tu64`:`htlc_minimum_msat`]
    1. type: 14 (`allowed_features`)
    2. data:
        * [`...*byte`:`features`]


<!-- omit in toc -->
#### Requisitos

Un destinatario N(r) que crea una ruta ciega `N(0) -> N(1) -> ... -> N(r)` hacia sí mismo:

- DEBE crear un ID de nodo ciego `B(i)` para cada nodo utilizando el siguiente algoritmo:
  - `e(0) <- {0;1}%256`
  - `E(0) = e(0) * G`
  - Para cada nodo de la ruta:
    - sea `N(i) = k(i) * G` el `node_id` (`k(i)` es la clave privada de `N(i)`)
    - `ss(i) = SHA256(e(i) * N(i)) = SHA256(k(i) * E(i))` (secreto compartido ECDH conocido solo por `N(r)` y `N( yo)`)
    - `B(i) = HMAC256('blinded_node_id', ss(i)) * N(i)` (`node_id` ciego para `N(i)`, clave privada conocida solo por `N(i)`)
    - `rho(i) = HMAC256('rho', ss(i))` (clave utilizada para cifrar la carga útil para `N(i)` por `N(r)`)
    - `e(i+1) = SHA256(E(i) || ss(i)) * e(i)` (clave privada efímera ciega, solo conocida por `N(r)`)
    - `E(i+1) = SHA256(E(i) || ss(i)) * E(i)` (NB: `N(i)` NO DEBE aprender `e(i)`)
- PUEDE reemplazar `E(i+1)` con un valor diferente, pero si lo hace:
  - DEBE establecer `encrypted_data_tlv(i).next_blinding_override` en `E(i+1)`
- PUEDE almacenar datos privados en `encrypted_data_tlv(r).path_id` para verificar que la ruta se usa en el contexto correcto y fue creada por ellos
- DEBERÍA agregar datos de relleno para garantizar que todos los `encrypted_data_tlv(i)` tengan la misma longitud
- DEBE encriptar cada `encrypted_data_tlv(i)` con ChaCha20-Poly1305 usando la clave `rho(i)` correspondiente y un solo cero para producir `encrypted_recipient_data(i)`
- DEBE comunicar los ID de nodo ocultos `B(i)` y `encrypted_recipient_data(i)` al remitente
- DEBE comunicar el ID de nodo real del punto de introducción `N(0)` al remitente
- DEBE comunicar la primera clave efímera cegadora `E(0)` al remitente

Un lector:

- Si recibe `blinding_point` (`E(i)`) del par anterior:
  - DEBE usar `b(i)` en lugar de su clave privada `k(i)` para descifrar la cebolla.
    Tenga en cuenta que el nodo puede modificar la clave efímera cebolla con `HMAC256('blinded_node_id', ss(i))` que logra el mismo resultado.
- De lo contrario:
  - DEBE usar `k(i)` para descifrar la cebolla, para extraer `current_blinding_point` (`E(i)`).
- DEBE calcular:
  - `ss(i) = SHA256(k(i) * E(i))` (ECDH estándar)
  - `b(i) = HMAC256('blinded_node_id', ss(i)) * k(i)`
  - `rho(i) = HMAC256('rho', ss(i))`
  - `E(i+1) = SHA256(E(i) || ss(i)) * E(i)`
- DEBE descifrar el campo `encrypted_data` usando `rho(i)` y usar los campos descifrados para ubicar el siguiente nodo
- Si falta el campo `encrypted_data` o no se puede descifrar:
  - DEBE devolver un error
- Si `encrypted_data` contiene `next_blinding_override`:
  - DEBE usarlo como el siguiente punto ciego en lugar de `E(i+1)`
- De lo contrario:
  - DEBE usar `E(i+1)` como el siguiente punto ciego
- DEBE reenviar la cebolla e incluir el siguiente punto ciego en el mensaje de relámpago para el siguiente nodo

El receptor final:

- DEBE calcular:
  - `ss(r) = SHA256(k(r) * E(r))` (ECDH estandard)
  - `b(r) = HMAC256("blinded_node_id", ss(r)) * k(r)`
  - `rho(r) = HMAC256("rho", ss(r))`
- DEBE descrifar el campo `encrypted_data` usando `rho(r)`
- Si falta el campo `encrypted_data` o no puede ser descifrado:
  - DEBE devolver un error
- DEBE ignorar el mensaje si `path_id` no coincide con la ruta ciega que creó

<!-- omit in toc -->
#### Racional

El enmascaramiento de rutas es una técnica ligera para brindar anonimato a los destinatarios.
Es más flexible que el enrutamiento de encuentro porque simplemente reemplaza las claves públicas de los nodos en la ruta con claves públicas aleatorias mientras permite que los remitentes elijan qué datos ponen en la cebolla para cada salto. Las rutas ciegas también son reutilizables en algunos casos (por ejemplo, mensajes de cebolla).

Cada nodo en la ruta ciega necesita recibir `E(i)` para poder descifrar la cebolla y la carga útil `encrypted_data`. Los protocolos que utilizan el enmascaramiento de rutas deben especificar cómo se propaga este valor entre los nodos.

Al concatenar dos rutas ciegas generadas por diferentes nodos, el último nodo de la primera ruta necesita conocer el primer `blinding_point` de la segunda ruta: el campo `next_blinding_override` debe usarse para transmitir esta información.

El destinatario final debe verificar que la ruta ciega se utiliza en el contexto correcto (por ejemplo, para un pago específico) y que fue creada por él. De lo contrario, un remitente malintencionado podría crear diferentes rutas ciegas a todos los nodos que sospecha que pueden ser el destinatario real y probarlos hasta que uno acepte el mensaje. El destinatario puede protegerse contra eso almacenando 'E(r)' y el contexto (por ejemplo, un 'payment_hash'), y verificando que coincidan al recibir la cebolla. De lo contrario, para evitar costos de almacenamiento adicionales, puede colocar información de contexto privada en el campo `path_id` (por ejemplo, `payment_preimage`) y verificarlo al recibir la cebolla. Tenga en cuenta que es importante utilizar información privada en ese caso, a la que los remitentes no pueden tener acceso.

Cada vez que el punto de introducción recibe una falla de la ruta oculta, debe agregar un retraso aleatorio antes de reenviar el error. Es probable que las fallas sean intentos de sondeo y la sincronización del mensaje puede ayudar al atacante a inferir su distancia al destinatario final.

El campo `padding` se puede utilizar para garantizar que todos los `encrypted_data` tengan la misma longitud. Es particularmente útil cuando se agregan saltos ficticios al final de una ruta oculta, para evitar que el remitente descubra qué nodo es el destinatario final.

Cuando se utiliza el enmascaramiento de rutas para los pagos, el destinatario especifica las tarifas y el vencimiento que los nodos ocultos deben aplicar al pago en lugar de permitir que el remitente los configure. El destinatario también agrega restricciones adicionales a los pagos que pueden pasar por esa ruta para protegerse contra ataques de sondeo que permitirían que los nodos maliciosos revelaran la identidad de los nodos cegados. Debe configurar `payment_constraints.max_cltv_expiry` para restringir la vida útil de una ruta cegada y reducir el riesgo de que un nodo intermedio actualice sus tarifas y rechace los pagos (lo que podría usarse para desenmascarar los nodos dentro de la ruta).

# Aceptar y Reenviar un Pago

Una vez que un nodo ha decodificado la carga útil, acepta el pago localmente o lo reenvía al par indicado como el siguiente salto en la carga útil.

## Reenvío No Estricto

Un nodo PUEDE reenviar un HTLC a lo largo de un canal saliente que no sea el especificado por `short_channel_id`, siempre que el receptor tenga la misma clave pública de nodo prevista por `short_channel_id`. Por lo tanto, si `short_channel_id` conecta los nodos A y B, el HTLC se puede reenviar a través de cualquier canal que conecte A y B.
Si no se adhiere, el receptor no podrá descifrar el siguiente salto en el paquete de cebolla.

<!-- omit in toc -->
### Racional

En el caso de que dos pares tengan múltiples canales, el nodo descendente podrá descifrar la carga útil del siguiente salto, independientemente del canal a través del cual se envíe el paquete.

Los nodos que implementan el reenvío no estricto pueden realizar evaluaciones en tiempo real de los anchos de banda del canal con un par en particular y utilizar el canal localmente óptimo.

Por ejemplo, si el canal especificado por `short_channel_id` que conecta A y B no tiene suficiente ancho de banda en el momento del reenvío, entonces A puede usar un canal diferente que sí lo tenga. Esto puede reducir la latencia de pago al evitar que el HTLC falle debido a restricciones de ancho de banda en `short_channel_id`, solo para que el remitente intente la misma ruta que difiere solo en el canal entre A y B.

El reenvío no estricto permite que los nodos utilicen canales privados que los conectan con el nodo receptor, incluso si el canal no se conoce en el gráfico de canales públicos.

<!-- omit in toc -->
### Recomendación

Las implementaciones que utilizan reenvío no estricto deben considerar aplicar el mismo programa de tarifas a todos los canales con el mismo par, ya que es probable que los remitentes seleccionen el canal que resulte en el costo total más bajo. Tener políticas distintas puede resultar en que el nodo de reenvío acepte tarifas basadas en el programa de tarifas más óptimo para el remitente, aunque proporcionen ancho de banda agregado en todos los canales con el mismo par.

Alternativamente, las implementaciones pueden optar por aplicar el reenvío no estricto solo a los canales de política similar para garantizar que sus ingresos por tarifas esperados no se desvíen al usar un canal alternativo.

## Carga útil para el último nodo

Al construir la ruta, el nodo de origen DEBE usar una carga útil para el nodo final con los siguientes valores:

* `payment_secret`: establecido en el secreto de pago especificado por el destinatario (por ejemplo,`payment_secret` de una factura de pago [BOLT #11](11-payment-encoding.md))
* `outgoing_cltv_value`: establecido en el vencimiento final especificado por el destinatario (por ejemplo, `min_final_cltv_expiry_delta` de una factura de pago [BOLT #11](11-payment-encoding.md))
* `amt_to_forward`: establecido en el importe final especificado por el destinatario (por ejemplo, `amount` de una factura de pago [BOLT #11](11-payment-encoding.md))

Esto permite que el nodo final verifique estos valores y devuelva errores si es necesario, pero también elimina la posibilidad de ataques de sondeo por parte del penúltimo nodo. Dichos ataques podrían, de lo contrario, intentar descubrir si el par receptor es el último al reenviar HTLC con diferentes cantidades/vencimientos.
El nodo final extraerá su carga útil de cebolla del HTLC que ha recibido y comparará sus valores con los del HTLC. Consulte la sección [Errores devueltos](#returning-errors) a continuación para obtener más detalles.

Si no fuera por lo anterior, dado que no necesita reenviar pagos, el nodo final podría simplemente descartar su carga útil.

# Secreto compartido

El nodo de origen establece un secreto compartido con cada salto a lo largo de la ruta utilizando Diffie-Hellman de curva elíptica entre la clave efímera del remitente en ese salto y la clave de identificación del nodo del salto. El punto de la curva resultante se serializa al
formato comprimido y hash usando `SHA256`. La salida hash se utiliza como secreto compartido de 32 bytes.

El Diffie-Hellman de curva elíptica (ECDH) es una operación en una clave privada EC y una clave pública EC que genera un punto de curva. Para este protocolo se utiliza la variante ECDH implementada en `libsecp256k1`, que se define sobre la curva elíptica `secp256k1`. Durante la construcción del paquete, el remitente usa la clave privada efímera y la clave pública del salto como entradas para ECDH, mientras que durante el reenvío de paquetes, el salto usa la clave pública efímera y su propia clave privada de ID de nodo. Debido a las propiedades de ECDH, ambos obtendrán el mismo valor.

# Claves efímeras cegadoras

Para garantizar que los múltiples saltos a lo largo de la ruta no puedan vincularse mediante las claves públicas efímeras que ven, la clave se oculta en cada salto. El ocultamiento se realiza de forma determinista que permite al remitente calcular las claves privadas ocultas correspondientes durante la construcción del paquete.

El cegamiento de una clave pública EC es una única multiplicación escalar del punto EC que representa la clave pública con un factor de cegamiento de 32 bytes. Debido a la propiedad conmutativa de la multiplicación escalar, la clave privada oculta es el producto multiplicativo de la clave privada correspondiente de la entrada con el mismo factor de ocultación.

El propio factor de cegamiento se calcula en función de la clave pública efímera y el secreto compartido de 32 bytes. Concretamente, es el valor hash `SHA256` de la concatenación de la clave pública serializada en su formato comprimido y el secreto compartido.

# Construcción de paquetes

En el siguiente ejemplo, se supone que un _nodo de envío_ (nodo de origen), `n_0`, quiere enrutar un paquete a un _nodo de recepción_ (nodo final), `n_r`.
Primero, el remitente calcula una ruta `{n_0, n_1, ..., n_{r-1}, n_r}`, donde `n_0` es el propio remitente y `n_r` es el destinatario final. Todos los nodos `n_i` y `n_{i+1}` DEBEN ser pares en la ruta de red superpuesta. Luego, el remitente recopila las claves públicas para `n_1` a `n_r` y genera una `sessionkey` aleatoria de 32 bytes.
Opcionalmente, el remitente puede pasar _datos asociados_, es decir, datos a los que se compromete el paquete pero que no están incluidos en el paquete en sí. Los datos asociados se incluirán en los HMAC y deben coincidir con los datos asociados proporcionados durante la verificación de integridad en cada salto.

Para construir la cebolla, el remitente inicializa la clave privada efímera para el primer salto `ek_1` a `sessionkey` y deriva de ella la clave pública efímera correspondiente `epk_1` multiplicándola por el punto base `secp256k1`. Para cada uno de los saltos `k` a lo largo de la ruta, el remitente calcula iterativamente el secreto compartido `ss_k` y la clave efímera para el siguiente salto `ek_{k+1}` de la siguiente manera:

 - El remitente ejecuta ECDH con la clave pública del salto y la clave privada efímera para obtener un punto de curva, que se codifica con `SHA256` para producir el secreto compartido `ss_k`.
 - El factor de cegamiento es el hash `SHA256` de la concatenación entre la clave pública efímera `epk_k` y el secreto compartido `ss_k`.
 - La clave privada efímera para el siguiente salto `ek_{k+1}` se calcula multiplicando la clave privada efímera actual `ek_k` por el factor de cegamiento.
 - La clave pública efímera para el siguiente salto `epk_{k+1}` se deriva de la clave privada efímera `ek_{k+1}` multiplicando por el punto base.

Una vez que el remitente tiene toda la información requerida arriba, puede construir el paquete. La construcción de un paquete enrutado sobre saltos `r` requiere claves públicas efímeras `r` de 32 bytes, secretos compartidos `r` de 32 bytes, factores de cegamiento `r` de 32 bytes y cargas útiles `hop_payload` de longitud variable `r`.
La construcción devuelve un solo paquete de 1366 bytes junto con la dirección del primer par receptor.

La construcción del paquete se realiza en el orden inverso de la ruta, es decir, las operaciones del último salto se aplican primero.

El paquete se inicializa con 1300 _random_ bytes derivados de un CSPRNG (ChaCha20). La tecla _pad_ a la que se hace referencia anteriormente se usa para extraer bytes aleatorios adicionales de un flujo de ChaCha20, usándolo como un CSPRNG para este propósito. Una vez que se ha obtenido la `paddingKey`, se usa ChaCha20 con un nonce todo cero, para generar 1300 bytes aleatorios. Esos bytes aleatorios luego se usan como el estado inicial del encabezado de mezcla que se creará.

Se genera un relleno (ver [Generación de relleno](#filler-generation)) utilizando el secreto compartido.

Para cada salto en la ruta, en orden inverso, el remitente aplica las siguientes operaciones:

 - La clave _rho_ y la clave _mu_ se generan utilizando el secreto compartido del salto.
 - `shift_size` se define como la longitud de `hop_payload` más la codificación de tamaño grande de la longitud y la longitud de ese HMAC. Por lo tanto, si la longitud de la carga útil es `l`, `shift_size` es `1 + l + 32` para `l < 253`; de lo contrario, `3 + l + 32` debido a la codificación de tamaño grande de `l`.
 - El campo `hop_payload` se desplaza a la derecha en bytes `shift_size`, descartando los últimos bytes `shift_size` que superan su tamaño de 1300 bytes.
 - La longitud serializada de tamaño grande, `hop_payload` y `hmac` serializados se copian en los siguientes bytes `shift_size`.
 - La clave _rho_ se usa para generar 1300 bytes de flujo de bytes pseudoaleatorios que luego se aplica, con `XOR`, al campo `hop_payloads`.
 - Si este es el último salto, es decir, la primera iteración, la parte final del campo `hop_payloads` se sobrescribe con la información de enrutamiento `filler`.
 - El siguiente HMAC se calcula (con la clave _mu_ como clave HMAC) sobre las `hop_payloads` concatenadas y los datos asociados.

El valor HMAC final resultante es el HMAC que utilizará el primer par receptor en la ruta.

La generación de paquetes devuelve un paquete serializado que contiene el byte de 'versión', la clave pública efímera para el primer salto, el HMAC para el primer salto y los 'hop_payloads' ofuscados.

El siguiente código Go es un ejemplo de implementación de la construcción del paquete:

```Go
func NewOnionPacket(paymentPath []*btcec.PublicKey, sessionKey *btcec.PrivateKey,
	hopsData []HopData, assocData []byte) (*OnionPacket, error) {

	numHops := len(paymentPath)
	hopSharedSecrets := make([][sha256.Size]byte, numHops)

	// Inicialice la clave efímera para el primer salto a la clave de sesión.
	var ephemeralKey big.Int
	ephemeralKey.Set(sessionKey.D)

	for i := 0; i < numHops; i++ {
		// Realice ECDH y haga un hash del resultado.
		ecdhResult := scalarMult(paymentPath[i], ephemeralKey)
		hopSharedSecrets[i] = sha256.Sum256(ecdhResult.SerializeCompressed())

		// Derivar clave pública efímera de clave privada.
		ephemeralPrivKey := btcec.PrivKeyFromBytes(btcec.S256(), ephemeralKey.Bytes())
		ephemeralPubKey := ephemeralPrivKey.PubKey()

		// Calcular el factor de cegamiento.
		sha := sha256.New()
		sha.Write(ephemeralPubKey.SerializeCompressed())
		sha.Write(hopSharedSecrets[i])

		var blindingFactor big.Int
		blindingFactor.SetBytes(sha.Sum(nil))

		// Clave ciega efímera para el próximo salto.
		ephemeralKey.Mul(&ephemeralKey, &blindingFactor)
		ephemeralKey.Mod(&ephemeralKey, btcec.S256().Params().N)
	}

	// Genere el relleno, llamado 'cadenas de relleno' en el papel.
	filler := generateHeaderPadding("rho", numHops, hopDataSize, hopSharedSecrets)

	// Asignar e inicializar campos a sectores llenos de ceros
	var mixHeader [routingInfoSize]byte
	var nextHmac [hmacSize]byte
        
        // Nuestro paquete inicial debe completarse con bytes aleatorios,
        // generar algunos de forma determinista utilizando la clave privada de la sesión.
        paddingKey := generateKey("pad", sessionKey.Serialize()
        paddingBytes := generateCipherStream(paddingKey, routingInfoSize)
        copy(mixHeader[:], paddingBytes)

  // Calcular la información de enrutamiento para cada salto junto con un
  // MAC de la información de enrutamiento usando la clave compartida para ese salto.
	for i := numHops - 1; i >= 0; i-- {
		rhoKey := generateKey("rho", hopSharedSecrets[i])
		muKey := generateKey("mu", hopSharedSecrets[i])

		hopsData[i].HMAC = nextHmac

		// Cambiar y ofuscar la información de enrutamiento
		streamBytes := generateCipherStream(rhoKey, numStreamBytes)

		rightShift(mixHeader[:], hopDataSize)
		buf := &bytes.Buffer{}
		hopsData[i].Encode(buf)
		copy(mixHeader[:], buf.Bytes())
		xor(mixHeader[:], mixHeader[:], streamBytes[:routingInfoSize])

		// Estos deben sobrescribirse, por lo que cada nodo genera un relleno correcto
		if i == numHops-1 {
			copy(mixHeader[len(mixHeader)-len(filler):], filler)
		}

		packet := append(mixHeader[:], assocData...)
		nextHmac = calcMac(muKey, packet)
	}

	packet := &OnionPacket{
		Version:      0x00,
		EphemeralKey: sessionKey.PubKey(),
		RoutingInfo:  mixHeader,
		HeaderMAC:    nextHmac,
	}
	return packet, nil
}
```

# Reenvío de paquetes

Esta especificación está limitada a paquetes `versión` `0`; la estructura de versiones futuras puede cambiar.

Al recibir un paquete, un nodo de procesamiento compara el byte de la versión del paquete con sus propias versiones admitidas y cancela la conexión si el paquete especifica un número de versión que no admite.
Para paquetes con números de versión admitidos, el nodo de procesamiento primero analiza el paquete en sus campos individuales.

A continuación, el nodo de procesamiento calcula el secreto compartido utilizando la clave privada correspondiente a su propia clave pública y la clave efímera del paquete, como se describe en [Secreto compartido](#secreto-compartido).

Los requisitos anteriores evitan que cualquier salto a lo largo de la ruta vuelva a intentar un pago varias veces, en un intento de rastrear el progreso de un pago a través del análisis de tráfico. Tenga en cuenta que la desactivación de dicho sondeo podría lograrse utilizando un registro de secretos compartidos anteriores o HMAC, que podrían olvidarse una vez que el HTLC no se aceptara de todos modos (es decir, después de que haya pasado `outgoing_cltv_value`). Tal registro puede usar una estructura de datos probabilísticos, pero DEBE limitar la tasa de compromisos como
necesario, para restringir los requisitos de almacenamiento en el peor de los casos o los falsos positivos de este registro.

A continuación, el nodo de procesamiento utiliza el secreto compartido para calcular una clave _mu_, que a su vez utiliza para calcular el HMAC de los `hop_payloads`. Luego, el HMAC resultante se compara con el HMAC del paquete.

La comparación del HMAC calculado y el HMAC del paquete DEBE ser constante en el tiempo para evitar fugas de información.

En este punto, el nodo de procesamiento puede generar una clave _rho_.

Luego, la información de enrutamiento se desofusca y se extrae la información sobre el siguiente salto.
Para hacerlo, el nodo de procesamiento copia el campo `hop_payloads`, agrega 1300 bytes `0x00`, genera `2*1300` bytes pseudoaleatorios (usando la tecla _rho_) y aplica el resultado usando `XOR`, a la copia de `hop_payloads`.
Los primeros bytes corresponden a la longitud codificada en tamaño grande `l` de `hop_payload`, seguidos de `l` bytes de la información de enrutamiento resultante que se convierte en `hop_payload` y el HMAC de 32 bytes.
Los siguientes 1300 bytes son `hop_payloads` para el paquete saliente.

Un valor `hmac` especial de 32 bytes `0x00` indica que el salto de procesamiento actual es el destinatario previsto y que el paquete no debe reenviarse.

Si el HMAC no indica la terminación de la ruta y si el siguiente salto es un par del nodo de procesamiento; luego se ensambla el nuevo paquete. El ensamblaje del paquete se logra ocultando la clave efímera con la clave pública del nodo de procesamiento, junto con el secreto compartido, y serializando los `hop_payloads`.
Luego, el paquete resultante se reenvía al par direccionado.

<!-- omit in toc -->
## Requisitos

El nodo de procesamiento:
  - si la clave pública efímera NO está en la curva `secp256k1`:
    - DEBE abortar el procesamiento del paquete.
    - DEBE informar una falla de ruta al nodo de origen.
  - si el paquete ha sido previamente reenviado o redimido localmente, es decir, el
  El paquete contiene información de enrutamiento duplicada a un paquete recibido previamente:
    - si se conoce la preimagen:
      - PUEDE canjear inmediatamente el HTLC usando la preimagen.
    - de lo contrario:
      - DEBE abortar el procesamiento y reportar una falla en la ruta.
  - si el HMAC calculado y el HMAC del paquete difieren:
    - DEBE abortar el procesamiento.
    - DEBE informar una falla en la ruta.
  - si el `realm` es desconocido:
    - DEBE soltar el paquete.
    - DEBE señalizar un fallo de ruta.
  - DEBE direccionar el paquete a otro par que sea su vecino directo.
  - si el nodo de procesamiento no tiene un par con la dirección coincidente:
    - DEBE soltar el paquete.
    - DEBE señalizar un fallo de ruta.


# Generación del relleno

Al recibir un paquete, el nodo de procesamiento extrae la información destinada a él de la información de la ruta y la carga útil por salto.
La extracción se realiza desofuscando y desplazando el campo a la izquierda.
Esto haría que el campo fuera más corto en cada salto, lo que permitiría a un atacante deducir la longitud de la ruta. Por este motivo, el campo se rellena previamente antes de reenviarlo.
Dado que el relleno es parte del HMAC, el nodo de origen tendrá que generar previamente un relleno idéntico (al que generará cada salto) para calcular los HMAC correctamente para cada salto.
El relleno también se usa para rellenar la longitud del campo, en el caso de que la ruta seleccionada sea más corta que 1300 bytes.

Antes de desofuscar los `hop_payloads`, el nodo de procesamiento lo rellena con 1300 bytes `0x00`, de modo que la longitud total es `2*1300`.
Luego genera el flujo de bytes pseudoaleatorios, de longitud coincidente, y lo aplica con `XOR` a `hop_payloads`.
Esto desofusca la información destinada a él, mientras que simultáneamente ofusca los bytes `0x00` agregados al final.

Para calcular el HMAC correcto, el nodo de origen tiene que generar previamente las `hop_payloads` para cada salto, incluido el relleno ofuscado incremental agregado por cada salto. Este relleno incrementalmente ofuscado se conoce como `filler`.

El siguiente código de ejemplo muestra cómo se genera el relleno en Go:

```Go
func generateFiller(key string, numHops int, hopSize int, sharedSecrets [][sharedSecretSize]byte) []byte {
	fillerSize := uint((numMaxHops + 1) * hopSize)
	filler := make([]byte, fillerSize)

	// El último salto no ofusca, ya no está reenviando.
	for i := 0; i < numHops-1; i++ {

		// Desplazar a la izquierda el campo
		copy(filler[:], filler[hopSize:])

		// Llenar con cero el último salto
		copy(filler[len(filler)-hopSize:], bytes.Repeat([]byte{0x00}, hopSize))

		// Generate pseudo-random byte stream
		streamKey := generateKey(key, sharedSecrets[i])
		streamBytes := generateCipherStream(streamKey, fillerSize)

		// Ofuscar
		xor(filler, filler, streamBytes)
	}

  // Cortar el relleno a la longitud correcta (numHops+1)*hopSize
  // bytes serán precedidos por la generación del paquete.
	return filler[(numMaxHops-numHops+2)*hopSize:]
}
```

Tenga en cuenta que esta implementación de ejemplo es solo para fines de demostración; el `filler` se puede generar de manera mucho más eficiente.
El último salto no necesita ofuscar el `filler`, ya que no reenviará más el paquete y, por lo tanto, tampoco necesitará extraer un HMAC.

# Devolviendo errores

El protocolo de enrutamiento de cebolla incluye un mecanismo simple para devolver mensajes de error cifrados al nodo de origen.
Los mensajes de error devueltos pueden ser fallos informados por cualquier salto, incluido el nodo final.
El formato del paquete de reenvío no se puede utilizar para la ruta de retorno, ya que ningún salto además del origen tiene acceso a la información requerida para su generación.
Tenga en cuenta que estos mensajes de error no son confiables, ya que no se colocan en la cadena debido a la posibilidad de fallo del salto.

Los saltos intermedios almacenan el secreto compartido de la ruta de reenvío y lo reutilizan para ofuscar cualquier paquete de retorno correspondiente durante cada salto.
Además, cada nodo almacena localmente datos sobre su propio par de envío en la ruta, por lo que sabe adónde devolver-reenviar cualquier paquete de retorno eventual.
El nodo que genera el mensaje de error (_erring node_) crea un paquete de retorno que consta de los siguientes campos:

1. datos:
   * [`32*byte`:`hmac`]
   * [`u16`:`fracaso_len`]
   * [`failure_len*byte`:`failuremsg`]
   * [`u16`:`pad_len`]
   * [`pad_len*byte`:`pad`]

Donde `hmac` es un HMAC (`Hashed Message Authentication Code`) que autentica el resto del paquete, con una clave generada utilizando el proceso anterior, con el tipo de clave `um`, `failuremsg` como se define a continuación, y `pad` como los bytes adicionales utilizados para ocultar la longitud.

El nodo erróneo luego genera una nueva clave, usando el tipo de clave `ammag`.
Esta clave luego se usa para generar un flujo pseudoaleatorio, que a su vez se aplica al paquete usando `XOR`.

El paso de ofuscación se repite en cada salto a lo largo de la ruta de retorno.
Al recibir un paquete de retorno, cada salto genera su `ammag`, genera el flujo de bytes pseudoaleatorios y aplica el resultado al paquete de retorno antes de devolverlo y reenviarlo.

El nodo de origen puede detectar que es el destinatario final previsto del mensaje de retorno porque, por supuesto, fue el originador del paquete de reenvío correspondiente.
Cuando un nodo de origen recibe un mensaje de error que coincide con una transferencia que inició (es decir, no puede volver a enviar el error más), genera las claves `ammag` y `um` para cada salto en la ruta.
Luego descifra iterativamente el mensaje de error, usando la clave `ammag` de cada salto, y calcula el HMAC, usando la clave `um` de cada salto.
El nodo de origen puede detectar al remitente del mensaje de error haciendo coincidir el campo `hmac` con el HMAC calculado.

La asociación entre los paquetes de reenvío y retorno se maneja fuera de este protocolo de enrutamiento de cebolla, p.e. mediante asociación con un HTLC en un canal de pago.

El manejo de errores para los HTLC con `blinding_point` es particularmente tenso, ya que las diferencias en las implementaciones (o versiones) pueden aprovecharse para eliminar el anonimato de los elementos de la ruta ciega. Por lo tanto, la decisión convierte cada error en `invalid_onion_blinding` que se convertirá en un error de cebolla normal en el punto de introducción.

<!-- omit in toc -->
### Requisitos

El _nodo erróneo_:
  - DEBE configurar `pad` de modo que `failure_len` más `pad_len` sea al menos 256.
  - DEBERÍA configurar `pad` de modo que `failure_len` más `pad_len` sea igual a 256. Si se desvía de esto, es posible que los nodos más antiguos no puedan analizar el mensaje de retorno.

El _nodo de origen_:
  - una vez descifrado el mensaje de respuesta:
    - DEBE almacenar una copia del mensaje.
    - DEBE continuar descifrando, hasta que el ciclo se haya repetido 20 veces.
    - DEBE usar las claves constantes `ammag` y `um` para ofuscar la longitud de la ruta.

## Mensajes de error

El mensaje de error encapsulado en `failuremsg` tiene un formato idéntico al de un mensaje normal: un tipo `failure_code` de 2 bytes seguido de los datos aplicables a ese tipo. Los datos del mensaje van seguidos de un [flujo TLV](01-messaging.md#formato-tipo-longitud-valor) opcional.

A continuación se muestra una lista de los valores de `failure_code` admitidos actualmente, seguidos de los requisitos de su caso de uso.

Tenga en cuenta que los `failure_code`s no son del mismo tipo que otros tipos de mensajes, definidos en otros BOLT, ya que no se envían directamente a la capa de transporte, sino que se envuelven dentro de los paquetes de retorno.
Los valores numéricos para `failure_code` pueden por lo tanto reutilizar valores, que también están asignados a otros tipos de mensajes, sin peligro de causar colisiones.

El byte superior de `failure_code` se puede leer como un conjunto de banderas:
* 0x8000 (BADONION): cebolla no analizable cifrada por el envío del par
* 0x4000 (PERM): fallo permanente (de lo contrario transitorio)
* 0x2000 (NODE): fallo de nodo (de lo contrario, canal)
* 0x1000 (ACTUALIZAR): nueva actualización de canal adjunta

Tenga en cuenta que el campo `channel_update` es obligatorio en los mensajes cuyo `failure_code` incluye el indicador `UPDATE`. Está codificado *con* el prefijo del tipo de mensaje, es decir, siempre debe comenzar con `0x0102`. Tenga en cuenta que las implementaciones históricas de Lightning serializaron esto sin el tipo de mensaje `0x0102`.

Se definen los siguientes `failure_code`s:

1. type: PERM|1 (`invalid_realm`)

El byte `realm` no fue entendido por el nodo de procesamiento.

1. type: NODE|2 (`temporary_node_failure`)

Fallo temporal general del nodo de procesamiento.

1. type: PERM|NODE|2 (`permanent_node_failure`)

Fallo general permanente del nodo de procesamiento.

1. type: PERM|NODE|3 (`required_node_feature_missing`)

El nodo de procesamiento tiene una característica requerida que no estaba en esta cebolla.

1. type: BADONION|PERM|4 (`invalid_onion_version`)
2. data:
   * [`sha256`:`sha256_of_onion`]

El nodo de procesamiento no entendió el byte `version`.

1. type: BADONION|PERM|5 (`invalid_onion_hmac`)
2. data:
   * [`sha256`:`sha256_of_onion`]

El HMAC de la cebolla era incorrecto cuando llegó al nodo de procesamiento.

1. type: BADONION|PERM|6 (`invalid_onion_key`)
2. data:
   * [`sha256`:`sha256_of_onion`]

El nodo de procesamiento no pudo analizar la clave efímera.

1. type: UPDATE|7 (`temporary_channel_failure`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

El canal del nodo de procesamiento no pudo manejar este HTLC, pero es posible que pueda manejarlo, u otros, más adelante.

1. type: PERM|8 (`permanent_channel_failure`)

El canal del nodo de procesamiento no puede manejar ningún HTLC.

1. type: PERM|9 (`required_channel_feature_missing`)

El canal del nodo de procesamiento requiere características que no están presentes en la cebolla.

1. type: PERM|10 (`unknown_next_peer`)

La cebolla especificó un `short_channel_id` que no coincide con ningún encabezado del nodo de procesamiento.

1. type: UPDATE|11 (`amount_below_minimum`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

La cantidad de HTLC estaba por debajo del `htlc_minimum_msat` del canal del nodo de procesamiento.

1. type: UPDATE|12 (`fee_insufficient`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

El monto de la tarifa estaba por debajo del requerido por el canal del nodo de procesamiento.

1. type: UPDATE|13 (`incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

El `cltv_expiry` no cumple con el `cltv_expiry_delta` requerido por el canal desde el nodo de procesamiento: no cumple con el siguiente requisito:

        cltv_expiry - cltv_expiry_delta >= outgoing_cltv_value

1. type: UPDATE|14 (`expiry_too_soon`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

El vencimiento de CLTV está demasiado cerca de la altura del bloque actual para que el nodo de procesamiento lo manipule de manera segura.

1. type: PERM|15 (`incorrect_or_unknown_payment_details`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u32`:`height`]

El `payment_hash` es desconocido para el nodo final, el `payment_secret` no coincide con el `payment_hash`, la cantidad para ese `payment_hash` es incorrecta, la caducidad CLTV del htlc está demasiado cerca de la altura del bloque actual por seguridad el manejo o `payment_metadata` no está presente cuando debería estarlo.

El parámetro `htlc_msat` es superfluo, pero se dejó por compatibilidad con versiones anteriores. El valor de `htlc_msat` siempre coincide con la cantidad especificada en la carga útil de cebolla de salto final. Por tanto, no tiene ningún valor informativo para el remitente. Un penúltimo salto que envía una cantidad o vencimiento diferente para el htlc se maneja a través de `final_incorrect_cltv_expiry` y `final_incorrect_htlc_amount`.

El parámetro `height` lo establece el nodo final en la altura de bloque más conocida en el momento de recibir el htlc. El remitente puede usar esto para distinguir entre enviar un pago con el vencimiento final de CLTV incorrecto y un salto intermedio que retrasa el pago para que ya no se cumpla el requisito de delta de CLTV de la factura del receptor.

Nota: Originalmente, PERM|16 (`incorrect_payment_amount`) y 17 (`final_expiry_too_soon`) se usaban para diferenciar los parámetros htlc incorrectos del hash de pago desconocido. Lamentablemente, el envío de esta respuesta permite realizar ataques de sondeo en los que un nodo que recibe un HTLC para reenviar puede verificar las conjeturas sobre su destino final mediante el envío de pagos con el mismo hash pero valores mucho más bajos o alturas de vencimiento a destinos potenciales y verificar la respuesta.

Las implementaciones deben tener cuidado para diferenciar el caso anteriormente no permanente para `final_expiry_too_soon` (17) de las otras fallas permanentes ahora representadas por `incorrect_or_unknown_payment_details` (PERM|15).

1. type: 18 (`final_incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]

El vencimiento de CLTV en el HTLC no coincide con el valor en la cebolla.

1. type: 19 (`final_incorrect_htlc_amount`)
2. data:
   * [`u64`:`incoming_htlc_amt`]

La cantidad en el HTLC no coincide con el valor en la cebolla.

1. type: UPDATE|20 (`channel_disabled`)
2. data:
   * [`u16`:`disabled_flags`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

El canal del nodo de procesamiento ha sido deshabilitado. No hay banderas para `disabled_flags` actualmente definidas, por lo que actualmente siempre son dos cero bytes.

1. type: 21 (`expiry_too_far`)

El vencimiento de CLTV en el HTLC está demasiado lejos en el futuro.

1. type: PERM|22 (`invalid_onion_payload`)
2. data:
   * [`bigsize`:`type`]
   * [`u16`:`offset`]

El nodo de procesamiento no entendió la carga útil de cebolla por salto descifrada o está incompleta. Si la falla se puede reducir a un tipo de tlv específico en la carga útil, el nodo erróneo puede incluir ese 'tipo' y su 'compensación' de bytes en el flujo de bytes descifrado.

1. type: 23 (`mpp_timeout`)

El monto total del pago de varias partes no se recibió dentro de un tiempo razonable.

1. type: BADONION|PERM|24 (`invalid_onion_blinding`)
2. data:
   * [`sha256`:`sha256_of_onion`]

Ocurrió un error dentro de la ruta ciega.

<!-- omit in toc -->
### Requisitos

Un _nodo erróneo_:
  - si se establece `blinding_point` en `update_add_htlc` entrante:
    - DEBE devolver un error `invalid_onion_blinding`.
  - si se establece `current_blinding_point` en la carga útil de cebolla y no es el nodo final:
    - DEBE devolver un error `invalid_onion_blinding`.
  - de lo contrario:
    - DEBE seleccionar uno de los códigos de error anteriores al crear un mensaje de error.
    - DEBE incluir los datos apropiados para ese tipo de error en particular.
    - si hay más de un error:
      - DEBERÍA seleccionar el primer error que encuentre de la lista anterior.

Un _nodo erróneo_ PUEDE:
  - si el byte `realm` es desconocido:
    - devuelve un error `invalid_realm`.
  - si la carga útil por salto en la cebolla no es válida (por ejemplo, no es una transmisión tlv válida) o falta la información requerida (por ejemplo, no se especificó la cantidad):
    - devuelve un error `invalid_onion_payload`.
  - si se produce un error transitorio no especificado en todo el nodo:
    - devuelve un error `temporal_node_failure`.
  - si se produce un error permanente no especificado en todo el nodo:
    - devuelve un error `permanent_node_failure`.
  - si un nodo tiene requisitos anunciados en sus `node_announcement` `features`, que NO estaban incluidos en la cebolla:
    - devuelve un error `required_node_feature_missing`.

Un _nodo de reenvío_ DEBE:
  - si se establece `blinding_point` en `update_add_htlc` entrante:
    - devuelve un error `invalid_onion_blinding`.
  - si se establece `current_blinding_point` en la carga útil de cebolla y no es el nodo final:
    - devuelve un error `invalid_onion_blinding`.
  - de lo contrario:
    - seleccione uno de los códigos de error anteriores al crear un mensaje de error.

Un _nodo de reenvío_ PUEDE, pero un _nodo final_ NO DEBE:
  - si se desconoce el byte de `versión` de cebolla:
    - devuelve un error `invalid_onion_version`.
  - si la cebolla HMAC es incorrecta:
    - devuelve un error `invalid_onion_hmac`.
  - si la clave efímera en la cebolla no se puede analizar:
    - devuelve un error `invalid_onion_key`.
  - si durante el reenvío a su par receptor, se produce un error transitorio no especificado en el canal de salida (por ejemplo, capacidad del canal alcanzada, demasiados HTLC en vuelo, etc.):
    - devuelve un error `temporal_channel_failure`.
  - si se produce un error permanente no especificado durante el reenvío a su par receptor (por ejemplo, canal cerrado recientemente):
    - devuelve un error `permanent_channel_failure`.
  - si el canal saliente tiene requisitos anunciados en las `características` de `channel_announcement`, que NO se incluyeron en la cebolla:
    - devuelve un error `required_channel_feature_missing`.
  - si NO se conoce el par receptor especificado por la cebolla:
    - devuelve un error `unknown_next_peer`.
  - si la cantidad de HTLC es inferior a la cantidad mínima especificada actualmente:
    - informar la cantidad del HTLC saliente y la configuración del canal actual para el canal saliente.
    - devuelve un error `amount_below_minimum`.
  - si el HTLC NO paga una tarifa suficiente:
    - informar la cantidad del HTLC entrante y la configuración del canal actual para el canal saliente.
    - devuelve un error `fee_insufficient`.
 - si el `cltv_expiry` entrante menos el `outgoing_cltv_value` está por debajo del `cltv_expiry_delta` para el canal saliente:
    - informar el `cltv_expiry` del HTLC saliente y la configuración del canal actual para el canal saliente.
    - devuelve un error `incorrect_cltv_expiry`.
  - si `cltv_expiry` está injustificadamente cerca del presente:
    - informar la configuración del canal actual para el canal saliente.
    - devuelve un error `expiry_too_soon`.
  - si `cltv_expiry` está irrazonablemente lejos en el futuro:
    - devuelve un error `expiry_too_far`.
  - si el canal está deshabilitado:
    - informar la configuración del canal actual para el canal saliente.
    - devuelve un error `channel_disabled`.

Un _salto intermedio_ NO DEBE, pero el _nodo final_:
  - si el `payment_hash` ya ha sido pagado:
    - PUEDE tratar el `payment_hash` como desconocido.
    - PUEDE lograr aceptar el HTLC.
  - si el `payment_secret` no coincide con el valor esperado para ese `payment_hash`, o si se requiere `payment_secret` y no está presente:
    - DEBE fallar el HTLC.
    - DEBE devolver un error `incorrect_or_unknown_payment_details`.
  - si la cantidad pagada es inferior a la cantidad esperada:
    - DEBE fallar el HTLC.
    - DEBE devolver un error `incorrect_or_unknown_payment_details`.
  - si se desconoce el hash de pago:
    - DEBE fallar el HTLC.
    - DEBE devolver un error `incorrect_or_unknown_payment_details`.
  - si la cantidad pagada es más del doble de la cantidad esperada:
    - DEBE fallar el HTLC.
    - DEBERÍA devolver un error `incorrect_or_unknown_payment_details`.
      - Nota: esto permite que el nodo de origen reduzca la fuga de información alterando el monto sin permitir el sobrepago bruto accidental.
  - si el valor de `cltv_expiry` está injustificadamente cerca del presente:
    - DEBE fallar el HTLC.
    - DEBE devolver un error `incorrect_or_unknown_payment_details`.
- si `cltv_expiry` del HTLC del nodo final está por debajo de `outgoing_cltv_value`:
    - DEBE devolver el error `final_incorrect_cltv_expiry`.
  - si `amount_msat` del HTLC del nodo final está por debajo de `amt_to_forward`:
    - DEBE devolver un error `final_incorrect_htlc_amount`.
  - si devuelve un `channel_update`:
    - DEBE establecer `short_channel_id` en `short_channel_id` utilizado por la cebolla entrante.

<!-- omit in toc -->
### Racional

En el caso de múltiples alias de `short_channel_id`, `channel_update` `short_channel_id` debe referirse al que espera el remitente original, tanto para evitar confusiones como para evitar filtraciones de información sobre otros alias (o la ubicación real del canal UTXO).

## Recibir códigos de error

<!-- omit in toc -->
### Requisitos

El _nodo de origen_:
  - DEBE ignorar cualquier byte extra en `failuremsg`.
  - si el _nodo final_ devuelve el error:
    - si el bit PERM está activado:
      - DEBE fallar el pago.
    - de lo contrario:
      - si el código de error se entiende y es válido:
        - PUEDE volver a intentar el pago. En particular, `final_expiry_too_soon` puede ocurrir si la altura del bloque ha cambiado desde el envío y, en este caso, `temporal_node_failure` podría resolverse en unos pocos segundos.
  - de lo contrario, un _salto intermedio_ está devolviendo el error:
    - si el bit NODO está establecido:
      - DEBE eliminar todos los canales conectados con el nodo erróneo de la consideración.
    - si el bit PERM NO está activado:
      - DEBERÍA restaurar los canales a medida que recibe nuevos `channel_update`s.
    - de lo contrario:
      - si se establece ACTUALIZAR, Y `channel_update` es válido y más reciente que `channel_update` utilizado para enviar el pago:
        - si `channel_update` NO debería haber causado la falla:
          - PUEDE tratar `channel_update` como no válido.
        - de lo contrario:
          - DEBERÍA aplicar el `channel_update`.
        - PUEDE poner en cola `channel_update` para su transmisión.
      - de lo contrario:
        - DEBERÍA eliminar de consideración el canal que sale del nodo erróneo.
        - si el bit PERM NO está activado:
          - DEBERÍA restaurar el canal a medida que recibe nuevas `channel_update`s.
    - DEBERÍA volver a intentar enrutar y enviar el pago.
  - PUEDE utilizar los datos especificados en los distintos tipos de fallos con fines de depuración.
# Test Vector

## Returning Errors

Los vectores de prueba utilizan los siguientes parámetros:

	pubkey[0] = 0x02eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619
	pubkey[1] = 0x0324653eac434488002cc06bbfb7f10fe18991e35f9fe4302dbea6d2353dc0ab1c
	pubkey[2] = 0x027f31ebc5462c1fdce1b737ecff52d37d75dea43ce11c74d25aa297165faa2007
	pubkey[3] = 0x032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
	pubkey[4] = 0x02edabbd16b41c8371b92ef2f04c1185b4f03b6dcd52ba9b78d9d7c89c8f221145

	nhops = 5
	sessionkey = 0x4141414141414141414141414141414141414141414141414141414141414141

	failure_source  = node 4
	failure_message = `incorrect_or_unknown_payment_details`
      htlc_msat = 100
      height    = 800000
      tlv data
        type  = 34001
        value = [128, 128, ..., 128] (300 bytes)

El siguiente es un seguimiento detallado de un ejemplo de creación de mensaje de error:

	# creando el mensaje de error
	encoded_failure_message = 400f0000000000000064000c3500fd84d1fd012c80808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808002c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
	payload = 0140400f0000000000000064000c3500fd84d1fd012c80808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808002c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	um_key = 4da7f2923edce6c2d85987d1d9fa6d88023e6c3a9c3d20f07d3b10b61a78d646
	raw_error_packet = fda7e11974f78ca6cc456f2d17ae54463664696e93842548245dd2a2c513a6260140400f0000000000000064000c3500fd84d1fd012c80808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808080808002c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	# forwarding error packet
	shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
	ammag_key = 2f36bb8822e1f0d04c27b7d8bb7d7dd586e032a3218b8d414afbba6f169a4d68
	stream = e9c975b07c9a374ba64fd9be3aae955e917d34d1fa33f2e90f53bbf4394713c6a8c9b16ab5f12fd45edd73c1b0c8b33002df376801ff58aaa94000bf8a86f92620f343baef38a580102395ae3abf9128d1047a0736ff9b83d456740ebbb4aeb3aa9737f18fb4afb4aa074fb26c4d702f42968888550a3bded8c05247e045b866baef0499f079fdaeef6538f31d44deafffdfd3afa2fb4ca9082b8f1c465371a9894dd8c243fb4847e004f5256b3e90e2edde4c9fb3082ddfe4d1e734cacd96ef0706bf63c9984e22dc98851bcccd1c3494351feb458c9c6af41c0044bea3c47552b1d992ae542b17a2d0bba1a096c78d169034ecb55b6e3a7263c26017f033031228833c1daefc0dedb8cf7c3e37c9c37ebfe42f3225c326e8bcfd338804c145b16e34e4f5984bc119af09d471a61f39e9e389c4120cadabc5d9b7b1355a8ccef050ca8ad72f642fc26919927b347808bade4b1c321b08bc363f20745ba2f97f0ced2996a232f55ba28fe7dfa70a9ab0433a085388f25cce8d53de6a2fbd7546377d6ede9027ad173ba1f95767461a3689ef405ab608a21086165c64b02c1782b04a6dba2361a7784603069124e12f2f6dcb1ec7612a4fbf94c0e14631a2bef6190c3d5f35e0c4b32aa85201f449d830fd8f782ec758b0910428e3ec3ca1dba3b6c7d89f69e1ee1b9df3dfbbf6d361e1463886b38d52e8f43b73a3bd48c6f36f5897f514b93364a31d49d1d506340b1315883d425cb36f4ea553430d538fd6f3596d4afc518db2f317dd051abc0d4bfb0a7870c3db70f19fe78d6604bbf088fcb4613f54e67b038277fedcd9680eb97bdffc3be1ab2cbcbafd625b8a7ac34d8c190f98d3064ecd3b95b8895157c6a37f31ef4de094b2cb9dbf8ff1f419ba0ecacb1bb13df0253b826bec2ccca1e745dd3b3e7cc6277ce284d649e7b8285727735ff4ef6cca6c18e2714f4e2a1ac67b25213d3bb49763b3b94e7ebf72507b71fb2fe0329666477ee7cb7ebd6b88ad5add8b217188b1ca0fa13de1ec09cc674346875105be6e0e0d6c8928eb0df23c39a639e04e4aedf535c4e093f08b2c905a14f25c0c0fe47a5a1535ab9eae0d9d67bdd79de13a08d59ee05385c7ea4af1ad3248e61dd22f8990e9e99897d653dd7b1b1433a6d464ea9f74e377f2d8ce99ba7dbc753297644234d25ecb5bd528e2e2082824681299ac30c05354baaa9c3967d86d7c07736f87fc0f63e5036d47235d7ae12178ced3ae36ee5919c093a02579e4fc9edad2c446c656c790704bfc8e2c491a42500aa1d75c8d4921ce29b753f883e17c79b09ea324f1f32ddf1f3284cd70e847b09d90f6718c42e5c94484cc9cbb0df659d255630a3f5a27e7d5dd14fa6b974d1719aa98f01a20fb4b7b1c77b42d57fab3c724339d459ee4a1c6b5d3bd4e08624c786a257872acc9ad3ff62222f2265a658d9f2a007229a5293b67ec91c84c4b4407c228434bad8a815ca9b256c776bd2c9f
	error packet for node 4: 146e94a9086dbbed6a0ab6932d00c118a7195dbf69b7d7a12b0e6956fc54b5e0a989f165b5f12fd45edd73a5b0c48630ff5be69500d3d82a29c0803f0a0679a6a073c33a6fb8250090a3152eba3f11a85184fa87b67f1b0354d6f48e3b342e332a17b7710f342f342a87cf32eccdf0afc2160808d58abb5e5840d2c760c538e63a6f841970f97d2e6fe5b8739dc45e2f7f5f532f227bcc2988ab0f9cc6d3f12909cd5842c37bc8c7608475a5ebbe10626d5ecc1f3388ad5f645167b44a4d166f87863fe34918cea25c18059b4c4d9cb414b59f6bc50c1cea749c80c43e2344f5d23159122ed4ab9722503b212016470d9610b46c35dbeebaf2e342e09770b38392a803bc9d2e7c8d6d384ffcbeb74943fe3f64afb2a543a6683c7db3088441c531eeb4647518cb41992f8954f1269fb969630944928c2d2b45593731b5da0c4e70d04a0a57afe4af42e99912fbb4f8883a5ecb9cb29b883cb6bfa0f4db2279ff8c6d2b56a232f55ba28fe7dfa70a9ab0433a085388f25cce8d53de6a2fbd7546377d6ede9027ad173ba1f95767461a3689ef405ab608a21086165c64b02c1782b04a6dba2361a7784603069124e12f2f6dcb1ec7612a4fbf94c0e14631a2bef6190c3d5f35e0c4b32aa85201f449d830fd8f782ec758b0910428e3ec3ca1dba3b6c7d89f69e1ee1b9df3dfbbf6d361e1463886b38d52e8f43b73a3bd48c6f36f5897f514b93364a31d49d1d506340b1315883d425cb36f4ea553430d538fd6f3596d4afc518db2f317dd051abc0d4bfb0a7870c3db70f19fe78d6604bbf088fcb4613f54e67b038277fedcd9680eb97bdffc3be1ab2cbcbafd625b8a7ac34d8c190f98d3064ecd3b95b8895157c6a37f31ef4de094b2cb9dbf8ff1f419ba0ecacb1bb13df0253b826bec2ccca1e745dd3b3e7cc6277ce284d649e7b8285727735ff4ef6cca6c18e2714f4e2a1ac67b25213d3bb49763b3b94e7ebf72507b71fb2fe0329666477ee7cb7ebd6b88ad5add8b217188b1ca0fa13de1ec09cc674346875105be6e0e0d6c8928eb0df23c39a639e04e4aedf535c4e093f08b2c905a14f25c0c0fe47a5a1535ab9eae0d9d67bdd79de13a08d59ee05385c7ea4af1ad3248e61dd22f8990e9e99897d653dd7b1b1433a6d464ea9f74e377f2d8ce99ba7dbc753297644234d25ecb5bd528e2e2082824681299ac30c05354baaa9c3967d86d7c07736f87fc0f63e5036d47235d7ae12178ced3ae36ee5919c093a02579e4fc9edad2c446c656c790704bfc8e2c491a42500aa1d75c8d4921ce29b753f883e17c79b09ea324f1f32ddf1f3284cd70e847b09d90f6718c42e5c94484cc9cbb0df659d255630a3f5a27e7d5dd14fa6b974d1719aa98f01a20fb4b7b1c77b42d57fab3c724339d459ee4a1c6b5d3bd4e08624c786a257872acc9ad3ff62222f2265a658d9f2a007229a5293b67ec91c84c4b4407c228434bad8a815ca9b256c776bd2c9f
	# forwarding error packet
	shared_secret = 21e13c2d7cfe7e18836df50872466117a295783ab8aab0e7ecc8c725503ad02d
	ammag_key = cd9ac0e09064f039fa43a31dea05f5fe5f6443d40a98be4071af4a9d704be5ad
	stream = 617ca1e4624bc3f04fece3aa5a2b615110f421ec62408d16c48ea6c1b7c33fe7084a2bd9d4652fc5068e5052bf6d0acae2176018a3d8c75f37842712913900263cff92f39f3c18aa1f4b20a93e70fc429af7b2b1967ca81a761d40582daf0eb49cef66e3d6fbca0218d3022d32e994b41c884a27c28685ef1eb14603ea80a204b2f2f474b6ad5e71c6389843e3611ebeafc62390b717ca53b3670a33c517ef28a659c251d648bf4c966a4ef187113ec9848bf110816061ca4f2f68e76ceb88bd6208376460b916fb2ddeb77a65e8f88b2e71a2cbf4ea4958041d71c17d05680c051c3676fb0dc8108e5d78fb1e2c44d79a202e9d14071d536371ad47c39a05159e8d6c41d17a1e858faaaf572623aa23a38ffc73a4114cb1ab1cd7f906c6bd4e21b29694f9830d12e8ad4b1ac320b3d5bfb4e534f02cefe9a983d66939581412acb1927eb93e8ed73145cddf24266bdcc95923ecb38c8c9c5f4465335b0f18bf9f2d03fa02d57f258db27983d94378bc796cbe7737180dd7e39a36e461ebcb7ec82b6dcdf9d3f209381f7b3a23e798c4f92e13b4bb972ee977e24f4b83bb59b577c210e1a612c2b035c8271d9bc1fb915776ac6560315b124465866830473aa238c35089cf2adb9c6e9f05ab113c1d0a4a18ba0cb9951b928c0358186532c36d4c3daa65657be141cc22e326f88e445e898893fd5f0a7dd231ee5bc972077b1e12a8e382b75d4b557e895a2adc757f2e451e33e0ae3fb54566ee09155da6ada818aa4a4a2546832a8ba22f0ef9ec6a1c78e03a7c29cb126bcaf81aea61cd8b07ab9f4e5e0ad0d9a3a0c66d2d0a00cc05884d183a68e816e76b75842d55895f5b91c5c1b9f7052763aae8a647aa0799214275b6e781f0816fc9ffb802a0101eb5a2de6b3375d3e3478f892b2de7f1900d8ca9bf188fcba89fc49d03c38fa2587a8ae119abfb295b15fa11cb188796bedc4fdfceef296e44fbfa7e84569cc6346389a421782e40a298e1e2b6f9cae3103c3f39d24541e4ab7b61dafe1a5f2fe936a59d87cccdaf7c226acc451ceec3e81bc4828b4925feeae3526d5e2bf93bd5f4fdc0e069010aea1ae7e0d480d438918598896b776bf08fea124f91b3a13414b56934857707902612fc97b0e5d02cbe6e5901ad304c7e8656390efccf3a1b22e18a2935181b78d5d2c89540ede8b0e6194d0d3945780bf577f622cb12deedbf8210eb1450c298b9ee19f3c7082aabc2bdbd3384f3539dc3766978567135549df0d48287735854a6098fa40a9e48eaa27e0d159beb65dd871e4c0b3fffa65f0375f0d3253f582193135ece60d5b9d8ba6739d87964e992cbec674b728d9eaaed595462c41d15fb497d4baa062368005d13fc99e1402563117a6c140c10363b05196a4cbb6b84ae807d62c748485c15e3316841e4a98c3aac81e3bc996b4baca77eac8cdbe99cc7c5ebeb85c907cefb4abe15cbe87fdc5dc2326019196235ac205934fcf8e3
	error packet for node 3: 7512354d6a26781d25e65539772ba049b7ed7c530bf75ab7ef80cf974b978a07a1c3dabc61940011585323f70fa98cfa1d4c868da30b1f751e44a72d9b3f79809c8c51c9f0843daa8fe83587844fedeacb7348362003b31922cbb4d6169b2087b6f8d192d9cfe5363254cd1fde24641bde9e422f170c3eb146f194c48a459ae2889d706dc654235fa9dd20307ea54091d09970bf956c067a3bcc05af03c41e01af949a131533778bf6ee3b546caf2eabe9d53d0fb2e8cc952b7e0f5326a69ed2e58e088729a1d85971c6b2e129a5643f3ac43da031e655b27081f10543262cf9d72d6f64d5d96387ac0d43da3e3a03da0c309af121dcf3e99192efa754eab6960c256ffd4c546208e292e0ab9894e3605db098dc16b40f17c320aa4a0e42fc8b105c22f08c9bc6537182c24e32062c6cd6d7ec7062a0c2c2ecdae1588c82185cdc61d874ee916a7873ac54cddf929354f307e870011704a0e9fbc5c7802d6140134028aca0e78a7e2f3d9e5c7e49e20c3a56b624bfea51196ec9e88e4e56be38ff56031369f45f1e03be826d44a182f270c153ee0d9f8cf9f1f4132f33974e37c7887d5b857365c873cb218cbf20d4be3abdb2a2011b14add0a5672e01e5845421cf6dd6faca1f2f443757aae575c53ab797c2227ecdab03882bbbf4599318cefafa72fa0c9a0f5a51d13c9d0e5d25bfcfb0154ed25895260a9df8743ac188714a3f16960e6e2ff663c08bffda41743d50960ea2f28cda0bc3bd4a180e297b5b41c700b674cb31d99c7f2a1445e121e772984abff2bbe3f42d757ceeda3d03fb1ffe710aecabda21d738b1f4620e757e57b123dbc3c4aa5d9617dfa72f4a12d788ca596af14bea583f502f16fdc13a5e739afb0715424af2767049f6b9aa107f69c5da0e85f6d8c5e46507e14616d5d0b797c3dea8b74a1b12d4e47ba7f57f09d515f6c7314543f78b5e85329d50c5f96ee2f55bbe0df742b4003b24ccbd4598a64413ee4807dc7f2a9c0b92424e4ae1b418a3cdf02ea4da5c3b12139348aa7022cc8272a3a1714ee3e4ae111cffd1bdfd62c503c80bdf27b2feaea0d5ab8fe00f9cec66e570b00fd24b4a2ed9a5f6384f148a4d6325110a41ca5659ebc5b98721d298a52819b6fb150f273383f1c5754d320be428941922da790e17f482989c365c078f7f3ae100965e1b38c052041165295157e1a7c5b7a57671b842d4d85a7d971323ad1f45e17a16c4656d889fc75c12fc3d8033f598306196e29571e414281c5da19c12605f48347ad5b4648e371757cbe1c40adb93052af1d6110cfbf611af5c8fc682b7e2ade3bfca8b5c7717d19fc9f97964ba6025aebbc91a6671e259949dcf40984342118de1f6b514a7786bd4f6598ffbe1604cef476b2a4cb1343db608aca09d1d38fc23e98ee9c65e7f6023a8d1e61fd4f34f753454bd8e858c8ad6be6403edc599c220e03ca917db765980ac781e758179cd93983e9c1e769e4241d47c
	# forwarding error packet
	shared_secret = 3a6b412548762f0dbccce5c7ae7bb8147d1caf9b5471c34120b30bc9c04891cc
	ammag_key = 1bf08df8628d452141d56adfd1b25c1530d7921c23cecfc749ac03a9b694b0d3
	stream = 6149f48b5a7e8f3d6f5d870b7a698e204cf64452aab4484ff1dee671fe63fd4b5f1b78ee2047dfa61e3d576b149bedaf83058f85f06a3172a3223ad6c4732d96b32955da7d2feb4140e58d86fc0f2eb5d9d1878e6f8a7f65ab9212030e8e915573ebbd7f35e1a430890be7e67c3fb4bbf2def662fa625421e7b411c29ebe81ec67b77355596b05cc155755664e59c16e21410aabe53e80404a615f44ebb31b365ca77a6e91241667b26c6cad24fb2324cf64e8b9dd6e2ce65f1f098cfd1ef41ba2d4c7def0ff165a0e7c84e7597c40e3dffe97d417c144545a0e38ee33ebaae12cc0c14650e453d46bfc48c0514f354773435ee89b7b2810606eb73262c77a1d67f3633705178d79a1078c3a01b5fadc9651feb63603d19decd3a00c1f69af2dab2595931ca50d8280758b1cc91ba2dc43dbbc3d91bf25c08b46c2ecef7a32cec64d4b61ee3a629ef563afe058b71e71bcb69033948bc8728c5ebe65ec596e4f305b9fc159d53f723dfc95b57f3d51717f1c89af97a6d587e89e62efcc92198a1b2bd66e2d875505ea4046c04389f8cb0ee98f0af03af2652e2f3d9a9c48430f2891a4d9b16e7d18099e4a3dd334c24aba1e2450792c2f22092c170da549d43a440021e699bd6c20d8bbf1961100a01ebcce06a4609f5ad93066287acf68294cfa9ea7cea03a508983b134a9f0118b16409a61c06aaa95897d2067cb7cd59123f3e2ccf0e16091571d616c44818f118bb7835a679f5c0eea8cf1bd5479882b2c2a341ec26dbe5da87b3d37d66b1fbd176f71ab203a3b6eaf7f214d579e7d0e4a3e59089ebd26ba04a62403ae7a793516ec16d971d51c5c0107a917d1a70221e6de16edca7cb057c7d06902b5191f298aa4d478a0c3a6260c257eae504ebbf2b591688e6f3f77af770b6f566ae9868d2f26c12574d3bf9323af59f0fe0072ff94ae597c2aa6fbcbf0831989e02f9d3d1b9fd6dd97f509185d9ecbf272e38bd621ee94b97af8e1cd43853a8f6aa6e8372585c71bf88246d064ade524e1e0bd8496b620c4c2d3ae06b6b064c97536aaf8d515046229f72bee8aa398cd0cc21afd5449595016bef4c77cb1e2e9d31fe1ca3ffde06515e6a4331ccc84edf702e5777b10fc844faf17601a4be3235931f6feca4582a8d247c1d6e4773f8fb6de320cf902bbb1767192782dc550d8e266e727a2aa2a414b816d1826ea46af71701537193c22bbcc0123d7ff5a23b0aa8d7967f36fef27b14fe1866ff3ab215eb29e07af49e19174887d71da7e7fe1b7aa1b3c805c063e0fafedf125fa6c57e38cce33a3f7bb35fd8a9f0950de3c22e49743c05f40bc55f960b8a8b5e2fde4bb229f125538438de418cb318d13968532499118cb7dcaaf8b6d635ac4001273bdafd12c8ea0702fb2f0dac81dbaaf68c1c32266382b293fa3951cb952ed5c1bdc41750cdbc0bd62c51bb685616874e251f031a929c06faef5bfcb0857f815ae20620b823f0abecfb5
	error packet for node 2: 145bc1c63058f7204abbd2320d422e69fb1b3801a14312f81e5e29e6b5f4774cfed8a25241d3dfb7466e749c1b3261559e49090853612e07bd669dfb5f4c54162fa504138dabd6ebcf0db8017840c35f12a2cfb84f89cc7c8959a6d51815b1d2c5136cedec2e4106bb5f2af9a21bd0a02c40b44ded6e6a90a145850614fb1b0eef2a03389f3f2693bc8a755630fc81fff1d87a147052863a71ad5aebe8770537f333e07d841761ec448257f948540d8f26b1d5b66f86e073746106dfdbb86ac9475acf59d95ece037fba360670d924dce53aaa74262711e62a8fc9eb70cd8618fbedae22853d3053c7f10b1a6f75369d7f73c419baa7dbf9f1fc5895362dcc8b6bd60cca4943ef7143956c91992119bccbe1666a20b7de8a2ff30a46112b53a6bb79b763903ecbd1f1f74952fb1d8eb0950c504df31fe702679c23b463f82a921a2c931500ab08e686cffb2d87258d254fb17843959cccd265a57ba26c740f0f231bb76df932b50c12c10be90174b37d454a3f8b284c849e86578a6182c4a7b2e47dd57d44730a1be9fec4ad07287a397e28dce4fda57e9cdfdb2eb5afdf0d38ef19d982341d18d07a556bb16c1416f480a396f278373b8fd9897023a4ac506e65cf4c306377730f9c8ca63cf47565240b59c4861e52f1dab84d938e96fb31820064d534aca05fd3d2600834fe4caea98f2a748eb8f200af77bd9fbf46141952b9ddda66ef0ebea17ea1e7bb5bce65b6e71554c56dd0d4e14f4cf74c77a150776bf31e7419756c71e7421dc22efe9cf01de9e19fc8808d5b525431b944400db121a77994518d6025711cb25a18774068bba7faaa16d8f65c91bec8768848333156dcb4a08dfbbd9fef392da3e4de13d4d74e83a7d6e46cfe530ee7a6f711e2caf8ad5461ba8177b2ef0a518baf9058ff9156e6aa7b08d938bd8d1485a787809d7b4c8aed97be880708470cd2b2cdf8e2f13428cc4b04ef1f2acbc9562f3693b948d0aa94b0e6113cafa684f8e4a67dc431dfb835726874bef1de36f273f52ee694ec46b0700f77f8538067642a552968e866a72a3f2031ad116663ac17b172b446c5bc705b84777363a9a3fdc6443c07b2f4ef58858122168d4ebbaee920cefc312e1cea870ed6e15eec046ab2073bbf08b0a3366f55cfc6ad4681a12ab0946534e7b6f90ea8992d530ec3daa6b523b3cf03101c60cadd914f30dec932c1ef4341b5a8efac3c921e203574cfe0f1f83433fddb8ccfd273f7c3cab7bc27efe3bb61fdccd5146f1185364b9b621e7fb2b74b51f5ee6be72ab6ff46a6359dc2c855e61469724c1dbeb273df9d2e1c1fb74891239c0019dc12d5c7535f7238f963b761d7102b585372cf021b64c4fc85bfb3161e59d2e298bba44cfd34d6859d9dba9dc6271e5047d525468c814f2ae438474b0a977273036da1a2292f88fcfb89574a6bdca1185b40f8aa54026d5926725f99ef028da1be892e3586361efe15f4a148ff1bc9
	# forwarding error packet
	shared_secret = a6519e98832a0b179f62123b3567c106db99ee37bef036e783263602f3488fae
	ammag_key = 59ee5867c5c151daa31e36ee42530f429c433836286e63744f2020b980302564
	stream = 0f10c86f05968dd91188b998ee45dcddfbf89fe9a99aa6375c42ed5520a257e048456fe417c15219ce39d921555956ae2ff795177c63c819233f3bcb9b8b28e5ac6e33a3f9b87ca62dff43f4cc4a2755830a3b7e98c326b278e2bd31f4a9973ee99121c62873f5bfb2d159d3d48c5851e3b341f9f6634f51939188c3b9ff45feeb11160bb39ce3332168b8e744a92107db575ace7866e4b8f390f1edc4acd726ed106555900a0832575c3a7ad11bb1fe388ff32b99bcf2a0d0767a83cf293a220a983ad014d404bfa20022d8b369fe06f7ecc9c74751dcda0ff39d8bca74bf9956745ba4e5d299e0da8f68a9f660040beac03e795a046640cf8271307a8b64780b0588422f5a60ed7e36d60417562938b400802dac5f87f267204b6d5bcfd8a05b221ec294d883271b06ca709042ed5dbb64a7328d2796195eab4512994fb88919c73b3e5dd7bf68b2136d34cff39b3be266b71e004509bf975a240800bb8ae5eed248423a991ae80ef751b2d03b67fb93ffdd7969d5b500fe446a4ffb4cd04d0767a5d367ebd3f8f260f38ae1e9d9f9a7bd1a99ca1e10ee36bd241f06fc2b481c9b7450d9c9704204666807783264a0e93468e22db4dc4a7a4db2963ddf4366d08e225cf94848aac794bcecb7e850113e38cc3647a03a5dfaa3442b1bb58b1de7fa7f436feb4d7c23cbd2de6d55d4025fcd383cc9d49c0b130e2fd5a9097c216683c842f898a8a2159761cca9aa1c818194e3b7bea6da6652d5189f3b6b0ca1d5398b6d14e311d9c7f00399c29e94deb98496f4cd97c5d7d6a65cabc3791f60d728d6422a422c0cff5f7dfd4ce2d7e8d38dd71ae18763acc832c57275497f61b2620cca13cc64c0c48353f3817016f91448d6fc1cc451ee1f4a429e43292bbcd54fcd807e2c47675bac1781d9d81e9e6dc69028d428f5ee261750f626bcaf416a0e7badadf73fe1922207ae6c5209d16849e4a108f4a6f38694075f55177105ac4c2b97f6a474b94c03257d8d12b0196e2905d914b8c2213a1b9dc9608e1a2a1e03fe0820a813275de83be5e9734875787a9e006eb8574c23ddd49e2347d1ecfcedf3caa0a5dd45666368525b48ac14225d6422f82dbf59860ee4dc78e845d3c57668ce9b9e7a8d012491cef242078b458a956ad67c360fb6d8b86ab201d6217e49b55fa02a1dea2dbe88d0b08d30670d1b93c35cc5e41e088fccb267e41d6151cf8560496e1beeefe680744d9dabb383a4957466b4dc3e2bce7b135211da483d998a22fa687cc609641126c5dee3ed87291067916b5b065f40582163291d48e81ecd975d0d6fd52a31754f8ef15e43a560bd30ea5bf21915bd2e7007e607abbc6261edc8430cc7f789675b1fe83e807c5c475bd5178eba2fc40674706b0a68c6a428e5dec36e413e653c6db1178923ff87e2389a78bf9e93b713de4f4753f9f9d6a361369b609e1970c91ff9bd191c472e0bf2e8681412260ad0ef5855dc39f2084d45
	error packet for node 1: 1b4b09a935ce7af95b336baae307f2b400e3a7e808d9b4cf421cc4b3955620acb69dcdb656128dae8857adbd4e6b37fbb1be9c1f2f02e61e9e59a630c4c77cf383cb37b07413aa4de2f2fbf5b40ae40a91a8f4c6d74aeacef1bb1be4ecbc26ec2c824d2bc45db4b9098e732a769788f1cff3f5b41b0d25c132d40dc5ad045ef0043b15332ca3c5a09de2cdb17455a0f82a8f20da08346282823dab062cdbd2111e238528141d69de13de6d83994fbc711e3e269df63a12d3a4177c5c149150eb4dc2f589cd8acabcddba14dec3b0dada12d663b36176cd3c257c5460bab93981ad99f58660efa9b31d7e63b39915329695b3fa60e0a3bdb93e7e29a54ca6a8f360d3848866198f9c3da3ba958e7730847fe1e6478ce8597848d3412b4ae48b06e05ba9a104e648f6eaf183226b5f63ed2e68f77f7e38711b393766a6fab7921b03eba82b5d7cb78e34dc961948d6161eadd7cf5d95d9c56df2ff5faa6ccf85eacdc9ff2fc3abafe41c365a5bd14fd486d6b5e2f24199319e7813e02e798877ffe31a70ae2398d9e31b9e3727e6c1a3c0d995c67d37bb6e72e9660aaaa9232670f382add2edd468927e3303b6142672546997fe105583e7c5a3c4c2b599731308b5416e6c9a3f3ba55b181ad0439d3535356108b059f2cb8742eed7a58d4eba9fe79eaa77c34b12aff1abdaea93197aabd0e74cb271269ca464b3b06aef1d6573df5e1224179616036b368677f26479376681b772d3760e871d99efd34cca5cd6beca95190d967da820b21e5bec60082ea46d776b0517488c84f26d12873912d1f68fafd67bcf4c298e43cfa754959780682a2db0f75f95f0598c0d04fd014c50e4beb86a9e37d95f2bba7e5065ae052dc306555bca203d104c44a538b438c9762de299e1c4ad30d5b4a6460a76484661fc907682af202cd69b9a4473813b2fdc1142f1403a49b7e69a650b7cde9ff133997dcc6d43f049ecac5fce097a21e2bce49c810346426585e3a5a18569b4cddd5ff6bdec66d0b69fcbc5ab3b137b34cc8aefb8b850a764df0e685c81c326611d901c392a519866e132bbb73234f6a358ba284fbafb21aa3605cacbaf9d0c901390a98b7a7dac9d4f0b405f7291c88b2ff45874241c90ac6c5fc895a440453c344d3a365cb929f9c91b9e39cb98b142444aae03a6ae8284c77eb04b0a163813d4c21883df3c0f398f47bf127b5525f222107a2d8fe55289f0cfd3f4bbad6c5387b0594ef8a966afc9e804ccaf75fe39f35c6446f7ee076d433f2f8a44dba1515acc78e589fa8c71b0a006fe14feebd51d0e0aa4e51110d16759eee86192eee90b34432130f387e0ccd2ee71023f1f641cddb571c690107e08f592039fe36d81336a421e89378f351e633932a2f5f697d25b620ffb8e84bb6478e9bd229bf3b164b48d754ae97bd23f319e3c56b3bcdaaeb3bd7fc02ec02066b324cb72a09b6b43dec1097f49d69d3c138ce6f1a6402898baf7568c
	# forwarding error packet
	shared_secret = 53eb63ea8a3fec3b3cd433b85cd62a4b145e1dda09391b348c4e1cd36a03ea66
	ammag_key = 3761ba4d3e726d8abb16cba5950ee976b84937b61b7ad09e741724d7dee12eb5
	stream = 3699fd352a948a05f604763c0bca2968d5eaca2b0118602e52e59121f050936c8dd90c24df7dc8cf8f1665e39a6c75e9e2c0900ea245c9ed3b0008148e0ae18bbfaea0c711d67eade980c6f5452e91a06b070bbde68b5494a92575c114660fb53cf04bf686e67ffa4a0f5ae41a59a39a8515cb686db553d25e71e7a97cc2febcac55df2711b6209c502b2f8827b13d3ad2f491c45a0cafe7b4d8d8810e805dee25d676ce92e0619b9c206f922132d806138713a8f69589c18c3fdc5acee41c1234b17ecab96b8c56a46787bba2c062468a13919afc18513835b472a79b2c35f9a91f38eb3b9e998b1000cc4a0dbd62ac1a5cc8102e373526d7e8f3c3a1b4bfb2f8a3947fe350cb89f73aa1bb054edfa9895c0fc971c2b5056dc8665902b51fced6dff80c4d247db977c15a710ce280fbd0ae3ca2a245b1c967aeb5a1a4a441c50bc9ceb33ca64b5ca93bd8b50060520f35a54a148a4112e8762f9d0b0f78a7f46a5f06c7a4b0845d020eb505c9e527aabab71009289a6919520d32af1f9f51ce4b3655c6f1aae1e26a16dc9aae55e9d4a6f91d4ba76e96fcb851161da3fc39d0d97ce30a5855c75ac2f613ff36a24801bcbd33f0ce4a3572b9a2fca21efb3b07897fc07ee71e8b1c0c6f8dbb7d2c4ed13f11249414fc94047d1a4a0be94d45db56af4c1a3bf39c9c5aa18209eaebb9e025f670d4c8cc1ee598c912db154eaa3d0c93cb3957e126c50486bf98c852ad326b5f80a19df6b2791f3d65b8586474f4c5dcb2aca0911d2257d1bb6a1e9fc1435be879e75d23290f9feb93ed40baaeca1c399fc91fb1da3e5f0f5d63e543a8d12fe6f7e654026d3a118ab58cb14bef9328d4af254215eb1f639828cc6405a3ab02d90bb70a798787a52c29b3a28fc67b0908563a65f08112abd4e9115cb01db09460c602aba3ddc375569dc3abe42c61c5ea7feb39ad8b05d8e2718e68806c0e1c34b0bc85492f985f8b3e76197a50d63982b780187078f5c59ebd814afaeffc7b2c6ee39d4f9c8c45fb5f685756c563f4b9d028fe7981b70752f5a31e44ba051ab40f3604c8596f1e95dc9b0911e7ede63d69b5eecd245fbecbcf233cf6eba842c0fec795a5adeab2100b1a1bc62c15046d48ec5709da4af64f59a2e552ddbbdcda1f543bb4b687e79f2253ff0cd9ba4e6bfae8e510e5147273d288fd4336dbd0b6617bf0ef71c0b4f1f9c1dc999c17ad32fe196b1e2b27baf4d59bba8e5193a9595bd786be00c32bae89c5dbed1e994fddffbec49d0e2d270bcc1068850e5d7e7652e274909b3cf5e3bc6bf64def0bbeac974a76d835e9a10bdd7896f27833232d907b7405260e3c986569bb8fdd65a55b020b91149f27bda9e63b4c2cc5370bcc81ef044a68c40c1b178e4265440334cc40f59ab5f82a022532805bfa659257c8d8ab9b4aef6abbd05de284c2eb165ef35737e3d387988c566f7b1ca0b1fc3e7b4ed991b77f23775e1c36a09a991384a33b78
	error packet for node 0: 2dd2f49c1f5af0fcad371d96e8cddbdcd5096dc309c1d4e110f955926506b3c03b44c192896f45610741c85ed4074212537e0c118d472ff3a559ae244acd9d783c65977765c5d4e00b723d00f12475aafaafff7b31c1be5a589e6e25f8da2959107206dd42bbcb43438129ce6cce2b6b4ae63edc76b876136ca5ea6cd1c6a04ca86eca143d15e53ccdc9e23953e49dc2f87bb11e5238cd6536e57387225b8fff3bf5f3e686fd08458ffe0211b87d64770db9353500af9b122828a006da754cf979738b4374e146ea79dd93656170b89c98c5f2299d6e9c0410c826c721950c780486cd6d5b7130380d7eaff994a8503a8fef3270ce94889fe996da66ed121741987010f785494415ca991b2e8b39ef2df6bde98efd2aec7d251b2772485194c8368451ad49c2354f9d30d95367bde316fec6cbdddc7dc0d25e99d3075e13d3de0822669861dafcd29de74eac48b64411987285491f98d78584d0c2a163b7221ea796f9e8671b2bb91e38ef5e18aaf32c6c02f2fb690358872a1ed28166172631a82c2568d23238017188ebbd48944a147f6cdb3690d5f88e51371cb70adf1fa02afe4ed8b581afc8bcc5104922843a55d52acde09bc9d2b71a663e178788280f3c3eae127d21b0b95777976b3eb17be40a702c244d0e5f833ff49dae6403ff44b131e66df8b88e33ab0a58e379f2c34bf5113c66b9ea8241fc7aa2b1fa53cf4ed3cdd91d407730c66fb039ef3a36d4050dde37d34e80bcfe02a48a6b14ae28227b1627b5ad07608a7763a531f2ffc96dff850e8c583461831b19feffc783bc1beab6301f647e9617d14c92c4b1d63f5147ccda56a35df8ca4806b8884c4aa3c3cc6a174fdc2232404822569c01aba686c1df5eecc059ba97e9688c8b16b70f0d24eacfdba15db1c71f72af1b2af85bd168f0b0800483f115eeccd9b02adf03bdd4a88eab03e43ce342877af2b61f9d3d85497cd1c6b96674f3d4f07f635bb26add1e36835e321d70263b1c04234e222124dad30ffb9f2a138e3ef453442df1af7e566890aedee568093aa922dd62db188aa8361c55503f8e2c2e6ba93de744b55c15260f15ec8e69bb01048ca1fa7bbbd26975bde80930a5b95054688a0ea73af0353cc84b997626a987cc06a517e18f91e02908829d4f4efc011b9867bd9bfe04c5f94e4b9261d30cc39982eb7b250f12aee2a4cce0484ff34eebba89bc6e35bd48d3968e4ca2d77527212017e202141900152f2fd8af0ac3aa456aae13276a13b9b9492a9a636e18244654b3245f07b20eb76b8e1cea8c55e5427f08a63a16b0a633af67c8e48ef8e53519041c9138176eb14b8782c6c2ee76146b8490b97978ee73cd0104e12f483be5a4af414404618e9f6633c55dda6f22252cb793d3d16fae4f0e1431434e7acc8fa2c009d4f6e345ade172313d558a4e61b4377e31b8ed4e28f7cd13a7fe3f72a409bc3bdabfe0ba47a6d861e21f64d2fac706dab18b3e546df4

# Referencias

[sphinx]: http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf
[RFC2104]: https://tools.ietf.org/html/rfc2104
[fips198]: http://csrc.nist.gov/publications/fips/fips198-1/FIPS-198-1_final.pdf
[sec2]: http://www.secg.org/sec2-v2.pdf
[rfc8439]: https://tools.ietf.org/html/rfc8439

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
