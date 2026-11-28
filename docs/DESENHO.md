# Desenho da pesquisa

Este documento registra o desenho do projeto e as decisões tomadas antes de olhar os resultados. O notebook segue este roteiro. As mudanças feitas depois estão na Seção 9 (desvios do plano), e os resultados, no notebook e no `README.md`.

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
- **Exclusões:** empates, *no contests*, desqualificações, resultados revertidos (`OVERTURNED`), `CNC`, `OTHER`, lutas sem estatísticas e lutas sem tempo de controle. Na janela saíram 47 empates, 59 *no contests* e 13 desqualificações; nenhuma luta ficou sem estatísticas ou sem tempo de controle.
- **Duração mínima:** 60 s. Lutas mais curtas inflam as taxas por minuto; saíram 178 KO/TKO e 33 finalizações. A Seção 13 testa 30 s e 120 s.
- **Estratos:** KO/TKO, Finalização, Decisão (`U-DEC`, `S-DEC`, `M-DEC`), sendo Decisão o estrato focal. KO/TKO inclui as interrupções médicas (*doctor's stoppage*, 40 lutas) e Decisão inclui 7 decisões técnicas (faltas acidentais, 11 a 15 min).
- **Tamanho final:** 5.519 lutas (2.837 decisões, 1.687 KO/TKO e 995 finalizações), das 5.849 da janela. Treino de 4.139 e teste de 1.380, estratificados por estrato × y.

## 5. Unidade de análise

Uma linha por luta. Um dos dois lutadores é sorteado como referência (semente 42); em cada estrato, exatamente metade das lutas recebe o vencedor. y = 1 se a referência venceu. Os atributos são diferenças (referência − oponente).

Isso elimina a duplicação espelhada e dá variância real a y. Atributos que não são diferenças (peso, gênero, rounds, ano) não predizem y com referência aleatória: entram só como moderadores, em subamostras.

## 6. Variáveis

Todas normalizadas pela duração da luta (`total_fight_seconds`, em minutos).

| Bloco | Variável |
|---|---|
| Grappling | golpes significativos no solo/min; tempo de controle / duração; quedas/min; precisão de quedas (sem tentativa = categoria própria); tentativas de finalização/min; reversões |
| Striking | golpes significativos à distância/min; no clinch/min; golpes totais/min; precisão de golpes significativos; knockdowns |

Reversões são raras (89,5% dos lutadores têm zero), mas ficam como contagem. Lutas com menos de 60 s saem da amostra (Seção 4).

**Blocos sem sobreposição (comparação de blocos, Seção 10.2 do notebook).** `total_pm` e `sig_acc` contam também golpes no solo, e `kd` é o mecanismo do KO. Por isso eles entram nos modelos com os 12 atributos, mas ficam fora dos dois blocos: o bloco *em pé* é `dist_pm`, `clinch_pm` e `pe_acc` (precisão à distância e no clinch), e o bloco de grappling é o da tabela acima.

## 7. Análise

1. **Descritiva.** Cohen's *d* com IC 95% por estrato (bootstrap sobre lutas), em forest plot comparável à Figura 1 do original.
2. **Replicação.** Árvores por estrato com mínimo de 10 casos por folha e validação cruzada 10-fold, para comparar com as regras de James et al.
3. **Teste formal de H3.** Regressão logística no conjunto completo com interações estrato × indicador. Erros-padrão diádicos (Fafchamps e Gubert, 2007): somam os escores de todo par de lutas com um lutador em comum, qualquer que seja o papel dele (referência ou oponente). O teste é a significância das interações (Wald). Comparar coeficientes "no olho" não é teste.
   - **H3c** é testada pela razão β_dist / β_ground (decisões contra agregado). Coeficientes logit de amostras diferentes não se comparam em nível, porque todos crescem quando o ajuste melhora (não colapsabilidade; Mood, 2010). A razão cancela esse fator comum.
   - Os três testes confirmatórios (H3a, H3b, H3c) são corrigidos por Holm.
4. **Comparação de modelos.** AUC e calibração. A contribuição do grappling é o ganho de AUC ao somar o bloco de grappling ao bloco em pé, dentro de cada estrato e no agregado.
5. **Secundária (exploratória).** Margem do placar dos juízes nas decisões, sem reivindicar novidade.

**Sensibilidade (14 variantes, Seção 13 do notebook):**
- **Recorte:** só lutas de 3 rounds; só lutas com 10 min ou mais; sem lutas de título; 2015–2016 e 2017+; só masculino e só feminino.
- **Reconstrução:** janela 2000+; duração mínima de 30 s e de 120 s; outra semente do sorteio.
- **Especificação:** sem `sub_pm`; escala por quantis.

## 8. Armadilhas antecipadas

- **Viés de seleção.** Condicionar na decisão é condicionar em um evento pós-tratamento: lutas que chegam à decisão são mais equilibradas, e o próprio grappling influencia a chance de chegar lá. A pergunta é descritiva e condicional ("entre as lutas que chegam à decisão, o que distingue vencedores?"), não causal.
- **Restrição de amplitude.** A menor variância dos indicadores nas decisões pode, sozinha, reduzir tamanhos de efeito padronizados. Padronizar pela amostra inteira só põe os coeficientes na mesma unidade; não corrige a restrição. Parte da menor variância vem também da duração: taxas por minuto variam menos em lutas longas, com ou sem seleção. A Seção 13 testa um recorte só com lutas de 10 min ou mais.
- **Circularidade dos juízes.** Nas decisões, a circularidade mecânica sai, mas entra outra: os juízes pontuam com base nos mesmos golpes e no mesmo controle usados como preditores. Nas decisões, a pergunta passa a ser o que os juízes premiam.
- **Múltiplos testes.** Hipóteses fixadas antes de rodar a Seção 8 do notebook; os três testes confirmatórios são corrigidos por Holm. As variantes de sensibilidade são descritivas.
- **Definição de tempo de controle.** O UFC Stats não publica glossário. Não está claro se inclui controle no clinch contra a grade.
- **Sem *offensive passes*.** Um dos indicadores do Modelo 1 de James et al. não existe no UFC Stats. O tempo de controle cobre a mesma dimensão (domínio posicional) com uma medida contínua.
- **Vazamento.** Ordem A/B pré-2010 lista o vencedor primeiro; cartel e médias de `fighters` incluem lutas futuras.

## 9. Desvios do plano

Registrados depois da revisão independente do notebook:

| Plano | O que foi feito | Por quê |
|---|---|---|
| Árvores com validação 10-fold | 5-fold (Seção 9) e 5 × 5 repetida (Seção 10) | Mesmo protocolo para todos os modelos |
| Razão de verossimilhança entre blocos | ΔAUC e fração do erro eliminada por bloco, na validação cruzada | Compara modelos lineares e não lineares na mesma escala |
| Erros-padrão agrupados pelos lutadores | Erros-padrão diádicos | O agrupamento por papel separava lutas do mesmo atleta |
| Fração do erro eliminada | Métrica criada depois de ver os dados | Exploratória; o teste formal de H3a é o Wald das interações |
| Sensibilidade com e sem imputação de controle | Não feita | Nenhuma luta da janela tem controle ausente |
| Seis variantes de sensibilidade | Catorze | Somadas: só feminino, só lutas com 10+ min, janela 2000+, duração mínima de 30 s e 120 s, sem `sub_pm`, escala por quantis |

## 10. Disponibilidade dos dados no UFC Stats

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

- Fafchamps, M., & Gubert, F. (2007). The formation of risk sharing networks. *Journal of Development Economics*, 83(2), 326–350.
- Gift, P. (2021). The impact of new judging criteria on 10-8 scores in MMA. *Journal of Sports Analytics*, 7(4), 235–246. https://doi.org/10.3233/JSA-200478
- James, L. P., Robertson, S., Haff, G. G., Beckman, E. M., & Kelly, V. G. (2016). Identifying the performance characteristics of a winning outcome in elite mixed martial arts competition. *Journal of Science and Medicine in Sport*, 20(3), 296–301. https://doi.org/10.1016/j.jsams.2016.08.001. Publicado online em 2016; 234 lutas do UFC de julho a dezembro de 2014.
- James, L. P., Sweeting, A. J., Kelly, V. G., & Robertson, S. (2019). Longitudinal analysis of tactical strategy in the men's division of the Ultimate Fighting Championship. *Frontiers in Artificial Intelligence*, 2, 29. https://doi.org/10.3389/frai.2019.00029. Usa 2.831 lutas masculinas com vencedor, de 2000 a 2015; nMDS e RIPPER.
- Latshaw, N. JudgeAI. Análise independente, sem revisão por pares. A URL original do blog não foi confirmada. Os resultados são descritos em https://agentmma.com/mma-lab/ufc-round-judging-criteria, que cita 1.670 decisões de 2011 a 2020 e 5.238 rounds. Conferir antes da entrega.
- Miarka, B., Brito, C. J., Moreira, D. G., & Amtmann, J. (2018). Differences by ending rounds and other rounds in time-motion analysis of mixed martial arts: Implications for assessment and training. *Journal of Strength and Conditioning Research*, 32(2), 534–544.
- Miarka, B., Vecchio, F. B. D., Camey, S., & Amtmann, J. A. (2016). Comparisons: Technical-tactical and time-motion analysis of mixed martial arts by outcomes. *Journal of Strength and Conditioning Research*, 30(7), 1975–1984.
- Mood, C. (2010). Logistic regression: Why we cannot do what we think we can do, and what we can do about it. *European Sociological Review*, 26(1), 67–82.
- Nadeau, C., & Bengio, Y. (2003). Inference for the generalization error. *Machine Learning*, 52(3), 239–281.
- Dados: UFC Stats (http://ufcstats.com), via `ufc-stats-scraper` (https://github.com/JuanHPassos/ufc-stats-scraper), coleta de 2026-09-21.

As referências ainda precisam ser conferidas nas fontes antes da entrega.
