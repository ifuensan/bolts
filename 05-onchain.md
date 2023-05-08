<!-- omit in toc -->
# BOLT #5: Recomendaciones para Manejar las transacciones On-chain

<!-- omit in toc -->
## Abstract

Lightning permite que dos partes (un nodo local y un nodo remoto) realicen transacciones fuera de la cadena otorgando a cada una de las partes una *transacción de compromiso con firma cruzada*, que describe el estado actual del canal (básicamente, el saldo actual) . Esta *transacción de compromiso* se actualiza cada vez que se realiza un nuevo pago y es gastable en todo momento.

Hay tres formas en las que un canal puede terminar:

1. El buen camino (*cierre de mutuo acuerdo*): en algún momento los nodos locales y remotos acuerdan cerrar el canal. Generan una *transacción de cierre* (que es similar a una transacción de compromiso, pero sin ningún pago pendiente) y la publican en la cadena de bloques (ver [BOLT #2: Channel Close](02-peer-protocol.md#cierre-de-canal)).
2. La forma mala (*cierre unilateral*): algo sale mal, posiblemente sin malas intenciones de ninguna de las partes. Tal vez una parte falló por completo, por ejemplo. Un lado publica su *última transacción de compromiso*.
3. La forma fea (*cierre de transacción revocada*): una de las partes intenta deliberadamente hacer trampa, publicando una *transacción de compromiso desactualizada* (presumiblemente, una versión anterior, que está más a su favor).

Debido a que Lightning está diseñado para no confiar, no hay riesgo de pérdida de fondos en ninguno de estos tres casos; siempre que la situación se maneje adecuadamente.
El objetivo de este documento es explicar exactamente cómo debe reaccionar un nodo cuando se encuentra con cualquiera de las situaciones anteriores, en la cadena.

<!-- omit in toc -->
# Índice
- [Nomenclatura General](#nomenclatura-general)
- [Transacción de compromiso o `commitment transaction`](#transacción-de-compromiso-o-commitment-transaction)
- [Fallando un Canal](#fallando-un-canal)
- [Manejo de cierre mutuo](#manejo-de-cierre-mutuo)
- [Manejo de Cierre Unilateral: Transacción de Compromiso Local](#manejo-de-cierre-unilateral-transacción-de-compromiso-local)
  - [Manejo de salida de HTLC: compromiso local, ofertas locales](#manejo-de-salida-de-htlc-compromiso-local-ofertas-locales)
  - [Manejo de salida de HTLC: compromiso local, ofertas remotas](#manejo-de-salida-de-htlc-compromiso-local-ofertas-remotas)
- [Manejo cercano unilateral: Transacción de compromiso remoto](#manejo-cercano-unilateral-transacción-de-compromiso-remoto)
  - [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
  - [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
    - [Requirements](#requirements-2)
- [Revoked Transaction Close Handling](#revoked-transaction-close-handling)
  - [Requirements](#requirements-3)
  - [Rationale](#rationale-2)
  - [Penalty Transactions Weight Calculation](#penalty-transactions-weight-calculation)
- [Generation of HTLC Transactions](#generation-of-htlc-transactions)
  - [Requirements](#requirements-4)
- [General Requirements](#general-requirements)
- [Appendix A: Expected Weights](#appendix-a-expected-weights)
  - [Expected Weight of the `to_local` Penalty Transaction Witness](#expected-weight-of-the-to_local-penalty-transaction-witness)
  - [Expected Weight of the `offered_htlc` Penalty Transaction Witness](#expected-weight-of-the-offered_htlc-penalty-transaction-witness)
  - [Expected Weight of the `accepted_htlc` Penalty Transaction Witness](#expected-weight-of-the-accepted_htlc-penalty-transaction-witness)
- [Authors](#authors)

# Nomenclatura General

Cualquier salida no gastada se considera *no resuelta* y se puede *resolver* como se detalla en este documento. Por lo general, esto se logra gastándolo con otra transacción de *resolución*. Aunque, a veces, simplemente anotar la salida para el gasto posterior de la billetera es suficiente, en cuyo caso la transacción que contiene la salida se considera que es su propia transacción *de resolución*.

Los resultados que han sido *resueltos* se consideran *resueltos irrevocablemente* una vez que la transacción de *resolución* del nodo remoto se incluye en un bloque de al menos 100 de profundidad, en la cadena de bloques de mayor trabajo. 100 bloques es mucho mayor que la bifurcación de Bitcoin más larga conocida y es el mismo tiempo de espera que se usa para las confirmaciones de las recompensas de los mineros (ver [Implementación de referencia](https://github.com/bitcoin/bitcoin/blob/4db82b7aab4ad64717f742a7318e3dc6811b41be/src/consensus/tx_verify.cpp#L223)).


<!-- omit in toc -->
## Requisitos

Un nodo:
  - una vez que haya transmitido una transacción de financiación O enviado una `commitment signature` para una `commitment transaction` que contenga un HTLC de salida:
    - hasta que todas las salidas sean *irrevocablemente resueltas*:
      - DEBE monitorizar la cadena de bloques en busca de transacciones que gasten cualquier salida que NO esté *resuelta irrevocablemente*.
  - DEBE *resolver* todas las salidas, como se especifica a continuación.
  - DEBE estar preparado para resolver salidas varias veces, en caso de reorganizaciones de la blockchain.
  - al gastarse la transacción de financiación, si el canal NO está ya cerrado:
    - PUEDE enviar un `error` descriptivo.
    - DEBE fallar el canal.
  - DEBERÍA ignorar las transacciones no válidas.

<!-- omit in toc -->
## Rational

Una vez que un nodo local tiene algunos fondos en juego, se requiere monitorear la cadena de bloques para garantizar que el nodo remoto no se cierre unilateralmente.

Cualquier persona puede generar transacciones no válidas (por ejemplo, firmas incorrectas) (y la cadena de bloques las ignorará de todos modos), por lo que no deberían desencadenar ninguna acción.

# Transacción de compromiso o `commitment transaction`

Los nodos local y remoto tienen cada uno una *transacción de compromiso*. Cada una de estas transacciones de compromiso tiene hasta seis tipos de salidas:

1. Salida principal del _nodo local_: Cero o una salida, a pagar al *nodo local* `delayed_pubkey`.
2. Salida principal del _nodo remoto_: Cero o una salida, a pagar al *nodo remoto* `delayed_pubkey`.
3. Salida ancla del _nodo local_: una salida que paga a la `funding_pubkey` del *nodo local*.
4. _salida ancla del nodo remoto_: una salida pagando a la `funding_pubkey` del *nodo remoto*.
5. HTLC ofrecidos por el _nodo local_: Cero o más pagos pendientes (*HTLCs*), para pagar al *nodo remoto* a cambio de una preimagen de pago.
6. HTLC ofrecidos por _nodo remoto_: Cero o más pagos pendientes (*HTLCs*), para pagar al *nodo local* a cambio de una preimagen de pago.

Para incentivar a los nodos locales y remotos a cooperar, un tiempo de espera relativo `OP_CHECKSEQUENCEVERIFY` grava las *salidas del nodo local* (en la *transacción de compromiso del nodo local*) y las *salidas del nodo remoto* (en la *transacción de compromiso del nodo remoto*) . Entonces, por ejemplo, si el nodo local publica su transacción de compromiso, tendrá que esperar para reclamar sus propios fondos, mientras que el nodo remoto tendrá acceso inmediato a sus propios fondos. Como consecuencia, las dos transacciones de compromiso no son idénticas, pero son (normalmente) simétricas.

Consulte [BOLT #3: Transacción de compromiso](03-transactions.md#commitment-transaction) para obtener más detalles.

# Fallando un Canal

Aunque el cierre de un canal se puede lograr de varias maneras, se prefiere la más eficiente.

Varios casos de error implican el cierre de un canal. Los requisitos para enviar mensajes de error a pares se especifican en [BOLT #1: The 'error' Message](01-messaging.md#the-error-message).

<!-- omit in toc -->
## Requisitos

Un nodo:
  - si una *transacción de compromiso local* NO ha contenido nunca un `to_local` o una salida HTLC:
    - PUEDE simplemente olvidar el canal.
  - de lo contrario:
    - si la *transacción de compromiso actual* NO contiene `to_local` u otras salidas HTLC:
      - PUEDE simplemente esperar a que el nodo remoto cierre el canal.
      - hasta que se cierre el nodo remoto:
        - NO DEBE olvidar el canal.
    - de lo contrario:
      - si ha recibido un mensaje `closing_signed` válido que incluye una tarifa suficiente:
        - DEBERÍA usar esta tarifa para realizar un *cierre mutuo*.
      - de lo contrario:
      - si el nodo sabe o asume que su estado de canal está desactualizado:
        - NO DEBE transmitir su *última transacción de compromiso*.
        - de lo contrario:
          - DEBE transmitir la *última transacción de compromiso*, para la cual tiene una firma, para realizar un *cierre unilateral*.
          - DEBE gastar cualquier salida `to_local_anchor`, proporcionando tarifas suficientes como incentivo para incluir la transacción de compromiso en un bloque.
          Se debe tener especial cuidado cuando se gasta a un tercero, porque esto vuelve a introducir la vulnerabilidad que se abordó al agregar el retraso de CSV a las salidas no `anchor`.
          - DEBE usar [replace-by-fee](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki) u otro mecanismo en la transacción de gasto si resulta insuficiente para la inclusión oportuna en un bloque .

<!-- omit in toc -->
## Racional

Dado que se supone que `dust_limit_satoshis` evita la creación de resultados no económicos (que de otro modo permanecerían para siempre, sin gastar en la cadena de bloques), todos los resultados de transacciones de compromiso DEBEN gastarse.

En las primeras etapas de un canal, es común que un lado tenga poco o ningún fondo en el canal; en este caso, al no tener nada en juego, un nodo no necesita consumir recursos monitorizando el estado del canal.

Existe un sesgo hacia la preferencia de los cierres mutuos sobre los cierres unilaterales, porque los resultados de los primeros no están gravados por un retraso y los monederos pueden gastarlos directamente. Además, las tarifas de cierre mutuo tienden a ser menos exageradas que las de las transacciones de compromiso (o en el caso de `option_anchors`, la transacción de compromiso puede requerir una transacción secundaria para que se extraiga). Por lo tanto, la única razón para no usar la firma de `closing_signed` sería si la tarifa ofrecida fuera demasiado pequeña para procesarla.

# Manejo de cierre mutuo

Una transacción de cierre *resuelve* el resultado de la transacción de financiación.

En el caso de un cierre mutuo, un nodo no necesita hacer nada más, ya que ya ha aceptado la salida, que se envía a su `scriptpubkey` especificado (ver [BOLT #2: Closing initiation: `shutdown`](02-peer-protocol.md#closing-initiation-shutdown)).

# Manejo de Cierre Unilateral: Transacción de Compromiso Local

Este es el primero de dos casos relacionados con cierres unilaterales. En este caso, un nodo descubre su *transacción de compromiso local*, que *resuelve* el resultado de la transacción de financiación.

Sin embargo, un nodo no puede reclamar fondos de las salidas de un cierre unilateral que inició, hasta que haya pasado el retraso `OP_CHECKSEQUENCEVERIFY` (según lo especificado por el campo `to_self_delay` del nodo remoto). En su caso, esta situación se indica a continuación.

<!-- omit in toc -->
## Requisitos

Un nodo:
  - al descubrir su *transacción de compromiso local*:
    - DEBE pasar la salida `to_local` a una dirección conveniente.
    - DEBE esperar hasta que haya pasado el retraso `OP_CHECKSEQUENCEVERIFY` (según lo especificado por el campo `to_self_delay` del nodo remoto) antes de gastar la salida.
      - Nota: si la salida se gasta (como se recomienda), la salida se *resuelve* mediante la transacción de gasto; de lo contrario, se considera *resuelta* por la propia transacción de compromiso.
    - PUEDE ignorar la salida `to_remote`.
      - Nota: El nodo local no requiere ninguna acción, ya que `to_remote` se considera *resuelto* por la propia transacción de compromiso.
    - DEBE manejar los HTLC ofrecidos por sí mismo como se especifica en [Manejo de salida de HTLC: compromiso local, ofertas locales] (#htlc-output-handling-local-commitment-local-offers).
    - DEBE manejar los HTLC ofrecidos por el nodo remoto como se especifica en [Manejo de salida HTLC: compromiso local, ofertas remotas] (#htlc-output-handling-local-commitment-remote-offers).

<!-- omit in toc -->
## Racional

Gastar la salida `to_local` evita tener que recordar el complicado script de testigo, asociado con ese canal en particular, para gastos posteriores.

La salida `to_remote` es enteramente asunto del nodo remoto y se puede ignorar.

## Manejo de salida de HTLC: compromiso local, ofertas locales

Cada salida de HTLC solo puede ser gastada por el *oferente local*, mediante el uso de la transacción de tiempo de espera de HTLC después de que se agote el tiempo, o el *destinatario remoto*, si tiene la preimagen de pago.

Puede haber HTLC que no estén representados por ninguna salida: ya sea porque se recortaron como polvo o porque la transacción solo se consolidó parcialmente.

La salida de HTLC ha *excedido el tiempo de espera* una vez que la altura del último bloque es igual o mayor que el `cltv_expiry` del HTLC.

<!-- omit in toc -->
### Requisitos

A node:
  - if the commitment transaction HTLC output is spent using the payment preimage, the output is considered *irrevocably resolved*:
    - MUST extract the payment preimage from the transaction input witness.
  - if the commitment transaction HTLC output has *timed out* and hasn't been *resolved*:
    - MUST *resolve* the output by spending it using the HTLC-timeout transaction.
    - once the resolving transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
      - MUST resolve the output of that HTLC-timeout transaction.
      - SHOULD resolve the HTLC-timeout transaction by spending it to a convenient address.
        - Note: if the output is spent (as recommended), the output is *resolved* by the spending transaction, otherwise it is considered
        *resolved* by the HTLC-timeout transaction itself.
      - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified by the remote node's `open_channel` `to_self_delay` field)
      before spending that HTLC-timeout output.
  - for any committed HTLC that does NOT have an output in this commitment transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - if no *valid* commitment transaction contains an output corresponding to the HTLC.
      - MAY fail the corresponding incoming HTLC sooner.

Un nodo:
  - si la salida de HTLC de la transacción de compromiso se gasta usando la preimagen de pago, la salida se considera *resuelta irrevocablemente*:
    - DEBE extraer la preimagen de pago de la `witness` de entrada de la transacción.
  - si la salida HTLC de la transacción de compromiso *ha agotado el tiempo de espera* y no se ha *resuelto*:
    - DEBE *resolver* la salida gastándola usando la transacción de tiempo de espera de HTLC.
    - una vez que la transacción de resolución ha alcanzado una profundidad razonable:
      - DEBE fallar el HTLC entrante correspondiente (si lo hay).
      - DEBE resolver la salida de esa transacción de tiempo de espera de HTLC.
      - DEBERÍA resolver la transacción de tiempo de espera de HTLC gastándola en una dirección conveniente.
        - Nota: si la salida se gasta (como se recomienda), la salida se *resuelve* mediante la transacción de gasto; de lo contrario, se considera *resuelta* por la propia transacción de tiempo de espera de HTLC.
      - DEBE esperar hasta que haya pasado el retraso `OP_CHECKSEQUENCEVERIFY` (según lo especificado por el campo `open_channel` `to_self_delay` del nodo remoto) antes de gastar esa salida de tiempo de espera de HTLC.
  - para cualquier HTLC comprometido que NO tenga una salida en esta transacción de compromiso:
    - una vez que la transacción de compromiso ha alcanzado una profundidad razonable:
      - DEBE fallar el HTLC entrante correspondiente (si lo hay).
    - si ninguna transacción de compromiso *válida* contiene una salida correspondiente al HTLC.
      - PUEDE fallar antes el HTLC entrante correspondiente.


<!-- omit in toc -->
### Racional

La preimagen de pago sirve para probar el pago (cuando el nodo de oferta originó el pago) o para canjear el HTLC entrante correspondiente de otro par (cuando el nodo de oferta está reenviando el pago). Una vez que un nodo ha extraído el pago, ya no le importa el destino de la transacción de gasto de HTLC en sí.

En los casos en los que ambas resoluciones son posibles (p. ej., cuando un nodo recibe un pago exitoso después del tiempo de espera), cualquiera de las interpretaciones es aceptable; es responsabilidad del destinatario gastarlo antes de que esto ocurra.

La transacción de tiempo de espera de HTLC local debe usarse para agotar el tiempo de espera del HTLC (para evitar que el nodo remoto lo complete y reclame los fondos) antes de que el nodo local pueda hacer retroceder cualquier HTLC entrante correspondiente, usando `update_fail_htlc` (presumiblemente por la razón `permanent_channel_failure`), como se detalla en [BOLT #2](02-peer-protocol.md#forwarding-htlcs).
Si el HTLC entrante también está en la cadena, un nodo simplemente debe esperar a que se agote el tiempo de espera: no hay forma de señalar una fallo temprano.

Si un HTLC es demasiado pequeño para aparecer en *cualquier transacción de compromiso*, se puede fallar de manera segura de inmediato. De lo contrario, si un HTLC no está en la *transacción de compromiso local*, un nodo debe asegurarse de que una reorganización de la blockchain, o `race`, no cambie a una transacción de compromiso que contenga el HTLC antes de que el nodo falle (de ahí la espera). El requisito de que el HTLC entrante falle antes de su propio tiempo de espera, aún se aplica como límite superior.

## Manejo de salida de HTLC: compromiso local, ofertas remotas

Cada resultado de HTLC solo puede ser gastado por el destinatario, utilizando la transacción `HTLC-success`, que solo puede completar si tiene la preimagen de pago. Si no tiene la preimagen (y no la descubre), es responsabilidad del oferente gastar la salida HTLC una vez que se agote el tiempo.

Hay varios casos posibles para un HTLC ofrecido:

1. El oferente NO se compromete irrevocablemente a ello. Por lo general, el destinatario no conocerá la preimagen, ya que no reenviará los HTLC hasta que estén completamente confirmados. Entonces, usar la preimagen revelaría que este destinatario es el salto final; por lo tanto, en este caso, es mejor permitir que el HTLC se agote.
2. El oferente está irrevocablemente comprometido con el HTLC ofrecido, pero el destinatario aún no se ha comprometido con un HTLC saliente. En este caso, el destinatario puede reenviar o agotar el tiempo de espera del HTLC ofrecido.
3. El destinatario se ha comprometido con un HTLC saliente, a cambio del HTLC ofrecido. En este caso, el destinatario deberá utilizar la preimagen, una vez que la reciba del HTLC saliente; de lo contrario, perderá fondos al enviar un pago saliente sin canjear el pago entrante.

<!-- omit in toc -->
### Requisitos

Un nodo local:
  - si recibe (o ya posee) una preimagen de pago por una salida HTLC no resuelta que se le ha ofrecido Y para la cual se ha comprometido con un HTLC saliente:
    - DEBE *resolver* la salida gastándola, utilizando la transacción HTLC-success.
    - NO DEBE revelar su propia preimagen cuando no es el destinatario final.<sup>[Preimage-Extraction](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-October/002857.html)</sup >
    - DEBE resolver el resultado de esa transacción HTLC-success.
  - de lo contrario:
    - si el *nodo remoto* NO está irrevocablemente comprometido con el HTLC:
      - NO DEBE *resolver* la salida gastándola.
  - DEBERÍA resolver esa salida de transacción HTLC-success gastándola en una dirección conveniente.
  - DEBE esperar hasta que haya pasado el retraso `OP_CHECKSEQUENCEVERIFY` (según lo especificado por el campo `to_self_delay` del *nodo remoto* `open_channel`), antes de gastar esa salida de transacción HTLC-success.

Si se gasta la salida (como se recomienda), la transacción de gasto *resuelve* la salida; de lo contrario, se considera *resuelta* por la propia transacción HTLC-success.

Si NO se resuelve de otra manera, una vez que la salida HTLC haya expirado, se considera *resuelto irrevocablemente*.

# Manejo cercano unilateral: Transacción de compromiso remoto

La transacción de compromiso del *nodo remoto* *resuelve* el resultado de la transacción de financiación.

En este caso, no hay demoras que restrinjan el comportamiento del nodo, por lo que es más fácil de manejar para un nodo que en el caso en que descubre su transacción de compromiso local (consulte [Manejo de Cierre Unilateral: Transacción de Compromiso Local](#manejo-de-cierre-unilateral-transacción-de-compromiso-local)).

<!-- omit in toc -->
## Requirements

A local node:
  - upon discovering a *valid* commitment transaction broadcast by a *remote node*:
    - if possible:
      - MUST handle each output as specified below.
      - MAY take no action in regard to the associated `to_remote`, which is simply a P2WPKH output to the *local node*.
        - Note: `to_remote` is considered *resolved* by the commitment transaction itself.
      - MAY take no action in regard to the associated `to_local`, which is a payment output to the *remote node*.
        - Note: `to_local` is considered *resolved* by the commitment transaction itself.
      - MUST handle HTLCs offered by itself as specified in [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
      - MUST handle HTLCs offered by the remote node as specified in [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
    - otherwise (it is NOT able to handle the broadcast for some reason):
      - MUST inform the user of potentially lost funds.

<!-- omit in toc -->
## Rationale

There may be more than one valid, *unrevoked* commitment transaction after a signature has been received via `commitment_signed` and before the corresponding `revoke_and_ack`. As such, either commitment may serve as the *remote node's* commitment transaction; hence, the local node is required to handle both.

In the case of data loss, a local node may reach a state where it doesn't recognize all of the *remote node's* commitment transaction HTLC outputs. It can detect the data loss state, because it has signed the transaction, and the commitment number is greater than expected. If both nodes support `option_data_loss_protect`, the local node will possess the remote's `per_commitment_point`, and thus can derive its own `remotepubkey` for the transaction, in order to salvage its own funds. Note: in this scenario, the node will be unable to salvage the HTLCs.

## HTLC Output Handling: Remote Commitment, Local Offers

Each HTLC output can only be spent by either the *local offerer*, after it's timed out, or by the *remote recipient*, by using the HTLC-success transaction if it has the payment preimage.

There can be HTLCs which are not represented by any outputs: either because the outputs were trimmed as dust, or because the remote node has two *valid* commitment transactions with differing HTLCs.

The HTLC output has *timed out* once the depth of the latest block is equal to or greater than the HTLC `cltv_expiry`.

<!-- omit in toc -->
### Requirements

A local node:
  - if the commitment transaction HTLC output is spent using the payment preimage:
    - MUST extract the payment preimage from the HTLC-success transaction input witness.
      - Note: the output is considered *irrevocably resolved*.
  - if the commitment transaction HTLC output has *timed out* AND NOT been *resolved*:
    - MUST *resolve* the output, by spending it to a convenient address.
  - for any committed HTLC that does NOT have an output in this commitment transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - otherwise:
      - if no *valid* commitment transaction contains an output corresponding to the HTLC:
        - MAY fail it sooner.


<!-- omit in toc -->
### Rationale

If the commitment transaction belongs to the *remote* node, the only way for it to spend the HTLC output (using a payment preimage) is for it to use the HTLC-success transaction.

The payment preimage either serves to prove payment (when the offering node is the originator of the payment) or to redeem the corresponding incoming HTLC from another peer (when the offering node is forwarding the payment). After a node has extracted the payment, it no longer need be concerned with the fate of the HTLC-spending transaction itself.

In cases where both resolutions are possible (e.g. when a node receives payment success after timeout), either interpretation is acceptable: it's the responsibility of the recipient to spend it before this occurs.

Once it has timed out, the local node needs to spend the HTLC output (to prevent the remote node from using the HTLC-success transaction) before it can back-fail any corresponding incoming HTLC, using `update_fail_htlc` (presumably with reason `permanent_channel_failure`), as detailed in [BOLT #2](02-peer-protocol.md#forwarding-htlcs).
If the incoming HTLC is also on-chain, a node simply waits for it to timeout, as there's no way to signal early failure.

If an HTLC is too small to appear in *any commitment transaction*, it can be safely failed immediately. Otherwise,
if an HTLC isn't in the *local commitment transaction* a node needs to make sure that a blockchain reorganization or race does not switch to a commitment transaction that does contain it before the node fails it: hence the wait. The requirement that the incoming HTLC be failed before its own timeout still applies as an upper bound.

## HTLC Output Handling: Remote Commitment, Remote Offers

The remote HTLC outputs can only be spent by the local node if it has the payment preimage. If the local node does not have the preimage (and doesn't discover it), it's the remote node's responsibility to spend the HTLC output once it's timed out.

There are actually several possible cases for an offered HTLC:

1. The offerer is not irrevocably committed to it. In this case, the recipient usually won't know the preimage, since it won't forward HTLCs until they're fully committed. As using the preimage would reveal that this recipient is the final hop, it's best to allow the HTLC to time out.
2. The offerer is irrevocably committed to the offered HTLC, but the recipient hasn't yet committed to an outgoing HTLC. In this case, the recipient can either forward it or wait for it to timeout.
3. The recipient has committed to an outgoing HTLC in exchange for an offered HTLC. In this case, the recipient must use the preimage, if it receives it from the outgoing HTLC; otherwise, it will lose funds by sending an outgoing payment without redeeming the incoming one.

### Requirements

A local node:
  - if it receives (or already possesses) a payment preimage for an unresolved HTLC output that it was offered AND for which it has committed to an outgoing HTLC:
    - MUST *resolve* the output by spending it to a convenient address.
  - otherwise:
    - if the remote node is NOT irrevocably committed to the HTLC:
      - MUST NOT *resolve* the output by spending it.

If not otherwise resolved, once the HTLC output has expired, it is considered *irrevocably resolved*.

# Revoked Transaction Close Handling

If any node tries to cheat by broadcasting an outdated commitment transaction (any previous commitment transaction besides the most current one), the other node in the channel can use its revocation private key to claim all the funds from the
channel's original funding transaction.

## Requirements

Once a node discovers a commitment transaction for which *it* has a revocation private key, the funding transaction output is *resolved*.

A local node:
  - MUST NOT broadcast a commitment transaction for which *it* has exposed the `per_commitment_secret`.
  - MAY take no action regarding the _local node's main output_, as this is a simple P2WPKH output to itself.
    - Note: this output is considered *resolved* by the commitment transaction itself.
  - MUST *resolve* the _remote node's main output_ by spending it using the revocation private key.
  - MUST *resolve* the _remote node's offered HTLCs_ in one of three ways:
    * spend the *commitment tx* using the payment revocation private key.
    * spend the *commitment tx* using the payment preimage (if known).
    * spend the *HTLC-timeout tx*, if the remote node has published it.
  - MUST *resolve* the _local node's offered HTLCs_ in one of three ways:
    * spend the *commitment tx* using the payment revocation private key.
    * spend the *commitment tx* once the HTLC timeout has passed.
    * spend the *HTLC-success tx*, if the remote node has published it.
  - MUST *resolve* the _remote node's HTLC-timeout transaction_ by spending it using the revocation private key.
  - MUST *resolve* the _remote node's HTLC-success transaction_ by spending it using the revocation private key.
  - SHOULD extract the payment preimage from the transaction input witness, if it's not already known.
  - if `option_anchors` applies:
    - MAY use a single transaction to *resolve* all the outputs.
    - if confirmation doesn't happen before reaching `security_delay` blocks from expiry:
      - SHOULD *resolve* revoked outputs in their own, separate penalty transactions. A previous penalty transaction claiming multiple revoked outputs at once may be blocked from confirming because of a transaction pinning attack.
  - otherwise:
    - MAY use a single transaction to *resolve* all the outputs.
  - MUST handle its transactions being invalidated by HTLC transactions.

## Rationale

A single transaction that resolves all the outputs will be under the standard size limit because of the 483 HTLC-per-party limit (see [BOLT #2](02-peer-protocol.md#the-open_channel-message)).

Note: if `option_anchors` applies, the cheating node can pin spends of its HTLC-timeout/HTLC-success outputs thanks to SIGHASH_SINGLE malleability.
Using a single penalty transaction for all revoked outputs is thus unsafe as it could be blocked to propagate long enough for the _local node's `to_local` output_ 's relative locktime to expire and the cheating party escaping the penalty on this
output. Though this situation doesn't prevent faithful punishment of the second-level revoked output if the pinning transaction confirms.

The `security_delay` is a fixed-point relative to the absolute expiration of the revoked output at which the punishing node must broadcast a single-spend transaction for the revoked output and actively fee-bump it until its confirmation.
The exact value of `security_delay` is left as a matter of node policy, though we recommend 18 blocks (similar to incoming HTLC deadline).

## Penalty Transactions Weight Calculation

There are three different scripts for penalty transactions, with the following witness weights (details of weight computation are in [Appendix A](#appendix-a-expected-weights)):

    to_local_penalty_witness: 160 bytes
    offered_htlc_penalty_witness: 243 bytes
    accepted_htlc_penalty_witness: 249 bytes

The penalty *txinput* itself takes up 41 bytes and has a weight of 164 bytes, which results in the following weights for each input:

    to_local_penalty_input_weight: 324 bytes
    offered_htlc_penalty_input_weight: 407 bytes
    accepted_htlc_penalty_input_weight: 413 bytes

The rest of the penalty transaction takes up 4+1+1+8+1+34+4=53 bytes of non-witness data: assuming it has a pay-to-witness-script-hash (the largest standard output script), in addition to a 2-byte witness header.

In addition to spending these outputs, a penalty transaction may optionally spend the commitment transaction's `to_remote` output (e.g. to reduce the total amount paid in fees). Doing so requires the inclusion of a P2WPKH witness and an
additional *txinput*, resulting in an additional 108 + 164 = 272 bytes.

In the worst case scenario, the node holds only incoming HTLCs, and the HTLC-timeout transactions are not published, which forces the node to spend from the commitment transaction.

With a maximum standard weight of 400000 bytes, the maximum number of HTLCs that can be swept in a single transaction is as follows:

    max_num_htlcs = (400000 - 324 - 272 - (4 * 53) - 2) / 413 = 966

Thus, 483 bidirectional HTLCs (containing both `to_local` and `to_remote` outputs) can be resolved in a single penalty transaction.
Note: even if the `to_remote` output is not swept, the resulting `max_num_htlcs` is 967; which yields the same unidirectional limit of 483 HTLCs.

# Generation of HTLC Transactions

If `option_anchors` does not apply to the commitment transaction, then HTLC-timeout and HTLC-success transactions are complete transactions with (hopefully!) reasonable fees and must be used directly.

Otherwise, `SIGHASH_SINGLE|SIGHASH_ANYONECANPAY` MUST be used on the HTLC signatures received from the peer, as this allows HTLC transactions to be combined with other transactions.  The local signature MUST use `SIGHASH_ALL`, otherwise
anyone can attach additional inputs and outputs to the tx.

If `option_anchors_zero_fee_htlc_tx` applies, then the HTLC-timeout and HTLC-success transactions are signed with the input and output having the same value. This means they have a zero fee and MUST be combined with other inputs
to arrive at a reasonable fee.

## Requirements

A node which broadcasts an HTLC-success or HTLC-timeout transaction for a commitment transaction:
  1. if `option_anchor_outputs` applies:
    - SHOULD combine it with inputs contributing sufficient fee to ensure timely inclusion in a block.
    - MAY combine it with other transactions.
  2. if `option_anchors_zero_fee_htlc_tx` applies:
    - MUST combine it with inputs contributing sufficient fee to ensure timely inclusion in a block.
    - MAY combine it with other transactions.

Note that `option_anchors_zero_fee_htlc_tx` has a stronger requirement for adding inputs to the final transactions than `option_anchor_outputs`, since the HTLC-success and HTLC-timeout transactions won't propagate without additional
inputs added.

# General Requirements

A node:
  - upon discovering a transaction that spends a funding transaction output which does not fall into one of the above categories (mutual close, unilateral close, or revoked transaction close):
    - MUST warn the user of potentially lost funds.
      - Note: the existence of such a rogue transaction implies that its private key has leaked and that its funds may be lost as a result.
  - MAY simply monitor the contents of the most-work chain for transactions.
    - Note: on-chain HTLCs should be sufficiently rare that speed need not be considered critical.
  - MAY monitor (valid) broadcast transactions (a.k.a the mempool).
    - Note: watching for mempool transactions should result in lower latency HTLC redemptions.

# Appendix A: Expected Weights

## Expected Weight of the `to_local` Penalty Transaction Witness

As described in [BOLT #3](03-transactions.md), the witness for this transaction is:

    <sig> 1 { OP_IF <revocationpubkey> OP_ELSE to_self_delay OP_CSV OP_DROP <local_delayedpubkey> OP_ENDIF OP_CHECKSIG }

The *expected weight* of the `to_local` penalty transaction witness is calculated as follows:

    to_local_script: 83 bytes
        - OP_IF: 1 byte
            - OP_DATA: 1 byte (revocationpubkey length)
            - revocationpubkey: 33 bytes
        - OP_ELSE: 1 byte
            - OP_DATA: 1 byte (delay length)
            - delay: 8 bytes
            - OP_CHECKSEQUENCEVERIFY: 1 byte
            - OP_DROP: 1 byte
            - OP_DATA: 1 byte (local_delayedpubkey length)
            - local_delayedpubkey: 33 bytes
        - OP_ENDIF: 1 byte
        - OP_CHECKSIG: 1 byte

    to_local_penalty_witness: 160 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - one_length: 1 byte
        - witness_script_length: 1 byte
        - witness_script (to_local_script)

## Expected Weight of the `offered_htlc` Penalty Transaction Witness

The *expected weight* of the `offered_htlc` penalty transaction witness is calculated as follows (some calculations have already been made in [BOLT #3](03-transactions.md)):

    offered_htlc_script: 133 bytes

    offered_htlc_penalty_witness: 243 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocation_key_length: 1 byte
        - revocation_key: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (offered_htlc_script)

## Expected Weight of the `accepted_htlc` Penalty Transaction Witness

The *expected weight*  of the `accepted_htlc` penalty transaction witness is calculated as follows (some calculations have already been made in [BOLT #3](03-transactions.md)):

    accepted_htlc_script: 139 bytes

    accepted_htlc_penalty_witness: 249 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocationpubkey_length: 1 byte
        - revocationpubkey: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (accepted_htlc_script)

# Authors

[FIXME:]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
