# BOLT #10: DNS Bootstrap y Ubicación asistida de Nodos

## Visión general

Esta especificación describe un mecanismo de descubrimiento de nodos basado en el Sistema de nombres de dominio (DNS).
Su finalidad es doble:

 - Bootstrap: proporciona el descubrimiento de nodo inicial para nodos que no tienen contactos conocidos en la red
 - Ubicación de nodo asistida: nodos de apoyo en el descubrimiento de la dirección de red actual de pares previamente conocidos

Un servidor de nombres de dominio que implementa esta especificación se denomina _DNS Seed_ y responde a las consultas DNS entrantes de tipo `A`, `AAAA` o `SRV`, como se especifica en RFC 1035<sup>[1](#ref-1 )</sup>, 3596<sup>[2](#ref-2)</sup> y 2782<sup>[3](#ref-3)</sup>, respectivamente.
El servidor DNS tiene autoridad para un subdominio, denominado _dominio raíz inicial_, y los clientes pueden consultarlo en busca de subdominios.

Los subdominios constan de una serie de _condiciones_ separadas por puntos que limitan aún más los resultados deseados.

## Índice

- [BOLT #10: DNS Bootstrap y Ubicación asistida de Nodos](#bolt-10-dns-bootstrap-y-ubicación-asistida-de-nodos)
  - [Visión general](#visión-general)
  - [Índice](#índice)
  - [Consultas de semillas de DNS](#consultas-de-semillas-de-dns)
    - [Semántica de las consultas](#semántica-de-las-consultas)
    - [Requisitos](#requisitos)
    - [Contrucción de respuesta](#contrucción-de-respuesta)
  - [Policies](#policies)
  - [Ejemplos](#ejemplos)
  - [Referencias](#referencias)
  - [Autores](#autores)

## Consultas de semillas de DNS

Un cliente PUEDE emitir consultas utilizando los tipos de consulta `A`, `AAAA` o `SRV`, especificando las condiciones para los resultados deseados que debe devolver la semilla.

Las consultas distinguen entre consultas _wildcard_ y consultas _node_, dependiendo de si la `l`-key está configurada o no.

### Semántica de las consultas

Las condiciones son pares clave-valor: la clave es una sola letra, mientras que el resto del par clave-valor es el valor.
Los siguientes pares clave-valor DEBEN ser compatibles con una semilla DNS:

 - `r`: byte de `realm`
   - se usa para especificar qué dominio deben admitir los nodos devueltos
   - valor por defecto: 0 (Bitcoin)
 - `a`: tipos de direcciones
   - un campo de bits que utiliza los tipos de [BOLT #7](07-routing-gossip.md) como índice de bits
   - se usa para especificar qué tipos de direcciones deben devolverse para las consultas `SRV`
   - PUEDE usarse solo para consultas `SRV`
   - valor predeterminado: 6 (es decir, `2 || 4`, ya que el bit 1 y el bit 2 están configurados para IPv4 y
     IPv6, respectivamente)
 - `l`: `id_nodo`
   - un `node_id` codificado en bech32 de un nodo específico
   - se utiliza para solicitar un solo nodo en lugar de una selección aleatoria
   - valor predeterminado: nulo
 - `n`: número de registros de respuesta deseados
   - valor predeterminado: 25

Las condiciones se pasan en la consulta inicial de DNS como componentes de subdominio individuales separados por puntos.

Por ejemplo, una consulta de `r0.a2.n10.lseed.bitcoinstats.com` implicaría: devolver 10 (`n10`) registros IPv4 (`a2`) para nodos compatibles con Bitcoin (`r0`).

### Requisitos

La semilla DNS:
  - DEBE evaluar las condiciones desde el _dominio raíz semilla_ 'subiendo por el árbol' (`going up-the-tree`), es decir, evaluando de derecha a izquierda en un nombre de dominio completo.
    - P.ej. para evaluar el caso anterior: primero evalúe `n10`, luego `a2` y finalmente `r0`.
  - si una condición (clave) se especifica más de una vez:
    - DEBE descartar cualquier valor anterior para esa condición Y usar el nuevo valor en su lugar.
      - P.ej. para `n5.r0.a2.n10.lseed.bitcoinstats.com`, el resultado es:
      ~~`n10`~~, `a2`, `r0`, `n5`.
  - DEBE devolver resultados que coincidan con todas las condiciones.
  - si NO implementa filtrado por una condición dada:
    - PUEDE ignorar la condición por completo (es decir, el filtrado de semillas es solo el mayor esfuerzo posible).
  - para consultas `A` y `AAAA`:
    - DEBE devolver solo los nodos que escuchan en el puerto predeterminado 9735, como se define en
    [BOLT #1](01-messaging.md).
  - para consultas `SRV`:
    - PUEDEN devolver nodos que están escuchando en puertos no predeterminados, ya que los registros `SRV` devuelven una tupla _(nombre de host, puerto)_.
  - al recibir una consulta _wildcard_:
    - DEBE seleccionar un subconjunto aleatorio de hasta `n` direcciones IPv4 o IPv6 de nodos que están escuchando conexiones entrantes.
  - al recibir una consulta de _nodo_:
    - DEBE seleccionar el registro que coincida con `node_id`, si corresponde, Y devolver todas las direcciones asociadas con ese nodo.

Consultando clientes:
  - NO DEBE depender de que los resultados cumplan una determinada condición.

### Contrucción de respuesta

Los resultados se serializan en una respuesta con un tipo de consulta que coincide con el tipo de consulta del cliente. Por ejemplo, las consultas `A`, `AAAA` y `SRV` resultan en respuestas `A`, `AAAA` y `SRV`, respectivamente. Además, las respuestas pueden ser ampliadas con registros adicionales (por ejemplo, para agregar registros `A` o `AAAA` que coincidan con los registros `SRV` devueltos).

Para las consultas `A` y `AAAA`, la respuesta contiene el nombre de dominio y la dirección IP de los resultados.

El nombre de dominio DEBE coincidir con el dominio de la consulta para no ser filtrado por resolutores intermedios.

Para las consultas `SRV`, la respuesta consiste en tuplas de (_nombres de host virtuales_, puerto).
Un nombre de host virtual es un subdominio del dominio raíz de la semilla que identifica de manera única un nodo en la red.
Se construye agregando la condición `node_id` al dominio raíz de la semilla.

La semilla DNS:
  - PUEDE devolver adicionalmente los registros `A` y `AAAA` correspondientes que indican la dirección IP de las entradas `SRV` en la sección adicional de la respuesta.
  - PUEDE omitir estos registros adicionales al detectar una consulta repetida.
    - Razón: debido al gran tamaño de la respuesta resultante, esta puede ser descartada por nodos intermedios que resuelvan los nombres.
  - si no hay entradas que cumplan todas las condiciones:
      - DEBE devolver una respuesta vacía.


## Policies

The DNS seed:
  - MUST NOT return replies with a TTL less than 60 seconds.
  - MAY filter nodes from its local views for various reasons, including faulty nodes, flaky nodes, or spam prevention.
  - MUST reply to random queries (i.e. queries to the seed root domain and to the `_nodes._tcp.` alias for `SRV` queries) with _random and unbiased_ samples from the set of all known good nodes, in accordance with the Bitcoin DNS Seed policy<sup>[4](#ref-4)</sup>.


El DNS seed:
  - NO DEBE devolver respuestas con un TTL inferior a 60 segundos.
  - PUEDE filtrar nodos de sus vistas locales por varias razones, incluyendo nodos defectuosos, nodos inestables o prevención de spam.
  - DEBE responder a consultas aleatorias (es decir, consultas al dominio raíz de la semilla y al alias `_nodes._tcp.` para consultas `SRV`) con muestras _aleatorias e imparciales_ del conjunto de todos los nodos conocidos y buenos, de acuerdo con la política de semillas DNS de Bitcoin<sup>[4](#ref-4)</sup>."

## Ejemplos

Realizando una consulta de registros AAAA:

	$ dig lseed.bitcoinstats.com AAAA
	lseed.bitcoinstats.com. 60      IN      AAAA    2a02:aa16:1105:4a80:1234:1234:37c1:9c9

Realizando una consulta de registros `SRV`:

	$ dig lseed.bitcoinstats.com SRV
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 6331 ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qv2w3tledmzczw227nnkqrrltvmydl8gu4w4d70g9td7avke6nmz2tdefqp.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qtynyymv99pqf0r9cuexvvqtxrlgejuecf8myfsa96vcpflgll5cqmr2xsu.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4280 ln1qdfvlysfpyh96apy3w3qdwlu8jjkdhnuxa689ka540tnde6gnx86cf7ga2d.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4281 ln1qwf789tlcpe4n34649xrqllxt97whsvfk5pm07ggqms3vrjwdj3cu6332zs.lseed.bitcoinstats.com.

Realizando una consulta de `A` para el primer nombre de host virtual del ejemplo anterior:

	$ dig ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com A
	ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com. 60 IN A 139.59.143.87

Realizando una consulta solo para nodos IPv4 (`a2`) mediante filtrado de semillas:

	$dig a2.lseed.bitcoinstats.com SRV
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1q2jy22cg2nckgxttjf8txmamwe9rtw325v4m04ug2dm9sxlrh9cagrrpy86.lseed.bitcoinstats.com.
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1qfrkq32xayuq63anmc2zp5vtd2jxafhdzzudmuws0hvxshtgd2zd7jsqv7f.lseed.bitcoinstats.com.

Realizando una consulta solo para nodos IPv6 (`a4`) que admiten Bitcoin (`r0`) mediante filtrado de semillas:

	$dig r0.a4.lseed.bitcoinstats.com SRV
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwx3prnvmxuwsnaqhzwsrrpwy4pjf5m8fv4m8kcjkdvyrzymlcmj5dakwrx.lseed.bitcoinstats.com.
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwr7x7q2gvj7kwzzr7urqq9x7mq0lf9xn6svs8dn7q8gu5q4e852znqj3j7.lseed.bitcoinstats.com.

## Referencias
- <a id="ref-1">[RFC 1035 - Domain Names](https://www.ietf.org/rfc/rfc1035.txt)</a>
- <a id="ref-2">[RFC 3596 - DNS Extensions to Support IP Version 6](https://tools.ietf.org/html/rfc3596)</a>
- <a id="ref-3">[RFC 2782 - A DNS RR for specifying the location of services (DNS SRV)](https://www.ietf.org/rfc/rfc2782.txt)</a>
- <a id="ref-4">[Expectations for DNS Seed operators](https://github.com/bitcoin/bitcoin/blob/master/doc/dnsseed-policy.md)</a>

## Autores

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
