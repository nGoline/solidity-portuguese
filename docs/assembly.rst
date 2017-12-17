#################
Solidity Assembly
#################

.. index:: ! assembly, ! asm, ! evmasm

O Solidity define uma linguagem de assembly que também pode ser utilizada sem o Solidity.
Essa linguagem de assembly também pode ser utilizada como "inline assembly", ou seja, pode-se utilizar o assembly em 
linha dentro do código fonte do Solidity. Vamos começar descrevendo como utilizar o assembly dentro do código e como 
isso se difere da utilização do assembly autônomo e então vamos ver as especificações do código assembly.

.. note::
    TODO: Escrever sobre como as regras de escopo do assembly em linha são um pouco diferentes
    e as complicações que surgem quando, por exemplo, utilizamos funções internas ou bibliotecas.
    Além disso, escrever sobre os símbolos definidos pelo compilador.

.. _inline-assembly:

Inline Assembly
===============

Para um controle mais fino, especialmente para aprimorar a linguagem com a escrita de bibliotecas,
é possível intercalar declarações Solidity com assembly em linha em uma linguagem parecida
com a utilizada pela máquina virtual. Graças ao fato da EVM ser uma maquina de empilhamento, muitas
vezes é difícil endereçar o slot correto da pilha e prover argumentos para os opcodes no ponto exato na pilha.
O assembly em linha do Solidity tenta facilitar isso e outros problemas decorrentes da escrita de assembly
manualmente com as seguintes ferramentas:

* opcodes estilo funcional: ``mul(1, add(2, 3))`` ao invés de ``push1 3 push1 2 add push1 1 mul``
* variáveis locais de assembly: ``let x := add(2, 3)  let y := mload(0x40)  x := add(x, y)``
* acesso à variáveis externas: ``function f(uint x) { assembly { x := sub(x, 1) } }``
* rótulos: ``let x := 10  repeat: x := sub(x, 1) jumpi(repeat, eq(x, 0))``
* loops: ``for { let i := 0 } lt(i, x) { i := add(i, 1) } { y := mul(2, y) }``
* declarações switch: ``switch x case 0 { y := mul(x, 2) } default { y := 0 }``
* chamadas de funções: ``function f(x) -> y { switch x case 0 { y := 1 } default { y := mul(x, f(sub(x, 1))) }   }``

Agora vamos descrever a linguagem assembly em linha de forma detalhada

.. warning::
    O assembly em linha é uma maneira de acessar a Máquina Virtual Ethereum
    em baixo nível. Isso descarta vários aspectos de segurança importantes do Solidity.

Exemplo
-------

O exemplo a seguir fornece códigos de bibliotecas que acessam o código de outro contrato
e o carregam na variável ``bytes``. Isso seria impossível utilizando somente "Solidity puro"
e a ideia é que bibliotecas assembly sejam utilizadas para aprimorar a linguagem.

.. code::

    pragma solidity ^0.4.0;

    library GetCode {
        function at(address _addr) returns (bytes o_code) {
            assembly {
                // recupera o tamanho do código, isso precisa de assembly
                let size := extcodesize(_addr)
                // aloca uma saída para o vetor de bytes - isso poderia ser feito sem utilizar assembly
                // utilizando o código: o_code = new bytes(size)
                o_code := mload(0x40)
                // nova "memory end" incluindo preenchimento
                mstore(0x40, add(o_code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
                // guarda o comprimento na memória
                mstore(o_code, size)
                // recupera o código, isso precisa de assembly
                extcodecopy(_addr, add(o_code, 0x20), 0, size)
            }
        }
    }

O assembly em linha também pode ser beneficial para casos em que o otimizador falha em produzir
código eficiente. Por favor, tenha em mente que assembly é muito mais difícil de escrever pois o
compilador não realiza checagens, então é necessário utilizar somente para tarefas complexas e se
você realmente sabe o que está fazendo.

.. code::

    pragma solidity ^0.4.12;

    library VectorSum {
        // Essa função é menos eficiente pois o otimizador falha em remover 
        // as checagens de limites no acesso ao vetor.
        function sumSolidity(uint[] _data) returns (uint o_sum) {
            for (uint i = 0; i < _data.length; ++i)
                o_sum += _data[i];
        }

        // Nós sabemos que só queremos acessar o vetor dentro de seus limites , então podemos
        // ignorar a checagem. 0x20 precisa ser adicionado ao vetor pois a primeira posição contém
        // o comprimento do vetor.
        function sumAsm(uint[] _data) returns (uint o_sum) {
            for (uint i = 0; i < _data.length; ++i) {
                assembly {
                    o_sum := add(o_sum, mload(add(add(_data, 0x20), mul(i, 0x20))))
                }
            }
        }

        // Mesmo código que o anterior, mas executa todo o código dentro do assembly em linha.
        function sumPureAsm(uint[] _data) returns (uint o_sum) {
            assembly {
               // Carrega o comprimento (primeiros 32 bytes)
               let len := mload(_data)

               // Ignora o campo de comprimento.
               //
               // Mantém uma variável para que seja incrementada localmente.
               //
               // NOTA: incrementing _data resultará em uma
               //       variável _data inutilizável após esse bloco de assembly
               let data := add(_data, 0x20)

               // Itera enquanto o limite não for atingido.
               for
                   { let end := add(data, len) }
                   lt(data, end)
                   { data := add(data, 0x20) }
               {
                   o_sum := add(o_sum, mload(data))
               }
            }
        }
    }


Síntese
-------

O assembly analisa comentários, literais e identificadores exatamente como o Solidity, assim você pode utilizar
os comentários ``//`` e ``/* */``. o assembly em linha é marcado por ``assembly { ... }`` e dentro das chaves
podem ser utilizados (veja as próximas seções para mais detalhes)

 - literais, ex. ``0x123``, ``42`` ou ``"abc"`` (textos de até 32 caracteres)
 - opcodes (em "estilo de instrução"), ex. ``mload sload dup1 sstore``, veja abaixo para uma lista
 - opcodes em estilo funcional, ex. ``add(1, mlod(0))``
 - rótulos, ex. ``name:``
 - declaração de variáveis, ex. ``let x := 7``, ``let x := add(y, 3)`` ou ``let x`` (valor inicial de (0) é atribuído)
 - identificadores (rótulos ou variáveis locais de assembly ou externas se utilizadas no assembly em linha), ex. ``jump(name)``, ``3 x add``
 - atribuições (em "estilo de instrução"), ex. ``3 =: x``
 - atribuições em estilo funcional, ex. ``x := add(y, 3)``
 - blocos onde variáveis locais tem escopo interno, ex. ``{ let x := 3 { let y := add(x, 1) } }``

Opcodes
-------

Esse documento não tenta ser uma descrição completa da Máquina Virtual Ethereum, mas a lista
a seguir pode ser utilizada como referência aos seus opcodes.

Se um opcode recebe argumentos (sempre do topo da pilha), eles são dados em parênteses.
Note que a ordem dos argumentos pode ser vista invertida em estilos não funcionais (explicado abaixo).
Opcodes marcados com ``-`` não inserem um item na pilha, os marcados com ``*`` são especiais e 
todas as outras inserem exatamente um item na pilha.

No código, ``mem[a...b)`` simboliza os bytes da memória iniciando na posição ``a`` até
(excluindo) a posição ``b`` e ``storage[p]`` simboliza a armazenagem de conteúdo na posição ``p``.
Os opcodes ``pushi`` e ``jumpdest`` não podem ser utilizados diretamente.

No dicionário, opcodes são representados como identificadores pré-definidos.

+-------------------------+------+-----------------------------------------------------------------+
| stop                    + `-`  | para a execução, idêntico à return(0,0)                         |
+-------------------------+------+-----------------------------------------------------------------+
| add(x, y)               |      | x + y                                                           |
+-------------------------+------+-----------------------------------------------------------------+
| sub(x, y)               |      | x - y                                                           |
+-------------------------+------+-----------------------------------------------------------------+
| mul(x, y)               |      | x * y                                                           |
+-------------------------+------+-----------------------------------------------------------------+
| div(x, y)               |      | x / y                                                           |
+-------------------------+------+-----------------------------------------------------------------+
| sdiv(x, y)              |      | x / y, para números assinados em complemento de 2               |
+-------------------------+------+-----------------------------------------------------------------+
| mod(x, y)               |      | x % y                                                           |
+-------------------------+------+-----------------------------------------------------------------+
| smod(x, y)              |      | x % y, para números assinados em complemento de 2               |
+-------------------------+------+-----------------------------------------------------------------+
| exp(x, y)               |      | x elevado a y                                                   |
+-------------------------+------+-----------------------------------------------------------------+
| not(x)                  |      | ~x, todos os bits de x são negados                              |
+-------------------------+------+-----------------------------------------------------------------+
| lt(x, y)                |      | 1 se x < y, senão 0                                             |
+-------------------------+------+-----------------------------------------------------------------+
| gt(x, y)                |      | 1 se x > y, senão 0                                             |
+-------------------------+------+-----------------------------------------------------------------+
| slt(x, y)               |      | 1 se x < y, senão 0, para números assinados em complemento de 2 |
+-------------------------+------+-----------------------------------------------------------------+
| sgt(x, y)               |      | 1 se x > y, senão 0, para números assinados em complemento de 2 |
+-------------------------+------+-----------------------------------------------------------------+
| eq(x, y)                |      | 1 se x == y, senão 0                                            |
+-------------------------+------+-----------------------------------------------------------------+
| iszero(x)               |      | 1 se x == 0, senão 0                                            |
+-------------------------+------+-----------------------------------------------------------------+
| and(x, y)               |      | bitwise and de x e y                                            |
+-------------------------+------+-----------------------------------------------------------------+
| or(x, y)                |      | bitwise or de x e y                                             |
+-------------------------+------+-----------------------------------------------------------------+
| xor(x, y)               |      | bitwise xor de x e y                                            |
+-------------------------+------+-----------------------------------------------------------------+
| byte(n, x)              |      | enésimo byte de x, onde o byte mais significante é o byte 0     |
+-------------------------+------+-----------------------------------------------------------------+
| addmod(x, y, m)         |      | (x + y) % m com precisão aritmética arbitrária                  |
+-------------------------+------+-----------------------------------------------------------------+
| mulmod(x, y, m)         |      | (x * y) % m com precisão aritmética arbitrária                  |
+-------------------------+------+-----------------------------------------------------------------+
| signextend(i, x)        |      | extensão de sinal do(i*8+7)th bit à partir do menos significante|
+-------------------------+------+-----------------------------------------------------------------+
| keccak256(p, n)         |      | keccak(mem[p...(p+n)))                                          |
+-------------------------+------+-----------------------------------------------------------------+
| sha3(p, n)              |      | keccak(mem[p...(p+n)))                                          |
+-------------------------+------+-----------------------------------------------------------------+
| jump(label)             | `-`  | pula para o rótulo / posição do código                          |
+-------------------------+------+-----------------------------------------------------------------+
| jumpi(label, cond)      | `-`  | pula para o rótulo se cond é nonzero                            |
+-------------------------+------+-----------------------------------------------------------------+
| pc                      |      | posição atual no código                                         |
+-------------------------+------+-----------------------------------------------------------------+
| pop(x)                  | `-`  | remove o elemento adicionado por x                              |
+-------------------------+------+-----------------------------------------------------------------+
| dup1 ... dup16          |      | copia a enésima posição (à partir do topo) da pilha para o topo |
+-------------------------+------+-----------------------------------------------------------------+
| swap1 ... swap16        | `*`  | troca o topmost (primeira posição da pilha) e a enésima posição |
|                         |      | da pilha                                                        |
+-------------------------+------+-----------------------------------------------------------------+
| mload(p)                |      | mem[p..(p+32))                                                  |
+-------------------------+------+-----------------------------------------------------------------+
| mstore(p, v)            | `-`  | mem[p..(p+32)) := v                                             |
+-------------------------+------+-----------------------------------------------------------------+
| mstore8(p, v)           | `-`  | mem[p] := v & 0xff    - modifica somente um único byte          |
+-------------------------+------+-----------------------------------------------------------------+
| sload(p)                |      | storage[p]                                                      |
+-------------------------+------+-----------------------------------------------------------------+
| sstore(p, v)            | `-`  | storage[p] := v                                                 |
+-------------------------+------+-----------------------------------------------------------------+
| msize                   |      | tamanho da memória, ex. maior índice de memória acessada        |
+-------------------------+------+-----------------------------------------------------------------+
| gas                     |      | gas disponível para execução                                    |
+-------------------------+------+-----------------------------------------------------------------+
| address                 |      | endereço do contrato atual / contexto de execução               |
+-------------------------+------+-----------------------------------------------------------------+
| balance(a)              |      | saldo em wei do endereço a                                      |
+-------------------------+------+-----------------------------------------------------------------+
| caller                  |      | disparador da chamada (exceto delegatecall)                     |
+-------------------------+------+-----------------------------------------------------------------+
| callvalue               |      | wei enviado com a chamada                                       |
+-------------------------+------+-----------------------------------------------------------------+
| calldataload(p)         |      | dados da chamada iniciando na posição p (32 bytes)              |
+-------------------------+------+-----------------------------------------------------------------+
| calldatasize            |      | tamanho da chamada em bytes                                     |
+-------------------------+------+-----------------------------------------------------------------+
| calldatacopy(t, f, s)   | `-`  | copia s bytes dos dados na posição f para mem na posição t      |
+-------------------------+------+-----------------------------------------------------------------+
| codesize                |      | tamanho do código do contrato atual / contexto de execução      |
+-------------------------+------+-----------------------------------------------------------------+
| codecopy(t, f, s)       | `-`  | copia s bytes do codigo na posição f para mem na posição t      |
+-------------------------+------+-----------------------------------------------------------------+
| extcodesize(a)          |      | tamanho do código no endereço a                                 |
+-------------------------+------+-----------------------------------------------------------------+
| extcodecopy(a, t, f, s) | `-`  | como codecopy(t, f, s) mas utiliza o código do endereço a       |
+-------------------------+------+-----------------------------------------------------------------+
| returndatasize          |      | tamanho do último returndata                                    |
+-------------------------+------+-----------------------------------------------------------------+
| returndatacopy(t, f, s) | `-`  | copia s bytes de returndata na posição f para mem na posição t  |
+-------------------------+------+-----------------------------------------------------------------+
| create(v, p, s)         |      | cria um novo contrato com o código de mem[p..(p+s)), envia v    |
|                         |      | wei e retorna o endereço do novo contrato                       |
+-------------------------+------+-----------------------------------------------------------------+
| create2(v, n, p, s)     |      | cria um novo contrato com o código de mem[p..(p+s)) no endereço |
|                         |      | keccak256(<address> . n . keccak256(mem[p..(p+s))) e envia v    |
|                         |      | wei e retorna o endereço do novo contrato                       |
+-------------------------+------+-----------------------------------------------------------------+
| call(g, a, v, in,       |      | chama um contrato no endereço a com os dados em                 |
| insize, out, outsize)   |      | mem[in..(in+insize)) provendo g gas e v wei e saída para        |
|                         |      | mem[out..(out+outsize)) retornando 0 em caso de erro            |
|                         |      | (ex. falta de gas) e 1 em caso de sucesso na execução           |
+-------------------------+------+-----------------------------------------------------------------+
| callcode(g, a, v, in,   |      | idêntico ao `call` mas só utiliza o código de a, além de manter |
| insize, out, outsize)   |      | o contexto de execução do contrato atual                        |
+-------------------------+------+-----------------------------------------------------------------+
| delegatecall(g, a, in,  |      | idêntico ao `callcode` mas também mantém ``caller``             |
| insize, out, outsize)   |      | e ``callvalue``                                                 |
+-------------------------+------+-----------------------------------------------------------------+
| staticcall(g, a, in,    |      | idêntico ao `call(g, a, 0, in, insize, out, outsize)` mas não   |
| insize, out, outsize)   |      | permite modifição de estado                                     |
+-------------------------+------+-----------------------------------------------------------------+
| return(p, s)            | `-`  | finaliza a execução, retorna os dados em mem[p..(p+s))          |
+-------------------------+------+-----------------------------------------------------------------+
| revert(p, s)            | `-`  | finaliza a execução, reverte as mudanças de estado e retorna os |
|                         |      | dados em mem[p..(p+s))                                          |
+-------------------------+------+-----------------------------------------------------------------+
| selfdestruct(a)         | `-`  | finaliza a execução, destrói o contrato atual e envia o saldo   |
|                         |      | para o contrato a                                               |
+-------------------------+------+-----------------------------------------------------------------+
| invalid                 | `-`  | finaliza a execução com instrução inválida                      |
+-------------------------+------+-----------------------------------------------------------------+
| log0(p, s)              | `-`  | loga sem tópicos os dados em mem[p..(p+s))                      |
+-------------------------+------+-----------------------------------------------------------------+
| log1(p, s, t1)          | `-`  | loga com os tópico t1 os dados em mem[p..(p+s))                 |
+-------------------------+------+-----------------------------------------------------------------+
| log2(p, s, t1, t2)      | `-`  | loga com os tópicos t1 e t2 os dados em mem[p..(p+s))           |
+-------------------------+------+-----------------------------------------------------------------+
| log3(p, s, t1, t2, t3)  | `-`  | loga com os tópicos t1, t2 e t3 os dados em mem[p..(p+s))       |
+-------------------------+------+-----------------------------------------------------------------+
| log4(p, s, t1, t2, t3,  | `-`  | loga com os tópicos t1, t2, t3 e t4 os dados em mem[p..(p+s))   |
| t4)                     |      |                                                                 |
+-------------------------+------+-----------------------------------------------------------------+
| origin                  |      | quem enviou a transação                                         |
+-------------------------+------+-----------------------------------------------------------------+
| gasprice                |      | preço do gas da transação                                       |
+-------------------------+------+-----------------------------------------------------------------+
| blockhash(b)            |      | hash do bloco b - somente para os últimos 256 blocos excluindo o|
|                         |      | bloco atual                                                     |
+-------------------------+------+-----------------------------------------------------------------+
| coinbase                |      | beneficiário da mineração                                       |
+-------------------------+------+-----------------------------------------------------------------+
| timestamp               |      | timestamp do bloco atual em segundos desde a época              |
|                         |      | (1970-01-01T00:00:00Z)                                          |
+-------------------------+------+-----------------------------------------------------------------+
| number                  |      | número do bloco atual                                           |
+-------------------------+------+-----------------------------------------------------------------+
| difficulty              |      | dificuldade do bloco atual                                      |
+-------------------------+------+-----------------------------------------------------------------+
| gaslimit                |      | limite de gas do bloco atual                                    |
+-------------------------+------+-----------------------------------------------------------------+

Literais
--------

Você pode utilizar constantes inteiras (integer) digitando-as em decimal ou hexadecimal
e a instrução apropriada ``PUSHi`` vai ser automaticamente gerada. O trecho a seguir cria
um código para adicionar 2 e 3 resultando em 5 e então computa o bitwise and com o texto "abc".
Textos são guardados alinhados à esquerda e não podem conter mais de 32 bytes.

.. code::

    assembly { 2 3 add "abc" and }

Estilo Funcional
----------------

Você pode digitar opcode após opcode da mesma maneira que eles ficarão em bytecode. Por exemplo
adicionar ``3`` ao conteúdo da memória na posição ``0x80`` seria

.. code::

    3 0x80 mload add 0x80 mstore

Como é difícil ver quais são os argumentos atuais para certos opcodes,
o aseembly em linha do Solidity provê uma notação de "estilo funcional"
onde o mesmo código seria escrito como

.. code::

    mstore(0x80, add(mload(0x80), 3))

As expressões em estilo funcional não podem utilizar estilo instrucional internamente, ex.
``1 2 mstore(0x80, add)`` não é um assembly válido, o correto seria escrever como
``mstore(0x80, add(2, 1))``. Para opcodes que não recebem argumentos, os parênteses podem ser omitidos.

Note que a ordem dos argumentos é invertida em estilo funcional, o oposto do estilo instrucional.
Se você utiliza estilo funcional, o primeiro argumento vai ficar no topo da pilha.


Acesso à Variaveis Externas e Funções
-------------------------------------

Vaiáveis de Solidity e outros identificadores podem ser acessados simplesmente
utilizando seu nome. Para variáveis de memória, isso insere seu endereço e não seu valor
na pilha. Variáveis de armazenamento são diferentes: Valores no armazenamento
podem não ocupar um espaço de armazenamento completo na memória, por isso seu "endereço"
é composto de um espaço na memória e um byte-offset dentro do espaço. Para recuperar o conteúdo
da variável ``x``, voce deve utilizar ``x_slot`` e para recuperar o byte-offset você deve utilizar
``x_offset``.

Em atribuições (veja abaixo), nós podemos utilizar variáveis locais do Solidity para atribuir um valor.

Funções externas ao assembly em linha também pode ser acessado: O assembly irá
inserir seu rótulo de entrada (com a resolução da função virtual aplicada). A semântica
para chamada em solidity é:
Functions external to inline assembly can also be accesse

 - o consumidor chama return label, arg1, arg2, ..., argn
 - a chamada retorna com ret1, ret2, ..., retm

Esse recurso ainda é um pouco moroso para usar, porque a pilha essencialmente
muda durante a chamada, e desta forma referências para variáveis locais estarão erradas.

.. code::

    pragma solidity ^0.4.11;

    contract C {
        uint b;
        function f(uint x) returns (uint r) {
            assembly {
                r := mul(x, sload(b_slot)) // ignore o offset, sabemos que é 0
            }
        }
    }

Rótulos
-------

Outro problema com assembly de EVM é que ``jump`` e ``jumpi`` utilizam endereços absolutos
que podem mudar facilmente. Assembly em linha do solidity provê rótulos para facilitar o
uso de pulos (jumps). Note que rótulos são um recurso de baixo nível e que é possível
escrever assemblies eficientes sem rótulos, utilizando apenas funções de assembly, loops e
instruções switch (veja abaixo). O código a seguir computa um elemento na série Fibonacci.

.. code::

    {
        let n := calldataload(4)
        let a := 1
        let b := a
    loop:
        jumpi(loopend, eq(n, 0))
        a add swap1
        n := sub(n, 1)
        jump(loop)
    loopend:
        mstore(0, a)
        return(0, 0x20)
    }

Perceba que o acesso automático às variáveis da pilha só funciona se o assembler
sabe a altura atual da pilha. Isso não funciona se a origem do salto e o destino
possuem diferentes alturas de pilha. Ainda é possível utilizar tais saltos, mas
você não deve acessar nenhuma variável da pilha (incluindo variáveis de assembly) neste caso.

Além disso, o analisador da altura de pilha analisa o código opcode por opcode
(e não de acordo com o fluxo de controle), então nesse caso, o assembler terá
a impressão incorreta sobre a altura da pilha no rótulo ``two``:

.. code::

    {
        let x := 8
        jump(two)
        one:
            // Aqui a altura da pilha é 2 (porque nós inserimos x e 7),
            // mas o assembler pensa que é 1 por que ele lê
            // do topo pra baixo.
            // Acessar a variável da pilha x aqui vai resultar em erro.
            x := 9
            jump(three)
        two:
            7 // insira algo na pilha
            jump(one)
        three:
    }

Esse problema pode ser resolvido ajustando manualmente a altura da pilha
para o assembler - você pode prover uma altura de pilha delta que é adicionada
à altura da pilha antes do rótulo.
Note que que você não precisa se preocupar com isso se utilizar loops e funções
de nivel de assembly.

O código a seguir é um exemplo de como isso pode ser feito em casos extremos.

.. code::

    {
        let x := 8
        jump(two)
        0 // Esse código é inacessível mas ajusta a altura da pilha corretamente
        one:
            x := 9 // Agora x pode ser acessado.
            jump(three)
            pop // Correção negativa similar.
        two:
            7 // insere algo na pilha
            jump(one)
        three:
        pop // Temos que remover o valor inserido manualmente novamente.
    }

Declarando Variáveis Locais de Assembly
---------------------------------------

Você pode usar a palavra ``let`` para declarar variáveis visíveis somente no
assembly em linha e somente no bloco ``{...}```atual. A instrução ``let`` vai 
criar um novo espaço na pilha reservado para a variável, que será automaticamente
removido quando o bloco chegar ao final. Você precisa prover um valor inicial
para a variável, que pode ser apenas ``0``, mas também pode ser alguma expressão
de estilo funcional complexo.

.. code::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint x) returns (uint b) {
            assembly {
                let v := add(x, 1)
                mstore(0x80, v)
                {
                    let y := add(sload(v), 1)
                    b := y
                } // y é "desalocado" aqui
                b := add(b, v)
            } // v é "desalocado" aqui
        }
    }


Atribuições
-----------

Atribuições são possíveis para variáveis locais do assembly e para variáveis
locais de função. Fique atento pois quando voce atribui valores que apontam
para memória ou armazenamento à uma variável, você vai alterar apenas o
ponteiro e não os dados.

Existem dois tipos de atribuições: de estilo funcional e de estilo instrucional.
Para atribuições de estilo funcional (``variable := value``), voce precisa fornecer
um valor em expressão funcional que resulte em exatamente um valor de pilha.
Para atribuições de estilo instrucional (``=: variable``), o valor é o mesmo do topo
da pilha.
Em ambos os casos, os "dois pontos" apontam para o nome da variável. A atribuiçao
é realizada substituindo o valor da variável na pilha pelo novo valor.

.. code::

    {
        let v := 0 // atribuição de estilo funcional realizada como parte da declaração da variável
        let g := add(v, 2)
        sload(10)
        =: v // atribuição dem estilo instrucional, colcoa o valor de sload(10) em v
    }

Switch
------

Você pode utilizar uma declaração de switch como uma versão básica de "if/else".
Ela pega o valor da expressão e compara à várias constantes. O código da constante
correspondente é selecionado. Diferente do comportamento visto em outras linguagens
de programação, o fluxo de controle não continua para o próximo caso após ser executado.
Pode existir um recurso padrão chamado de ``default`` que é executado caso todos os testes
falhem.

.. code::

    {
        let x := 0
        switch calldataload(4)
        case 0 {
            x := calldataload(0x24)
        }
        default {
            x := calldataload(0x44)
        }
        sstore(0, div(x, 2))
    }

A lista de casos não reuqer chaves, mas o corpo do caso sim.

Loops
-----

O assembly suporta um loop estilo for simples. Loops for possuem um
cabeçalho contendo uma parte de inicialização, uma condição e uma parte
pós iteração. A condição deve ser uma expressão de estilo funcional enquanto
as outras duas são blocos. Se a parte de inicialização declara qualquer variável,
o escopo dessas variáveis é extendido para o corpo da função (incluindo a 
condição e a parte pós iteração).

O exemplo a seguir computa a soma de uma área na memória.

.. code::

    {
        let x := 0
        for { let i := 0 } lt(i, 0x100) { i := add(i, 0x20) } {
            x := add(x, mload(i))
        }
    }

Loops for podem também ser escritos de uma maneira para que se comportem
como loops while:
Basta deixar a parte de inicialização e pós iteração vazias.

.. code::

    {
        let x := 0
        let i := 0
        for { } lt(i, 0x100) { } {     // while(i < 0x100)
            x := add(x, mload(i))
            i := add(i, 0x20)
        }
    } 

Funções
-------

O código assembly permite a definição de funções de baixo nível. Essas
funções recebem seus argumentos (e retornam um PC) da pilha e também 
inserem seus resultados na pilha. Chamar uma função é bem parecido com
executar um opcode de estilo funcional.

Funções podem ser definidaas em qualquer local do código e é visível
no bloco onde foram declaradas. Dentro da função você não pode acessar
variáveis locais definidas fora da função. Não existe ``return`` explícito.

Se você chamar uma função que retorna múltiplos valores, você deve atribuí-los
À um tuple utilizando ``a, b := f(x)`` ou ``let a, b := f(x)``.

O exemplo a seguir implementa a função potência utilizando raiz quadrada e multiplicação.

.. code::

    {
        function power(base, exponent) -> result {
            switch exponent
            case 0 { result := 1 }
            case 1 { result := base }
            default {
                result := power(mul(base, base), div(exponent, 2))
                switch mod(exponent, 2)
                    case 1 { result := mul(base, result) }
            }
        }
    }

Coisas a Evitar
---------------

Assembly em linha pode parecer uma linguagem de alto nível, mas é uma linguagem
extramente de baixo nível. Chamadas de funções, loops e switches são convertidos
simplesmente reescrevendo regras e, depois disso, a única coisa que o assembler
faz par você é rearranjar opcodes de estilo funcional, administrando rótulos de jump,
contando a altura da pilha para acesso à variáveis e removendo espaços de pilha para
variáveis locais de assembly ao final dos blocos. Especialmente para os dois últimos
dois casos, é importante saber que o assembler só conta a altura da pilha do topo para
baixo, não necessariamente seguindo o fluxo de controle. Além disso, operações como swap
só vão modificar o conteúdo da pilha e não o local das variáveis.

Convenções em Solidity
----------------------

Em contraste ao assembly EVM, o Solidity sabe os tipos que são menores que 256 bits,
ex. ``uint24``. Para ser mais eficientes, a maior parte das operações aritmética
as trata como números de 256-bit e os bits de ordem maior só são limpos quando
necessário, ex. pouco antes de serem escritos na memória ou de comparações serem efetuadas.
Isso faz com que se você acessar esse tipo de variável de dentro do assembly em linha,
você deve limpar manualmente os bits de ordem maior primeiro.

O solidity gerencia a memória de uma maneira simples: Existe um "ponteiro de memória livre"
na posição ``0x40`` da memória. Se você quiser alocar memória basta utilizar a memória daquele
ponto e realizar as alterações necessárias.

Elementos em vetores de memória no Solidity sempre ocupa múltiplos de 32 bytes (sim,
até mesmo para ``byte[]``, mas não para ``bytes`` e ``string``). Vetores multi dimensionais
de memória são ponteiros para vetores de memória. O comprimento de um vetor dinâmico
é armazenado na primeira posição do vetor, depois disso os elemntos são armazenados normalmente.

.. warning::
    Vetores de memória estáticamente dimensionados não possuem um espaço para o comprimento,
    mas isso vai ser alterado em breve para melhor convertibilidade entre vetores estáticos
    e dinâmicos. Por favor, não confie nesta conversão.


Assembly Autônomo
=================

A linguagem assembly descrita como assembly em linha também pode ser utilizada
de forma autônoma. Na verdade o plano é que ela seja utilizada como linguagem
intermediária para o compilador Solidity. Desta forma, ela tenta atingir diversos objetivos:

1. Programas escritos na linguagem devem ser legíveis, mesmo que o código tenha sido gerado por um compilador Solidity.
2. A tradução do assembly para o bytecode deve conter a menor quantidade de "surpresas" possível.
3. O controle de fluxo deve ser fácil de detectar para ajudar na verificação formal e otimização.

Para atingir o primeiro e último objetivos, o assembly fornece conceitos de alto nível
como loops ``for``, declarações de ``switch`` e chamadas de função. Deve ser possível
escrever programas assembly que não fazem uso explícito das declarações ``SWAP``, ``DUP``,
``JUMP`` e ``JUMPI``, pois os dois primeiros ofuscam o fluxo de data e os dois últimos
ofuscam o fluxo de controle. Além disso, declarações funcionais na forma ``mul(add(x, y), 7)``
são preferidas ao invés de opcode puro como ``7 y x add mul`` pois na primeira forma é muito
mais fácil ver qual operador é utilizado para cada opcode.

O segundo objetivo é atingido introduzindo a fase de desaçucarização que remove
somente os níveis mais altos de conceitos de uma maneira padrão que ainda permite
a inspeção do código assembly de baixo nível gerado pelo assembler. A única operação
não local feita pelo assembler é a pesquisa de nomes para identificadores definidos
pelo usuário (funções, variáveis, ...), que seguem regras simples de escopo e limpeza
de variáveis locais da pilha.

Âmbito: Um identificador que é declarado (rótulo, variável, função, assembly)
só é visível no bloco onde foi declarado (incluindo blocos aninhados detro do
bloco corrente). Não é permitido acessar variáveis locais entre fronteiras de
funções, mesmo que elas estejam no mesmo escopo. Shadowing não é permitido.
Variáveis locais não podem ser acessadas antes de serem declaradas, mas rótulos,
funções e assemblies podem. Assemblies são blocos especiais que por ex.
retornem código de tempo de execução ou criem contratos. Nenhum identificador
de um assembly externo é visível em um sub-assembly.

Se o fluxo de controle passar do fim de um bloco, instruções de pop são inseridas
para corresponder ao número de variáveis locais declaradas no bloco.
Sempre que uma variável local é referenciada, o gerador de código precisa
saber a posição relativa na pilha e consequentemente precisa manter o registro
da então altura da pilha. Já que todas as variáveis locais são removidas ao final
do bloco, a altura do bloco deverá ser a mesma. Se este não for o caso, um alerta
é disparado.

Por que utilizamos conceitos de alto nível como ``switch``, ``for`` e funções:

Com a utilização de ``switch``, ``for`` e funções, é possível escrever códigos
complexos sem a necessidade de utilizar ``jump`` ou ``jumpi`` manualmente.
Isso facilita a análize do controle de fluxo, o que melhora a verificação
formal e otimização.

Além disso, se saltos (jump) manuais fossem permitidos, computar a altura da pilha seria
muito complicado.
A posição de todas as variáveis na pilha precisam ser conhecidas, senão nenhuma das referências
às variáveis locais nem a remoção automática de variáveis locais iriam funcionar. O mecanismo de
desaçucarização insere operações corretamente em blocos inatingíveis para ajustar a altura do
bloco adequadamente em casos de saltos (jump) que não possuem um controle de fluxo contínuo.

Exemplo:

Vamos ver um exemplo de compilção de Solidity para um assembly desaçucarizado.
Considere o bytecode de tempo de execução do seguinte programa Solidity::

    pragma solidity ^0.4.0;

    contract C {
      function f(uint x) returns (uint y) {
        y = 1;
        for (uint i = 0; i < x; i++)
          y = 2 * y;
      }
    }

O assembly a seguir será gerado::

    {
      mstore(0x40, 0x60) // guarda o "ponteiro de memória livre"
      // function dispatcher
      switch div(calldataload(0), exp(2, 226))
      case 0xb3de648b {
        let (r) = f(calldataload(4))
        let ret := $allocate(0x20)
        mstore(ret, r)
        return(ret, 0x20)
      }
      default { revert(0, 0) }
      // alocação de memória
      function $allocate(size) -> pos {
        pos := mload(0x40)
        mstore(0x40, add(pos, size))
      }
      // a função do contrato
      function f(x) -> y {
        y := 1
        for { let i := 0 } lt(i, x) { i := add(i, 1) } {
          y := mul(2, y)
        }
      }
    }

Após a desaçucarização esse é o resultado::

    {
      mstore(0x40, 0x60)
      {
        let $0 := div(calldataload(0), exp(2, 226))
        jumpi($case1, eq($0, 0xb3de648b))
        jump($caseDefault)
        $case1:
        {
          // a chamada da função - colocamos o rótulo de retorno e argumentos na pilha
          $ret1 calldataload(4) jump(f)
          // Esse código é inacessível. Opcodes são adicionados para espelhar o
          // o efeito da função na altura da pilha: argumentos
          // são removidos e valores de retorno inseridos.
          pop pop
          let r := 0
          $ret1: // o ponto real de retorno
          $ret2 0x20 jump($allocate)
          pop pop let ret := 0
          $ret2:
          mstore(ret, r)
          return(ret, 0x20)
          // apesar de ser inútil, o salto (jump) é automaticamente inserido,
          // já que o processo de desaçucarização é uma operação puramente sintática
          // que não analisa o o fluxo de controle
          jump($endswitch)
        }
        $caseDefault:
        {
          revert(0, 0)
          jump($endswitch)
        }
        $endswitch:
      }
      jump($afterFunction)
      allocate:
      {
        // saltamos o código inacessível que introduz os argumentos da função
        jump($start)
        let $retpos := 0 let size := 0
        $start:
        // variáveis de saída vivem no mesmo escopo que os argumentos e estão
        // atualmente alocados.
        let pos := 0
        {
          pos := mload(0x40)
          mstore(0x40, add(pos, size))
        }
        // Esse código substitui os argumentos pelos valores de retorno e salta (jump) de volta.
        swap1 pop swap1 jump
        // Novamente código inacessível corrige a altura da pilha.
        0 0
      }
      f:
      {
        jump($start)
        let $retpos := 0 let x := 0
        $start:
        let y := 0
        {
          let i := 0
          $for_begin:
          jumpi($for_end, iszero(lt(i, x)))
          {
            y := mul(2, y)
          }
          $for_continue:
          { i := add(i, 1) }
          jump($for_begin)
          $for_end:
        } // Aqui, uma instrução pop será inserida para i
        swap1 pop swap1 jump
        0 0
      }
      $afterFunction:
      stop
    }


O assembly acontece em quatro estágios:

1. Análise
2. desaçucarização (remove switch, for e funções)
3. Gera o fluxo de opcodes
4. Geração de bytecode

Vamos especificar os passos de um a três de uma maneira pseudo formal.
Especificações mais formais virão em seguida.


Análise / Dicionário
--------------------

As tarefas do analisador são as seguintes:

- Transforma o fluxo de bytes em um fluxo de tokens, descarta comentários estilo C++
  (um comentário especial existe para referência de origem, mas não será explicado aqui).
- Transforma o fluxo de tokens em um AST de acordo com o dicionário abaixo
- Registra identificadores com o bloco onde estão definidos (anotação no nó AST) e
  anota à partir de qual ponto as variáveis podem ser acessadas.

O lexer do assembly é definido pelo próprio Solidity.

Espaço em branco é utilizado para delimitar tokens e consiste nos caracteres Espaço,
Tab e nova linha. Comentários no padrão JavaScript/C++ são interpretados como espaço em branco.

Grammar::

    AssemblyBlock = '{' AssemblyItem* '}'
    AssemblyItem =
        Identifier |
        AssemblyBlock |
        FunctionalAssemblyExpression |
        AssemblyLocalDefinition |
        FunctionalAssemblyAssignment |
        AssemblyAssignment |
        LabelDefinition |
        AssemblySwitch |
        AssemblyFunctionDefinition |
        AssemblyFor |
        'break' | 'continue' |
        SubAssembly | 'dataSize' '(' Identifier ')' |
        LinkerSymbol |
        'errorLabel' | 'bytecodeSize' |
        NumberLiteral | StringLiteral | HexLiteral
    Identifier = [a-zA-Z_$] [a-zA-Z_0-9]*
    FunctionalAssemblyExpression = Identifier '(' ( AssemblyItem ( ',' AssemblyItem )* )? ')'
    AssemblyLocalDefinition = 'let' IdentifierOrList ':=' FunctionalAssemblyExpression
    FunctionalAssemblyAssignment = IdentifierOrList ':=' FunctionalAssemblyExpression
    IdentifierOrList = Identifier | '(' IdentifierList ')'
    IdentifierList = Identifier ( ',' Identifier)*
    AssemblyAssignment = '=:' Identifier
    LabelDefinition = Identifier ':'
    AssemblySwitch = 'switch' FunctionalAssemblyExpression AssemblyCase*
        ( 'default' AssemblyBlock )?
    AssemblyCase = 'case' FunctionalAssemblyExpression AssemblyBlock
    AssemblyFunctionDefinition = 'function' Identifier '(' IdentifierList? ')'
        ( '->' '(' IdentifierList ')' )? AssemblyBlock
    AssemblyFor = 'for' ( AssemblyBlock | FunctionalAssemblyExpression)
        FunctionalAssemblyExpression ( AssemblyBlock | FunctionalAssemblyExpression) AssemblyBlock
    SubAssembly = 'assembly' Identifier AssemblyBlock
    LinkerSymbol = 'linkerSymbol' '(' StringLiteral ')'
    NumberLiteral = HexNumber | DecimalNumber
    HexLiteral = 'hex' ('"' ([0-9a-fA-F]{2})* '"' | '\'' ([0-9a-fA-F]{2})* '\'')
    StringLiteral = '"' ([^"\r\n\\] | '\\' .)* '"'
    HexNumber = '0x' [0-9a-fA-F]+
    DecimalNumber = [0-9]+


Desaçucarização
---------------

Uma transformação AST remove conceitos de for, switch e funções. O resultado
ainda é analisável pelo mesmo analisador, mas não utilizará certos conceitos.
Se jumpdests são adicionados e são apenas saltados para e não continuados em, informação
sobre a pilha é adicionada, ao menos que variáveis locais de fora do escopo não sejam acessadas
e que a altura da pilha seja a mesma que na última instrução.

Pseudocode::

    desugar item: AST -> AST =
    match item {
    AssemblyFunctionDefinition('function' name '(' arg1, ..., argn ')' '->' ( '(' ret1, ..., retm ')' body) ->
      <name>:
      {
        jump($<name>_start)
        let $retPC := 0 let argn := 0 ... let arg1 := 0
        $<name>_start:
        let ret1 := 0 ... let retm := 0
        { desugar(body) }
        swap and pop items so that only ret1, ... retm, $retPC are left on the stack
        jump
        0 (1 + n times) to compensate removal of arg1, ..., argn and $retPC
      }
    AssemblyFor('for' { init } condition post body) ->
      {
        init // cannot be its own block because we want variable scope to extend into the body
        // find I such that there are no labels $forI_*
        $forI_begin:
        jumpi($forI_end, iszero(condition))
        { body }
        $forI_continue:
        { post }
        jump($forI_begin)
        $forI_end:
      }
    'break' ->
      {
        // find nearest enclosing scope with label $forI_end
        pop all local variables that are defined at the current point
        but not at $forI_end
        jump($forI_end)
        0 (as many as variables were removed above)
      }
    'continue' ->
      {
        // find nearest enclosing scope with label $forI_continue
        pop all local variables that are defined at the current point
        but not at $forI_continue
        jump($forI_continue)
        0 (as many as variables were removed above)
      }
    AssemblySwitch(switch condition cases ( default: defaultBlock )? ) ->
      {
        // find I such that there is no $switchI* label or variable
        let $switchI_value := condition
        for each of cases match {
          case val: -> jumpi($switchI_caseJ, eq($switchI_value, val))
        }
        if default block present: ->
          { defaultBlock jump($switchI_end) }
        for each of cases match {
          case val: { body } -> $switchI_caseJ: { body jump($switchI_end) }
        }
        $switchI_end:
      }
    FunctionalAssemblyExpression( identifier(arg1, arg2, ..., argn) ) ->
      {
        if identifier is function <name> with n args and m ret values ->
          {
            // find I such that $funcallI_* does not exist
            $funcallI_return argn  ... arg2 arg1 jump(<name>)
            pop (n + 1 times)
            if the current context is `let (id1, ..., idm) := f(...)` ->
              let id1 := 0 ... let idm := 0
              $funcallI_return:
            else ->
              0 (m times)
              $funcallI_return:
              turn the functional expression that leads to the function call
              into a statement stream
          }
        else -> desugar(children of node)
      }
    default node ->
      desugar(children of node)
    }

Geração de Fluxo de Opcode
--------------------------

Durante a geração do fluxo de opcode mantemos registro da altura da pilha em um contador
para que seja possível acessar as variáveis à partir de seus nomes. A altura da pilha é
modificada com cada opcode que modifica a pilha e com cada rótulo que não é anotado com um
ajuste de pilha. Toda vez que uma nova variável local é introduzida ela é registrada com a
altura da pilha. Se uma variável é acessada (seja para cópia do valor ou para atribuição),
a instrução aprorpiada ``DUP`` ou ``SWAP`` é selecionada dependendo da diferença entre a
altura atual da pilha e a altura da pilha no momento em que a variável foi introduzida.

Pseudocode::

    codegen item: AST -> opcode_stream =
    match item {
    AssemblyBlock({ items }) ->
      join(codegen(item) for item in items)
      if last generated opcode has continuing control flow:
        POP for all local variables registered at the block (including variables
        introduced by labels)
        warn if the stack height at this point is not the same as at the start of the block
    Identifier(id) ->
      lookup id in the syntactic stack of blocks
      match type of id
        Local Variable ->
          DUPi where i = 1 + stack_height - stack_height_of_identifier(id)
        Label ->
          // referência  que deve ser resolvida durante a geração de bytecode
          PUSH<bytecode position of label>
        SubAssembly ->
          PUSH<bytecode position of subassembly data>
    FunctionalAssemblyExpression(id ( arguments ) ) ->
      join(codegen(arg) for arg in arguments.reversed())
      id (which has to be an opcode, might be a function name later)
    AssemblyLocalDefinition(let (id1, ..., idn) := expr) ->
      register identifiers id1, ..., idn as locals in current block at current stack height
      codegen(expr) - assert that expr returns n items to the stack
    FunctionalAssemblyAssignment((id1, ..., idn) := expr) ->
      lookup id1, ..., idn in the syntactic stack of blocks, assert that they are variables
      codegen(expr)
      for j = n, ..., i:
      SWAPi where i = 1 + stack_height - stack_height_of_identifier(idj)
      POP
    AssemblyAssignment(=: id) ->
      look up id in the syntactic stack of blocks, assert that it is a variable
      SWAPi where i = 1 + stack_height - stack_height_of_identifier(id)
      POP
    LabelDefinition(name:) ->
      JUMPDEST
    NumberLiteral(num) ->
      PUSH<num interpreted as decimal and right-aligned>
    HexLiteral(lit) ->
      PUSH32<lit interpreted as hex and left-aligned>
    StringLiteral(lit) ->
      PUSH32<lit utf-8 encoded and left-aligned>
    SubAssembly(assembly <name> block) ->
      append codegen(block) at the end of the code
    dataSize(<name>) ->
      assert that <name> is a subassembly ->
      PUSH32<size of code generated from subassembly <name>>
    linkerSymbol(<lit>) ->
      PUSH32<zeros> and append position to linker table
    }
