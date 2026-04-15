# Desafio SBPO 2025 — O Problema da Seleção de Pedidos Ótima

Solução desenvolvida para o [Desafio de Otimização SBPO 2025](https://github.com/mercadolibre/challenge-sbpo-2025), proposto pelo Mercado Livre. O objetivo é encontrar uma *wave* de pedidos ótima em um armazém, maximizando o número de itens coletados por corredor visitado.

---

## Contexto do Problema

Em centros de distribuição, pedidos são agrupados em *waves* para serem coletados juntos. A seleção criteriosa dos pedidos que compõem uma wave permite reduzir o número de corredores visitados, aumentando a produtividade da operação.

Formalmente, dado um conjunto de pedidos $\mathcal{O}$, um conjunto de corredores $\mathcal{A}$, e limites operacionais LB e UB, o objetivo é encontrar um subconjunto de pedidos $O' \subseteq \mathcal{O}$ e um subconjunto de corredores $A' \subseteq \mathcal{A}$ que maximizem:

$$\max \frac{\sum_{o \in O'} \sum_{i \in I_o} u_{oi}}{|A'|}$$

sujeito às restrições de capacidade mínima e máxima da wave, e de disponibilidade de estoque nos corredores selecionados.

---

## Abordagem Inicial — Heurística MILP (Google Colab)

A solução inicial foi desenvolvida e testada no **Google Colab**, no notebook `sot_V4.ipynb`.

### Modelagem

O problema foi modelado como um **Programa Linear Inteiro Misto (MILP)**, resolvido com o solver **CBC** via [OR-Tools](https://developers.google.com/optimization).

#### Variáveis de decisão

| Variável | Tipo | Significado |
|---|---|---|
| `X[(i, o, a)]` | Inteira ≥ 0 | Unidades do item `i` do pedido `o` coletadas no corredor `a` |
| `Yo[o]` | Binária | 1 se o pedido `o` está incluído na wave |
| `Ya[a]` | Binária | 1 se o corredor `a` é visitado |

A variável `X` é uma **variável de transporte tridimensional**: ela decide simultaneamente quais pedidos entram na wave, quais corredores são ativados e de qual corredor específico cada item será retirado.

#### Restrições

**Tamanho da wave:** o total de unidades deve respeitar os limites operacionais.

$$ \text{LB} \leq \sum_{o \in O} \text{contO}[o] \cdot Y_o \leq \text{UB} $$

**Consistência pedido ↔ transporte:** se um pedido é selecionado, todas as suas unidades devem ser transportadas; caso contrário, nada é transportado.

$$\sum_{i \in I_o} \sum_{a \in A} X_{ioa} = \text{contO}[o] \cdot Y_o \quad \forall o \in O$$

**Capacidade dos corredores por item:** o total retirado de cada item em cada corredor não pode exceder o estoque disponível. Esta restrição também ativa `Ya[a]` quando o corredor é utilizado.

$$\sum_{o \in O} X_{ioa} \leq u_{ai} \cdot Y_a \quad \forall i \in I,\; a \in A$$

#### Função objetivo — heurística de linearização

O objetivo original é **fracionário** (não-linear), pois tanto o numerador quanto o denominador dependem das variáveis de decisão. Para permitir o uso do CBC, o objetivo foi substituído pela seguinte heurística linear:

$$\max \sum_{o \in O} \text{contO}[o] \cdot Y_o \;-\; \sum_{a \in A} \text{Uailim}[a] \cdot Y_a$$

onde `Uai_lim[a]` é a capacidade total do corredor `a`. A ideia é **maximizar os itens coletados e penalizar os corredores usados**, com o peso da penalidade proporcional ao tamanho de cada corredor.

### Limitação desta abordagem

Esta linearização é uma **heurística**: o objetivo que o solver minimiza não é equivalente ao objetivo original do desafio. A solução encontrada pode ser muito boa na prática, mas **não garante otimalidade** em relação à métrica real (itens/corredor). É, no entanto, um bom ponto de partida — rápida de implementar, encontra soluções viáveis e de qualidade razoável.

---

## Métodos Exatos Estudados

Para garantir a otimalidade, foram estudadas abordagens que resolvem o problema fracionário de forma exata, baseadas no **Teorema de Dinkelbach (1967)**:

> O valor ótimo $p^*$ do problema fracionário satisfaz: $g(p^*) = \max_x \left[ N(x) - p^* \cdot D(x) \right] = 0$

Isso transforma o problema em uma busca pela raiz de $g(p) = 0$, onde cada avaliação de $g(p)$ é um MILP linear — resolvível pelo CBC.

### Bisseção Paramétrica (`sot_bissecao.ipynb`)

Busca binária no intervalo $[0, \text{UB}]$:

```
enquanto hi - lo > tolerância:
    p_mid = (lo + hi) / 2
    resolver MILP com objetivo: max N(x) - p_mid * D(x)
    se g(p_mid) > 0:  lo = p_mid   # solução com métrica > p_mid existe
    senão:            hi = p_mid   # não existe solução melhor que p_mid
```

Converge em $O(\log(\text{UB} / \varepsilon))$ iterações — tipicamente 14 a 20 chamadas ao solver para precisão de 0,01.

### Dinkelbach Puro (`sot_dinkelbach.ipynb`)

Em vez de dividir o intervalo ao meio, usa a própria solução encontrada para atualizar $p$:

```
enquanto g(p) > tolerância:
    resolver MILP com objetivo: max N(x) - p * D(x)
    p ← N(x*) / D(x*)    # métrica real da solução atual
```

Converge **superlinearmente** — tipicamente 3 a 6 iterações — porque cada novo $p$ já parte do valor objetivo que a solução corrente atingiu.

### Comparação dos métodos

| | Heurística original | Bisseção | Dinkelbach |
|---|---|---|---|
| Garante otimalidade | Não | Sim | Sim |
| Chamadas ao solver | 1 | ~15–20 | ~3–6 |
| Implementação | Simples | Moderada | Moderada |
| Notebook | `sot_V4.ipynb` | `sot_bissecao.ipynb` | `sot_dinkelbach.ipynb` |

---

## Estrutura do Repositório

```
.
├── sot_V4.ipynb           # Solução inicial (heurística, Google Colab)
├── sot_bissecao.ipynb     # Solução exata via bisseção paramétrica
├── sot_dinkelbach.ipynb   # Solução exata via algoritmo de Dinkelbach
└── README.md
```

---

## Como Executar

Os notebooks foram desenvolvidos para o **Google Colab** e carregam os dados diretamente do repositório oficial do desafio. Para executar:

1. Abra o notebook desejado no Google Colab.
2. Ajuste a variável `instancia` (de 1 a 20) na segunda célula.
3. Execute todas as células em ordem.

Para instalar as dependências localmente:

```bash
pip install ortools pandas numpy
```

---

## Referência

Dinkelbach, W. (1967). *On nonlinear fractional programming*. Management Science, 13(7), 492–498.
