# Aumento de limite e gasto no cartão: quando a diferença de médias engana

Simulação de um problema clássico de crédito: um banco aumenta o limite do cartão de parte dos clientes e quer saber quanto isso muda o gasto mensal. Como os dados são simulados, o efeito verdadeiro é conhecido (R$ 150), e dá para medir exatamente quanto cada estimador erra e por quê.

O objetivo não é o modelo, é a pergunta: **em que condições comparar quem recebeu com quem não recebeu dá a resposta certa?**

## A pergunta

Quanto um aumento de 50% no limite do cartão muda o gasto mensal do cliente?

Na prática, o banco não sorteia quem recebe o aumento, mas oferece para quem tem score alto. Isso muda tudo quando queremos medir o efeito causal.

## Como os dados foram gerados

20.000 clientes com três características:

- **Renda:** lognormal com mediana de R$ 4.000
- **Score:** de 300 a 1000, construído a partir do percentil de renda mais um ruído
- **Tempo de casa:** uniforme entre 0 e 15 anos

Para cada cliente, foram gerados os dois resultados potenciais:

- `y0`: gasto sem o aumento de limite, que depende de renda, score e tempo de casa
- `y1 = y0 + 150`: gasto com o aumento

Ou seja, o efeito verdadeiro é de **R$ 150 para todo mundo**. Qualquer estimador correto tem que chegar perto disso.

A mesma população foi submetida a duas regras de atribuição do tratamento:

| Cenário | Quem recebe o aumento |
| --- | --- |
| A | Sorteio, 50% de chance para cada cliente |
| B | Clientes com score acima da média |

Só o que muda entre os cenários é a regra. Os clientes, a renda, o score e os resultados potenciais são os mesmos.

## O que a diferença de médias encontra

| | Cenário A | Cenário B |
| --- | --- | --- |
| Diferença de médias | 144.73 | 1116.20 |
| IC 95% (bootstrap) | [9.89, 281.89] | [1023.93, 1215.26] |
| Efeito verdadeiro | 150 | 150 |

No cenário A, a diferença de médias acerta. No B, erra para cima, e muito: os clientes tratados já gastavam mais antes de qualquer aumento.

Como `y0` é conhecido na simulação, dá para decompor a diferença de médias em efeito mais viés, exatamente como no capítulo 1 do *Causal Inference for the Brave and True*:

```
E[y|T=1] - E[y|T=0] = (E[y1|T=1] - E[y0|T=1]) + (E[y0|T=1] - E[y0|T=0]) = ATT + viés
```

| | Cenário A | Cenário B |
| --- | --- | --- |
| ATT | 150 | 150 |
| Viés | -3.43 | 968.55 |

O ATT é 150 nos dois cenários. O que muda é o viés: perto de zero quando o tratamento é sorteado, e de 968.55 quando é dado por score. Toda a diferença entre os cenários está no termo `E[y0|T=1] - E[y0|T=0]`, que é a pergunta "os tratados já eram diferentes antes?".

## O grafo causal

![Cenário A](..\figs\grafo_cenario_a.png)
![Cenário B](..\figs\grafo_cenario_b.png)

No cenário A, nenhuma seta chega em T: a atribuição é aleatória e não existe confounder. No cenário B aparece a seta `Score → T`. Como o score também causa o gasto, ele é um confounder, e abre uma porta dos fundos entre T e Y que passa por `Score` e, indiretamente, por `Renda`.

Uma observação que vale registrar: o score é mediador do efeito da renda sobre o gasto. Se a pergunta fosse "quanto a renda muda o gasto?", controlar por score seria um erro. Como a pergunta é sobre T, e o score é medido antes de T, ele não está no caminho entre tratamento e resultado. Mediador é sempre relativo à pergunta.

## O que a regressão encontra

Regressão de `y` sobre `T`, com e sem controles (renda, score e tempo de casa):

| Modelo | Cenário A | Cenário B |
| --- | --- | --- |
| `y ~ T` | 146.56 [125.92, 167.21] | 1118.55 [1102.87, 1134.23] |
| `y ~ T + renda + score + tempo_casa` | 150.72 [146.13, 151.67] | 155.13 [150.32, 159.94] |

Com os controles, os dois cenários voltam para perto de 150. No cenário B, controlar por score fecha a porta dos fundos; no A, não havia porta aberta, mas os controles explicam parte da variação de `y` e apertam o intervalo.

Na regressão, só o coeficiente de T tem leitura causal. Os coeficientes de renda, score e tempo de casa são ajustes, não efeitos: o de renda, por exemplo, mede apenas a parte que não passa pelo score.

## Controlar por tudo não funciona: o caso da utilização

Depois dos resultados acima, surge a tentação de controlar por todas as variáveis disponíveis. Para testar isso, foi criada a `utilizacao = gasto / limite`, uma métrica muito comum em crédito, e incluída na regressão.

| Modelo | Cenário A | Cenário B |
| --- | --- | --- |
| Com controles | 150.72 | 178.27 |
| Com controles + utilização | 155.13 | 186.31 |

O coeficiente de T se afasta de 150 nos dois cenários.

![Cenário B com utilização](../figs/cenario_a_collider.png)
![Cenário B com utilização](../figs/cenario_b_collider.png)

A utilização é um collider: é consequência do tratamento (que muda o limite) e do resultado (o gasto). Condicionar nela cria uma associação entre T e os resultados potenciais que não existia. A conta mostra isso: para dois clientes com a mesma utilização e o mesmo limite base, o tratado precisa de um `y0` maior para chegar à mesma utilização, porque o denominador dele é 50% maior.

A regra prática que sai daqui: **nada que seja medido depois do tratamento entra como controle.** Em banco, isso aparece sempre que alguém controla por comportamento pós-oferta (utilização, fatura, atraso) ao avaliar uma campanha.

## O que isso significa para uma campanha real

Quando o banco oferece aumento de limite para quem tem score alto e depois compara o gasto de quem recebeu com o de quem não recebeu, o número que sai superestima o efeito da campanha. A parte do "efeito" que vem do score já existia antes.

Para medir de verdade, as opções são: sortear parte das ofertas (teste A/B), ou comparar clientes com score parecido, que é o que a regressão com controle faz, com a hipótese de que o score explica toda a regra de atribuição. Na simulação essa hipótese é verdadeira por construção, em dados reais é a parte que precisa ser defendida.

## Como reproduzir

```
pip install numpy pandas matplotlib networkx statsmodels
jupyter notebook simulacao.ipynb
```

As sementes estão fixadas no notebook; os números das tabelas são os que ele gera.