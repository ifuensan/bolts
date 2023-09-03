# BOLT #0: Introducción e Índice

¡Bienvenido, amigo! Estos documentos de `Basis of Lightning Technology` (BOLT) describen un protocolo de capa 2 para la transferencia de Bitcoin fuera de la cadena (`off-chain`) mediante cooperación mutua, confiando en transacciones en la cadena (`on-chain`) para la aplicación si es necesario.

Algunos requisitos son sutiles; hemos tratado de resaltar las motivaciones y el razonamiento detrás de los resultados que ves aquí. Seguramente nos hemos quedado cortos; si encuentras alguna parte confusa o incorrecta, por favor contáctanos y ayúdanos a mejorar.

Esta es la traducción de la versión 0.

1. [BOLT #1](01-messaging.md): Protocolo Base
2. [BOLT #2](02-peer-protocol.md): Protocolo de pares para la gestión de canales
3. [BOLT #3](03-transactions.md): Formatos de Transacciones y Scripts de Bitcoin
4. [BOLT #4](04-onion-routing.md): Protocolo Onion Routing
5. [BOLT #5](05-onchain.md): Recomendaciones para Manejar las transacciones On-chain
7. [BOLT #7](07-routing-gossip.md): Descubrimiento de canales y nodos P2P
8. [BOLT #8](08-transport.md): Transporte cifrado y autenticado
9. [BOLT #9](09-features.md): Feature Flags Asignados
10. [BOLT #10](10-dns-bootstrap.md): DNS Bootstrap y Ubicación Asistida de Nodos
11. [BOLT #11](11-payment-encoding.md): Invoice Protocol for Lightning Payments

## La Chispa: Una Breve Introducción a Lightning

Lightning es un protocolo para realizar pagos rápidos con Bitcoin utilizando una red de canales.

### Canales

Lightning funciona estableciendo canales: dos participantes crean un canal de pago en Lightning que contiene una cierta cantidad de bitcoin (por ejemplo, 0.1 bitcoin) que han bloqueado en la red de Bitcoin. Solo es gastable con las firmas de ambos.

Inicialmente, cada uno de ellos tiene una transacción de bitcoin que envía todos los bitcoins (por ejemplo, 0.1 bitcoin) de vuelta a una de las partes. Más adelante, pueden firmar una nueva transacción de bitcoin que distribuye estos fondos de manera diferente, por ejemplo, 0.09 bitcoin para una parte y 0.01 bitcoin para la otra, e invalidar la transacción de bitcoin anterior para que no se gaste.

Consulta [BOLT #2: Establecimiento de Canales](02-peer-protocol.md#establecimiento-de-canal) para obtener más información sobre el establecimiento de canales y [BOLT #3: Salida de la Transacción de Financiación](03-transactions.md#funding-transaction-output) para conocer el formato de la transacción de bitcoin que crea el canal. Consulta [BOLT #5: Recomendaciones para el Manejo de Transacciones](05-onchain.md) para conocer los requisitos cuando los participantes no están de acuerdo o fallan, y la transacción de bitcoin con firmas cruzadas debe ser gastada.

### Pagos condicionales

Un canal Lightning solo permite pagos entre dos participantes, pero los canales se pueden conectar entre sí para formar una red que permite pagos entre todos los miembros de la red. Esto requiere la tecnología de un pago condicional, que se puede agregar a un canal, por ejemplo "obtienes 0,01 bitcoin si revelas el secreto dentro de las 6 horas". Una vez que el destinatario presenta el secreto, esa transacción de bitcoin se reemplaza por una que carece del pago condicional y se agregan los fondos a la salida de ese destinatario.

Consulte el [BOLT #2: Agregar un HTLC](02-peer-protocol.md#añadiendo-un-htlc-update_add_htlc) para conocer los comandos que usa un participante para agregar un pago condicional, y el [BOLT #3: Commitment Transaction o `Transacción de compromiso`](03-transactions.md#commitment-transaction) para conocer el formato completo de la transacción bitcoin.

### Reenvío
Un pago condicional de este tipo se puede enviar de forma segura a otro participante con un límite de tiempo más bajo, por ejemplo "obtienes 0,01 bitcoin si revelas el secreto dentro de las 5 horas". Esto permite encadenar canales en una red sin confiar en los intermediarios.

Consulte [BOLT #2: Reenvío de HTLCs](02-peer-protocol.md#reenvío-de-htlcs) para obtener detalles sobre el reenvío de pagos, [BOLT #4: Estructura de paquetes](04-onion-routing.md#estructura-del-paquete) para saber cómo se transportan las instrucciones de pago.

### Topología de la red
Para realizar un pago, un participante necesita saber a través de qué canales puede enviar. Los participantes se cuentan entre sí sobre la creación y las actualizaciones de canales y nodos.

Consulte el [BOLT #7: Descubrimiento de canales y nodos P2P](07-routing-gossip.md) para obtener detalles sobre el protocolo de comunicación, y el [BOLT #10: Arranque DNS y ubicación asistida de nodo](10-dns-bootstrap.md) para el arranque inicial de la red.

### Facturación de Pagos

Un participante recibe facturas que le indican qué pagos realizar.

Consulte [BOLT #11: Protocolo de factura para pagos Lightning](11-payment-encoding.md) para conocer el protocolo que describe el destino y el propósito de un pago, de modo que el pagador pueda demostrar posteriormente que el pago fue exitoso.

## Glosario y guía de la terminología

* #### *Anuncio*:
   * Un mensaje de chisme enviado entre *[pares](#pares)* destinado a ayudar al descubrimiento de un *[canal](#canal)* o un *[nodo](#nodo)*.

* #### `chain_hash`:
   * El hash de identificación único de la cadena de bloques de destino (generalmente el hash génesis). Esto permite que *[nodos](#nodo)* creen y hagan referencia a *canales* en varias cadenas de bloques. Los nodos deben ignorar cualquier mensaje que haga referencia a un `chain_hash` que les sea desconocido. A diferencia de `bitcoin-cli`, el hash no se invierte sino que se usa directamente.

     Para la cadena principal de la cadena de bloques de Bitcoin, el valor `chain_hash` DEBE ser (codificado en hexadecimal): `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.

* #### *Canal*: 
    * Un método rápido, fuera de la cadena, de intercambio mutuo entre dos *[pares](#pares)*. Para realizar transacciones de fondos, los pares intercambian firmas para crear una *[transacción de compromiso](#transacción-de-compromiso)* actualizada. 
    * _Ver métodos de cierre: [cierre mutuo](#cierre-mutuo), [cierre de transacción revocada](#cierre-de-transacción-revocada), [cierre unilateral](#cierre-unilateral)_ 
    * _Ver relacionado: [ruta](#ruta)_
  
* #### *Cierre de transacción*: 
    * Una transacción generada como parte de un *[cierre mutuo](#cierre-mutuo)*. Una transacción de cierre es similar a una _transacción de compromiso_, pero sin pagos pendientes. 
    * _Ver relacionado: [transacción de compromiso](#transacción-de-compromiso), [transacción de financiación](#transacción-de-financiación), [transacción de penalización](#transacción-de-penalización)_

* #### *Número de compromiso*: 
    * Un contador incremental de 48 bits para cada *[transacción de compromiso](#transaccion-de-compromiso)*; Los contadores son independientes para cada *par* en el *canal* y comienzan en 0. 
    * _Ver contenedor: [transacción de compromiso](#transacción-de-compromiso)_ 
    * _Ver relacionado: [transacción de cierre](#transacción-de-cierre), [transacción de financiación](#transacción-de-financiación), [transacción de penalización](#transacción-de-penalización)_

* #### *Clave privada de revocación de compromiso*: 
    * Cada *[transacción de compromiso](#transacción-de-compromiso)* tiene un valor de clave privada de revocación de compromiso único que permite al otro *par* gastar todos los resultados inmediatamente: revelar esta clave es cómo se revocan las transacciones de compromiso antiguas. Para admitir la revocación, cada resultado de la transacción de compromiso hace referencia a la clave pública de revocación del compromiso. 
    * _Ver contenedor: [transacción de compromiso](#transacción-de-compromiso)_ 
    * _Ver autor: [secreto por compromiso](#secreto-por-medio-del-compromiso)_

* #### *Transacción de compromiso*: 
    * Una transacción que gasta la *[transacción de financiación](#transacción-de-financiación)*. Cada *par* tiene la firma del otro par para esta transacción, de modo que cada uno siempre tiene una transacción de compromiso que puede gastar. Después de negociar una nueva transacción de compromiso, la anterior se *revoca*. 
    * _Ver partes: [número de compromiso](#número-compromiso), [clave privada de revocación de compromiso](#clave-privada-de-revocación-de-compromiso), [HTLC](#HTLC-Contrato-bloqueado-por-hash-de-tiempo), [por-secreto de compromiso](#por-secreto-de-compromiso), [punto de salida](#punto-de-salida)_ 
    * _Ver relacionado: [transacción de cierre](#transacción-de-cierre), [transacción de financiación](#transacción-de-financiación), [transacción de penalización](#transacción-de-penalización)_ 
    * _Ver tipos: [transacción de compromiso revocada](#transacción-de-compromiso-revocada)_

* #### *Fallo del canal*: 
   * Este es un cierre forzoso del canal. Muy pronto (antes de abrir), esto puede no requerir ninguna acción más que olvidar la existencia del canal. Por lo general, requiere firmar y transmitir la última transacción de compromiso, aunque durante el cierre mutuo también se puede realizar firmando y transmitiendo una transacción de cierre mutuo. Consulte [BOLT #5](05-onchain.md#fallando-un-canal).

* #### *Cerrar la conexión*: 
   * Esto significa cerrar la comunicación con el par (como cerrar el socket TCP). No implica cerrar ningún canal con el par, pero sí provoca el descarte del estado no comprometido para conexiones con canales: consulte [BOLT #2](02-peer-protocol.md#retransmisión-de-mensaje).

* #### *Nodo final*: 
    * El destinatario final de un paquete que enruta un pago desde un *[nodo de origen](#nodo-de origen)* a través de algún número de *[saltos](#salto-hop)*. También es el *[par receptor](#par-receptor)* final de una cadena. 
    * _Ver categoría: [nodo](#nodo)_ 
    * _Ver relacionado: [nodo origen](#nodo-origen), [nodo enrutador](#nodo-enrutador)_

* #### *Transacción de financiación*: 
    * Una transacción irreversible en cadena que paga a ambos *[peers](#peers)* en un *[channel](#channel)*. Sólo puede gastarse de mutuo acuerdo. 
    * _Ver relacionado: [transacción de cierre](#transacción-de-cierre), [transacción de compromiso](#transacción-de-compromiso), [transacción-de-penalización](#transacción-de-penalización)_

* #### *Salto `Hop`*: 
    * Un *[nodo](#nodo)*. Generalmente, un nodo intermedio que se encuentra entre un *[nodo origen](#nodo-origen)* y un *[nodo final](#nodo-final)*. 
    * _Ver categoría: [nodo](#nodo)_

* #### *HTLC*: Contrato bloqueado por hash de tiempo. 
    * Un pago condicional entre dos *[pares](#peers)*: el destinatario puede gastar el pago presentando su firma y una *preimagen de pago*, de lo contrario, el pagador puede cancelar el contrato gastándolo después de un tiempo determinado. Estos se implementan como resultados de la *[transacción de compromiso](#transacción-compromiso)*. 
    * _Ver contenedor: [transacción de compromiso](#transacción-compromiso)_ 
    * _Ver partes: [Hash de pago](#Hash-de-pago), [Preimagen de pago](#Preimagen-de-pago)_

* #### *Factura*: Una solicitud de fondos en Lightning Network, posiblemente 
     incluido el tipo de pago, el monto del pago, el vencimiento y otra información. Así es como se realizan los pagos en Lightning Network, en lugar de utilizar direcciones estilo Bitcoin. 

 * #### *Está bien ser impar*:
    * En inglés `It's ok to be odd`. Una regla aplicada a algunos campos numéricos que indica compatibilidad opcional u obligatoria para funciones. Los números pares indican que ambos puntos finales DEBEN admitir la característica en cuestión, mientras que los números impares indican que el otro punto final PUEDE ignorar la característica. 

 * #### *MSAT*: 
    * Un milisatoshi, utilizado a menudo como nombre de campo.

* #### *Cierre mutuo*: 
    * Un cierre cooperativo de un *[canal](#canal)*, logrado mediante la transmisión de un gasto incondicional de la *[transacción de financiación](#transacción-de-financiación)* con una salida para cada *par* (a menos que una salida sea demasiado pequeño y por lo tanto no está incluido). 
    * _Ver relacionado: [cierre de transacción revocada](#cierre-de-transacción-revocada), [cierre unilateral](#cierre-unilateral)_

* #### *Nodo*: 
    * Una computadora u otro dispositivo que sea parte de la red Lightning. 
    * _Ver relacionado: [pares](#pares)_ 
    * _Ver tipos: [nodo final](#nodo-final), [salto](#salto-hop), [nodo origen](#nodo-origen), [nodo enrutador](#nodo-enrutador), [nodo receptor]( #nodo-receptor), [nodo emisor](#nodo-emisor)_
* #### *Nodo origen*: 
    * El *[nodo](#nodo)* que origina un paquete que enrutará un pago a través de una cierta cantidad de [saltos](#salto-hop) a un *[nodo final](#nodo-final)*. También es el primer [par emisor](#par-emisor) de una cadena. 
    * _Ver categoría: [nodo](#nodo)_ 
    * _Ver relacionado: [nodo final](#nodo-final), [nodo enrutador](#nodo-enrutador)_

* #### *Punto de salida*:
   * Un hash de transacción y un índice de salida que identifican de forma única una salida de transacción no gastada. Necesario para componer una nueva transacción, como entrada. 
   * _Ver relacionado: [transacción de financiación](#transacción-de-financiación), [transacción de compromiso](#transacción-de-compromiso)_

* #### *Hash de pago*: 
    * El *[HTLC](#HTLC-Hashed-Time-Locked-Contract)* contiene el hash de pago, que es el hash de la *[preimagen de pago](#Payment-preimage)*. 
    * _Ver contenedor: [HTLC](#HTLC-Hashed-Time-Locked-Contract)_ 
    * _Ver autor: [Preimagen de pago](#Preimagen-de-pago)_

* #### *Preimagen de pago*: 
    * Comprobante de recepción del pago, en poder del destinatario final, que es la única persona que conoce este secreto. El destinatario final libera la preimagen para liberar fondos. La preimagen de pago tiene un hash como *[hash de pago](#hash-de-pago)* en el *[HTLC](#htlc-contrato-bloqueado-por-hash-de-tiempo)*. 
    * _Ver contenedor: [HTLC](#htlc-contrato-bloqueado-por-hash-de-tiempo)_ 
    * _Ver derivación: [hash de pago](#Hash-de-pago)_

* #### *Pares*: 
    * Dos *[nodos](#nodo)* que están en comunicación entre sí. 
       * Dos compañeros pueden cotillear entre sí antes de configurar un canal. 
       * Dos pares pueden establecer un *[canal](#canal)* a través del cual realizan transacciones. 
    * _Ver relacionado: [nodo](#nodo)_

* #### *Transacción de penalización*: 
    * Una transacción que gasta todos los resultados de una *[transacción de compromiso revocada](#transacción-de-compromiso-revocada)*, utilizando la *clave privada de revocación de compromiso*. Un *[par](#pares)* usa esto si el otro par intenta "hacer trampa" transmitiendo una *[transacción de compromiso revocada](#transacción-de-compromiso-revocada)*. 
    * _Ver relacionado: [transacción de cierre](#transacción-de-cierre), [transacción de compromiso](#transacción-de-compromiso), [transacción-de-financiación](#transacción-de-financiación)_

* #### *Secreto por medio del compromiso*:
    * Cada *[transacción de compromiso](#transacción-de-compromiso)* deriva sus claves de un secreto por medio del compromiso, que se genera de manera que la serie de secretos por compromiso para todos los compromisos anteriores se pueda almacenar de forma compacta. 
    * _Ver contenedor: [transacción de compromiso](#transacción-de-compromiso)_ 
    * _Ver derivación: [clave privada de revocación de compromiso](#clave-privada-de-revocación-de-compromiso)_

* #### *Nodo enrutador*: 
    * Un *[nodo](#nodo)* que está procesando un paquete que se originó con un *[nodo origen](#nodo-origen)* y que se envía hacia un *[nodo final](#nodo-final)* para enrutar un pago. Actúa como un *[par receptor](#par-receptor)* para recibir el mensaje, luego un [par remitente](#par-emisor) para enviar el paquete. 
    * _Ver categoría: [nodo](#nodo)_ 
    * _Ver relacionado: [nodo final](#nodo-final), [nodo origen](#nodo-origen)_

* #### *Nodo receptor*: 
    * Un *[nodo](#nodo)* que está recibiendo un mensaje. 
    * _Ver categoría: [nodo](#nodo)_ 
    * _Ver relacionado: [nodo emisor](#nodo-emisor)_

* #### *Par receptor*: 
    * Un *[nodo](#nodo)* que está recibiendo un mensaje de un *par* conectado directamente. 
    * _Ver categoría: [pares](#pares)_ 
    * _Ver relacionado: [par emisor](#par-emisor)_

* #### *Transacción de compromiso revocada*: 
    * Una *[transacción de compromiso](#transacción-de-compromiso)* antigua que ha sido revocada porque se ha negociado una nueva transacción de compromiso. 
    * _Ver categoría: [transacción de compromiso](#transacción-de-compromiso)_

* #### *Cierre de transacción revocada*: 
    * Un cierre no válido de un *[canal](#canal)*, logrado mediante la transmisión de una *transacción de compromiso revocada*. Dado que el otro *par* conoce la *clave secreta de revocación de compromiso*, puede crear una *[transacción de penalización](#transacción-de-penalización)*. 
    * _Ver relacionado: [cierre mutuo](#cierre-mutuo), [cierre unilateral](#cierre-unilateral)_

* #### *Ruta*: 
   * Una ruta a través de Lightning Network que permite un pago desde un *nodo de origen* a un *[nodo final](#nodo-final)* a través de uno o más *[saltos](#salto-hop)*. 
   * _Ver relacionado: [canal](#canal)_

* #### *Nodo emisor*: 
    * Un *[nodo](#nodo)* que está enviando un mensaje. 
    * _Ver categoría: [nodo](#nodo)_ 
    * _Ver relacionado: [nodo receptor](#nodo-receptor)_
  
* #### *Par emisor*: 
    * Un *[nodo](#nodo)* que envía un mensaje a un *par* conectado directamente. 
    * _Ver categoría: [pares](#pares)_ 
    * _Ver relacionado: [par receptor](#par-receptor)_.

* #### *Cierre unilateral*: 
    * Un cierre no cooperativo de un *[canal](#canal)*, logrado mediante la transmisión de una *[transacción de compromiso](#transacción-de-compromiso)*. Esta transacción es más grande (es decir, menos eficiente) que una *[transacción de cierre](#transacción-de-cierre)*, y el *[par](#pares)* cuyo compromiso se transmite no puede acceder a sus propios resultados durante un tiempo previamente negociado.
    * _Ver relacionado: [cierre mutuo](#cierre-mutuo), [cierre de transacción revocada](#cierre-de-transacción-revocada)_
## Tema Musical
   [Escucha el tema musical aquí.](https://youtu.be/edItjMHez48)

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)

 
      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## Autores

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
