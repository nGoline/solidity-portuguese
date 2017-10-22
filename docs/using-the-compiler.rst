*******************
Usando o compilador
*******************

.. index:: ! commandline compiler, compiler;commandline, ! solc, ! linker

.. _commandline-compiler:

Usando o Comando em linha do Compilador
***************************************

Um dos objetivos de construção do repositório Solidity é ``solc``, o compilador de linha de comando do Solidity.
Usar ``solc --help`` fornecerá uma explicação de todas as opções. O compilador pode produzir várias saídas, que vão 
desde binários simples e montagem em uma árvore de sintaxe abstrata (árvore de análise) até estimativas do uso de gás.

Se você quiser compilar apenas um único arquivo, você pode executá-lo como ``solc --bin sourceFile.sol`` e ele irá imprimir o binário. 
Antes de implantar seu contrato, ative o otimizador ao compilar usando ``solc --optimize --bin sourceFile.sol``. Se você quiser obter algumas variantes mais avançadas de saída do ``solc``, provavelmente é melhor contar para que ele saia tudo para separar arquivos usando ``solc -o outputDirectory --bin --ast --asm sourceFile.sol``.

O compilador de linha de comando lê automaticamente arquivos importados do sistema de arquivos, mas
também é possível fornecer redirecionamentos de caminho usando ``prefix=path`` da seguinte maneira:

::

    solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ =/usr/local/lib/fallback file.sol

Isso essencialmente instrui o compilador a procurar qualquer coisa começando com
``github.com/ethereum/dapp-bin/`` em ``/usr/local/lib/dapp-bin`` e se não
encontrar o arquivo lá, ele verá ``/usr/local/lib/fallback`` (o prefixo vazio
sempre irá corresponder). ``solc`` não lerá arquivos do sistema de arquivos que ficam fora de
os destinos de remapeamento e fora dos diretórios onde os arquivos fontes explicitamente residem, 
então coisas como  ``import "/etc/passwd";`` só funcionam se você adicionar ``=/`` como um remapeamento.

Se houver várias correspondências devido a remapeamentos, é selecionado aquele com o prefixo comum mais longo.

Por motivos de segurança, o compilador tem restrições sobre quais diretórios ele pode acessar. Os caminhos (e seus subdiretórios) dos arquivos de origem especificados na linha de comando e os caminhos definidos pelos remakes são permitidos para instruções de importação, mas tudo o resto é rejeitado. Caminhos adicionais (e seus subdiretórios) podem ser permitidos através do parâmetro ``--allow-paths /sample/path,/another/sample/path``.

Se seus contratos usam :ref:`libraries <libraries>`, você notará que o bytecode contém substrings do formulário ``__LibraryName______``. Você pode usar ``solc`` como um vinculador, o que significa que ele irá inserir os endereços da biblioteca para você nesses pontos:

Ou adicione ``--libraries "Math:0x12345678901234567890 Heap:0xabcdef0123456"`` para o seu comando fornecer um endereço para cada biblioteca ou armazenar a string em um arquivo (uma biblioteca por linha) e executar ``solc`` usando ``--libraries fileName``.

Se ``solc`` é chamado com a opção ``--link``, todos os arquivos de entrada são interpretados como binários não vinculados (codificados em hexadecimal) no ``__LibraryName____``-formato acima e estão ligados no local (se a entrada é lida a partir de stdin, está escrito para stdout). Todas as opções, exceto ``--libraries``, são ignoradas (incluindo ``-o``) neste caso.

Se ``solc`` é chamado com a opção ``--standard-json``, espera uma entrada JSON (conforme explicado abaixo) na entrada padrão e retorna uma saída JSON na saída padrão.

.. _compiler-api:


Entrada e saída do compilador Descrição JSON
********************************************

Esses formatos JSON são usados pela API do compilador, bem como estão disponíveis através de ``solc``. Estes estão sujeitos a alterações,
alguns campos são opcionais (como observado), mas é destinado a apenas fazer alterações compatíveis com versões anteriores.

A API do compilador espera uma entrada formatada JSON e produz o resultado da compilação em uma saída formatada JSON.

Os comentários não são, naturalmente, permitidos e são utilizados aqui apenas para fins explicativos.


Descrição da Entrada
--------------------

.. code-block:: none

    {
      // Required: Source code language, such as "Solidity", "serpent", "lll", "assembly", etc.
      language: "Solidity",
      // Required
      sources:
      {
        // The keys here are the "global" names of the source files,
        // imports can use other files via remappings (see below).
        "myFile.sol":
        {
          // Optional: keccak256 hash of the source file
          // It is used to verify the retrieved content if imported via URLs.
          "keccak256": "0x123...",
          // Required (unless "content" is used, see below): URL(s) to the source file.
          // URL(s) should be imported in this order and the result checked against the
          // keccak256 hash (if available). If the hash doesn't match or none of the
          // URL(s) result in success, an error should be raised.
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // Optional: keccak256 hash of the source file
          "keccak256": "0x234...",
          // Required (unless "urls" is used): literal contents of the source file
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // Optional
      settings:
      {
        // Optional: Sorted list of remappings
        remappings: [ ":g/dir" ],
        // Optional: Optimizer settings (enabled defaults to false)
        optimizer: {
          enabled: true,
          runs: 500
        },
        // Metadata settings (optional)
        metadata: {
          // Use only literal content and not URLs (false by default)
          useLiteralContent: true
        },
        // Addresses of the libraries. If not all libraries are given here, it can result in unlinked objects whose output data is different.
        libraries: {
          // The top level key is the the name of the source file where the library is used.
          // If remappings are used, this source file should match the global path after remappings were applied.
          // If this key is an empty string, that refers to a global level.
          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        }
        // The following can be used to select desired outputs.
        // If this field is omitted, then the compiler loads and does type checking, but will not generate any outputs apart from errors.
        // The first level key is the file name and the second is the contract name, where empty contract name refers to the file itself,
        // while the star refers to all of the contracts.
        //
        // The available output types are as follows:
        //   abi - ABI
        //   ast - AST of all source files
        //   legacyAST - legacy AST of all source files
        //   devdoc - Developer documentation (natspec)
        //   userdoc - User documentation (natspec)
        //   metadata - Metadata
        //   ir - New assembly format before desugaring
        //   evm.assembly - New assembly format after desugaring
        //   evm.legacyAssembly - Old-style assembly format in JSON
        //   evm.bytecode.object - Bytecode object
        //   evm.bytecode.opcodes - Opcodes list
        //   evm.bytecode.sourceMap - Source mapping (useful for debugging)
        //   evm.bytecode.linkReferences - Link references (if unlinked object)
        //   evm.deployedBytecode* - Deployed bytecode (has the same options as evm.bytecode)
        //   evm.methodIdentifiers - The list of function hashes
        //   evm.gasEstimates - Function gas estimates
        //   ewasm.wast - eWASM S-expressions format (not supported atm)
        //   ewasm.wasm - eWASM binary format (not supported atm)
        //
        // Note that using a using `evm`, `evm.bytecode`, `ewasm`, etc. will select every
        // target part of that output.
        //
        outputSelection: {
          // Enable the metadata and bytecode outputs of every single contract.
          "*": {
            "*": [ "metadata", "evm.bytecode" ]
          },
          // Enable the abi and opcodes output of MyContract defined in file def.
          "def": {
            "MyContract": [ "abi", "evm.opcodes" ]
          },
          // Enable the source map output of every single contract.
          "*": {
            "*": [ "evm.sourceMap" ]
          },
          // Enable the legacy AST output of every single file.
          "*": {
            "": [ "legacyAST" ]
          }
        }
      }
    }


Descrição da Saída
------------------

.. code-block:: none

    {
      // Optional: not present if no errors/warnings were encountered
      errors: [
        {
          // Optional: Location within the source file.
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          ],
          // Mandatory: Error type, such as "TypeError", "InternalCompilerError", "Exception", etc
          type: "TypeError",
          // Mandatory: Component where the error originated, such as "general", "ewasm", etc.
          component: "general",
          // Mandatory ("error" or "warning")
          severity: "error",
          // Mandatory
          message: "Invalid keyword"
          // Optional: the message formatted with source location
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
      ],
      // This contains the file-level outputs. In can be limited/filtered by the outputSelection settings.
      sources: {
        "sourceFile.sol": {
          // Identifier (used in source maps)
          id: 1,
          // The AST object
          ast: {},
          // The legacy AST object
          legacyAST: {}
        }
      },
      // This contains the contract-level outputs. It can be limited/filtered by the outputSelection settings.
      contracts: {
        "sourceFile.sol": {
          // If the language used has no contract names, this field should equal to an empty string.
          "ContractName": {
            // The Ethereum Contract ABI. If empty, it is represented as an empty array.
            // See https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
            abi: [],
            // See the Metadata Output documentation (serialised JSON string)
            metadata: "{...}",
            // User documentation (natspec)
            userdoc: {},
            // Developer documentation (natspec)
            devdoc: {},
            // Intermediate representation (string)
            ir: "",
            // EVM-related outputs
            evm: {
              // Assembly (string)
              assembly: "",
              // Old-style assembly (object)
              legacyAssembly: {},
              // Bytecode and related details.
              bytecode: {
                // The bytecode as a hex string.
                object: "00fe",
                // Opcodes list (string)
                opcodes: "",
                // The source mapping as a string. See the source mapping definition.
                sourceMap: "",
                // If given, this is an unlinked object.
                linkReferences: {
                  "libraryFile.sol": {
                    // Byte offsets into the bytecode. Linking replaces the 20 bytes located there.
                    "Library1": [
                      { start: 0, length: 20 },
                      { start: 200, length: 20 }
                    ]
                  }
                }
              },
              // The same layout as above.
              deployedBytecode: { },
              // The list of function hashes
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // Function gas estimates
              gasEstimates: {
                creation: {
                  codeDepositCost: "420000",
                  executionCost: "infinite",
                  totalCost: "infinite"
                },
                external: {
                  "delegate(address)": "25000"
                },
                internal: {
                  "heavyLifting()": "infinite"
                }
              }
            },
            // eWASM related outputs
            ewasm: {
              // S-expressions format
              wast: "",
              // Binary format (hex string)
              wasm: ""
            }
          }
        }
      }
    }
