# BOLT #9: `Feature Flags` Asignados

Este documento rastrea la asignación de `features flags` en el mensaje `init` ([BOLT #1](01-messaging.md)), así como campos `features` en los mensajes `channel_announcement` y `node_announcement` ([BOLT #7](07-routing-gossip.md)). Los `flags` se rastrean por separado, ya que es probable que se agreguen nuevos `flags` con el tiempo.

Los indicadores se numeran desde el bit menos significativo, en el bit 0 (es decir, 0x1, un bit _par_). Por lo general, se asignan en pares para que las funciones se puedan introducir como opcionales (_bits_ impares) y luego actualizarse para que sean obligatorias (_bits_ pares), que serán rechazadas por los nodos obsoletos: consulte [BOLT #1: The `init` Message]( 01-messaging.md#the-init-message).

Algunas funciones no tienen sentido por canal o por nodo, por lo que cada función define cómo se presenta en esos contextos. Es posible que se requieran algunas funciones para abrir un canal, pero no un requisito para el uso del canal, por lo que la presentación de esas funciones depende de la función en sí.

La columna Contexto se decodifica de la siguiente manera:

* `I`: presentado en el mensaje `init`.
* `N`: presentado en los mensajes `node_announcement`. 
* `C`: presentado en el mensaje `channel_announcement`.
* `C-`: presentado en el mensaje `channel_announcement`, pero siempre impar (opcional).
* `C+`: presentado en el mensaje `channel_announcement`, pero siempre par (requerido).
* `9`: presentado en las facturas [BOLT 11](11-payment-encoding.md).
* `B`: presentado en el campo `allowed_features` de una ruta `blinded`.

| Bits  | Nombre                            | Descrición                                                                     | Contexto | Dependencias              | Enlace                                                                |
|-------|-----------------------------------|--------------------------------------------------------------------------------|----------|---------------------------|-----------------------------------------------------------------------|
| 0/1   | `option_data_loss_protect`        | Requiere o admite campos `channel_reestablish` adicionales                     | IN       |                           | [BOLT #2][bolt02-retransmit]                                          |
| 3     | `initial_routing_sync`            | Quien envío necesita un volcado de información de enrutamiento completo        | I        |                           | [BOLT #7][bolt07-sync]                                                |
| 4/5   | `option_upfront_shutdown_script`  | Se compromete con un scriptpubkey de apagado al abrir el canal                 | IN       |                           | [BOLT #2][bolt02-open]                                                |
| 6/7   | `gossip_queries`                  | Control de chismes más sofisticado                                             | IN       |                           | [BOLT #7][bolt07-query]                                               |
| 8/9   | `var_onion_optin`                 | Requiere/admite cargas útiles de cebolla de enrutamiento de longitud variable  | IN9      |                           | [Routing Onion Specification][bolt04]                                 |
| 10/11 | `gossip_queries_ex`               | Las consultas de chismes pueden incluir información adicional                  | IN       | `gossip_queries`          | [BOLT #7][bolt07-query]                                               |
| 12/13 | `option_static_remotekey`         | Clave estática para salida remota                                              | IN       |                           | [BOLT #3](03-transactions.md)                                         |
| 14/15 | `payment_secret`                  | El nodo admite el campo `payment_secret`                                       | IN9      | `var_onion_optin`         | [Routing Onion Specification][bolt04]                                 |
| 16/17 | `basic_mpp`                       | El nodo puede recibir pagos multiparte básicos                                 | IN9      | `payment_secret`          | [BOLT #4][bolt04-mpp]                                                 |
| 18/19 | `option_support_large_channel`    | Puede crear canales grandes                                                    | IN       |                           | [BOLT #2](02-peer-protocol.md#the-open_channel-message)               |
| 20/21 | `option_anchor_outputs`           | `Anchor outputs`                                                               | IN       | `option_static_remotekey` | [BOLT #3](03-transactions.md)                                         |
| 22/23 | `option_anchors_zero_fee_htlc_tx` | `Anchor commitment` Tipo con transacciones HTLC de tarifa cero                 | IN       | `option_static_remotekey` | [BOLT #3][bolt03-htlc-tx], [lightning-dev][ml-sighash-single-harmful] |
| 24/25 | `option_route_blinding`           | El nodo admite rutas ciegas                                                    | IN9      | `var_onion_optin`         | [BOLT #4](bolt04-route-blinding)                                      |
| 26/27 | `option_shutdown_anysegwit`       | Futuras versiones de segwit permitidas en `shutdown`                           | IN       |                           | [BOLT #2][bolt02-shutdown]                                            |
| 44/45 | `option_channel_type`             | El nodo admite el campo `channel_type` en abrir/aceptar                        | IN       |                           | [BOLT #2](02-peer-protocol.md#the-open_channel-message)               |
| 46/47 | `option_scid_alias`               | Suministro del alias ​​del canal para enrutamiento                               | IN       |                           | [BOLT #2][bolt02-channel-ready]                                       |
| 48/49 | `option_payment_metadata`         | Metadatos de pago en registro tlv                                              | 9        |                           | [BOLT #11](11-payment-encoding.md#tagged-fields)                      |
| 50/51 | `option_zeroconf`                 | Entiende los tipos de canales de zeroconf                                      | IN       | `option_scid_alias`       | [BOLT #2][bolt02-channel-ready]                                       |

## Definiciones

Nosotros definimos `option_anchors` como `option_anchor_outputs || option_anchors_zero_fee_htlc_tx`.

## Requisitos

El nodo de origen:
  * Si es compatible con una función anterior, DEBERÍA establecer el bit impar correspondiente en todos los campos de función indicados por la columna Contexto, a menos que se indique que debe configurar el bit de función par en su lugar.
  * Si requiere una función anterior, DEBE configurar el bit de función par correspondiente en todos los campos de función indicados por la columna Contexto, a menos que se indique que debe configurar el bit de función impar en su lugar.
  * NO DEBE establecer bits de función que no admita.
  * NO DEBE establecer bits de función en campos no especificados en la tabla anterior.
  * DEBE establecer todas las dependencias de características transitivas.


El nodo de origen DEBE soportar:
  * `var_onion_optin`

Los requisitos para recibir bits específicos se definen en las secciones vinculadas de la tabla anterior.
Los requisitos para los bits de función que no están definidos anteriormente se pueden encontrar en [BOLT #1: The `init` Message](01-messaging.md#the-init-message).

## Racional

No hay un bit _par_ para `initial_routing_sync`, ya que no tendría mucho sentido: un nodo local no puede determinar si un nodo remoto cumple, y debe interpretar el `flag`, como se define en la especificación inicial.

Tenga en cuenta que para los `feature flags` que están disponibles en los contextos de factura `node_announcement` y [BOLT 11](11-payment-encoding.md), las funciones establecidas en [BOLT 11](11-payment-encoding.md) la factura debe anular los establecidos en `node_announcement`. Esto mantiene las cosas consistentes con el comportamiento de las `features` desconocidas como se especifica en [BOLT 7](07-routing-gossip.md#the-node_announcement-message).

El origen debe establecer todas las dependencias de características transitivas para crear un vector de características bien formado. Al validar todas las dependencias conocidas por adelantado, esto simplifica la puerta lógica en un solo bit de función; se sabe que las dependencias de la `feature` están establecidas y no es necesario validarlas en cada entrada de `feature`.

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

[bolt02-retransmit]: 02-peer-protocol.md#message-retransmission
[bolt02-open]: 02-peer-protocol.md#the-open_channel-message
[bolt03-htlc-tx]: 03-transactions.md#htlc-timeout-and-htlc-success-transactions
[bolt02-shutdown]: 02-peer-protocol.md#closing-initiation-shutdown
[bolt02-channel-ready]: 02-peer-protocol.md#the-channel_ready-message
[bolt04]: 04-onion-routing.md
[bolt07-sync]: 07-routing-gossip.md#initial-sync
[bolt07-query]: 07-routing-gossip.md#query-messages
[bolt04-mpp]: 04-onion-routing.md#basic-multi-part-payments
[bolt04-route-blinding]: 04-onion-routing.md#route-blinding
[ml-sighash-single-harmful]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-September/002796.html
