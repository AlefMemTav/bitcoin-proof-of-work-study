# O que é Proof of Work?

Proof of Work é um mecanismo no qual o participante precisa encontrar um
valor que faça o hash de um candidato ficar abaixo de um alvo. Como o hash
aparenta ser aleatório, a estratégia geral é testar candidatos até encontrar
um resultado válido.

Quanto menor o alvo, menor a probabilidade de sucesso em uma tentativa e
maior o trabalho esperado. A rede ajusta a dificuldade para conservar,
aproximadamente, o intervalo desejado entre blocos.

O nonce é um dos campos alteráveis do cabeçalho. Quando seu espaço se esgota,
o minerador pode alterar outros campos permitidos, como o `ExtraNonce` da
transação Coinbase, o que também altera a raiz de Merkle e cria novos
candidatos.

As páginas seguintes distinguem duas perguntas diferentes:

- é possível encontrar um candidato válido com menos tentativas?
- é possível avaliar cada tentativa com menor custo?

Essa distinção é essencial para interpretar corretamente otimizações e
resultados de criptoanálise.
