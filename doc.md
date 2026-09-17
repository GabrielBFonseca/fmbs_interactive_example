# Implementação por força bruta para análise de empates no FMBS

## Objetivo

Esta seção apresenta uma implementação simplificada, por **força bruta**, para investigar os valores de conexão utilizados no **Fuzzy Marker-Based Segmentation (FMBS)** e, em particular, identificar situações em que diferentes pixels pertencentes a um marcador produzem o mesmo valor ótimo de conexão para um determinado pixel da imagem.

A implementação eficiente do FMBS utilizada anteriormente explora diretamente a estrutura hierárquica para calcular a conexão de cada pixel com um marcador fuzzy sem precisar avaliar explicitamente todos os pares `(pixel, marcador)`.

Nesta implementação experimental, seguimos uma estratégia diferente:

1. identificamos individualmente os pixels que pertencem ao marcador;
2. para cada pixel `x` da imagem;
3. calculamos explicitamente sua conexão com cada pixel `m` do marcador;
4. selecionamos o menor valor de conexão;
5. contamos quantos marcadores produziram exatamente esse mesmo valor mínimo.

Assim, além do valor de conexão fuzzy utilizado pelo FMBS, podemos investigar a **multiplicidade do mínimo**, isto é, quantos marcadores estão empatados como melhores conexões para cada pixel.

> **Importante:** esta implementação foi construída como código experimental para análise dos empates. Ela deve ser validada matematicamente e numericamente em relação à formulação original do FMBS antes de seus resultados serem utilizados em experimentos.

---

## 1. Ideia geral

Considere um pixel `x` e um conjunto de pixels pertencentes a um marcador:

\[
M = \{m_1,m_2,\ldots,m_k\}.
\]

A implementação por força bruta calcula explicitamente:

\[
C(x,m_1), C(x,m_2), \ldots, C(x,m_k).
\]

O valor de conexão de `x` com o marcador é então obtido por:

\[
C(x,M) = \min_{m \in M} C(x,m).
\]

Além disso, queremos calcular:

\[
T(x,M)
=
\left|
\left\{
m \in M :
C(x,m)=C(x,M)
\right\}
\right|.
\]

Ou seja, `T(x,M)` representa **quantos pixels do marcador atingem o mesmo valor mínimo de conexão com `x`**.

### Exemplo simples

Suponha que um pixel `x` possua quatro possíveis conexões com pixels pertencentes ao marcador:

| Marcador | Valor de conexão |
|---|---:|
| `m1` | 0.42 |
| `m2` | 0.18 |
| `m3` | 0.18 |
| `m4` | 0.31 |

Então:

\[
C(x,M)=0.18
\]

e:

\[
T(x,M)=2.
\]

Portanto, existem **dois marcadores empatados na melhor conexão** com `x`.

---

# 2. Lowest Common Ancestor (LCA)

A primeira função auxiliar é:

```python
def lowest_common_ancestor(tree, x, y):
```

Ela calcula o **Lowest Common Ancestor (LCA)** de dois nós da árvore.

Em uma hierarquia de regiões, o LCA de dois pixels corresponde ao menor nó da hierarquia que contém simultaneamente os dois pixels.

Para dois pixels `x` e `m`, podemos representar simplificadamente uma árvore como:

```text
                  R
                /   \
               A     B
              / \
             x   C
                / \
               m   z
```

Nesse exemplo, o menor nó contendo simultaneamente `x` e `m` é `A`. Portanto:

```text
LCA(x,m) = A
```

A altitude associada a esse nó:

```python
altitudes[lca]
```

representa o nível da hierarquia no qual `x` e `m` passam a pertencer à mesma região.

---

## 2.1 Como a função encontra o LCA

Primeiramente são obtidos os pais de todos os nós:

```python
parents = tree.parents()
```

Em seguida, o código percorre o caminho de `x` até a raiz:

```python
ancestors_x = set()

node = x

while True:
    ancestors_x.add(node)

    parent = parents[node]

    if parent == node:
        break

    node = parent
```

Assim, se o caminho de `x` até a raiz for:

```text
x -> A -> D -> R
```

o conjunto será:

```text
{x, A, D, R}
```

Depois, percorremos o caminho de `y` até encontrar o primeiro nó que também pertence aos ancestrais de `x`:

```python
node = y

while node not in ancestors_x:
    node = parents[node]
```

O primeiro nó comum encontrado é o LCA.

---

## 2.2 O que deve ser verificado nesta função

A implementação é deliberadamente simples e foi escolhida pela facilidade de leitura.

É importante verificar:

- se a estrutura de árvore fornecida pelo Higra utiliza a raiz como seu próprio pai;
- se todos os pixels são representados pelas folhas da árvore;
- se o índice das folhas corresponde corretamente aos índices utilizados pelos marcadores;
- se o LCA obtido corresponde ao menor componente da hierarquia contendo os dois pixels.

Também pode ser interessante posteriormente substituir esta implementação por uma função otimizada de LCA fornecida pelo Higra, principalmente caso o tempo de execução se torne um problema.

---

# 3. Conexão fuzzy entre um pixel e um marcador individual

A função:

```python
def pairwise_fuzzy_connection_value(
    tree,
    altitudes,
    marker,
    x,
    m,
    alpha=lambda x: 1 / (x + 1e-9)
):
```

é o ponto central da implementação experimental.

Ela tenta calcular explicitamente o valor de conexão entre:

- um pixel `x`;
- um pixel específico `m` pertencente ao marcador.

Primeiramente obtemos o grau de pertinência fuzzy do marcador:

```python
mu_m = marker[m]
```

Portanto:

\[
\mu(m) = \texttt{marker[m]}.
\]

A função `alpha` utilizada é:

```python
alpha = lambda x: 1 / (x + 1e-9)
```

de modo que valores maiores de pertinência produzem valores menores de `alpha`.

---

# 4. Caso `x == m`

Se o pixel analisado é o próprio pixel marcador:

```python
if x == m:
    return alpha(mu_m) * (1 - mu_m)
```

é utilizado:

\[
C(m,m)
=
\alpha(\mu(m))(1-\mu(m)).
\]

### Exemplo

Suponha:

\[
\mu(m)=0.8.
\]

Então, aproximadamente:

\[
\alpha(0.8)=\frac{1}{0.8}=1.25.
\]

Assim:

\[
C(m,m)
=
1.25(1-0.8)
=
1.25 \times 0.2
=
0.25.
\]

O pequeno valor `1e-9` presente no código serve para evitar uma divisão por zero.

---

# 5. Caso `x != m`

Quando `x` e `m` são pixels diferentes, primeiro calculamos:

```python
lca = lowest_common_ancestor(tree, x, m)
```

Em seguida, a implementação atual utiliza:

```python
return alpha(mu_m) * (
    1 - mu_m + altitudes[lca]
)
```

ou seja:

\[
C(x,m)
=
\alpha(\mu(m))
\left(
1-\mu(m)+\lambda(\operatorname{LCA}(x,m))
\right),
\]

onde:

- \(\mu(m)\) é o grau de pertinência do pixel `m` ao marcador fuzzy;
- \(\alpha\) é a transformação decrescente utilizada pelo FMBS;
- \(\lambda(\operatorname{LCA}(x,m))\) é a altitude do menor componente da hierarquia contendo simultaneamente `x` e `m`.

### Exemplo numérico

Considere:

\[
\mu(m)=0.8
\]

e:

\[
\lambda(\operatorname{LCA}(x,m))=0.3.
\]

Então:

\[
\alpha(0.8)\approx1.25.
\]

Logo:

\[
C(x,m)
=
1.25(1-0.8+0.3)
\]

\[
=
1.25(0.5)
\]

\[
=
0.625.
\]

Esse valor seria armazenado como a conexão entre `x` e esse marcador específico `m`.

---

# 6. Ponto crítico a ser validado

A função:

```python
pairwise_fuzzy_connection_value(...)
```

é a principal parte que precisa ser **matematicamente verificada**.

A implementação eficiente original do FMBS trabalha com operações sobre regiões da hierarquia, incluindo valores como:

```python
min_alpha_marker
max_mu
cv_to_sib
regional_connection_values
```

e realiza propagações de mínimos ao longo da árvore.

A versão por força bruta assume que a conexão entre um pixel `x` e um marcador individual `m` pode ser expressa diretamente a partir da altitude do LCA:

\[
C(x,m)
=
\alpha(\mu(m))
\left(
1-\mu(m)+\lambda(\operatorname{LCA}(x,m))
\right).
\]

Essa equivalência **não deve ser assumida sem validação**.

Portanto, antes de utilizar a contagem de empates em experimentos, deve-se conferir essa expressão diretamente com a definição matemática apresentada no artigo e/ou na formulação teórica do FMBS.

Uma validação particularmente importante consiste em comparar os resultados desta versão por força bruta com os valores produzidos pela implementação eficiente original.

---

# 7. Identificação dos pixels pertencentes ao marcador

Na função:

```python
fuzzy_connection_values_bruteforce(...)
```

primeiramente identificamos quais folhas possuem valor de marcador diferente de zero:

```python
marker_indices = np.flatnonzero(marker > 0)
```

Assim, estamos assumindo que:

```text
marker[x] == 0
```

significa que `x` **não pertence ao marcador**, enquanto:

```text
marker[x] > 0
```

significa que `x` possui algum grau de pertinência ao marcador fuzzy.

### Exemplo

Para:

```python
marker = np.array([
    0.0,
    0.7,
    0.0,
    1.0,
    0.3
])
```

teremos:

```python
marker_indices = [1, 3, 4]
```

Portanto, existem três pixels considerados marcadores.

> **Verificar:** esta interpretação deve estar de acordo com a definição utilizada no FMBS. Em particular, deve-se confirmar se pixels com pertinência exatamente zero devem ser completamente ignorados na análise das conexões individuais.

---

# 8. Matriz completa de conexões

A implementação cria uma matriz:

```python
connections = np.empty(
    (n_leaves, n_markers),
    dtype=np.float64
)
```

Essa matriz possui:

- uma linha para cada pixel da imagem;
- uma coluna para cada pixel pertencente ao marcador.

Portanto:

```python
connections[x, j]
```

representa a conexão entre o pixel `x` e o `j`-ésimo pixel pertencente ao marcador.

### Exemplo

Suponha uma imagem com 5 pixels e um marcador contendo 3 pixels.

A matriz pode ter a forma:

| Pixel | `m1` | `m2` | `m3` |
|---|---:|---:|---:|
| `x1` | 0.40 | 0.20 | 0.20 |
| `x2` | 0.10 | 0.35 | 0.50 |
| `x3` | 0.60 | 0.60 | 0.60 |
| `x4` | 0.25 | 0.30 | 0.45 |
| `x5` | 0.42 | 0.21 | 0.21 |

Cada valor dessa matriz é calculado explicitamente por:

```python
connections[x, j] = pairwise_fuzzy_connection_value(
    tree,
    altitudes,
    marker,
    x,
    m,
    alpha
)
```

Esse é justamente o comportamento que torna esta versão simples de analisar, mas computacionalmente cara.

---

# 9. Cálculo do valor de conexão com o marcador

Depois de preencher toda a matriz, o código executa:

```python
fcv = np.min(connections, axis=1)
```

Para cada pixel, selecionamos o menor valor encontrado entre todos os pixels pertencentes ao marcador.

Matematicamente:

\[
C(x,M)
=
\min_{m\in M}C(x,m).
\]

No exemplo anterior:

| Pixel | `m1` | `m2` | `m3` | FCV |
|---|---:|---:|---:|---:|
| `x1` | 0.40 | 0.20 | 0.20 | **0.20** |
| `x2` | 0.10 | 0.35 | 0.50 | **0.10** |
| `x3` | 0.60 | 0.60 | 0.60 | **0.60** |
| `x4` | 0.25 | 0.30 | 0.45 | **0.25** |
| `x5` | 0.42 | 0.21 | 0.21 | **0.21** |

Assim:

```python
fcv
```

seria aproximadamente:

```python
[0.20, 0.10, 0.60, 0.25, 0.21]
```

---

# 10. Contagem dos empates

A principal finalidade desta implementação aparece em:

```python
tie_count = np.sum(
    connections == fcv[:, None],
    axis=1
)
```

A expressão:

```python
fcv[:, None]
```

transforma `fcv` em uma coluna, permitindo comparar o menor valor de cada pixel com todas as conexões daquela mesma linha.

Para o exemplo anterior:

| Pixel | `m1` | `m2` | `m3` | FCV | `tie_count` |
|---|---:|---:|---:|---:|---:|
| `x1` | 0.40 | **0.20** | **0.20** | 0.20 | **2** |
| `x2` | **0.10** | 0.35 | 0.50 | 0.10 | **1** |
| `x3` | **0.60** | **0.60** | **0.60** | 0.60 | **3** |
| `x4` | **0.25** | 0.30 | 0.45 | 0.25 | **1** |
| `x5` | 0.42 | **0.21** | **0.21** | 0.21 | **2** |

Portanto:

```python
tie_count
```

seria:

```python
[2, 1, 3, 1, 2]
```

---

# 11. Como interpretar `tie_count`

É importante notar que:

```python
tie_count[x] == 1
```

não significa que existe um empate.

Significa que existe **um único marcador responsável pelo valor mínimo**.

Por outro lado:

```python
tie_count[x] > 1
```

indica que múltiplos pixels pertencentes ao marcador produzem exatamente o mesmo valor ótimo de conexão.

Por exemplo:

```python
tie_count[x] == 4
```

significa que quatro pixels do marcador atingem o mesmo valor mínimo de conexão com `x`.

---

# 12. Número de pixels que apresentam empate

Para contar quantos pixels da imagem possuem pelo menos dois marcadores empatados:

```python
pixels_with_ties = np.sum(tie_count > 1)
```

A porcentagem pode ser calculada por:

```python
percentage_pixels_with_ties = (
    100 * np.sum(tie_count > 1) / len(tie_count)
)
```

Por exemplo:

```text
Pixels com empate: 125
Percentual de pixels com empate: 4.37%
```

---

# 13. Número total de conexões adicionais empatadas

Outra medida possível é contar a quantidade de conexões adicionais que aparecem devido aos empates.

Como um valor:

```python
tie_count[x] == 1
```

representa uma solução única, ele corresponde a zero conexões adicionais empatadas.

Assim, podemos calcular:

```python
total_additional_ties = np.sum(tie_count - 1)
```

Por exemplo:

```python
tie_count = np.array([1, 1, 2, 3, 1])
```

corresponde a:

```text
pixel 0 -> 1 ótimo  -> 0 empates adicionais
pixel 1 -> 1 ótimo  -> 0 empates adicionais
pixel 2 -> 2 ótimos -> 1 empate adicional
pixel 3 -> 3 ótimos -> 2 empates adicionais
pixel 4 -> 1 ótimo  -> 0 empates adicionais
```

Portanto:

```python
np.sum(tie_count - 1)
```

resulta em:

```text
3
```

Essas duas métricas respondem perguntas diferentes:

- `np.sum(tie_count > 1)` mede **quantos pixels são afetados por empates**;
- `np.sum(tie_count - 1)` mede **a multiplicidade total adicional das soluções ótimas**.

---

# 14. Recuperando quais marcadores estão empatados

Quando:

```python
return_matrix=True
```

a função retorna também:

```python
marker_indices
connections
```

Isso permite descobrir exatamente quais marcadores estão empatados para um determinado pixel.

Por exemplo:

```python
x = 100

best_value = fcv[x]

tied_markers = marker_indices[
    connections[x] == best_value
]

print("Pixel:", x)
print("Melhor valor de conexão:", best_value)
print("Número de marcadores empatados:", len(tied_markers))
print("Marcadores empatados:", tied_markers)
```

Uma saída possível seria:

```text
Pixel: 100
Melhor valor de conexão: 0.283
Número de marcadores empatados: 3
Marcadores empatados: [231 245 298]
```

Isso pode ser particularmente útil para investigar **onde os empates aparecem na hierarquia e quais marcadores são responsáveis por eles**.

---

# 15. Comparação com a implementação eficiente

Antes de utilizar os resultados de `tie_count`, deve-se verificar se os valores de `fcv` produzidos pela implementação por força bruta coincidem com aqueles produzidos pela implementação eficiente original.

Um teste inicial pode ser:

```python
fcv_efficient = fuzzy_connection_values(
    tree,
    altitudes,
    marker
)

fcv_bruteforce, tie_count = fuzzy_connection_values_bruteforce(
    tree,
    altitudes,
    marker
)

print(
    "Valores equivalentes:",
    np.allclose(fcv_efficient, fcv_bruteforce)
)

print(
    "Maior diferença:",
    np.max(np.abs(fcv_efficient - fcv_bruteforce))
)
```

Idealmente:

```python
np.allclose(fcv_efficient, fcv_bruteforce)
```

deve retornar:

```text
True
```

e a maior diferença deve ser zero ou suficientemente próxima de zero para ser explicada apenas por precisão numérica.

Se os valores forem diferentes, **a contagem de empates da versão por força bruta ainda não deve ser considerada validada**.

Nesse caso, o primeiro ponto a investigar é a função:

```python
pairwise_fuzzy_connection_value(...)
```

e, especificamente, a expressão utilizada para representar a conexão entre `x` e `m`.

---

# 16. Igualdade exata e ponto flutuante

Atualmente os empates são detectados por:

```python
connections == fcv[:, None]
```

Isso representa uma definição de **igualdade exata**.

Esse comportamento pode ser desejável se o objetivo do experimento for identificar empates matematicamente produzidos pela estrutura do algoritmo.

Entretanto, como os valores são representados em ponto flutuante, também pode ser interessante testar uma versão baseada em tolerância numérica:

```python
ties = np.isclose(
    connections,
    fcv[:, None],
    rtol=1e-12,
    atol=1e-12
)

tie_count = np.sum(ties, axis=1)
```

Essas duas definições respondem perguntas ligeiramente diferentes:

- `==` identifica valores representados exatamente da mesma forma;
- `np.isclose` considera valores suficientemente próximos como equivalentes.

A definição utilizada nos experimentos deve ser explicitamente documentada.

---

# 17. Custo computacional

Se existem:

- \(N\) pixels;
- \(K\) pixels pertencentes ao marcador;

a matriz:

```python
connections
```

possui aproximadamente:

\[
N \times K
\]

valores.

Além disso, a implementação atual calcula o LCA separadamente para cada par `(x,m)`.

Portanto, esta versão **não deve ser considerada uma alternativa eficiente ao algoritmo original**.

Seu objetivo é servir como uma implementação de referência, simples de inspecionar e útil para estudar os empates.

Por esse motivo, o tamanho da imagem utilizado nos experimentos foi reduzido significativamente.

Para imagens grandes ou marcadores contendo muitos pixels, o tempo de execução e o consumo de memória podem crescer rapidamente.

---

# 18. Testes recomendados

Antes de utilizar esta implementação em experimentos maiores, recomenda-se realizar alguns testes controlados.

### Teste 1: um único marcador

Criar um marcador contendo apenas um pixel.

Nesse caso, esperamos:

```python
tie_count[x] == 1
```

para todos os pixels, pois não existe outro marcador com o qual possa ocorrer empate.

---

### Teste 2: dois marcadores simétricos

Construir uma hierarquia pequena na qual dois marcadores possuam exatamente a mesma relação hierárquica com um determinado pixel `x`.

Se ambos possuírem também o mesmo grau de pertinência fuzzy, espera-se:

```python
tie_count[x] == 2
```

---

### Teste 3: marcadores com diferentes pertinências

Criar dois marcadores com:

```text
mu(m1) = 1.0
mu(m2) = 0.5
```

e verificar manualmente os valores:

\[
C(x,m_1)
\]

e:

\[
C(x,m_2).
\]

Esse teste é importante para verificar o papel da transformação `alpha` e do termo `1 - mu_m`.

---

### Teste 4: comparação com o algoritmo eficiente

Para diferentes imagens e diferentes conjuntos de marcadores, comparar:

```python
fcv_efficient
```

com:

```python
fcv_bruteforce
```

Esse é o teste mais importante antes da análise dos empates.

---

# 19. TODO

- [ ] **Revisar matematicamente `pairwise_fuzzy_connection_value`.**
      Conferir no artigo e na formulação original do FMBS se a conexão entre
      um pixel `x` e um marcador individual `m` pode ser expressa diretamente
      pela altitude do LCA da forma utilizada nesta implementação.

- [ ] **Comparar a implementação por força bruta com a implementação eficiente.**
      Os valores finais de FCV devem coincidir antes que a contagem de empates
      seja considerada válida.

- [ ] **Criar exemplos sintéticos pequenos.**
      Utilizar árvores nas quais os valores possam ser calculados manualmente
      para validar cada etapa do código.

- [ ] **Testar casos contendo empates conhecidos.**
      Construir situações com dois ou mais marcadores que necessariamente
      produzam a mesma conexão com um determinado pixel.

- [ ] **Verificar a interpretação de `marker > 0`.**
      Confirmar se somente folhas com pertinência estritamente positiva devem
      ser consideradas marcadores na análise por força bruta.

- [ ] **Definir o critério de igualdade.**
      Decidir se os experimentos utilizarão igualdade exata (`==`) ou uma
      tolerância numérica (`np.isclose`).

- [ ] **Analisar separadamente marcadores de objeto e fundo.**
      Aplicar o procedimento aos dois marcadores utilizados pelo FMBS e
      comparar a frequência e a distribuição dos empates.

- [ ] **Avaliar otimizações somente após a validação.**
      Esta implementação prioriza clareza e verificabilidade. Depois de
      validada, operações como LCA podem ser otimizadas sem alterar a
      definição utilizada para os empates.

---

# 20. Resumo do fluxo

O fluxo completo da implementação por força bruta pode ser resumido como:

```text
Hierarquia + altitudes + marcador fuzzy
                  |
                  v
      identificar marker > 0
                  |
                  v
       para cada pixel x
                  |
                  v
       para cada marcador m
                  |
                  v
           calcular LCA(x,m)
                  |
                  v
            calcular C(x,m)
                  |
                  v
     armazenar em connections[x,m]
                  |
                  v
       C(x,M) = min_m C(x,m)
                  |
                  v
 contar quantos m atingem esse mínimo
                  |
                  v
        fcv[x] + tie_count[x]
```

A principal vantagem dessa versão é a **transparência**: todas as conexões
individuais ficam disponíveis para inspeção.

A principal desvantagem é o **custo computacional**, já que a estratégia abandona
as otimizações da implementação eficiente baseada na hierarquia.

Por isso, esta implementação deve ser entendida principalmente como uma
**implementação experimental/de referência para validar e estudar a ocorrência
de empates no FMBS**, e não como substituta da implementação eficiente.