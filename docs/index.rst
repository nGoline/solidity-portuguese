Solidity
========

.. image:: logo.svg
    :width: 120px
    :alt: Solidity logo
    :align: center

Solidity é uma linguagem de programação de alto nível, orientada a contratos, com a síntese parecida
com a de JavaScript e desenhada para ser executada na Máquina Virtual Ethereum (EVM).

Solidity é estatisticamente tipada, suporta herança, bibliotecas e tipos
complexos definidos pelo usuário entre outras características.

Como você verá, é possível criar contratos para votação,
vaquinhas (crowdfunding), leilões às cegas, carteiras multi-assinadas e mais.

.. note::
    A melhor maneira de experimentar com Solidity é utilizando
    `Remix <https://remix.ethereum.org/>`_
    (pode demorar para carregar, seja paciente).

Links úteis
------------

* `Ethereum <https://ethereum.org>`_

* `Lista de mudanças <https://github.com/ethereum/solidity/blob/develop/Changelog.md>`_

* `Story Backlog <https://www.pivotaltracker.com/n/projects/1189488>`_

* `Código Fonte <https://github.com/ethereum/solidity/>`_

* `Ethereum Stackexchange <https://ethereum.stackexchange.com/>`_

* `Gitter Chat <https://gitter.im/ethereum/solidity/>`_

Integraçoes Disponíveis para Solidity
-------------------------------------

* `Remix <https://remix.ethereum.org/>`_
    IDE baseada em Browser com compilador integrado e ambiente de tempo de execução Solidity sem componentes de servidor.

* `IntelliJ IDEA plugin <https://plugins.jetbrains.com/plugin/9475-intellij-solidity>`_
    Plugin Solidity para IntelliJ IDEA (e todas as outras IDEs JetBrains)

* `Extensão para Visual Studio <https://visualstudiogallery.msdn.microsoft.com/96221853-33c4-4531-bdd5-d2ea5acc4799/>`_
    Plugin Solidity para Microsoft Visual Studio que inclui o compilador Solidity.

* `Pacote para SublimeText — síntese da linguagem Solidity <https://packagecontrol.io/packages/Ethereum/>`_
    Marcação de síntese Solidity para o editor de texto SublimeText.

* `Etheratom <https://github.com/0mkara/etheratom>`_
    Plugin para o editor Atom que conta com marcação de síntese, compilação e ambiente de tempo de execução (Compatível com backend node & VM).

* `Atom Solidity Linter <https://atom.io/packages/linter-solidity>`_
    Plugin para o editor Atom para Solidity linting.

* `Atom Solium Linter <https://atom.io/packages/linter-solium>`_
    Solidty linter configurável para Atom utilizando Solium como base.

* `Solium <https://github.com/duaraghav8/Solium/>`_
    Lint de linha de comando para Solidity que segue estritamente as regras descritas no `Guia de Estilos para Solidity <http://solidity.readthedocs.io/en/latest/style-guide.html>`_.

* `Extensão para Visual Studio Code <http://juan.blanco.ws/solidity-contracts-in-visual-studio-code/>`_
    Plugin de Solidity para o Microsoft Visual Studio Code que inclui marcação de síntese e o compilador Solidity.

* `Emacs Solidity <https://github.com/ethereum/emacs-solidity/>`_
    Plugin para o editor Emacs que provê marcação de síntese e aviso de erros de compilação.

* `Vim Solidity <https://github.com/tomlion/vim-solidity/>`_
    Plugin para o editor Vim que provê marcação de síntese.

* `Vim Syntastic <https://github.com/scrooloose/syntastic>`_
    Plugin para o editor Vim aviso de erros de compilação.

Descontinuado:

* `Mix IDE <https://github.com/ethereum/mix/>`_
    IDE baseada em para desenhar, debugar e testar smart contracts escritos em Solidity.

* `Ethereum Studio <https://live.ether.camp/>`_		
    IDE Web especializada que também provê acesso à linha de comando completa do ambiente Ethereum.

Ferramentas Solidity
--------------------

* `Dapp <https://dapp.readthedocs.io>`_
    Ferramenta de build, gerenciador de pacotes e assistente de publicação para Solidity.

* `Solidity REPL <https://github.com/raineorshine/solidity-repl>`_
    Experimente Solidity instantaneamente através da linha de comando Solidity.

* `solgraph <https://github.com/raineorshine/solgraph>`_
    Ferramenta para visualizar o fluxo de controle e mostrar potenciais falhas de segurança no seu contrato inteligente Solidity.

* `evmdis <https://github.com/Arachnid/evmdis>`_
    EVM Disassembler que realiza análise estática no código para garantir um maior nível de abstração em comparação com operações EVM puras.

* `Doxity <https://github.com/DigixGlobal/doxity>`_
    Gerador de Documentação para Solidity.

Interpretador e Dicionários de terceiros para Solidity
------------------------------------------------------

* `solidity-parser <https://github.com/ConsenSys/solidity-parser>`_
    Interpretador de Solidity para JavaScript

* `Solidity Grammar for ANTLR 4 <https://github.com/federicobond/solidity-antlr4>`_
    Dicionário de Solidity para o interpretador ANTLR 4.

Documentação da Linguagem
-------------------------

Nas próximas páginas vamos ver um :ref:`contrato inteligente simples <simple-smart-contract>` escrito
em Solidity seguido de conceitos básicos sobre :ref:`blockchains <blockchain-basics>`
e a :ref:`Máquina Virtual Ethereum <the-ethereum-virtual-machine>`.

A próxima sessão vai explicar várias *funcionalidades* do Solidity através de
:ref:`exemplos de contratos úteis <voting>`
Lembre-se que você pode testar os contratos
`no seu browser <https://remix.ethereum.org>`_!

A última e mais extensa seção vai cobrir todos os aspectos do Solidity profundamente.

Se você ainda tiver dúvidas você pode procurar ou perguntar no site do
`Ethereum Stackexchange <https://ethereum.stackexchange.com/>`_
ou acessar nosso `canal gitter <https://gitter.im/ethereum/solidity/>`_.
Ideias para melhorar o Solidity ou essa documentação são sempre bem vindas!

Veja também a `versão em Russo (русский перевод) <https://github.com/ethereum/wiki/wiki/%5BRussian%5D-%D0%A0%D1%83%D0%BA%D0%BE%D0%B2%D0%BE%D0%B4%D1%81%D1%82%D0%B2%D0%BE-%D0%BF%D0%BE-Solidity>`_.

Conteúdo
========

:ref:`Palavras Chave <genindex>`, :ref:`Página de Busca <search>`

.. toctree::
   :maxdepth: 2

   introduction-to-smart-contracts.rst
   installing-solidity.rst
   solidity-by-example.rst
   solidity-in-depth.rst
   security-considerations.rst
   using-the-compiler.rst
   abi-spec.rst
   style-guide.rst
   common-patterns.rst
   bugs.rst
   contributing.rst
   frequently-asked-questions.rst
