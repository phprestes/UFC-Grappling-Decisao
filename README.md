# O grappling vence lutas ou convence juízes?

**Reanálise dos indicadores de desempenho no UFC estratificada por método de vitória**

**Disciplina:** SME0828 - Introdução à Ciência de Dados  
**Instituição:** ICMC-USP  
**Docente:** Prof. Francisco A. Rodrigues  
**Alunos:**
- Pedro Henrique de Sousa Prestes (15507819)
- Laura Pazini Medeiros (15468452)
- Pedro Lunkes Villela (15484287)
- Juan Henriques Passos (15464826)

---

## 1. Motivação

James et al. (2016) concluíram que o grappling (golpes no solo e precisão de quedas) é o que mais distingue vencedores de perdedores no UFC. A amostra deles, porém, mistura nocautes, finalizações e decisões. Em um nocaute por *ground and pound*, os golpes no solo não predizem o resultado: eles são o resultado.

Este projeto pergunta se a vantagem do grappling se mantém quando ninguém termina a luta, ou seja, nas lutas que vão à decisão dos juízes.

## 2. Pergunta e hipóteses

**Pergunta.** A relação entre grappling e vitória é diferente nas lutas terminadas por nocaute ou finalização e nas lutas decididas pelos juízes?

| Hipótese | Enunciado | Resultado |
|---|---|---|
| **H3** (principal) | A vantagem aparente do grappling se dissolve quando a amostra é restrita a lutas que foram à decisão | **Refutada na forma forte:** o grappling encolhe nas decisões, mas não se dissolve |
| H3a | O efeito dos indicadores de grappling é menor nas decisões do que nos nocautes e finalizações | Não confirmada na forma prevista: os efeitos mudam entre estratos, mas sem sinal uniforme; o bloco inteiro contribui menos nas decisões |
| H3b | Nas decisões, o tempo de controle é o indicador de grappling mais ligado à vitória, à frente dos golpes no solo | Inconclusiva: o sinal depende da escala e da duração |
| H3c | Nas decisões, os golpes à distância ganham a importância que perdem no modelo com todas as lutas | Apoio parcial: o bloco em pé acrescenta mais nas decisões, mas a razão entre os coeficientes de distância e de chão não muda |

O desenho, as decisões metodológicas, os desvios do plano e a literatura estão em [`docs/DESENHO.md`](docs/DESENHO.md).

## 3. Dados

[UFC Stats](http://ufcstats.com). Retrato do site em **2026-09-21**, em `data/ufc-stats-2026-09-21/` (Parquet).

> **Observação:** os dados foram coletados do UFC Stats por um *scraper* desenvolvido pelo grupo, [`ufc-stats-scraper`](https://github.com/JuanHPassos/ufc-stats-scraper). O repositório está privado no momento e será tornado público depois.

| tabela | linhas | granularidade |
|---|---|---|
| `events` | 789 | um evento (1994–2026) |
| `fights` | 8.897 | uma luta |
| `fight_stats` | 17.752 | um lutador por luta |
| `round_stats` | 41.792 | um lutador por round |
| `fighters` | 4.618 | um lutador |

Leia `data/ufc-stats-2026-09-21/LEIA-ME.md` antes de analisar: a ordem A/B e as estatísticas de carreira de `fighters` causam vazamento de dados se usadas sem cuidado.

**Amostra analítica** (Seção 3 do notebook):

| Etapa | Lutas |
|---|---|
| Lutas de 2015-01-01 a 2026-09-19 | 5.849 |
| Sem empates (47), *no contests* (59) e desqualificações (13) | 5.730 |
| Com duração de pelo menos 60 s (saem 178 KO/TKO e 33 finalizações) | **5.519** |

São 2.837 decisões, 1.687 KO/TKO e 995 finalizações. O treino tem 4.139 lutas e o teste tem 1.380, com divisão estratificada por estrato e y.

## 4. Método em resumo

- **Unidade de análise:** uma linha por luta. Em cada estrato, metade das lutas recebe o vencedor como lutador de referência; y = 1 se a referência venceu. Os atributos são diferenças (referência − oponente) de taxas por minuto.
- **Atributos:** 7 de grappling (`ground_pm`, `ctrl_share`, `td_pm`, `td_acc`, `td_sem_tentativa`, `sub_pm`, `rev`) e 5 de striking (`dist_pm`, `clinch_pm`, `total_pm`, `sig_acc`, `kd`). Na comparação de blocos, o bloco em pé é `dist_pm`, `clinch_pm` e `pe_acc`, porque `total_pm`, `sig_acc` e `kd` misturam golpes no solo ou são o próprio KO.
- **Teste formal (Seção 8):**
  - logística com interação estrato × indicador;
  - erros-padrão diádicos (pares de lutas com um lutador em comum);
  - Wald para H3a e H3b;
  - razão β_dist / β_ground para H3c;
  - correção de Holm nos três testes.
- **Modelos (Seções 9–12):** logística (L1, L2, Elastic Net), árvore, floresta aleatória e *gradient boosting*, por estrato e no agregado. A seleção usa validação cruzada no treino, e o teste é usado uma única vez, na Seção 12.
- **Contribuição do grappling (Seção 10.2, exploratória):** é a fração do erro do modelo só com golpes em pé que o grappling elimina, medida em validação cruzada 5 × 5 com o *t* corrigido de Nadeau e Bengio.
- **Sensibilidade (Seção 13):** 14 variantes de recorte, reconstrução e especificação.

## 5. Resultados

**Contribuição do grappling** (logística, validação cruzada 5 × 5 no treino; [`figures/fig17_delta_auc_blocos.png`](figures/fig17_delta_auc_blocos.png)):

| | KO/TKO | Finalização | Decisão | Agregado |
|---|---|---|---|---|
| Fração do erro eliminada pelo grappling | 0,80 | 0,97 | **0,58** [0,53; 0,63] | 0,70 |
| Idem, sem `sub_pm` | 0,80 | 0,53 | **0,57** | 0,58 |
| ΔAUC do bloco em pé | 0,022 | 0,001 | **0,096** | 0,045 |

**Testes formais** ([`figures/fig12_h3_coeficientes_estrato.png`](figures/fig12_h3_coeficientes_estrato.png)):
- **H3a:** χ²(14) = 172,4, p < 0,001. Os efeitos mudam entre estratos, mas com sinais mistos: o chão pesa um pouco mais em KO/TKO, e o controle e as quedas pesam mais nas decisões.
- **H3b:** nas decisões, β(controle) − β(chão) = −1,59 [−2,29; −0,88] na escala por desvio-padrão. Com a escala por quantis, o valor é 0,04 [−0,28; 0,36].
- **H3c:** β_dist / β_ground vale 0,64 nas decisões e 0,60 no agregado. A diferença é 0,04 [−0,15; 0,25], com p (Holm) = 0,60.

**Sensibilidade** ([`figures/fig27_sensibilidade.png`](figures/fig27_sensibilidade.png)): nas 14 variantes, a fração do erro nas decisões fica entre 0,56 e 0,64.

**Modelos no teste:** AUC de 0,996 em KO/TKO, 0,986 em Finalização, 0,925 em Decisão e 0,953 no agregado.

**Placar dos juízes (Seção 14, exploratória):** mantidos os demais indicadores, o striking à distância e o controle pesam igual na margem dos cartões (Wald, p = 0,55).

## 6. Conclusão

A hipótese foi refutada na forma forte: o grappling encolhe nas decisões, mas não se dissolve. A parte que some é a da mecânica do fim da luta. Sem as tentativas de finalização, as decisões empatam com o modelo agregado. KO/TKO, em que o grappling inclui o *ground and pound*, fica acima. Nas decisões, o grappling diz mais sobre o que os juízes premiam do que sobre o que vence lutas.

**Limitações:**
- **Viés de seleção:** condicionar na decisão é condicionar em um evento posterior ao tratamento, então os resultados são condicionais, não causais.
- **Circularidade dos juízes:** eles pontuam com as mesmas estatísticas usadas como preditoras.
- **Duração das lutas:** taxas por minuto variam menos em lutas longas.
- **Escala da logit:** coeficientes de estratos com ajustes diferentes não se comparam diretamente.
- **Medida exploratória:** a fração do erro foi definida depois de ver os dados.

**Próximos passos:**
- análise round a round com `round_stats`;
- métricas fixadas antes da análise, com margem de equivalência;
- modelo multinomial do desfecho;
- validação cruzada aninhada.

## 7. Estrutura do repositório

```
UFC-Grappling-Decisao/
├── UFC_Grappling_Decisao.ipynb   notebook-relatório (entrega)
├── data/ufc-stats-2026-09-21/    tabelas Parquet, dicionário e relatórios de qualidade
├── docs/
│   ├── DESENHO.md                desenho da pesquisa, desvios do plano e referências
│   └── enunciado-projeto-final.pdf
├── figures/                      28 figuras geradas pelo notebook (fig01 a fig28)
├── results/                      datasets intermediários e caches (fora do git)
└── requirements.txt
```

Roteiro do notebook e passos do enunciado:

| Seções | Conteúdo | Passo do enunciado |
|---|---|---|
| 1–2 | Introdução, hipóteses e obtenção dos dados | Pergunta, hipótese e coleta |
| 3–4 | Preparação, limpeza, unidade de análise e atributos | 1, 7 |
| 5–6 | Estatística descritiva e análise exploratória | 2, 3 |
| 7–8 | Tipo de problema e teste formal da hipótese | 4, 5, 8 |
| 9–11 | Modelos, comparação e melhorias (PCA, seleção, transformações, validação temporal) | 5, 6, 7 |
| 12–14 | Resultados no teste, importância, sensibilidade e placar dos juízes | 8 |
| 15–16 | Discussão e conclusão | 9, 10 |

## 8. Como reproduzir

Requer Python 3.13 e as versões de `requirements.txt`.

```bash
python -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/jupyter lab UFC_Grappling_Decisao.ipynb
```

Para executar tudo de uma vez:

```bash
.venv/bin/jupyter nbconvert --to notebook --execute --inplace UFC_Grappling_Decisao.ipynb
```

- **Diretório:** execute a partir da raiz do repositório, porque os caminhos são relativos.
- **Tempo:** a execução completa leva cerca de 5 minutos numa máquina de 16 *threads*. A maior parte vai nas buscas de hiperparâmetros da Seção 9.
- **Caches:** as Seções 9, 10 e 13 gravam caches em `results/`:
  - `sec9_buscas.joblib`;
  - `sec10_cv.parquet`;
  - `sec10_blocos_sem_sobreposicao.parquet`;
  - `sec13_sensibilidade_v2.parquet`.

  Os caches não percebem mudanças no código ou nos dados. Apague-os antes de reexecutar depois de qualquer alteração.
- **Sementes:** todas são fixas (`SEED = 42`), e uma reexecução reproduz os números do notebook.
