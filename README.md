# solidity-portuguese

## Translation of Solidity Documentation to Portuguese (Brasil)

## Tradução da Documentação do Solidity para Português (Brasil)

### Como compilar a documentação?

É necessário compilar a documentação antes de fazer um pull-request para garantir que tudo vai funcionar no site: [read the docs](http://solidity-portuguese.readthedocs.io).

#### Pré-requisitos:

1. Python (versão 3.x) 
   [Python Download](https://www.python.org/downloads/)

2. docutils
   ```bash
   pip install 'docutils'
   ```
  
3. sphinx
   ```bash
   pip install 'sphinx'
   ```

#### Comandos

Você pode utilizar o comando `make.bat` para ver uma lista de todas as opções disponíveis.

* Para checar rapidamente se tudo está funcionando utilize o parâmetro `html`
  ```bash
  ./docs/make.bat html
  ```
  O comando cria o diretório `./docs/_build` com a documentação no formato HTML.

Caso haja qualquer erro durante a compilação você será informado do arquivo e linha com problemas:
```bash
C:\Users\n\Projects\solidity-portuguese\docs\introduction-to-smart-contracts.rst:495: WARNING: Explicit markup ends without a blank line; unexpected unin
dent.
```

#### Atualização da documentação

     O site Read The Docs atualiza automaticamente a documentação assim que a versão deste repositório é alterada.