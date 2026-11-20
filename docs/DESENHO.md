# Desenho da pesquisa

Este documento registra o desenho do projeto e as decisões tomadas antes de olhar os resultados. O notebook segue este roteiro.

## 1. Origem da hipótese

O ponto de partida é James et al. (2016, *Journal of Science and Medicine in Sport*): 234 lutas masculinas do UFC (jul–dez/2014), árvores CHAID e análise discriminante. Os indicadores decisivos foram golpes significativos no solo por minuto, precisão de quedas e precisão de golpes.

| Modelo | Acurácia | Sensibilidade | Especificidade |
|---|---|---|---|
| Árvore (bruto) | 71,8% ± 3,6% | 88,9% | 54,7% |
| Árvore (taxa/min) | 76,3% ± 5,1% | 73,5% | 79,1% |
| DFA (bruto) | 71,4% (70,1% CV) | 75,6% | 67,1% |
| DFA (taxa/min) | 71,2% (69,4% CV) | 76,1% | 66,2% |

Regras ilustrativas:
- mais de 4 golpes significativos no solo → 80,4% de vitórias; somando precisão de quedas > 25% → 84,9%;
- mais de 0,850 golpes significativos no solo por minuto → 91,5%; somando mais de 4,190 golpes conectados por minuto → 96,3%.

### Fragilidades do estudo original que motivam este trabalho

1. **Circularidade.** Os indicadores são medidos durante a luta. Em nocautes por *ground and pound*, golpes no solo são o próprio desfecho.
2. **Não independência.** Duas linhas espelhadas por luta (468 linhas para 234 lutas): o mesmo confronto cai no treino e no teste da validação cruzada.
3. **Tipo de desfecho não controlado.** Nocautes de 40 s e decisões de 25 min no mesmo modelo.
4. **Variáveis omitidas** que o UFC Stats fornece: tempo de controle, knockdowns, tentativas de finalização, reversões.
5. Sem estratificação por categoria de peso, número de rounds ou título; sem lutas femininas; amostra de seis meses.

## 2. Literatura relacionada e posicionamento

- **James et al. (2019, *Frontiers in AI*)**: 2.831 lutas masculinas (2000–2015), nMDS e RIPPER. A duração aparece em 3 das 8 regras; o grappling ofensivo segue como fator-chave, influenciado pela duração.
- **Miarka et al. (2016, *JSCR* 30(7); 2018, *JSCR* 32(2))**: comparam frequências de técnicas entre KO/TKO, finalização e decisão. A comparação descritiva entre desfechos **já foi feita**.
- **Latshaw (JudgeAI)**: 1.670 decisões (2011–2020), 83,3% de acurácia fora da amostra para o vencedor do round. **Gift (2021, *J. Sports Analytics*)**: efeito da revisão de critérios de 2017 sobre placares 10-8.

**Contribuição.** Ninguém testou se a **estrutura do modelo classificatório** muda entre estratos, ou seja, se os preditores de James et al. são estáveis ou dirigidos pelo subconjunto de lutas finalizadas. A contribuição é o **teste formal de interação estrato × indicador**, com o tempo de controle como variável nova. Modelar juízes fica como análise exploratória secundária.

## 3. Hipóteses

- **H3.** A vantagem aparente do grappling se dissolve quando a amostra é restrita a lutas que foram à decisão.
- **H3a.** O efeito dos indicadores de grappling (golpes no solo/min, quedas/min, precisão de quedas) é significativamente menor nas decisões do que nas finalizações.
- **H3b.** Nas decisões, o tempo de controle passa a ser o indicador de grappling mais associado à vitória, superando os golpes no solo.
- **H3c.** Nas decisões, indicadores de striking à distância recuperam a importância relativa perdida no modelo agregado.

Os dois resultados são informativos: se o efeito encolhe, o achado original estava inflado pela mecânica da finalização; se persiste, ele passa a dizer o que os juízes premiam.

## 4. Amostra

- **Fonte:** UFC Stats, coleta de 2026-09-21 (`data/ufc-stats-2026-09-21/`).
- **Janela:** 2015-01-01 até a coleta. É posterior à amostra original, tem cobertura completa de tempo de controle e inclui a revisão de regras de 2017.
- **Exclusões:** empates, *no contests*, desqualificações, resultados revertidos (`OVERTURNED`), `CNC`, `OTHER`, lutas sem estatísticas e lutas sem tempo de controle (reportar quantas e testar se a ausência é aleatória).
- **Estratos:** KO/TKO, Finalização, Decisão (`U-DEC`, `S-DEC`, `M-DEC`), sendo Decisão o estrato focal.
- **Tamanho esperado:** cerca de 5.700 lutas na janela, das quais cerca de 2.850 decisões, 1.850 KO/TKO e 1.000 finalizações.

## 5. Unidade de análise

Uma linha por luta. Um dos dois lutadores é sorteado como referência (semente registrada). y = 1 se a referência venceu. Os atributos são diferenças (referência − oponente).

Isso elimina a duplicação espelhada e dá variância real a y. Atributos que não são diferenças (peso, gênero, rounds, ano) não predizem y com referência aleatória: entram só como moderadores, em subamostras.

## 6. Variáveis

Todas normalizadas pela duração da luta (`total_fight_seconds`, em minutos).

| Bloco | Variável |
|---|---|
| Grappling | golpes significativos no solo/min; tempo de controle / duração; quedas/min; precisão de quedas (sem tentativa = categoria própria); tentativas de finalização/min; reversões |
| Striking | golpes significativos à distância/min; no clinch/min; golpes totais/min; precisão de golpes significativos; knockdowns |

Reversões são raras: verificar a distribuição antes de manter (talvez binária). Lutas muito curtas inflam as taxas por minuto: definir e justificar uma duração mínima.

## 7. Análise

1. **Descritiva.** Cohen's *d* com IC 95% por estrato (bootstrap sobre lutas), em forest plot comparável à Figura 1 do original.
2. **Replicação.** Árvores por estrato com mínimo de 10 casos por folha e validação cruzada 10-fold, para comparar com as regras de James et al.
3. **Teste formal de H3.** Regressão logística no conjunto completo com interações estrato × indicador. Erros-padrão agrupados pelos lutadores. O teste é a significância das interações (Wald). Comparar coeficientes "no olho" não é teste.
4. **Comparação de modelos.** AUC, calibração e razão de verossimilhança entre modelos só-striking, só-grappling e completo, dentro de cada estrato.
5. **Secundária (exploratória).** Margem do placar dos juízes nas decisões, sem reivindicar novidade.

**Sensibilidade:** só lutas de 3 rounds; sem lutas de título; 2015–2016 × 2017+; só masculino; com e sem imputação de tempo de controle; outra semente do sorteio.

## 8. Armadilhas antecipadas

- **Viés de seleção.** Condicionar na decisão é condicionar em um evento pós-tratamento: lutas que chegam à decisão são mais equilibradas, e o próprio grappling influencia a chance de chegar lá. A pergunta é descritiva e condicional ("entre as lutas que chegam à decisão, o que distingue vencedores?"), não causal.
- **Restrição de amplitude.** A menor variância dos indicadores nas decisões pode, sozinha, reduzir tamanhos de efeito padronizados. Por isso os coeficientes do teste formal usam a escala da amostra inteira.
- **Múltiplos testes.** Hipóteses fixadas antes de rodar a Seção 8 do notebook.
- **Definição de tempo de controle.** O UFC Stats não publica glossário. Não está claro se inclui controle no clinch contra a grade.
- **Sem *offensive passes*.** Um dos indicadores do Modelo 1 de James et al. não existe no UFC Stats. O tempo de controle cobre a mesma dimensão (domínio posicional) com uma medida contínua.
- **Vazamento.** Ordem A/B pré-2010 lista o vencedor primeiro; cartel e médias de `fighters` incluem lutas futuras.

## 9. Disponibilidade dos dados no UFC Stats

| Variável | Disponível | Origem |
|---|---|---|
| Golpes significativos por posição (distância, clinch, solo) | sim | `fight_stats` |
| Golpes significativos por alvo, golpes totais | sim | `fight_stats` |
| Quedas, tentativas de finalização, reversões, knockdowns | sim | `fight_stats` |
| Tempo de controle | sim (desde 2000) | `fight_stats.ctrl_seconds` |
| Método, round e tempo do fim, formato, categoria, gênero, título | sim | `fights` |
| Juízes e placares totais | sim (desde 2010) | `fights.judges_scores` |
| Duração da luta | calculada pelo scraper | `fights.total_fight_seconds` |
| Precisão de quedas e de golpes | calculada | `landed / attempted` |
| *Offensive passes* | **não** | — |

## Referências

- James, L. P., Robertson, S., Haff, G. G., Beckman, E. M., & Kelly, V. G. (2016). Identifying the performance characteristics of a winning outcome in elite mixed martial arts competition. *Journal of Science and Medicine in Sport*.
- James, L. P. et al. (2019). *Frontiers in Artificial Intelligence*.
- Miarka, B. et al. (2016). *Journal of Strength and Conditioning Research*, 30(7), 1975–1984.
- Miarka, B. et al. (2018). *Journal of Strength and Conditioning Research*, 32(2), 534–544.
- Gift, P. (2021). *Journal of Sports Analytics*.
- Latshaw, N. JudgeAI.

Títulos, volumes e DOIs ainda precisam ser conferidos.
