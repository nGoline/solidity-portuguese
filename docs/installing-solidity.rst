.. index:: ! installing

.. _installing-solidity:

###################
Instalando Solidity
###################

Versionamento
=============

As versões do Solidity seguem a `semantic versioning <https://semver.org>`_ e em adição aos
releases, **nightly development builds** são sempre disponibilizadas. As montagens noturnas
não tem garantia de funcionamento e apesar dos melhores esforços, elas podem conter mudanças
não documentadas ou quebradas. Nós recomendamos usar a última versão. Os pacotes de instação
abaixo irão usar a última versão. 

Remix
=====

Se você deseja usar o Solicity para small contracts, você
pode tentar usar o `Remix <https://remix.ethereum.org/>`_
que não necessita instalação. Se você deseja usar 
sem conexão com a Internet, você pode ir para
https://github.com/ethereum/browser-solidity/tree/gh-pages 
e baixar o arquivo zipado (.ZIP file), conforme explicado nesta página.

npm / Node.js
=============

Este é provavelmente o meio mais conveniente e portável de instalar o Solidity localmente.
Uma bibiloteca  de Java-Script independente da plataforma é provida através da compilação da 
fonte C++ dentro do JavaScript usando Emscripten. Pode ser usado nos projetos diretamente
(como o Remix). 
Por gentileza, consulte o diretório `solc-js <https://github.com/ethereum/solc-js>`_ 
para maiores instruções.

Ele também contém uma ferramenta de linha de comando chamada `solcjs`, que pode 
ser instalada via nmp:

.. code:: bash

    npm install -g solc

.. note::

    As opções da linha de comando `solcjs` não são compatíveis com `solc` e ferramentas (como o Geth)
    esperando que o comportamento de `solc` não irão funcionar com `solcjs`.


Docker
======

Nós fornecemos builders atualizadas para o date docker para o compilador. 
O repositório ``stable`` contém versões liberadas enquanto o repositório ``nightly``
contém mudanças potencialmente instáveis no segmento de desenvolvimento. 


.. code:: bash

    docker run ethereum/solc:stable solc --version


Atualmente, a imagem Docker contém somente o compilador executável,
então você terá algum trabalho adicional para linkar a fonte e os
diretórios de saída.


Binary Packages
===============

Os pacotes binários do olicity estão disponíveis em
`solidity/releases <https://github.com/ethereum/solidity/releases>`_.

Nós também temos PPAs para Ubuntu. Para a última versão estável.

.. code:: bash

    sudo add-apt-repository ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get install solc

Se você quiser a versão de desenvolvimento mais avançada:

.. code:: bash

    sudo add-apt-repository ppa:ethereum/ethereum
    sudo add-apt-repository ppa:ethereum/ethereum-dev
    sudo apt-get update
    sudo apt-get install solc
    

Nós também estamos liberando um `snap package <https://snapcraft.io/>`_,, que é instalável em todas as `supported Linux distros <https://snapcraft.io/docs/core/install>`_. Para instalar a última versão estável do solc:

.. code:: bash

    sudo snap install solc

Ou se voc~e quiser ajudar a testar o solc instável, com as versões mais recentes do ramo de desenvolvimento:

.. code:: bash

    sudo snap install solc --edge


Arch Linux também tem pacotes, embora limitados à mais recente versão de desenvolvimento:

.. code:: bash

    pacman -S solidity

Para Homebres, até o momento, estão faltando os pre-built bottles,
seguindo uma migração de Jenkis para TravisCI, mas para Homebrew
ainda deverá funcionar como um meio de construir direto da fonte.
Iremos re-adicionar brevemente os pre-built bottles.

.. code:: bash

    brew update
    brew upgrade
    brew tap ethereum/ethereum
    brew install solidity
    brew linkapps solidity

Se você quiser uma versão específica doSolicity, você pode instalar
a fórmula Homebrew diretamente do Github.

Veja em `solidity.rb commits on Github <https://github.com/ethereum/homebrew-ethereum/commits/master/solidity.rb>`_.

Siga os links do histórico até ter um link de arquivo bruto de um
compromisso específico de `` solidity.rb``.

Instale-o usando ``brew``:

.. code:: bash

    brew unlink solidity
    # Install 0.4.8
    brew install https://raw.githubusercontent.com/ethereum/homebrew-ethereum/77cce03da9f289e5a3ffe579840d3c5dc0a62717/solidity.rb


Gentoo Linux também fornecer um pacote solidity que pode ser instalado usando ``emerge``:

.. code:: bash

    emerge dev-lang/solidity

.. _building-from-source:

Construindo à partir do Fonte
=============================

Clone o repositório
-------------------

Para clonar o código fonte, execute o seguinte comando:

.. code:: bash

    git clone --recursive https://github.com/ethereum/solidity.git
    cd solidity

Se você deseja ajudar à desenvolver o Solidity, 
você deve derivar o Solidity e adicionar sua derivação pessoal como um segundo remote:


.. code:: bash

    cd solidity
    git remote add personal git@github.com:[username]/solidity.git

O Solicity tem sub-módulos git. Assegure-se de que eles foram carregados adequadamente:

.. code:: bash

   git submodule update --init --recursive

Pré-requisitos - MacOS
----------------------

Para o macOS, assegure-se de que você tenha a última versão do
`Xcode installed <https://developer.apple.com/xcode/download/>`_.

Isto contém o `Clang C++ compiler <https://en.wikipedia.org/wiki/Clang>`_, the
`Xcode IDE <https://en.wikipedia.org/wiki/Xcode>`_ e outras ferramentas Apple de 
desenvolvimento que são requeridas para construir aplicações C++ no OS X.
Se você estiver instalado Xcode pela primeira vez, or simplemente instalado uma 
nova versão, você terá que concordar com a licença de uso antes de poder fazer
compilações de linhas de comando:

.. code:: bash

    sudo xcodebuild -license accept

Nossas compilações do OS X exigem que você `install the Homebrew <http://brew.sh>`_,
gerenciador de pacotes para instalar dependências externas.
Eis como desinstalar o Homebrew, caso você queira iniciar novamente do início
`uninstall Homebrew
<https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/FAQ.md#how-do-i-uninstall-homebrew>`_,

Pré-Requisitos - Windows
------------------------

Você irá necessitar instalar as seguintes dependências para montar a versão do Solicity no Windows:


+------------------------------+-------------------------------------------------------+
| Software                     | Notas                                                 |
+==============================+=======================================================+
| `Git for Windows`_           | Ferramenta para linha de comando para recuperação dos |
|                              | fontes a partir do GitHub.                            |
+------------------------------+-------------------------------------------------------+
| `CMake`_                     | Gerador de Arquivos de compilãção entre plataformas   |
+------------------------------+-------------------------------------------------------+
| `Visual Studio 2015`_        | Compilador C++ e ambiente de desenolvimento           |
+------------------------------+-------------------------------------------------------+


.. _Git for Windows: https://git-scm.com/download/win
.. _CMake: https://cmake.org/download/
.. _Visual Studio 2015: https://www.visualstudio.com/products/vs-2015-product-editions


External Dependencies
Dependências Externas
---------------------

Nós agora temos um script "botão único" que instala todas as dependências externas requeridas
em macOS, Windows e numerosas Distros Linux. Isto é usado para ser um processo manual
multi-passos, mas agora em uma linha.


.. code:: bash

    ./scripts/install_deps.sh

Ou, no Windows:

.. code:: bat

    scripts\install_deps.bat


Command-Line Build
------------------

O projeto Solidity usa CMake para configurar a compilação.
Building Solidity é bastante semelhante ao Linux, MacOS e outros Unices:

.. code:: bash

    mkdir build
    cd build
    cmake .. && make

ou ainda mais fácil:

.. code:: bash
    
    #note: this will install binaries solc and soltest at usr/local/bin
    ./scripts/build.sh

Ou ainda para o Windows:

.. code:: bash

    mkdir build
    cd build
    cmake -G "Visual Studio 14 2015 Win64" ..


Este último conjunto de instruções deve resultar na criação de
**solidity.sln** nesse diretório de compilação. Clicando duas vezes nesse arquivo
deve resultar na ativação do Visual Studio. Sugerimos construir
uma configuração **RelWithDebugInfo**, mas todos os outros irão funcionar.

Alternativamente, você pode construir para o Windows na linha de comando, dessa maneira:

.. code:: bash

    cmake --build . --config RelWithDebInfo

Opções CMake
============

Se você está interessado quais opções CMake estão disponíveis, execute ``cmake .. -LH``.

A sequência de versão em detalhes
=================================

A versão de string do Solidity contém quatro partes:

- O número de versão
- Tag de pré-lançamento, normalmente marcado para ``develop.YYYY.MM.DD`` ou ``nightly.YYYY.MM.DD`` 
- Commit no formato ``commit.GITHASH``
- A plataforma tem um número arbitrário de itns, contendo detalhes sobre a plataforma e o compilador. 

Se existirem modificações locais, o commit será pós-fixado com ``.mod``.

Essas partes são combinadas conforme exigido pela Semver, onde a marca de pré-lançamento da Solidity é igual ao pré-lançamento do Semver
e o Commit do Soliddity e a plataforma combinadas compõem os metadados de construção do Semver.

Um exemplo de release:``0.4.8+commit.60cc1668.Emscripten.clang``.

Um exeplo de pre-release: ``0.4.9-nightly.2017.1.17+commit.6ecb4aa3.Emscripten.clang``

Informação importante sobre versionamento
=========================================

Depois que um lançamento é feito, o nível da versão do patch é superado, porque assumimos que apenas
as mudanças de nível de patch seguem. Quando as alterações são mescladas, a versão deve ser superada de acordo com
severidades da mudança. Finalmente, uma versão sempre é feita com a versão
da construção noturna atual, mas sem o especificador `` prerelease``.


Exemplo:

0. the 0.4.0 release is made
1. nightly build has a version of 0.4.1 from now on
2. non-breaking changes are introduced - no change in version
3. a breaking change is introduced - version is bumped to 0.5.0
4. the 0.5.0 release is made

Este comportamento funciona bem com a :ref:`version pragma <version_pragma>`. 

