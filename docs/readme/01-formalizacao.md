## 1. Formalização Matemática do Problema e Intuição Estrutural

### 1.1. A Equação Fundamental do Proof of Work

Seja $H \in \{0,1\}^{640}$ o cabeçalho de um bloco candidato do Bitcoin, composto exatamente por 80 bytes (640 bits). Seja $\text{SHA-256}: \{0,1\}^* \to \{0,1\}^{256}$ a função hash criptográfica padronizada pelo NIST (FIPS 180-4). O problema de mineração consiste em encontrar uma tupla de parâmetros livres dentro de $H$ tal que a aplicação dupla da função satisfaça a inequação inteira:

$$\text{int}_{\text{LE}}\left(\text{SHA-256}\left(\text{SHA-256}(H)\right)\right) \le T$$

onde $\text{int}_{\text{LE}}(\cdot)$ interpreta o vetor de saída de 256 bits (32 bytes) como um inteiro sem sinal em formato *little-endian*, e $T \in \{0, 1, \dots, 2^{256}-1\}$ é o limiar numérico (*target*) derivado do campo compacto de 32 bits `nBits` ($0\text{x}EEFFFFFF$ na forma base-256 científica):

$$T = \text{mantissa} \times 2^{8 \times (\text{expoente} - 3)}$$

**Intuição Didática (A Máquina de Moer Dados):** Imagine o $\text{SHA-256}^2$ como um liquidificador industrial com 64 lâminas giratórias seguidas por outro liquidificador idêntico de 64 lâminas. Você coloca uma ficha de 80 letras (o cabeçalho do bloco) e ele devolve um número de 256 dígitos binários (zeros e uns). O protocolo do Bitcoin exige que o número sorteado seja menor que o alvo $T$ — o que equivale a exigir que o resultado comece com uma sequência longa de zeros consecutivos (por exemplo, $d \approx 76$ zeros binários iniciais). Como o liquidificador embaralha os bits de forma caótica, mudar um único bit na entrada altera cerca de metade dos 256 bits da saída (efeito avalanche).

### 1.2. Diferenciação Formal de Problemas Criptográficos

Muitas abordagens falham por confundir quatro problemas com complexidades assintóticas distintas:

| Problema | Definição Matemática Formal | Graus de Liberdade | Complexidade Clássica | Complexidade Quântica |
| --- | --- | --- | --- | --- |
| **Colisão** | Encontrar $x \neq y$ tal que $F(x) = F(y)$ | Irrestrito em $x, y$ | $\mathcal{O}(2^{n/2}) = 2^{128}$ | $\mathcal{O}(2^{n/3}) \approx 2^{85,3}$ |
| **Pré-imagem Completa** | Dado $Y$, achar $x$ tal que $F(x) = Y$ | Irrestrito em $x$ | $\mathcal{O}(2^n) = 2^{256}$ | $\mathcal{O}(2^{n/2}) = 2^{128}$ |
| **Mineração (PoW)** | Achar $x \in \mathcal{V}$ tal que $F(x) \le T$ | Restrito a $\mathcal{V} \subset \{0,1\}^{640}$ | $\mathcal{O}(2^{256}/T) = 2^d$ | $\mathcal{O}(2^{d/2})$ |
| **Criptoanálise** | Distinguir $F(x)$ de permutação aleatória | Análise algébrica de $F$ | $\mathcal{O}(2^{k}) \ll 2^d$ | Depende do subespaço |

* **Por que busca de colisão não resolve mineração:** Encontrar dois cabeçalhos $H_1 \neq H_2$ com o mesmo hash não garante que esse hash seja menor ou igual a $T$.
* **Por que mineração é uma pré-imagem parcial com domínio restrito:** Não buscamos um ponto único $Y$, mas qualquer ponto no intervalo $[0, T]$. Em compensação, a entrada $x$ não pode ser qualquer string de 640 bits: os primeiros 68 bytes (versão, bloco anterior, raiz de Merkle) devem obedecer estritamente às regras de consenso do protocolo.

### 1.3. Decomposição do Cabeçalho de 80 Bytes nos Blocos do SHA-256

O algoritmo SHA-256 processa mensagens em blocos de 512 bits (64 bytes) usando a construção de Merkle-Damgård com função de compressão Davies-Meyer $\text{CF}: \{0,1\}^{256} \times \{0,1\}^{512} \to \{0,1\}^{256}$. Como o cabeçalho do Bitcoin possui 640 bits (80 bytes), uma única tentativa de mineração exige **três invocações da função de compressão**:

1. **Primeiro Bloco da 1ª Hash ($M^{(0)}$, 512 bits / Bytes 0 a 63):**
* `nVersion` (Bytes 0–3, 32 bits): Versão do bloco (permite *version rolling* nos bits de propósito geral segundo o BIP 320).
* `hashPrevBlock` (Bytes 4–35, 256 bits): Hash duplo do cabeçalho do bloco anterior (fixo durante toda a busca).
* `hashMerkleRoot[0:27]` (Bytes 36–63, 224 bits): Os primeiros 28 bytes da raiz da Árvore de Merkle das transações.
* **Propriedade Crítica:** Todo o bloco $M^{(0)}$ independe de `nTime` e `nNonce`. Logo, o estado intermediário $S_{\text{mid}} = \text{CF}(\text{IV}, M^{(0)})$ é constante para $2^{32}$ variações de `nNonce`!


2. **Segundo Bloco da 1ª Hash ($M^{(1)}$, 512 bits / Bytes 64 a 127 com Padding):**
* $W_0$ (Bytes 64–67): Últimos 4 bytes da `hashMerkleRoot` (`merkle_tail`).
* $W_1$ (Bytes 68–71): `nTime` (timestamp Unix de 32 bits).
* $W_2$ (Bytes 72–75): `nBits` (alvo compactado de 32 bits, fixo por época de 2016 blocos).
* $W_3$ (Bytes 76–79): `nNonce` (espaço de busca primário de 32 bits).
* $W_4$ a $W_{15}$ (Bytes 80–127): Padding determinístico do padrão FIPS 180-4 (`0x80000000` em $W_4$, zeros de $W_5$ a $W_{14}$, e o comprimento da mensagem $640_{10} = \text{0x00000280}$ em $W_{15}$).


3. **Bloco Único da 2ª Hash ($M^{(2)}$, 512 bits):**
* $W_0^{(2)}$ a $W_7^{(2)}$: Os 256 bits resultantes de $H^{(1)} = \text{CF}(S_{\text{mid}}, M^{(1)})$.
* $W_8^{(2)}$ a $W_{15}^{(2)}$: Padding fixo para mensagem de 256 bits (`0x80000000` em $W_8^{(2)}$, zeros de $W_9^{(2)}$ a $W_{14}^{(2)}$, e comprimento $256_{10} = \text{0x00000100}$ em $W_{15}^{(2)}$).



### 1.4. Estrutura Algébrica Interna da Função de Compressão SHA-256

Cada chamada à função de compressão executa 64 rodadas sobre um vetor de estado de oito palavras de 32 bits $V_t = (a_t, b_t, c_t, d_t, e_t, f_t, g_t, h_t) \in (\mathbb{Z}_{2^{32}})^8$, alimentado por um cronograma de expansão de mensagem (*Message Schedule*) $W_0, \dots, W_{63}$:

$$W_t = \begin{cases} M_t & \text{se } 0 \le t \le 15 \\ \sigma_1(W_{t-2}) \boxplus W_{t-7} \boxplus \sigma_0(W_{t-15}) \boxplus W_{t-16} & \text{se } 16 \le t \le 63 \end{cases}$$

onde $\boxplus$ representa adição modular $\pmod{2^{32}}$, e as funções lineares de dispersão bit a bit sobre $\mathbb{F}_2^{32}$ são:


$$\sigma_0(x) = (x \ggg 7) \oplus (x \ggg 18) \oplus (x \gg 3)$$

$$\sigma_1(x) = (x \ggg 17) \oplus (x \ggg 19) \oplus (x \gg 10)$$

Como ilustrado no diagrama da função de compressão acima, a cada rodada $t \in [0, 63]$, apenas **duas palavras de estado** sofrem transformação aritmética direta ($a_{t+1}$ e $e_{t+1}$), enquanto as outras seis apenas deslocam de posição ($b_{t+1}=a_t, c_{t+1}=b_t, d_{t+1}=c_t, f_{t+1}=e_t, g_{t+1}=f_t, h_{t+1}=g_t$):

$$T_1 = h_t \boxplus \Sigma_1(e_t) \boxplus \text{Ch}(e_t, f_t, g_t) \boxplus K_t \boxplus W_t$$

$$T_2 = \Sigma_0(a_t) \boxplus \text{Maj}(a_t, b_t, c_t)$$

$$e_{t+1} = d_t \boxplus T_1, \quad a_{t+1} = T_1 \boxplus T_2$$

com as funções não-lineares em $\mathbb{F}_2$ (grau algébrico 2):


$$\text{Ch}(e,f,g) = (e \wedge f) \oplus (\neg e \wedge g), \quad \text{Maj}(a,b,c) = (a \wedge b) \oplus (a \wedge c) \oplus (b \wedge c)$$


e rotações cíclicas $\Sigma_0(a) = (a \ggg 2) \oplus (a \ggg 13) \oplus (a \ggg 22)$ e $\Sigma_1(e) = (e \ggg 6) \oplus (e \ggg 11) \oplus (e \ggg 25)$.

**A Origem da Dificuldade Criptoanalítica (Conflito de Estruturas Algébricas):** As rotações $\Sigma_i, \sigma_i$ e o $\oplus$ (XOR) são lineares no corpo finito $\mathbb{F}_2^{32}$, mas a adição $\boxplus$ com transporte de *carry* (*vai-um*) é linear no anel $\mathbb{Z}_{2^{32}}$ e **fortemente não-linear em $\mathbb{F}_2^{32}$** (o bit mais significativo do *carry* tem grau algébrico 32 em $\mathbb{F}_2$). Simultaneamente, $\text{Ch}$ e $\text{Maj}$ quebram a linearidade em $\mathbb{Z}_{2^{32}}$. Essa alternância ARX (*Add-Rotate-XOR*) destrói qualquer estrutura de subespaço vetorial após poucas rodadas.

### 1.5. Estatística, Ajuste de Dificuldade e Espaço de Busca

* **Processo de Bernoulli e Poisson:** Modelando $\text{SHA-256}^2$ como uma família pseudoaleatória uniforme, cada avaliação de um cabeçalho distinto tem probabilidade independente de sucesso $p = \frac{T + 1}{2^{256}} \approx \frac{1}{D \cdot 2^{32}}$, onde $D = \frac{T_{\text{max}}}{T}$ é a dificuldade da rede ($T_{\text{max}} = (2^{16}-1)\cdot 2^{208}$).
* **Propriedade Sem Memória (*Memorylessness*):** Se um minerador já testou $10^{20}$ hashes sem sucesso, a probabilidade de encontrar o bloco na próxima tentativa continua sendo exatamente $p$. O número de tentativas até o sucesso segue uma **Distribuição Geométrica** com média $\mathbb{E}[N] = 1/p$, e o tempo até encontrar um bloco a uma taxa de hash $R$ segue uma **Distribuição Exponencial** $f(t) = \lambda e^{-\lambda t}$ com $\lambda = R \cdot p$.
* **Exaustão do Nonce e Custo da Merkle Tree:** Como $2^{32} \approx 4,29 \times 10^9$ esgota o campo `nNonce` em microssegundos, alterar o `ExtraNonce` na transação *Coinbase* (folha $L_0$ da árvore de Merkle) requer recalcular apenas o caminho logarítmico de irmãos ($\log_2 N_{\text{tx}}$ compressões SHA-256), gerando uma nova `hashMerkleRoot` e um novo $S_{\text{mid}}$.

---
