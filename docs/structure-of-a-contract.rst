.. index:: contract, state variable, function, event, struct, enum, function;modifier

.. _contract_structure:

************************
Estrutura de um contrato
************************

Contratos em Solidity são similares a classes em linguagens orientadas a objetos.
Cada contrato pode conter declarações de :ref:`structure-state-variables`, :ref:`structure-functions`,
:ref:`structure-function-modifiers`, :ref:`structure-events`, :ref:`structure-structs-types` and :ref:`structure-enum-types`.
Além disso, contratos podem herdar de outros contratos.

Contracts in Solidity are similar to classes in object-oriented languages.
Each contract can contain declarations of :ref:`structure-state-variables`, :ref:`structure-functions`,
:ref:`structure-function-modifiers`, :ref:`structure-events`, :ref:`structure-structs-types` and :ref:`structure-enum-types`.
Furthermore, contracts can inherit from other contracts.

.. _structure-state-variables:

Variáveis de Estado
===================

Variáveis de estado são valores armazenados permanentemente na memória de contrato.

::

  pragma solidity ^0.4.0;

  contract SimpleStorage {
      uint storedData; // Variáveis de estado
      // ...
  }

Veja a seção :ref:`types` para tipos válidos de variáveis de estado e 
:ref:`visibility-and-getters` para possíveis escolhad de visibilidade.

.. _structure-functions:

Funções
=======

Funções são unidades executáveis de código dentro de um contrato.

::

  pragma solidity ^0.4.0;

  contract SimpleAuction {
      function bid() payable { // Função
          // ...
      }
  }

:ref:`function-calls` podem acontecer internamente ou externamente
e tem diferentes níveis de visibilidade (:ref:`visibility-and-getters`)
para outros contratos.

.. _structure-function-modifiers:

Modificadores de Função
=======================

Modificadores de função podem ser usados para alterar a semântica das funções de forma declarativa
(veja :ref:`modifiers` na seção contratos).

::

  pragma solidity ^0.4.11;

  contract Purchase {
      address public seller;

      modifier onlySeller() { // Modificador 
          require(msg.sender == seller);
          _;
      }

      function abort() onlySeller { // Uso do Modificador
          // ...
      }
  }

.. _structure-events:

Eventos
=======

Eventos são interfaces de conveniência com facilidades de log na EVM.

::

  pragma solidity ^0.4.0;

  contract SimpleAuction {
      event HighestBidIncreased(address bidder, uint amount); // Evento

      function bid() payable {
          // ...
          HighestBidIncreased(msg.sender, msg.value); // Evento Trigger
      }
  }

Consulte: ref: `eventos` na seção de contratos para obter informações sobre como eventos são declarados
e podem ser usados dentro de uma dapp.

.. _structure-structs-types:

Tipos de Estrutura
==================

Estruturas são tipos personalizados que podem agrupar diversas variáveis (ver
: ref: `structs` na seção de tipos).

::

  pragma solidity ^0.4.0;

  contract Ballot {
      struct Voter { // Estrutura
          uint weight;
          bool voted;
          address delegate;
          uint vote;
      }
  }

.. _structure-enum-types:

Tipos de Enum 
=============

Enums podem ser usados para criar tipos personalizados com um conjunto finito de valores (veja
: ref: `enums` na seção de tipos).

::

  pragma solidity ^0.4.0;

  contract Purchase {
      enum State { Created, Locked, Inactive } // Enum
  }
