.. index:: Bugs

.. _known_bugs:

########################
Lista de Bugs Conhecidos
########################

Abaixo, você consegue encontrar em um formato JSON uma lista dos bugs relevantes para segurança, encontrados 
no compilador do Solidity. O arquivo em si, é hospedado em um `repositório no GitHub
<https://github.com/ethereum/solidity/blob/develop/docs/bugs.json>`_.
A lista se estende até a versão 0.3.0, bugs conhecidos que estão presentes somente nas
versões precedentes a esta não estão listados.

Existe outro arquivo chamado `bugs_by_version.json
<https://github.com/ethereum/solidity/blob/develop/docs/bugs_by_version.json>`_,
que pode ser utilizado para checar quais bugs afetam uma versão específica do compilador.

Ferramentas de verificação de fontes de contratos e outras ferramentas que interajam com contratos
devem consultar esta lista de acordo com os seguintes critérios:

 - É meio suspeito que um contrato foi compilado com uma versão nightly do compilador ao invés
   de uma versão de release. Esta lista não cobre versões unreleased ou nightly.
 - Também é meio suspeito que um contrato foi compilado com uma versão que não seja a mais recente no 
   momento da criação. Para contratos criados a partir de outros contratos, 
   você deve seguir a corrente de criação até a transação e usar a data desta transação como
   data de criação.
 - É altamente suspeito que um contrato foi compilado com um compilador que 
   contém um bug conhecido e o contrato foi criado quando uma nova versão de
   compilador contendo a correção foi lançada.

O arquivo JSON de bugs conhecidos abaixo, é um array de objetos, um para cada bug, 
com as seguintes chaves:

name
    Nome único dado a um bug
summary
    Descrição curta do bug
description
    Descrição detalhada do bug
link
    URL de um website com informações mais detalhadas, opcional
introduced
    A primeira versão publicada do compilador que possuía o bug, optional
fixed
    A primeira versão publicada do compilador que já não possuía mais o bug
publish
    A data em que o bug se tornou conhecido publicamente, opcional
severity
    Gravidade do bug, low (baixa), medium (média), high (alta). Leva em consideração
    a facilidade de ser encontrado em testes de contrato, probabilidade de ocorrência e
    danos potenciais por exploits.
conditions
    Condições que precisam ser atendidas para disparar o bug. Atualmente, esse objeto pode conter um valor boolean ``optimizer``, 
    que significa que o optimizer (otimizador) precisa ser ligado para ativar o bug.
    Se nenhuma condição for dada, suponha que o erro está presente.

.. literalinclude:: bugs.json
   :language: js
