###############################
Introdução aos Smart Contracts
###############################

.. _simple-smart-contract:

*************************
Um simples Smart Contract
*************************

Deixe-nos começar com o exemplo mais básico. 
Sem problemas se você não entender tudo neste momento; 
Iremos entrar em maiores detalhes depois. 

Armazenamento
=============

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData;

        function set(uint x) {
            storedData = x;
        }

        function get() constant returns (uint) {
            return storedData;
        }
    }

A primeira linha simplesmente nos diz que o código fonte foi escrito para 
a versão 0.4.0 do Solidity ou mais recente e não quebra sua funcionalidade
(até, mas não incluindo a versão 0.5.0). Isso é para garantir que o 
contrato não se comporte de forma diferente com uma nova versão do compilador. 
A palavra chave ``pragma`` é chamada desta maneira, porque, no geral
pragmas são instruções para o compilador sobre como tratar o 
código fonte (por exemplo `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`).

Um contrato, no conceito do Solidity, é um conjunto de códigos (*functions*) e
dados (*state*), que residem em um específico endereço na rede blockchain Ethereum.
A linha ``uint storedData;`` declara uma variável de estado chamada ``storedData`` do 
tipo ``uint`` (integer não assinado de 256 bits). Você pode pensar nisso como um único slot 
em um banco de dados que pode ser consultado e alterado chamando funções do 
código que gerencia o banco de dados. No caso do Ethereum, ele é sempre o contrato 
proprietário. E neste caso, as funções ``set`` e ``get`` podem ser usadas para modificar
ou recuperar o valor da variável.

Para acessar uma variável de estado, você não precisa do prefixo ``this.``, como é comum em
outras linguagens. 

Este contrato ainda não faz muita coisa (devido à infra-estrutura 
construida pelo Ethereum), além de permitir que qualquer um possa armazenar um simples número que é acessível por
qualquer um no mundo sem uma maneira (viável) de impedir você de publicar 
este número. Naturalmente, qualquer um pode somente chamar novamente a função ``set``, com um valor diferente
e sobrescrever seu número original, mas o número permanecerá armazenado no histórico 
do blockchain. Mais tarde, iremos ver como você pode impor restrições de acesso, 
de maneira que somente você possa alterar este número.

.. note::
    Todos os identificadores (nomes de contratos, nomes de funções e nomes de varáveis) estão restritos
    à tabela de caracteres ASCII. É possível armazenar dados codificados UTF-8 em variáveis de string.

.. warning::
    Seja cuidadoso com o uso do texto Unicode, pois caracteres similares (ou mesmo idênticos) podem ter 
    pontos de código diferentes e, como tal, serão codificados como uma matriz de bytes diferente.
    

.. index:: ! submoeda

Exemplo de Sub Moeda
====================

O contrato seguinte irá implementar a forma mais simples de uma 
criptomoeda. É possível gerar moedas do nada, mas
somente a pessoa que criou o contrato terá a capacidade de fazer isto (é trivial
implementar um esquema de emissão diferente).
Além disso, qualquer um pode enviar moedas para outros, sem necessidade
de se registrar com usuário e senha. Tudo o que você precisa é de um par de
chaves Ethereum.

::

    pragma solidity ^0.4.0;

    contract Coin {
        // The keyword "public" makes those variables
        // readable from outside.
        address public minter;
        mapping (address => uint) public balances;

        // Events allow light clients to react on
        // changes efficiently.
        event Sent(address from, address to, uint amount);

        // This is the constructor whose code is
        // run only when the contract is created.
        function Coin() {
            minter = msg.sender;
        }

        function mint(address receiver, uint amount) {
            if (msg.sender != minter) return;
            balances[receiver] += amount;
        }

        function send(address receiver, uint amount) {
            if (balances[msg.sender] < amount) return;
            balances[msg.sender] -= amount;
            balances[receiver] += amount;
            Sent(msg.sender, receiver, amount);
        }
    }


Este contrato introduz alguns novos conceitos, Passaremos por eles passo à passo.

A linha ``address public minter;`` declara uma variável de estado do tipo endereço (address),
que é publicamente acessível.  O tipo ``address`` é um valor de 160 bits
que não permite nenhuma operação aritmética. É adequada para armazenar endereços de contratos ou 
pares de chaves pertencentes à entidades externas. A palavra-chave ``public`` automaticamente
gera uma função que lhe permite acessar o valor atual do estado da variável.
Sem esta palavra-chave, outros contratos podem não ter acesso à variável.
A função irá aparecer dessa maneira:

    function minter() returns (address) { return minter; }

Naturalmente, adicionando-se a função, exatamente como está, não irá funcionar
porque nós devemos ter a função e uma variável de estado com o mesmo nome, mas felizmente, 
você pegou a ideia - o compilador irá entender para você.

.. index:: mapping

A próxima linha, ``mapping (address => uint) public balances;``, também
cria uma variável públca de estado, mas é um tipo de dado mais complexo.
O tipo mapa endereça integers não assinadas.
Mappings podem ser vistas como `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_ 

Praticamente inicializado de tal forma que cada chave possível existe e é mapeado para 
um valor que representação em bytes é toda em zeros. Esta analogia não vai muito 
longe, porém, como não é possível obter uma lista de todas as chaves 
de um mapeamento nem uma lista de todos os valores. Tenha em mente (ou 
melhor, mantenha uma lista ou use um tipo de dados mais avançado) o que você
adicionou no mapeamento ou use em um contexto onde isto não é necessário,
como este. A função getter :ref:`getter function<getter-functions>` criada pela palavra chave ``public``
é um pouco mais complexa neste caso. Parece aproximadamente como o seguinte::


    function balances(address _account) returns (uint) {
        return balances[_account];
    }

Como você pode ver, você pode usar esta função para facilmente pesquisar o saldo de 
uma conta simples.


.. index:: event

A linha ``event Sent(address from, address to, uint amount);`` declara
um auto-nomeado "event" que é eliminado na última linha da função  
``send``. Interfaces de usuário (assim como uma aplicação de servidor, naturalmente) pode
escutar estes eventos sendo eliminados no blockchain sem muito custo.
Assim que é eliminado, o listener irá receber o
argumento ``from``, ``to`` e ``amount``, que torna fácil rastrear
transações. Para escutar estes eventos, você pode usar ::


    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

 
 Perceba como a função ``balances``, gerada automaticamente, é chamada a partir
 do interface do usuário. 

.. index:: coin

A função especial ``Coin`` é o contrutor que é executado durante a criação do contrato e
não pode ser chamada depois. Ela armazena permanentemente o endereço da pessoa que criou
o contrato; ``msg`` (junto com ``tx`` e ``block``) é uma variável global mágica que 
contém algumas propriedades que permitem acessar o blockchain. ``msg.sender`` é 
sempre o endereço onde a função corrente (externa) é originada.

Finalmente, a função que era realmente encerrar com o contrato e pode ser chamada
pelos usuários e contratos são ``mint`` and ``send``.
Se ``mint`` é chamada por alguém que não a conta que criou o contrato, 
nada irá acontecer. Por outro lado, ``send`` pode ser usado por qualquer um (que já 
tenha alguma dessa moeda) para enviar moedas para alguém. Perceba que se você usar
este contrato para enviar moedas para um endereço, você não irá ver nada quando você
olhar para este endereço através de um explorador blockchain, porque o fato de você ter enviado
as moedas e os saldos alterados só são armazenados no armazenamento de dados deste
Contrato de moeda particular. Com o uso de eventos, é relativamente fácil criar
um "blockchain explorer" que rastreia transações e saldos da sua nova moeda.
 
.. _blockchain-basics:

************************
Princípios do Blockchain
************************

Blockchain como um conceito não é muito difícil de entender por programadores. A razão é que
a maioria das complicações (mining, `hashing <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `peer-to-peer networks <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.)
é justamente para prover um conjunto de características e promessas. Uma vez que você aceitas
estas características como certas, você não precisa se preocupar com a tecnologia adjacente - ou você precisa
saber como a Amazon´s AWZ trabalham internamente para usá-las?


.. index:: transaction

Transações
============

O blockchain é uma base de dados transacional globalmente distribuída.
Isso significa que qualquer um pode ler as entradas do banco de dados, somente por participar da rede.
Se você deseja mudar alguma coisa no banco de dados, você tem que criar uma transação 
que precisa ser aceita por todos os demais (participantes).

A palava transação implica que a mudança que você quer fazer (assumindo que você queira mudar
dois valores ao mesmo tempo) não é feita nem aplicada completamente. Além disso,
enquanto sua transação é aplicada ao banco de dados, nenhuma outra transação pode alterá-la.

Por exemplo, imagine uma tabela que lista os saldos de todas as contas em um
moeda eletrônica. Se uma transferência de uma conta para outra for solicitada,
a natureza transacional do banco de dados garante que, se o valor for
subtraído de uma conta, é sempre adicionado à outra conta.  Se, por qualquer motivo, 
não for possível adicionar o valor à conta destino, a conta de origem também não será modificada.

Por exemplo, imagine uma tabela que lista os saldos de todas as contas de uma
moeda eletrônica. Se uma transferência de uma conta para outra for solicitada,
a natureza transacional do banco de dados garante que, se o valor for
subtraído de uma conta, é sempre adicionado à outra conta. Se, por qualquer motivo, 
não for possível adicionar o valor à conta destino, a conta de origem também não será modificada.

Além disso, uma transação sempre é criptograficamente assinada pelo remetente (seu criador).
Isso torna direto proteger o acesso a modificações específicas da
base de dados. No exemplo da moeda eletrônica, uma verificação simples garante que
apenas a pessoa que possua as chaves da conta possa transferir dinheiro com ela.


.. index:: ! block

Blocks
======

Um dos principais obstáculos a superar é o que, em termos Bitcoin, é chamado de "ataque de dupla despesa":
O que acontece se ocorrer duas transações na rede que querem esvaziar uma conta,
um chamado conflito?

A resposta abstrata à isso é que você não precisa se preocupar. Uma ordem das transações
será selecionada para você, as transações serão empacotadas no que é chamado de "bloco" (block)
e então eles serão executados e distribuídos entre todos os nós participantes.
Se duas transações se contradizem, o que acaba sendo a segunda transação será
rejeitada e não se tornará parte do bloco.

Esses blocos formam uma seqüência linear no tempo e é aí que deriva o termo "bloco em cadeia" (blockchain). 
Os blocos são adicionados à cadeia em intervalos bastante regulares - para o
Ethereum é aproximadamente a cada 17 segundos.


Como parte do "mecanismo de seleção de pedidos" (que é chamado de (mining) "mineração"), pode acontecer 
que os blocos são revertidos de tempos em tempos, mas apenas na "ponta" da corrente. Quanto mais 
blocos são adicionados no topo, menos provável é. Então, pode ser que suas transações
sejam revertidas e até mesmo removidas do blockchain, mas quanto mais você aguardar, menor a probabilidade.

.. _the-ethereum-virtual-machine:

.. index:: !evm, ! ethereum virtual machine

********************************
A Máquina Virtual Ethereum (EVM)
********************************

Visão Geral
===========

A Máquina Virtual Ethereum (Ethereum Virtual Machine - EVM) é o ambiente em tempo de execução
para contratos inteligentes (smart contracts) no Ethereum. Não é apenas sandbox, mas
realmente isolado completamente, o que significa que o código está sendo executado
pelo EVM não tem acesso à rede, sistema de arquivos ou outros processos.
Os contratos inteligentes (smart contracts) têm acesso limitado à outros contratos inteligentes.


.. index:: ! account, address, storage, balance

Contas (Accounts)
=================

Existem dois tipos de contas (accounts) no Ethereum que compartilham o mesmo
espaço de endereço: **External accounts** (contas externas) que são controladas por
pares de chaves público-privadas (isto é, por humanos) e **contract accounts** (contas contratuais) que são
controladas pelo código armazenado junto com a conta.

O endereço de uma conta externa é determinado a partir 
da chave pública enquanto o endereço de um contrato é
determinado no momento em que o contrato é criado
(é derivado do endereço do criador e do número
das transações enviadas a partir desse endereço, o chamado "nonce").

Independentemente ou não do código de armazenamento das contas, os dois tipos são
tratados igualmente pelo EVM.

Cada conta possui um valor-chave persistente mapeado de 256-bits 
chamado **storage** de 256-bits

Além disso, cada conta tem **balance** (saldo) em Ether (em "Wei" para ser exato) 
que pode ser modificado enviando transações que incluem Ether.

.. index:: ! transaction

Transações (Transactions)
=========================

Cada transação (transaction) é uma mensagem que é enviada de uma conta para outra conta 
(que pode ser a mesma ou uma conta especial zero-account, veja abaixo).
Pode incluir dados binários (sua carga útil) e Ether.

Se a conta alvo contiver um código, esse código é executado e
a carga útil é fornecida como dados de entrada.

Se a conta alvo for do tipo conta zero (a conta com o
endereço ``0``), a transação cria um novo contrato **new contract**.

Como já mencionado, o endereço desse contrato não é
o endereço zero, mas um endereço derivado do remetente e
o número de transações enviadas (o "nonce"). A carga útil
de tal transação de criação de contrato é considerada como sendo
Bytecode EVM e executado. A saída desta execução é
permanentemente armazenada como o código do contrato.
Isso significa que, para criar um contrato, você não
envia o código real do contrato, mas, de fato, o código que
retorna esse código.


.. index:: ! gas, ! gas price

Gas
===

Após a criação, cada transação é cobrada com uma certa quantidade de **gas**,
cujo objetivo é limitar a quantidade de trabalho que é necessária para executar
a transação e para pagar esta execução. Enquanto o EVM executa o
transação, o gás é gradualmente usado de acordo com regras específicas.

O **gas price** é um valor definido pelo criador da transação, quem
tem que pagar o o resultado de ``gas_price * gas`` antes da conta de envio.
Se algum gás for deixado após a execução, ele será reembolsado da mesma maneira.

Se o gás for esgotado em qualquer ponto (isto é, o saldo disponível de Gas fica negativo),
é desencadeada uma exceção em gas (out-of-gas), que reverte todas as modificações
feito para o estado no quadro de chamada atual.

.. index:: ! storage, ! memory, ! stack

Armazenamento, Memória e Pilha
==============================

Cada conta tem uma área de memória persistente chamada **storage**.
Storage é um valor-chave que mapeia palavras de 256-bits para palavras de 256-bits.
Não é possível enumerar o armazenamento dentro de um contrato
e é comparativamente caro para ler e ainda mais, para modificar armazenamento. 
Um contrato não pode ler nem escrever em qualquer armazenamento separado
por conta própria.

A segunda área de memória é chamada de **memory**, de que um contrato obtém
uma instância recém autorizada para cada chamada de mensagem. A memória é linear e pode ser
endereçada no nível do byte, mas as leituras são limitadas a uma largura de 256 bits, enquanto escrever
pode ser de 8 bits ou 256 bits de largura. A memória é expandida por uma palavra (256 bits), enquanto
acessando (quer lendo ou escrevendo) uma palavra de memória anteriormente intacta (ou seja, qualquer deslocamento
dentro de uma palavra). No momento da expansão, o custo em Gas deve ser pago. A memória é mais
cara a medida que cresce (em escala quadrática).

O EVM não é uma register machine mas uma stack machine, portanto
todas os comandos são realizados em uma área chamada **stack**. 
Tem o tamanho máximo de 1024 elementos e contém palavras de 256 bits. O acesso à pilha (stack) 
é limitado ao topo (top end) na seguinte maneira:
É possível copiar um dos os 16 elementos superiores para o topo da pilha ou trocar o
elemento superior com um dos 16 elementos abaixo.
Todas as outras operações levam os dois primeiros (ou uma, ou mais, dependendo de
a operação) elementos da pilha e empurre o resultado para a pilha.
Naturalmente é possível mover elementos de pilha para armazenamento ou memória,
mas não é possível acessar elementos arbitrários mais profundos na pilha
sem primeiro remover o topo da pilha.


.. index:: ! instruction

Lista de Instruções
===================

A lista de instruções do EVM é mantido no mínimo para evitar
implementações incorretas que podem causar problemas de consenso.
Todas as instruções operam no tipo básico de dados, palavra de 256-bits.
As operações mais usuais de aritmética, bit, lógica e comparação estão presentes.
Desvios condicionais e não-condicionais são possíveis.
Além disso, contratos podem acessar propriedades relevantes do bloco atual
como seu número e carimbo de tempo (timestamp).


.. index:: ! message call, function;call

Chamadas por Mensagem
=====================

Contratos podem chamar outros contratos ou enviar Ether para 
contas não-contrato através de chamadas de mensagens.
As chamadas de mensagens são semelhantes
para as transações, na medida em que eles têm uma origem, um destino, carga útil de dados,
dados de Ether, Gas e informações de retorno. Na verdade, cada transação consiste em
uma chamada de mensagem de nível superior que, por sua vez, pode criar mais chamadas de mensagens.

Um contrato pode decidir quanto do **gas** restante deve ser enviado
com a chamada de mensagem interna e quanto ele deseja reter.
Se ocorrer uma exceção out-of-gas (sem gas) na chamada interna (ou qualquer
outra exceção), isso será sinalizado por um valor de erro colocado na pilha.
Neste caso, somente o gas enviado junto com a chamada é consumido.
No Solidity, o contrato de chamada causa uma exceção manual por padrão em
tais situações, de modo que as exceções "expandam" a pilha de chamadas.

Como já dissemos, o contrato chamado (que pode ser o mesmo que o chamador)
receberá uma instância de memória recém autorizada e terá acesso à
chamada carga útil - que será fornecida em uma área separada chamada **calldata**.
Depois de terminar a execução, pode retornar dados que serão armazenados em
uma localização na memória do chamador pré-alocada pelo chamador.

As chamadas são limitadas (**limited**) para uma profundidade de 1024, o que significa que para operações mais complexas, 
loops devem ser preferidos em relação a chamadas recursivas.


.. index:: delegatecall, callcode, library

Delegatecall / Callcode and Libraries
=====================================

Existe uma variante especial de chamada de mensagem, chamada **delegatecall**
que é identica à chamada de mensagem, exceto pelo fato que 
o código no endereço de destino é executado no contexto do contrato
chamado e ``msg.sender`` e ``msg.value`` não alteram seus valores.

Isto significa que um contrato pode dinamicamente carregar código de diferentes
endereços em tempo de execução. Armazenamento (Storage), Endereço Atual (current address) e saldo (Balance)
podem ainda se referir ao contrato chamador, apenas o código é retirado do endereço chamado.

Isto torna possível a implementação da característica de "library" no Solidity:
Códigos reusáveis de "library" podem ser aplicados ao armazenamento (storage) dos contratos, normalmente
para implementar estrutura de dsos complexas.

.. index:: log

Logs
====

É possível armazenar dados em uma estrutura de dados especialmente indexada
que mapeia até o nível do bloco. Esta característica chamada **logs**
é usada pelo Solidity para implementar eventos (**events**).
Os contratos não podem acessar os dados de log depois de terem sido criados, mas eles
pode ser acessado de forma eficiente, fora da cadeia de blocos.
Desde que algumas partes dos dados do log são armazenados no 
`bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_,
é possível pesquisar por estes dados de uma maneira eficiente e criptograficamente 
segura, sendo assim, outros participantes (clientes) da rede que não baixarem o blockchain todo 
("light clients"), podem, asinda, assim, encontrar estes logs. 


.. index:: contract creation

Create
======

Contratos podem ainda criar outros contratos usando um opcode especial 
(em geral, eles não chamam o zero address). A única diferença entre 
esses **create calls** e uma chamada de mensagem normal é que a carga útil (payload)
é executada e o resultado é armazenado como code e o chamador/criador 
recebe o endereço deste novo contrato na pilha (stack).

.. index:: selfdestruct

Self-destruct
=============

A única maneira de um código ser removido do blockchain é 
quando o contrato que ele endereça realizar a operação ``selfdestruct``.
O Ether remanescente, armazenado neste endereço, é enviado para um
destino designado e então o armazenamento e o código é removido.

.. warning:: mesmo quando o código do contrato não contém uma chamada para ``selfdestruct``,
  ele pode ainda realizar esta operação chamando ``delegatecall`` or ``callcode``.

.. note:: A poda de contratos antigos pode ou não ser implementada pelos Clientes Ethereum.
  Além disso, os nós de arquivo podem optar por manter o armazenamento do contrato e código indefinidamente.

.. note:: Atualmente **external accounts** não podem ser removidas do estado.
