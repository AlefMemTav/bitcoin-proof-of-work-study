## 2. Mapa do Conhecimento Científico e Estado da Arte

### 2.1. Criptoanálise Clássica e Solvers Algébricos (Linha C)

A literatura acadêmica (IACR CRYPTO, EUROCRYPT, FSE, ASIACRYPT) investigou intensivamente o SHA-256 desde sua publicação em 2001:

1. **Criptoanálise Diferencial e *Local Collisions*:**
* *Mendel, Nad & Schläffer (EUROCRYPT 2013)* e *Li, Liu & Wang (CRYPTO 2024)* demonstraram colisões práticas em **31 a 39 rodadas** do SHA-256 usando diferenciais modulares assinados (*signed bit differentials*) e condições de mensagem neutras.
* **Limitação para Mineração:** A propagação de características diferenciais exige liberdade total nas 16 palavras $W_0 \dots W_{15}$ do bloco de entrada. No bloco $M^{(1)}$ do Bitcoin, 12 das 16 palavras ($W_4 \dots W_{15}$) são constantes rígidas de *padding*, e no bloco $M^{(2)}$, 8 palavras ($W_8 \dots W_{15}$) são constantes de *padding* e as 8 primeiras ($W_0 \dots W_7$) são a saída caótica do primeiro hash, impedindo o controle de bits neutros além da rodada 18.


2. **Ataques de Pré-imagem (*Meet-in-the-Middle* e *Bicliques*):**
* *Aoki et al. (2009)* e *Khovratovich, Rechberger & Savelieva (2011)* introduziram a criptoanálise por **Bicliques**, alcançando ataques de pré-imagem em **45 rodadas** (e pseudo-pré-imagem em **52 rodadas**) com complexidade temporal de $\approx 2^{255,5}$ (ganho de um fator de apenas $1,4\times$ sobre força bruta).
* **Limitação para Mineração:** 52 rodadas ainda estão a 12 rodadas de distância de uma única chamada ao SHA-256 (64 rodadas) e a 76 rodadas de distância do $\text{SHA-256}^2$ (128 rodadas totais). Além disso, ataques *Meet-in-the-Middle* exigem acesso livre à variável de encadeamento de saída, que no Bitcoin é ocultada pela segunda camada do $\text{SHA-256}$.


3. **SAT Solvers, SMT e Bases de Gröbner (Criptoanálise Algébrica):**
* *Mironov & Zhang (2006)*, *Nossum (2012)* e *Nejati et al. (2017–2020)* codificaram a inversão parcial do SHA-256 em Forma Normal Conjuntiva (CNF) via transformação de Tseitin, resultando em $\approx 65.000$ variáveis booleanas e $\approx 280.000$ cláusulas por bloco de compressão.
* **Resultado Experimental Reproduzível:** Solvers CDCL modernos (*Kissat*, *CaDiCaL*, *CryptoMiniSat* com eliminação Gaussiana XOR) invertem pré-imagens de **até 22–24 rodadas** em minutos explorando propagação unitária nos bits do *padding*.Acima de 28 rodadas, a árvore de conflitos CDCL sofre explosão exponencial sem encontrar cláusulas aprendidas curtas, tornando-se $\approx 10^6$ vezes mais lenta por candidato que uma avaliação direta em hardware.



### 2.2. Inteligência Artificial e Aprendizado de Máquina (Linha B)

1. **Distinguishers Neurais em Cifras ARX:**
* A partir do trabalho seminal de *Gohr (CRYPTO 2019)* sobre a cifra ARX *Speck32/64*, pesquisadores testaram redes ResNet, MLP e Transformers para prever bits do *nonce* a partir de hashes ou distinguir saídas de rodadas reduzidas do SHA-256 de sequências verdaderamente aleatórias.
* **Limite Físico-Estatístico da Difusão:** No SHA-256, o critério estrito de avalanche (SAC — *Strict Avalanche Criterion*, onde inverter 1 bit de entrada inverte cada bit de saída com probabilidade $0,50 \pm \epsilon$) é atingido completamente na **rodada 9** de uma única compressão. Em experimentos controlados, redes neurais detectam correlações residuais estatisticamente significativas ($\text{acurácia} \gt 50,5\%$) em até **6 a 7 rodadas**. Em 64 rodadas (e 128 no hash duplo), a mútua informação condicional sobre qualquer subconjunto polinomial de features é indistinguível de zero, pois a função de grau algébrico em $\mathbb{F}_2$ atinge grau máximo ($2^{256}$ monômios potenciais).


2. **Onde a IA Realmente Funciona na Mineração:**
* Embora redes neurais **não possam prever bits de saída de 64 rodadas**, algoritmos de **Reinforcement Learning (RL) e Otimização Combinatória em Grafos** (como *PrefixRL* da NVIDIA e *AlphaDev*) têm eficácia comprovada na **síntese lógica de circuitos combinacionais** das rodadas do SHA-256, descobrindo topologias de somadores *Carry-Save* (CSA) e árvores de compressores 3:2 / 4:2 que reduzem a contagem de transistores e o atraso crítico de propagação (*critical path delay*) em ASICs.



### 2.3. Otimização Algorítmica e Arquitetural (Linhas A e D)

Aqui residem as **únicas vitórias matemáticas reais e operacionais** já demonstradas nas 64 rodadas completas:

1. **Midstate Caching (Reaproveitamento da 1ª Compressão):** Elimina 64 das 192 rodadas totais de cada tentativa (redução de **33,33%** no trabalho total).
2. **Pré-cálculo das Rodadas $R_0, R_1, R_2$ do Bloco $M^{(1)}$:** Como `nNonce` ocupa a palavra $W_3$, as rodadas $t=0$ ($W_0$), $t=1$ ($W_1$) e $t=2$ ($W_2$) da segunda compressão **não dependem do `nNonce**`! Logo, os estados $(a_3, b_3, c_3, d_3, e_3, f_3, g_3, h_3)$ podem ser pré-calculados uma única vez por segundo (`nTime`) e compartilhados por $2^{32}$ nonces, economizando mais 3 rodadas completas e metade da rodada $t=3$ (onde $T_2 = \Sigma_0(a_3) \boxplus \text{Maj}(a_3, b_3, c_3)$ também é constante!).
3. **ASICBoost (Colisões Estruturais no *Message Schedule* $W_t$):**
* Formalizado por *Timo Hanke e Sergio Demian Lerner (2016)*, baseia-se no fato de que a expansão de mensagem $W_{16} \dots W_{63}$ do bloco $M^{(1)}$ depende apenas de $W_0 \dots W_{15}$.
* Se gerarmos múltiplos candidatos que compartilham o mesmo final de `hashMerkleRoot` ($W_0$ idêntico em $M^{(1)}$, chamado *Overt ASICBoost* via *Version Rolling* no bloco $M^{(0)}$, ou *Covert ASICBoost* encontrando colisões nos últimos 4 bytes da raiz de Merkle), então para cada `nNonce` ($W_3$), **todo o vetor de expansão $W_0 \dots W_{63}$ de $M^{(1)}$ é 100% idêntico** para $k$ cabeçalhos distintos! Isso permite calcular a expansão de mensagem ($\sigma_0, \sigma_1$ e somas modulares de 48 palavras) apenas **uma vez** para múltiplos blocos candidatos, economizando até **15% a 20% da energia total do chip**.


4. **Poda Algébrica e *Early-Abort* nas Rodadas $R_{57}$ a $R_{63}$ do Bloco $M^{(2)}$:**
* Ao final da segunda compressão ($M^{(2)}$), o hash final é dado pelo *feed-forward* de Davies-Meyer:

$$H_0^{(2)} = \text{IV}_0 \boxplus a_{64}, \quad H_1^{(2)} = \text{IV}_1 \boxplus b_{64}, \dots, \quad H_7^{(2)} = \text{IV}_7 \boxplus h_{64}$$


* No Bitcoin, devido à codificação *little-endian* do número de 256 bits, os 32 bits mais significativos do número correspondem à última palavra do hash, $H_7^{(2)} = \text{IV}_7 \boxplus h_{64}$, e os 32 bits seguintes correspondem a $H_6^{(2)} = \text{IV}_6 \boxplus g_{64}$!
* Observe o deslocamento de registradores do SHA-256:

$$h_{64} = g_{63} = f_{62} = e_{61} = d_{60} \boxplus T_1^{(60)}$$


$$g_{64} = f_{63} = e_{62} = d_{61} \boxplus T_1^{(61)}$$


* **Descoberta Algébrica Crucial:** Para que $H_7^{(2)} = 0$ (exigido sempre que a dificuldade pede pelo menos 32 bits zero iniciais, o que ocorre no Bitcoin desde 2009!), é **matematicamente obrigatório** que:

$$e_{61} = h_{64} = -\text{IV}_7 \pmod{2^{32}} = 2^{32} - \text{0x5be0cd19} = \text{0xa41f32e7}$$


* Isso significa que **na rodada 60** (após calcular $e_{61}$), já sabemos com 100% de certeza determinística se os 32 bits mais significativos do hash serão zero! Em $1 - 2^{-32} \approx 99,99999997\%$ das tentativas, $e_{61} \neq \text{0xa41f32e7}$, permitindo **abortar imediatamente as rodadas 61, 62 e 63**, além de todas as 8 somas finais de Davies-Meyer e a expansão de $W_{61}, W_{62}, W_{63}$!
* Mais ainda: sabendo que $e_{61}$ deve assumir a constante fixa $\text{0xa41f32e7}$ para qualquer bloco válido, na rodada 61 todas as funções que dependem de $e_{61}$ — como $\Sigma_1(e_{61})$ — tornam-se **constantes conhecidas a priori**!



### 2.4. Computação Quântica e Algoritmo de Grover (Linha D)

* **Fundamento:** O algoritmo de Grover (*1996*) aplica amplificação de amplitude sobre uma superposição uniforme de $N = 2^k$ nonces, encontrando uma solução marcada em $\frac{\pi}{4}\sqrt{2^d}$ invocações do oráculo reversível $U_{\text{SHA-256}^2}$.
* **Análise Física Concreta (*Aggarwal et al., 2018*):**
* Cada chamada ao oráculo quântico exige implementar $\text{SHA-256}^2$ de forma reversível (portas Toffoli e CNOT, sem descarte de qubits de lixo via *uncomputation*), consumindo cerca de **2.400 qubits lógicos** e $\approx 1,5 \times 10^5$ portas T por avaliação.
* Com correção de erros quânticos baseada em *Surface Codes* (taxa de erro físico de $10^{-3}$), são necessários $\approx 4,5 \times 10^6$ qubits físicos.
* **O Gargalo da Paralelização Quântica:** Ao contrário dos ASICs clássicos, onde dividir o trabalho entre $P$ chips reduz o tempo linearmente ($1/P$), dividir uma busca de Grover entre $P$ processadores quânticos independentes reduz o ganho para $\sqrt{2^d / P}$ (perde-se vantagem quadrática na paralelização!). Para vencer a rede clássica dentro da janela de $\tau \approx 600\text{ s}$ na dificuldade atual ($d \approx 78$), um computador quântico precisaria executar $\approx 2^{39} \approx 5,5 \times 10^{11}$ iterações de Grover **em série**, exigindo velocidades de porta lógica na casa dos terahertz (3 a 4 ordens de grandeza acima dos qubits supercondutores atuais).



---
