###################
Solidity by Example
###################

.. index:: voting, ballot

.. _voting:

*******
Votando
*******

O contrato seguinte é bastante complexo, mas dmonstra
muitas das características do Solidity. Ele implementa 
um contrato de votação. Naturalmente, o problema principal 
do voto eletrônico é como atribuir direitos de voto a pessoa
e como prevenir manipulações. Não serão resolvidos todos os
problemas aqui, mas ao menos mostraremos como delegar 
direito de voto, pode ser feito de tal forma a que a contagem 
seja **automática e completamente transparente** ao mesmo 
tempo.

A idéia é criar um contrato por cédula,
fornecendo um nome curto para cada opção.
Em seguida, o criador do contrato que serve como
presidente dará o direito de votar a cada
endereço individualmente.

As pessoas por trás dos endereços podem então
escolher entre votar elas mesmas ou delegar o 
voto a outra pessoa de confiança.

No final do tempo de voto, ``winningProposal()``
retornará a proposta com maior número de votos. 

::

    pragma solidity ^0.4.11;

	/// @title Votação com delegação.
    contract Ballot {
        // Aqui é declarado um novo tipo complexo que será 
		// usado pelas variáveis mais tarde.
		// Represntará um votante único.
		
        struct Voter {
            uint weight; // peso é acumulado por delegação // weight is accumulated by delegation
            bool voted;  // se for verdadeiro, aquela pessoa já votou // if true, that person already voted
            address delegate; // pessoa a quem será delegado // person delegated to
            uint vote;   // índice do voto proposto // index of the voted proposal
        }

        // Este é um tipo de proposta única
		
		struct Proposal {
            bytes32 name;   // nome curto (até 32 bytes) // short name (up to 32 bytes)
            uint voteCount; // número de votos acumulados // number of accumulated votes
        }

		address public chairperson;
		
        // Aqui é declarada a variável de estado que  
        // armazena uma estrutura de "Votante" para cada possível endereço.
		
        mapping(address => Voter) public voters;
		
		// Uma estrutura de "Proposta" tipo array dinamicamente dimensionada.
		
        Proposal[] public proposals;

        /// Criar uma nova cédula para escolher uma das "proposalNames".
		
		function Ballot(bytes32[] proposalNames) {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;

            // Para cada nome de proposta, criar um novo
            // objeto proposta e adicione este objeto ao
            // fim do array.

            for (uint i = 0; i < proposalNames.length; i++) {
                
                // "Proposal({...})" cria um objeto temporário
                // e "proposal.push(...)" adiciona este objeto
                // ao fim das "propostas"

                proposals.push(Proposal({
                    name: proposalNames[i],
                    voteCount: 0
                }));
            }
        }

        // Dar ao "votante"  o direito de voto nesta cédula.
        // Pode somente ser chamada pelo "Presidente".

        function giveRightToVote(address voter) {
            
            // Se o argumento de "require" for determinado como "false",
            // é terminado e todas alterações são revertidas assim
            // como o saldo de Ether retorna ao valor antes da operação.
            // É normalmente uma boa ideia usar isto se funções são 
            // chamadas incorretamente. Mas fique atento, isso pode 
            // também consumir todo o "gas" disponível.
            // (está planejado para ser mudado no futuro).

            require((msg.sender == chairperson) && !voters[voter].voted && (voters[voter].weight == 0));
            voters[voter].weight = 1;
        }
		
		/// Delegar seu voto ao votante "to".
		
        function delegate(address to) {
			// atribuir referência 
			
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);

			// Auto-delegação não é permitida.
			
            require(to != msg.sender);

			// Encaminhar a atribuição desde que "to" também seja atribuido.
			
            // Em geral, estes tipos de loops são muito perigosos,
			// porque se demorarem muito tempo executando, podem
			// causar necessidade de mais "gas" do que é disponível 
			// para o bloco.
			// Neste caso, a atribuição não será executada, mas,
			// em outras situações, estes loops podem causar o 
			// "travamento" completo do contrato.
			
            while (voters[to].delegate != address(0)) {
                to = voters[to].delegate;

                // Se encontramos um loop na atribuição, não permitido.
				
				// We found a loop in the delegation, not allowed. 
                require(to != msg.sender);
            }

            // Desde que "sender" é uma referência, este
			// modifica "voters[msg.sender].voted"
			
			sender.voted = true;
            sender.delegate = to;
            Voter storage delegate = voters[to];
            if (delegate.voted) {
                // Se o atribuido já votou,
				// some diretamente no número de votos.
				
			    proposals[delegate.vote].voteCount += sender.weight;
            } else {
                // Se o atribuido ainda não votou, 
				// some ao seu peso.
				
			    delegate.weight += sender.weight;
            }
        }

        /// Dê o seu voto (incluindo votos atribuidos a você)
		/// a proposta "proposals[proposal].name".
		
		function vote(uint proposal) {
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);
            sender.voted = true;
            sender.vote = proposal;
			
			// Se "proposal" está fora do range do array,
			// será rejeitada automaticamente e todas as 
			// alterações revertidas.
			
            proposals[proposal].voteCount += sender.weight;
        }

        /// @dev calcula a proposta vencedora levando todos
		/// os votos prévios em consideração.
		
		function winningProposal() constant
                returns (uint winningProposal)
        {
            uint winningVoteCount = 0;
            for (uint p = 0; p < proposals.length; p++) {
                if (proposals[p].voteCount > winningVoteCount) {
                    winningVoteCount = proposals[p].voteCount;
                    winningProposal = p;
                }
            }
        }

        /// Chama a função "winningProposal()" para selecionar
		/// o indíce do vencedor contido no array de propostas e
		/// então retorna o nome do vencedor.
		
		function winnerName() constant
                returns (bytes32 winnerName)
        {
            winnerName = proposals[winningProposal()].name;
        }
    }

Possíveis Melhorias
===================

Atualmente, são necessárias muitas transações para atribuir os direitos
de votar para todos os participantes. Você pode pensar em uma maneira melhor?

.. index:: auction;blind, auction;open, blind auction, open auction

*************
Leilão cego
*************

Nesta seção, mostraremos quão fácil é criar um
contrato de leilão completamente cego no Ethereum.
Começaremos com um leilão aberto onde todos
podem ver os lances efetuados e, em seguida, estender 
esse contrato em um leilão cego onde não é
possível ver o lance real até que o período de lances
tenha finalizado.

.. _simple_auction:

Leilão aberto Simples
=====================

A idéia geral do contrato de leilão simples a seguir
é que todos podem enviar suas propostas durante
o período de licitação. Os lances já incluem envio de
dinheiro / ethers, a fim de vincular os licitantes ao
seu lance. Se o lance mais alto for superado, o 
melhor lance anterior recebe seu dinheiro de volta.
Após o final do período de licitação, o contrato deve
ser chamado manualmente para o beneficiário receber
seu dinheiro - contratos não podem ativar-se.

::

    pragma solidity ^0.4.11;

    contract SimpleAuction {
        // Parâmetros do leilão. Tempos são dados em "Unix timestamps" absolutos (segundos desde 01-01-1970)
        // ou períodos de tempo em segundos.

        address public beneficiary;
        uint public auctionStart;
        uint public biddingTime;

        // Estado do leilão corrente.

        address public highestBidder;
        uint public highestBid;

        // Permitidas retiradas de propostas prévias
        mapping(address => uint) pendingReturns;

        // Colocar em "true" no final, desabilitando qualquer mudança
        bool ended;

        // Eventos que serão ativados com as mudanças.
        
        event HighestBidIncreased(address bidder, uint amount);
        event AuctionEnded(address winner, uint amount);

        // A seguir vem o assim chamado "natspec comment"
        // reconhecível por três barras.
        // Será mostrado quando o usuário é requisitado para
        // confirmar a transação.
        
        /// Criar um leilão simples com "_biddingTime" segundos, 
        /// período de proposta em nome do endereço de beneficiário
        /// "_beneficiary".
        function SimpleAuction(
            uint _biddingTime,
            address _beneficiary
        ) {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingTime = _biddingTime;
        }

        /// Proposta no leilão com o valor enviado
        /// junto com esta transação.
        /// O valor somente será devolvido se a pro-
        /// posta não for vencedora.

        function bid() payable {
            // Não necessita de argumentos, toda 
            // informação faz já parte da transação.
            // A "keyword payable" é requerida para 
            // a função estar habilitada a receber Ethers.

            // Reverte a chamada se o período de proposta
            // for encerrado.
            require(now <= (auctionStart + biddingTime));

            // Se a proposta não for a mais alta, enviar o
            // dinheiro de volta

            require(msg.value > highestBid);

            if (highestBidder != 0) {
                // Restituir o dinheiro simplesmente udando
                // " highestBidder.send(highestBid)" é um risco de
                // segurança porque poderia ter executado um contrato
                // não confiável.
                // É sempre mais seguro deixar os destinatários das restiuições
                // resgatar seus valores por eles mesmos.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBidder = msg.sender;
            highestBid = msg.value;
            HighestBidIncreased(msg.sender, msg.value);
        }

        /// Restituir uma proposta que foi superada.
        function withdraw() returns (bool) {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // É importante colocar esta variável em zero para que o destinatário
                // possa chamar esta função novamente como parte da chamada recebida 
                // antes de "send" retornar.
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)) {
                    // Não necessário chamar aqui, somente dar um reset no valor devido.
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// Fim do leilão e envio da proposta mais alta 
        /// para o beneficiário.
        function auctionEnd() {
            // É uma boa diretriz estruturar as função que interagem 
            // com outros contratos (isto é, elas chamam funções para enviar Ethers)
            // em três fases:
            // 1. verificar condições;
            // 2. realizar ações (condições potenciais de mudança);
            // 3. interagir com outros contratos.
            // Se essas fases forem misturadas, o outro contrato pode
            // chamar de volta dentro do corrente contrato e modificar o estado ou 
            // efeito de causa (pagamento de ethers) a ser realizado multiplas vezes.
            // Se as funções chamadas internamente incluiem interação com contratos 
            // externos, eles também tem que considerar interações com estes.
            // 1. Condições
            
            require(now >= (auctionStart + biddingTime)); // leilão não encerrado ainda
            require(!ended); // função já foi chamada

            // 2. Efeitos
            ended = true;
            AuctionEnded(highestBidder, highestBid);

            // 3. Interação
            beneficiary.transfer(highestBid);
        }
    }


Leilão Cego
===========
	
O leilão aberto anterior é estendido para um leilão cego
na sequência. A vantagem de um leilão cego é
que não há pressão no tempo para o final do período 
de licitação. Criando um leilão cego em uma
plataforma de computação transparente pode soar como
uma contradição, mas a criptografia vem ao resgate.

Durante o ** período de licitação**, um licitante não
envia seu lance de verdade, mas apenas uma versão hash
do mesmo. Como atualmente é considerado praticamente impossível
para encontrar dois valores (suficientemente longos) cujos
hash sejão iguais, o licitante é associado ao lance por esse 
meio. Após o final do período de licitação, os concorrentes têm
que revelar suas propostas: eles enviam seus valores
não criptografados e o contrato verifica se o valor de hash
é o mesmo que o fornecido durante o período de licitação.

Outro desafio é como fazer o leilão
** vinculativo e cego ** ao mesmo tempo: a única maneira de
impedir que o licitante apenas não envie o dinheiro
depois que ele ganhou o leilão é fazer com que ele o envie
juntamente com a oferta. Como as transferências de valor não podem
ser cegas na rede Ethereum, qualquer um pode ver o valor.

O contrato a seguir resolve esse problema aceitando 
qualquer valor maior do que o mais alto lance. 
Uma vez que, claro, isso só pode ser verificado durante
a fase de revelação, algumas propostas podem ser ** inválidas ** 
e isso pode ser proposital (até mesmo fornece um flag explícito
para colocar lances inválidos com transferências de alto valor):
Participantes podem confundir a concorrência colocando vários
lances altos ou baixos inválidos.

::

    pragma solidity ^0.4.11;

    contract BlindAuction {
        struct Bid {
            bytes32 blindedBid;
            uint deposit;
        }

        address public beneficiary;
        uint public auctionStart;
        uint public biddingEnd;
        uint public revealEnd;
        bool public ended;

        mapping(address => Bid[]) public bids;

        address public highestBidder;
        uint public highestBid;

        // Retiradas permitidas por negociações prévias
		
        mapping(address => uint) pendingReturns;

        event AuctionEnded(address winner, uint highestBid);

        /// Modificadores são um meio conveniente para validas
		/// entradas nas funções. "onlyBefore" é aplicado ao negócio
		/// abaixo:
		/// O novo corpo da função é o modificador do corpo onde "_"
		/// é substituido pelo corpo antigo da função.
		
        modifier onlyBefore(uint _time) { require(now < _time); _; }
        modifier onlyAfter(uint _time) { require(now > _time); _; }

        function BlindAuction(
            uint _biddingTime,
            uint _revealTime,
            address _beneficiary
        ) {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingEnd = now + _biddingTime;
            revealEnd = biddingEnd + _revealTime;
        }

		/// Colocar uma negociação "cega" com `_blindedBid` = keccak256(value,
        /// fake, secret).
		/// Os ethers remetidos somente são devolvido se a negociação 
		/// for corretamente revelada na fase de revelação de propostas. A negociação
		/// é valida se o valor enviado junto com a negociação é ao menos "value" e
		/// fake não é verdadeiro.
		/// Configurar fake para "true" (verdadeiro) e enviar nao exatamente o valor são
		/// maneiras de esconder a oferta real mas ainda fazer o depósito requerido. O mesmo
		/// endereço pode colocar multiplas ofertas.
		
        function bid(bytes32 _blindedBid)
            payable
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

		/// Revelar as ofertas "cegas". Você ira restituir para todos
		/// ofertas inválidas corretamente cegas e para todas as ofertas
		/// exceto para a mais alta de todas.	
		
        function reveal(
            uint[] _values,
            bool[] _fake,
            bytes32[] _secret
        )
            onlyAfter(biddingEnd)
            onlyBefore(revealEnd)
        {
            uint length = bids[msg.sender].length;
            require(_values.length == length);
            require(_fake.length == length);
            require(_secret.length == length);

            uint refund;
            for (uint i = 0; i < length; i++) {
                var bid = bids[msg.sender][i];
                var (value, fake, secret) =
                        (_values[i], _fake[i], _secret[i]);
                if (bid.blindedBid != keccak256(value, fake, secret)) {
                    // Oferta não foi realmente revelada.
					// Não faça o depósito de restituição.
					
                    continue;
                }
                refund += bid.deposit;
                if (!fake && bid.deposit >= value) {
                    if (placeBid(msg.sender, value))
                        refund -= value;
                }
                // Torne impossível para o remetente reinvindicar
				// o mesmo depósito.
				
                bid.blindedBid = bytes32(0);
            }
            msg.sender.transfer(refund);
        }
		// Esta é uma função "interna" que significa que só 
		// pode ser chamada pelo próprio contrato (ou por contra-
		// tos derivados)
        
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid) {
                return false;
            }
            if (highestBidder != 0) {
                // Resituir o maior ofertante.
				
				pendingReturns[highestBidder] += highestBid;
            }
            highestBid = value;
            highestBidder = bidder;
            return true;
        }
		
		/// Retirar uma oferta que foi superada.
		
        function withdraw() {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
				// É importante colocar em zero para que o receptor 
				// possa chamar essa função novamente como parte do recebimento
				// da chamada antes de "send" retornar (veja o comentário acima sobre condições --> 
				// efeitos --> interações). 
					
			
                pendingReturns[msg.sender] = 0;

                msg.sender.transfer(amount);
            }
        }

        /// Fim do leilão e envio da maior proposta 
		/// para o beneficiário.
		
        function auctionEnd()
            onlyAfter(revealEnd)
        {
            require(!ended);
            AuctionEnded(highestBidder, highestBid);
            ended = true;
            
			// Enviaremos todo o dinheiro existente, porque
			// algumas das restiuições podem ter falhado.
			
            beneficiary.transfer(this.balance);
        }
    }

.. index:: purchase, remote purchase, escrow

********************
Compra Remota Segura
********************

::

    pragma solidity ^0.4.11;

    contract Purchase {
        uint public value;
        address public seller;
        address public buyer;
        enum State { Created, Locked, Inactive }
        State public state;

        // Garantir que 'msg.value' é um número par.
        // Divisão será truncada se for um número impar.
        // Verificar via multiplicação que não é um número impar.

        function Purchase() payable {
            seller = msg.sender;
            value = msg.value / 2;
            require((2 * value) == msg.value);
        }

        modifier condition(bool _condition) {
            require(_condition);
            _;
        }

        modifier onlyBuyer() {
            require(msg.sender == buyer);
            _;
        }

        modifier onlySeller() {
            require(msg.sender == seller);
            _;
        }

        modifier inState(State _state) {
            require(state == _state);
            _;
        }

        event Aborted();
        event PurchaseConfirmed();
        event ItemReceived();

        /// Abortar a compra e reinvindicar os ether.
        /// Pode somente ser chamado pelo vendedor antes
        /// do contrato ser travado.

        function abort()
            onlySeller
            inState(State.Created)
        {
            Aborted();
            state = State.Inactive;
            seller.transfer(this.balance);
        }

        /// Confirme a compra como comprador.
        /// Transação tem que incluir '2 * valor' ether.
        /// Os ether ficarão presos até a função confirmReceived
        /// for chamada.

        function confirmPurchase()
            inState(State.Created)
            condition(msg.value == (2 * value))
            payable
        {
            PurchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }

        /// Confirmar que você (o comprador) recebeu o item.
        /// Isto irá liberar os ether presos.

        function confirmReceived()
            onlyBuyer
            inState(State.Locked)
        {
            ItemReceived();
            // É importante mudar o estado primeiro porque
            // de outra forma, o contrato chamado usando 'send'
            // abaixo pode chamar novamente aqui.

            state = State.Inactive;

            // NOTA: Isto efetivamente permite o comprador e o vendedor 
            // bloquear a restituição - a retirada padrão deve ser usada.

            buyer.transfer(value);
            seller.transfer(this.balance);
        }çp~´
    }

********************
Micropayment Channel
********************

To be written.
