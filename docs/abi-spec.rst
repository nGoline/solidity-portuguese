.. index:: abi, application binary interface

.. _ABI:

************************************************
Especificação da interface binária do aplicativo
************************************************

Design Básico
=============

A interface binária do aplicativo é a maneira padrão de interagir com os contratos no ecossistema Ethereum, ambos
de fora do blockchain e para a interação contrato-contrato. Os dados são codificados de acordo com seu tipo,
conforme descrito nesta especificação. A codificação não é auto-descritiva e, portanto, requer um esquema para decodificar.

Assumimos que as funções de interface de um contrato são fortemente digitadas, conhecidas no momento da compilação e estáticas. Nenhum mecanismo de introspecção será fornecido. Assumimos que todos os contratos terão as definições de interface de quaisquer contratos que eles chamem disponíveis em tempo de compilação.

Esta especificação não aborda contratos cuja interface é dinâmica ou conhecida apenas em tempo de execução. Se esses casos se tornarem importantes, eles podem ser tratados adequadamente como instalações construídas no ecossistema Ethereum.


Função Selector
===============

Os primeiros quatro bytes dos dados de chamada para uma chamada de função especificam a função a ser chamada. É o
primeiro (esquerda, alta-ordem em big-endian) quatro bytes do Keccak (SHA-3) hash da assinatura da função. A assinatura é definida como a expressão canônica do protótipo básico, em geral, o nome da função com a lista entre parênteses dos tipos de parâmetros. Os tipos de parâmetros são divididos por uma única vírgula - não são utilizados espaços.


Codificação de Argumento
========================

A partir do quinto byte, seguem os argumentos codificados. Esta codificação também é usada em outros lugares, p.ex. os valores de retorno e também os argumentos de eventos são codificados da mesma maneira, sem os quatro bytes especificando a função.


Tipos
=====

existem os seguintes tipos elementares:


- ``uint<M>``: tipo integer não assinado de ``M`` bits, ``0 < M <= 256``, ``M % 8 == 0``. normalmente ``uint32``, ``uint8``, ``uint256``.

- ``int<M>``: o complemento de dois complementos de tipo inteiro de ``M`` bits, ``0 < M <= 256``, ``M % 8 == 0``.

- ``address``: equivalente ao ``uint160``, exceto pela interpretação assumida e digitação de idioma.

- ``uint``, ``int``: sinônimos para ``uint256``, ``int256`` respectivamente (não deve ser usado para computar o seletor de funções).

- ``bool``: equivalente para ``uint8`` restrito para o valor 0 e 1

- ``fixed<M>x<N>``: Número decimal de ponto fixo assinado de ``M`` bits, ``8 <= M <= 256``, ``M % 8 ==0``, e ``0 < N <= 80``, que denota o valor ``v`` como ``v / (10 ** N)``.

- ``ufixed<M>x<N>``: variante não assinada de ``fixed<M>x<N>``.

- ``fixed``, ``ufixed``: sinônimos para ``fixed128x19``, ``ufixed128x19`` respectivamente (não deve ser usado para computar o seletor de funções).

- ``bytes<M>``: tipo binário de ``M`` bytes, ``0 < M <= 32``.

- ``function``: equivalente para ``bytes24``: um endereço, seguido por um seletor de função

Existe o seguinte tipo de matriz (tamanho fixo):

- ``<type>[M]``: uma matriz de comprimento fixo do tipo de comprimento fixo fornecido.


Existem os seguintes tipos de tamanho não fixo:

- ``bytes``: Seqüência de bytes de tamanho dinâmico.

- ``string``: Cadeia unicode de tamanho dinâmico assumida como codificada em UTF-8.

- ``<type>[]``: uma matriz de comprimento variável do tipo de comprimento fixo fornecido.

Os tipos podem ser combinados para estruturas anônimas, encerrando um número finito não negativo
deles entre parênteses, separados por vírgulas:

- ``(T1,T2,...,Tn)``: estrutura anônima (tupla ordenada) constituída pelos tipos ``T1``, ..., ``Tn``, ``n >= 0``


É possível formar estruturas de estruturas, arrays de estruturas e assim por diante.

Especificação Formal da Codificação
===================================

Nós iremos agora especificar formalmente a codificação, de modo que ele terá o seguinte
propriedades, que são especialmente úteis se alguns argumentos forem arrays aninhados:

Propriedades:

1. O número de leituras necessárias para acessar um valor é, no máximo, a profundidade do valor dentro da estrutura da matriz do argumento, ou seja, são necessárias quatro leituras para recuperar ``a_i[k][l][r]``.. Em uma versão anterior do ABI, o número de leituras escalonadas linearmente com o número total de parâmetros dinâmicos no pior dos casos.

2. Os dados de uma variável ou elemento de matriz não são entrelaçados com outros dados e são relocáveis, isto é, ele usa apenas "endereços" relativos

Nós distinguimos tipos estáticos e dinâmicos. Os tipos estáticos são codificados no local e os tipos dinâmicos são codificados em um local alocado separadamente após o bloco corrente.

**Definição:** Os seguintes tipos são chamados "dinâmicos" (dynamic):
* ``bytes``
* ``string``
* ``T[]`` para qualquer ``T``
* ``T[k]`` para qualquer dinâmico ``T`` e qualquer ``k > 0``
* ``(T1,...,Tk)`` se qualquer ``Ti`` é dinâmico para ``1 <= i <= k``

Todos os outros tipos são chamados "estáticos" (static)

**Definição:** ``len(a)`` é o n´mero de bytes em uma string binária``a``.
O tipo de ``len(a)`` é assumido para ser ``uint256``. 

Definimos ``enc``, a codificação real, como um mapeamento de valores dos tipos ABI para cadeias binárias, como
que ``len(enc(X))`` depende do valor de ``X`` se e somente se o tipo de ``X`` for dinâmico.

**Definição:** Para qualquer valor ABI ``X``, nós recursivamente definimos ``enc(X)``, dependendo
do tipo de ``X`` sendo

- ``(T1,...,Tk)`` para ``k >= 0`` e quaisquer tipos ``T1``, ..., ``Tk``

  ``enc(X) = head(X(1)) ... head(X(k-1)) tail(X(0)) ... tail(X(k-1))``

  onde ``X(i)`` é o componente ``ith`` do valor, e
  ``head`` e ``tail`` são definidos para ``Ti`` sendo um tipo estático como 

    ``head(X(i)) = enc(X(i))`` e ``tail(X(i)) = ""`` (a string vazia)


  e como 

    ``head(X(i)) = enc(len(head(X(0)) ... head(X(k-1)) tail(X(0)) ... tail(X(i-1))))``
    ``tail(X(i)) = enc(X(i))``

    caso contrário, em geral se ``Ti`` é um tipo dinâmico.

   Observe que no caso dinâmico, ``head(X(i))`` está bem definido, pois os comprimentos das 
   partes principais apenas dependem dos tipos e não dos valores. Seu valor é o deslocamento
   do começo do ``tail(X(i))`` relativo ao começo de ``enc(X)``.
  
- ``T[k]`` para qualquer ``T`` e ``k``:

  ``enc(X) = enc((X[0], ..., X[k-1]))``
  
  isto é codificado como se fosse uma estrutura anônima com elementos ``k``
  do mesmo tipo.

- ``T[]`` onde ``X`` tem ``k`` elementos (``k`` é assumido para ser do tipo ``uint256``):

  ``enc(X) = enc(k) enc([X[1], ..., X[k]])``

  isto é codificado como se fosse uma matriz de tamanho estático ``k``, prefixado com
  o número de elementos.

- ``bytes``, de tamanho ``k`` (que é assumido ser do tipo ``uint256``):

 ``enc(X) = enc(k) pad_right(X)``, isto é, o número de bytes é codificado como um
   ``uint256`` seguido pelo valor real de ``X`` como uma seqüência de bytes, seguida por
   o número mínimo de zero-bytes tal que ``len(enc(X))`` é um múltiplo de 32.

- ``string``:

``enc(X) = enc(enc_utf8(X))``, ou seja, ``X`` é codificado em utf-8 e esse valor é interpretado como de tipo ``bytes`` e codificado ainda mais. Observe que o comprimento usado nesta codificação subseqüente é o número de bytes da seqüência codificada utf-8, não o número de caracteres.

- ``uint<M>``: ``enc(X)`` é a codificação big-endian de ``X``, preenchida no lado de ordem superior (esquerda) com zero-bytes, de modo que o comprimento é um múltiplo de 32 bytes.
- ``address``: como no caso ``uint160``

- ``int<M>``: ``enc(X)`` é o complemento de código de dois big-endian ``X``, preenchido na mais alta ordem (esquerda) com ``0xff`` para ``X`` negativo e com zero bytes para `` X 'positivo, de modo que o comprimento seja um múltiplo de 32 bytes.
- ``bool``: como no caso ``uint8``, onde ``1`` é usado para ``true`` e ``0`` para ``false``
- ``fixed<M>x<N>``: ``enc(X)`` é ``enc(X * 10**N)`` onde ``X * 10**N`` é interprestado como ``int256``.
- ``fixed``: como no caso ``fixed128x19``
- ``ufixed<M>x<N>``: ``enc(X)`` é ``enc(X * 10**N)`` onde ``X * 10**N`` é interpretado como ``uint256``.
- ``ufixed``: como no caso ``ufixed128x19``
- ``bytes<M>``: ``enc(X)`` é a sequência de bytes em ``X`` preenchido com zero-bytes para um comprimento de 32.

perceba que para qualquer `X``, ``len(enc(X))`` é um múltiplo de 32.

Função Seletor e Codificação de Argumento
=========================================

Em suma, uma chamada para a função ``f`` com os parâmetros ``a_1, ..., a_n`` está codificada como

  ``function_selector(f) enc((a_1, ..., a_n))``

e os valores de retorno ``v_1, ..., v_k`` de ``f`` são codificados como 

  ``enc((v_1, ..., v_k))``

ou seja, os valores são combinados em uma estrutura anônima e codificados.

Exemplos
========

Dado o contrato:

::

    pragma solidity ^0.4.0;

    contract Foo {
      function bar(bytes3[2] xy) {}
      function baz(uint32 x, bool y) returns (bool r) { r = x > 32 || y; }
      function sam(bytes name, bool z, uint[] data) {}
    }

Assim, para o nosso exemplo ``Foo`` se queríamos chamar ``baz`` com os parâmetros ``69`` e ``true``, passaríamos 68 bytes no total, que podem ser divididos em:

- ``0xcdcd77c0``: o Method ID. Isto é derivado como os primeiros 4 bytes do hash Keccak do formato ASCII da assinatura ``baz(uint32,bool)``.
- ``0x0000000000000000000000000000000000000000000000000000000000000045``: o primeiro parâmetro, um uint32 de valor ``69`` preenchido para 32 bytes
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: o segundo parâmetro - booleano ``true``, preenchido para 32 bytes

No total::

    0xcdcd77c000000000000000000000000000000000000000000000000000000000000000450000000000000000000000000000000000000000000000000000000000000001


Isso irá retornar um único ``bool``. Se, por exemplo, devesse retornar ``false``, sua saída seria a matriz de byte ``0x0000000000000000000000000000000000000000000000000000000000000000``, um único bool.

Se quiséssemos chamar ``bar`` com o argumento ``["abc", "def"]``, passaríamos 68 bytes totais, divididos em:

- ``0xfce353f6``: o Method ID. Isto é derivado da assinatura ``bar(bytes3[2])``.
- ``0x6162630000000000000000000000000000000000000000000000000000000000``: a primeira parte do primeiro parâmetro, o valor ``bytes3`` para ``"abc"`` (alinhado à esquerda).
- ``0x6465660000000000000000000000000000000000000000000000000000000000``: a segunda parte do primeiro parâmetro, o valor ``bytes3`` para ``"def"`` (alinhado á esquerda).

No total::

    0xfce353f661626300000000000000000000000000000000000000000000000000000000006465660000000000000000000000000000000000000000000000000000000000

Se quisermos chamar ``sam`` com os argumentos ``"dave"``, ``true`` e ``[1,2,3]``, passaríamos 292 bytes totais, divididos em:

- ``0xa5643bf2``: o Method ID. É derivado da assinatura``sam(bytes,bool,uint256[])``. Perceba que ``uint`` é substituída por sua representação canônica``uint256``.
- ``0x0000000000000000000000000000000000000000000000000000000000000060``: a localização da parte de dados do primeiro parâmetro (tipo dinâmico), medida em bytes a partir do início do bloco de argumentos. Nesse caso, ``0x60``.
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: o segundo parâmetro = booleano de true.
- ``0x00000000000000000000000000000000000000000000000000000000000000a0``: a localização da parte de dados do terceiro parâmetro (tipo dinâmico), medida em bytes. Nesse caso, ``0xa0``.
- ``0x0000000000000000000000000000000000000000000000000000000000000004``: a parte de dados do primeiro argumento, ele começa com o comprimento da matriz de bytes em elementos, neste caso, 4.
- ``0x6461766500000000000000000000000000000000000000000000000000000000``: o conteúdo do primeiro argumento: a codificação UTF-8 (igual a ASCII neste caso) de ``"dave"``, preenchida à direita para 32 bytes.
- ``0x0000000000000000000000000000000000000000000000000000000000000003``: a parte de dados do terceiro argumento, ele começa com o comprimento da matriz em elementos, neste caso, 3.
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: A primeira entrada do terceiro parâmetro.
- ``0x0000000000000000000000000000000000000000000000000000000000000002``: A segunda entrada do terceiro parâmetro.
- ``0x0000000000000000000000000000000000000000000000000000000000000003``: A terceira entrada do terceiro parâmetro.

No total::

    0xa5643bf20000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000464617665000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003

Uso dos Tipos Dinâmicos
=======================

Uma chamada para uma função com a assinatura ``f(uint,uint32[],bytes10,bytes)`` com valores ``(0x123, [0x456, 0x789], "1234567890", "Hello, world!")`` está codificado da seguinte maneira:

Nós pegamos os primeiros quatro bytes de ``sha3("f(uint256,uint32[],bytes10,bytes)")``, isto é ``0x8be65246``

Então codificamos as partes principais de todos os quatro argumentos. Para os tipos estáticos ``uint256`` e ``bytes10``, estes são diretamente os valores que queremos passar, enquanto que para os tipos dinâmicos ``uint32 [] `` e ``bytes``, usamos o deslocamento em bytes para o início de sua área de dados, medida a partir do início da codificação do valor (ou seja, não contando os quatro primeiros bytes contendo o hash da assinatura da função). Esses são:

 - ``0x0000000000000000000000000000000000000000000000000000000000000123`` (``0x123`` preenchido para 32 bytes)
 - ``0x0000000000000000000000000000000000000000000000000000000000000080`` (offset para o início da parte de dados do segundo parâmetro, 4*32 bytes, exatamente o tamanho da parte principal)
 - ``0x3132333435363738393000000000000000000000000000000000000000000000`` (``"1234567890"`` preenchido para 32 bytes à direita)
 - ``0x00000000000000000000000000000000000000000000000000000000000000e0`` (offset para o início da parte de dados do quarto parâmetro = offset para o início da parte de dados do primeiro parâmetro dinâmico + o tamanho da parte de dados do primeiro parâmetro dinâmico = 4\*32 + 3\*32 (veja abaixo))

Após isto, a parte de dados do primeiro argumento dinâmico ``[0x456, 0x789]`` conforme segue:

 - ``0x0000000000000000000000000000000000000000000000000000000000000002`` (número de elementos do array, 2)
 - ``0x0000000000000000000000000000000000000000000000000000000000000456`` (primeiro elemento)
 - ``0x0000000000000000000000000000000000000000000000000000000000000789`` (segundo elemento)

Finalmente, nós codificamos a parte de dados do segundo argumento dinâmico ``"Hello, world!"``:

 - ``0x000000000000000000000000000000000000000000000000000000000000000d`` (número de elementos (bytes neste caso): 13)
 - ``0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000`` (``"Hello, world!"`` preenchido para 32 bytes à direita)

Tudo junto, a codificação é (nova linha após o seletor de função e cada 32 bytes para maior clareza):

::

    0x8be65246
      0000000000000000000000000000000000000000000000000000000000000123
      0000000000000000000000000000000000000000000000000000000000000080
      3132333435363738393000000000000000000000000000000000000000000000
      00000000000000000000000000000000000000000000000000000000000000e0
      0000000000000000000000000000000000000000000000000000000000000002
      0000000000000000000000000000000000000000000000000000000000000456
      0000000000000000000000000000000000000000000000000000000000000789
      000000000000000000000000000000000000000000000000000000000000000d
      48656c6c6f2c20776f726c642100000000000000000000000000000000000000

Eventos
=======

Os eventos são uma abstração do protocolo Ethereum para logging/event-watching. As entradas do registro fornecem o endereço do contrato, uma série de até quatro tópicos e alguns dados binários de comprimento arbitrário. Os eventos aproveitam a função ABI existente para interpretar isso (juntamente com uma especificação de interface) como uma estrutura corretamente digitada.

Dado um nome de evento e uma série de parâmetros de eventos, os dividimos em duas subséries: as que estão indexadas e as que não são. Aqueles que são indexados, que podem numerar até 3, são usados ao lado do hash Keccak da assinatura do evento para formar os tópicos da entrada de log. Aqueles que, como não indexados, formam a matriz de bytes do evento.

De fato, uma entrada de log usando este ABI é descrito como:


- ``address``: o endereço do contrato (intrinsicamente fornecido pelo Ethereum);
- ``topics[0]``: ``keccak(EVENT_NAME+"("+EVENT_ARGS.map(canonical_type_of).join(",")+")")`` (``canonical_type_of`` é uma função que simplesmente retorna o tipo canônico de um determinado argumento, em geral para ``uint indexed foo``, ele retornaria ``uint256``). Se o evento for declarado como "anônimo", o ``topics [0]`` não é gerado;
- ``topics[n]``: ``EVENT_INDEXED_ARGS[n - 1]`` (``EVENT_INDEXED_ARGS`` é uma série de ``EVENT_ARGS`` que são indexados);
- ``data``: ``abi_serialise(EVENT_NON_INDEXED_ARGS)`` (``EVENT_NON_INDEXED_ARGS`` é uma série de ``EVENT_ARGS`` que não são indexados, ``abi_serialise`` é a função de serialização ABI usado para retornar uma série de valores digitados de uma função conforme descrito acima. 

JSON
====

O formato JSON para a interface de um contrato é fornecido por uma série de funções e/ou descrições de eventos. Uma descrição de função é um objeto JSON com os campos:

- ``type``: ``"function"``, ``"constructor"``, ou ``"fallback"`` (a :ref:`unnamed "default" function <fallback-function>`);
- ``name``: o nome da função;
- ``inputs``: uma tabela de objetos, cada um contendo:
  * ``name``: o nome do parâmetro;
  * ``type``: o tipo canônico do parâmetro.
- ``outputs``: uma tabela de objetos similar aos ``inputs``, podendo ser omitido se a função não retornar algo;
- ``payable``: ``true`` se a função aceitar ether, por default para ``false``;
- ``stateMutability``: uma string com um dos seguintes valores: ``pure`` (:ref:`specified to not read blockchain state <pure-functions>`), ``view`` (:ref:`specified to not modify the blockchain state <view-functions>`), ``nonpayable`` e ``payable`` (o mesmo que ``payable`` acima).
- ``constant``: ``true`` se a função é ambos ``pure`` ou ``view``

``type`` pode ser omitido, defaulting to ``"function"``.

As funções Constructor e fallback nunca tem ``name`` ou ``outputs``. A função de retorno também não possui ``inputs``.

O envio de Ether não-zero para função não pagável será executada. Não faça isso.

Uma descrição do evento é um objeto JSON com campos bastante semelhantes:


- ``type``: sempre ``"event"``
- ``name``: o nome do evento;
- ``inputs``: uma tabela de objetos, cada um contendo:
  * ``name``: o nome do parâmetro;
  * ``type``: o tipo canônico do parâmetro.
  * ``indexed``: ``true`` se o campo for parte dos tópicos do log, ``false`` se um dos segmentos de dados do log.
- ``anonymous``: ``true`` se o evento foi declarado como ``anonymous``.

Por examplo, 

::

  pragma solidity ^0.4.0;

  contract Test {
    function Test(){ b = 0x12345678901234567890123456789012; }
    event Event(uint indexed a, bytes32 b)
    event Event2(uint indexed a, bytes32 b)
    function foo(uint a) { Event(a, b); }
    bytes32 b;
  }

pode resultar no JSON:

.. code:: json

  [{
  "type":"event",
  "inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
  "name":"Event"
  }, {
  "type":"event",
  "inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
  "name":"Event2"
  }, {
  "type":"function",
  "inputs": [{"name":"a","type":"uint256"}],
  "name":"foo",
  "outputs": []
  }]
