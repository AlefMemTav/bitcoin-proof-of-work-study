# Hashes e SHA-256

Uma função hash transforma uma entrada de tamanho variável em uma saída de
tamanho fixo. Para ser útil em sistemas criptográficos, ela deve produzir
saídas difíceis de prever, tornar difícil encontrar entradas com uma saída
específica e reagir de forma sensível a pequenas alterações na entrada.

O SHA-256 produz 256 bits. O Bitcoin aplica o SHA-256 duas vezes ao cabeçalho
do bloco, operação normalmente escrita como `SHA-256^2`.

A função hash não cifra dados e não deve ser tratada como uma operação
reversível. Na mineração, ela funciona como um teste rápido: cada cabeçalho
candidate gera uma saída e essa saída é comparada com o alvo definido pela
rede.
