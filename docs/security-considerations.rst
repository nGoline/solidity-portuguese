.. _security_considerations:

##########################
Considerações de Segurança
##########################

Embora geralmente seja bastante fácil criar software que funcione como o esperado,
é muito mais difícil verificar que ninguém possa usá-lo de uma forma **não** antecipada.

No Solidity, isso é ainda mais importante porque você pode usar contratos inteligentes (smart contracts)
para manipular tokens ou, possivelmente, coisas ainda mais valiosas. Além disso, cada
execução de um contrato inteligente acontece em público e, adicionalmente,
o código-fonte está frequentemente disponível.

É claro que você sempre tem que considerar o quanto está em jogo:
Você pode comparar um smart contract com um serviço da Web aberto para o
público (e, portanto, também para atores mal-intencionados) e talvez até fonte aberta.
Se você apenas armazena sua lista de compras nesse serviço da Web, talvez você não tenha
que se preocupar muito, mas se você gerencia sua conta bancária usando esse serviço web,
você deve ser mais cuidadoso.

Esta seção listará algumas armadilhas e recomendações gerais de segurança, mas
pode, é claro, nunca estar completa. Além disso, tenha em mente que, mesmo que o seu
código do contrato inteligente seja livre de erros, o compilador ou a própria plataforma pode
tem um bug. Uma lista de alguns erros conhecidos de segurança pública do compilador
pode ser encontrado no 
:ref:`list of known bugs<known_bugs>`, que também é legível por máquina. 

Note que existe um programa de recompensa de erros (bugs) que abrange o gerador de código do
compilador de Solidity.

Como sempre, com documentação open source, ajude-nos a expandir esta seção 
Especialmente, com alguns exemplos que não causem problemas!


*********************
Armadilhas (Pitfalls)
*********************

Informações Privativas e Aleatoriedade
======================================

Tudo o que você usa em Smart Contracts está visível ao público, mesmo
variáveis locais (local variables) e variáveis de estado (state variables) 
marcadas como privativas ``private``

Usar números aleatórios em contratos inteligentes é bastante complicado se você não quiser
mineiros habilitados para trapacear.


Re-Entrancy
===========

Qualquer interação de um contrato (A) com outro contrato (B) e qualquer transferência
de Ether, entrega o controle a esse contrato (B). Isso permite que (B)
chamar de volta para (A) antes que esta interação seja concluída. 
Para dar um exemplo, O código a seguir contém um erro (é apenas um trecho e não um
contrato completo):

::

  pragma solidity ^0.4.0;

  // THIS CONTRACT CONTAINS A BUG - DO NOT USE
  contract Fund {
      /// Mapping of ether shares of the contract.
      mapping(address => uint) shares;
      /// Withdraw your share.
      function withdraw() {
          if (msg.sender.send(shares[msg.sender]))
              shares[msg.sender] = 0;
      }
  }


O problema não é muito grave aqui por causa do gás limitado como parte
de ``send``, mas ainda assim expõe uma fraqueza: transferência de Ether sempre
inclui a execução do código, para que o destinatário possa ser um contrato que chama
de volta para ``withdraw``. Isso permitiria obter reembolsos múltiplos e
basicamente, recupera todo o Ether no contrato.

Para evitar re-entrancy, você pode usar o padrão Checks-Effects-Interactions como
descrito abaixo:


::

  pragma solidity ^0.4.11;

  contract Fund {
      /// Mapping of ether shares of the contract.
      mapping(address => uint) shares;
      /// Withdraw your share.
      function withdraw() {
          var share = shares[msg.sender];
          shares[msg.sender] = 0;
          msg.sender.transfer(share);
      }
  }


Perceba que re-entrancy não somente um efeito de transferência de Ether, 
mas de qualquer função chamada em outro contrato. Além disso, você pode ter
situações de multi-contratos dentro da conta. Um contrato chamado pode modificar o
estado do outro contrato de que você depende.


Gas Limit e Loops
===================

Loops que não tenham um número fixo de interações, por exemplo, loops que dependem de valores armazenados, 
tem que ser usados com muito cuidado:
Devido ao limite de Gas por bloco, as transações podem consumir somente uma certa quantidade de Gas.
Explicitamente ou apenas devido a operação normal, o número de interações em um loop pode crescer além do 
limite de gás de bloqueio, que pode causar o completo contrato para ser parado em um determinado ponto. 
Isso pode não se aplicar às funções ``constant`` que são executadas apenas
para ler dados da cadeia de blocos. Ainda assim, tais funções podem ser chamadas por outros contratos como parte das operações na cadeia
e paralisar aqueles. 

Seja pormenorizado sobre tais casos na documentação dos seus contratos.


Mandando e Recebendo Ether
============================

- Nem os contratos nem "external accounts" atualmente podem impedir que alguém lhes envie Ether.
  Os contratos podem reagir e rejeitar uma transferência regular, mas existem formas
  para mover Ether sem criar uma chamada de mensagem. Uma maneira é simplesmente "mine to" (minerar para)
  o endereço do contrato e a segunda maneira é usar ``selfdestruct(x)``.

- Se um contrato receber Ether (sem uma função chamada), a função de retorno (fallback function) é executada.
  Se não tiver uma função de retorno, o Ether será rejeitado (jogando uma exceção).
  Durante a execução da função de retorno, o contrato só pode confiar
  sobre o "gas stipend" (gasto de gas) (2300 gas) que estava disponível naquele momento. 
  Este gasto não é suficiente para acessar o armazenamento de qualquer maneira.
  Para ter certeza de que seu contrato pode receber Ether dessa maneira, verifique os requisitos de gas da função de retorno
  (por exemplo, na seção "details" em Remix).

- Existe uma maneira de encaminhar mais gas para o contrato de recebimento usando
  a função ``addr.call.value(x)()``. Isto é essencialmente o mesmo que a função
  ``addr.transfer(x)``, só que encaminhando o gas restante e abre a capacidade de
  destinatário para executar ações mais caras (e apenas retorna um código de falha
  e não propaga automaticamente o erro). Isso pode incluir chamar de volta
  no contrato de envio ou outras mudanças de estado em que você talvez não tenha pensado.
  Assim, permite uma grande flexibilidade para usuários honestos, mas também para atores maliciosos.

- Se você quiser enviar Ether usando ``address.transfer``, existem certos detalhes para se dar conta:
 
  1. Se o destinatário for um contrato, ele faz com que sua função de retorno seja executada, o que pode, por sua vez, chamar de volta o contrato de envio.
  2. O envio de Ether pode falhar devido à profundidade da chamada acima de 1024. Como o chamador (caller) está no controle total da profundidade da chamada,
     ele (caller) pode forçar a transferência para provocar a falhar; Tire essa possibilidade em consideração ou use ``send`` e certifique-se sempre de 
     verificar  seu valor de retorno. Melhor ainda, escreva seu contrato usando um padrão em que o destinatário pode retirar Ether em vez disso.
  3. O envio de Ether pode falhar porque a execução do contrato do destinatário
     requer mais do que a quantidade atribuída de gás (``require``,
     ``assert``, ``revert``, ``throw``ou porque a operação é muito cara) - "runs out of gas" (OOG).
     Se você usar ``transfer`` ou ``send`` com uma verificação de valor de retorno, isso pode fornecer um
     meio do destinatário bloquear o progresso no contrato de envio. Mais uma vez, a melhor prática aqui é usar
     :ref:`"withdraw" pattern instead of a "send" pattern <withdrawal_pattern>`.


Callstack Depth
===============

Chamadas de funções externas podem falhar a qualquer tempo porque excedem a 
profundidade de chamadas de 1024 pilhas. Em tais situações, o Solidity gera uma exceção.
Atores maliciosos podem forçar uma chamada de pilha para este alto valor
antes de interagir com seu contrato.

Perceba que ``.send()`` não não **not** lança uma exceção se a pilha de chamadas for
sendo empobrecida, mas sim retorna ``false`` nesse caso. As funções de baixo nível
``.call()``, ``.callcode()`` e ``.delegatecall()`` comportam-se da mesma maneira.

tx.origin
=========

Nunca use tx.origin para autorização. Vamos supor que você tenha um contrato de wallet conforme segue:

::

    pragma solidity ^0.4.11;

    // THIS CONTRACT CONTAINS A BUG - DO NOT USE
    contract TxUserWallet {
        address owner;

        function TxUserWallet() {
            owner = msg.sender;
        }

        function transferTo(address dest, uint amount) {
            require(tx.origin == owner);
            dest.transfer(amount);
        }
    }

Agora, alguém o engana para enviar Ether para o endereço desta carteira de ataque::

    pragma solidity ^0.4.11;

    interface TxUserWallet {
        function transferTo(address dest, uint amount);
    }

    contract TxAttackWallet {
        address owner;

        function TxAttackWallet() {
            owner = msg.sender;
        }

        function() {
            TxUserWallet(msg.sender).transferTo(owner, msg.sender.balance);
        }
    }

Se a sua carteira tivesse verificado ``msg.sender`` para obter autorização, obteria o endereço da carteira de ataque, em vez do endereço do proprietário.
Mas, ao verificar `tx.origin``, obtém o endereço original que iniciou a transação, que ainda é o endereço do proprietário. A carteira de ataque drena instantaneamente todos os seus fundos.


Detalhes Menores
================

- Em ``for (var i = 0; i < arrayName.length; i++) { ... }``, o tipo de ``i`` será ``uint8``, porque este é o menor tipo que é necessário 
  para manter o valor ``0``. Se a matriz tiver mais de 255 elementos, o loop não será encerrado.

- A palavra-chave ``constant`` para funções atualmente não é aplicada pelo compilador.
  Além disso, não é aplicada pelo EVM, então uma função de contrato que "reivindica"
  para ser uma constante pode ainda causar alterações no estado.

- Tipos que não ocupam os 32 bytes completos podem conter "bits sujos de ordem superior" (dirty higher order bits).
  Isto é especialmente importante se você acessar ``msg.data`` - isso representa um risco de maleabilidade:
  Você pode criar transações que chamam a função ``f(uint8 x)`` com um argumento de byte bruto
  de ``0xff000001`` e com ``0x00000001``. Ambos são alimentados ao contrato e ambos irão
  parecer que o número ``1`` até ``x`` está em causa, mas ``msg.data`` irá
  parecer diferente, então, se você usar ``keccak256(msg.data)`` para qualquer coisa, você obterá resultados diferentes.


*************
Recomendações
*************

Restringir o total de Ether
===========================

Restringir o total de Ether (ou outros tokens) que pode ser armazenado em um smart
contract. Se o seu código fonte, o compilador ou a plataforma tiver um erro (bug), estes
fundos poderão ser perdidos. Se você quiser limitar suas perdas, limite o total de Ether.

Mantenha Pequeno e Modular
==========================

Mantenha seu contrato pequeno e facilmente compreensível. Demonstre funcionalidades
não relacionadas em outros contratos ou bibliotecas. Recomendações gerais sobre
a qualidade do código fonte naturalmente se aplicam: Limite o total de variáveis locais,
o comprimento das funções e assim por diante; Documente suas funções para que outros
podem ver quais eram suas intenções e se é diferente do que o código executa. 


Use o padrão Checks-Effects-Interactions
========================================

A maioria das funções irá executar primeiro algumas verificações (quem chamou a função,
quais são os argumentos, se foi enviado Ether suficiente, se a pessoa
tem tokens, etc.). Essas verificações devem ser feitas primeiro.

Como o segundo passo, se todas as verificações passaram, os efeitos nas variáveis de estado
do contrato atual devem ser feitos. Interação com outros contratos devem ser o último passo 
em qualquer função.

Contratos iniciais retardam alguns efeitos e aguardaram a função externa
chamada para retornar em um estado sem erro. Este é muitas vezes um grave erro
devido ao problema de re-entrada (re-entrancy) explicado acima.

Observe também que as chamadas para contratos conhecidos podem, por sua vez, causar chamadas para
contratos desconhecidos, por isso provavelmente é melhor simplesmente aplicar esse padrão.

Incluir um Modo de Falha Segura (Fail-Fase Mode)
================================================

Ao tornar seu sistema totalmente descentralizado, você removerá qualquer intermediário,
Pode ser uma boa ideia, especialmente para o novo código, incluir algum tipo
de mecanismo de segurança:

Você pode adicionar uma função no seu contrato inteligente que executa algumas
auto-verificações como "Algum Ether vazou?" (Has any Ether leaked?),
"A soma dos tokens é igual ao saldo do contrato?"  ("Is the sum of the tokens equal to 
the balance of the contract?") e situações similares.

Tenha em mente que você não pode usar muito gas para isso, por isso pensar fora da
cadeia (off-chain) pode ser necessário aqui.

Se a auto-verificação falhar, o contrato muda automaticamente para algum tipo
do modo "failsafe", que, por exemplo, desabilita a maioria dos recursos, entrega
o controle para uma terceira parte fixa e confiável ou simplesmente converte o contrato em
um simples contrato "devolva meu dinheiro" (give me back my money).


******************
Verificação Formal
******************

Usando a verificação formal, é possível realizar uma prova matemática automatizada
de que o seu código fonte cumpre uma determinada especificação formal. A especificação 
ainda é formal (assim como o código fonte), mas geralmente muito mais simples.

Observe que a verificação formal em si só pode ajudá-lo a entender a
diferença entre o que você fez (a especificação) e como você fez isso
(a implementação real). Você ainda precisa verificar se a especificação
é o que você queria e que não perdeu os efeitos não intencionais.