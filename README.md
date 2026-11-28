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

Este projeto pergunta se a vantagem do grappling se mantém quando ninguém finaliza a luta.

## 2. Pergunta e hipótese

**Pergunta.** A associação entre indicadores de grappling e vitória difere entre lutas finalizadas (KO/TKO, finalização) e lutas decididas pelos juízes?

**Hipótese (H3).** A vantagem aparente do grappling se dissolve quando a amostra é restrita a lutas que foram à decisão.

| | Hipótese derivada |
|---|---|
| H3a | O efeito do grappling sobre a vitória é menor nas decisões do que nas finalizações |
| H3b | Nas decisões, o tempo de controle supera os golpes no solo como indicador de grappling |
| H3c | Nas decisões, o striking à distância recupera importância relativa |

O desenho completo, as decisões metodológicas e a literatura estão em [`docs/DESENHO.md`](docs/DESENHO.md).

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

## 4. Estrutura do repositório

```
UFC-Grappling-Decisao/
├── UFC_Grappling_Decisao.ipynb   notebook-relatório (entrega)
├── data/ufc-stats-2026-09-21/    tabelas Parquet, dicionário e relatórios de qualidade
├── docs/
│   ├── DESENHO.md                desenho da pesquisa, decisões e referências
│   └── enunciado-projeto-final.pdf
├── figures/                      figuras geradas pelo notebook
├── results/                      tabelas e datasets intermediários gerados pelo notebook
└── requirements.txt
```

## 5. Como reproduzir

```bash
python -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/jupyter lab UFC_Grappling_Decisao.ipynb
```

O notebook deve ser executado a partir da raiz do repositório (os caminhos são relativos).

## 6. Resultados

Amostra analítica: **5.519 lutas** (2.837 decisões, 1.687 KO/TKO, 995 finalizações), com 4.139 no treino e 1.380 no teste.

**Contribuição do grappling** (logística, validação cruzada 5 × 5 no treino). É a fração do erro do modelo só com golpes em pé (`dist_pm`, `clinch_pm`, `pe_acc`) que o bloco de grappling elimina. É uma medida exploratória.

| | KO/TKO | Finalização | Decisão | Agregado |
|---|---|---|---|---|
| Fração do erro eliminada pelo grappling | 0,80 | 0,97 | **0,58** [0,53; 0,63] | 0,70 |
| Idem, sem `sub_pm` | 0,80 | 0,53 | **0,57** | 0,58 |

**Testes formais** (logística com interação estrato × indicador, erros-padrão diádicos, Holm):

- **H3a:** os efeitos de grappling mudam entre estratos (χ²(14) = 172,4, p < 0,001), mas sem o padrão uniforme previsto.
- **H3b:** inconclusiva. O sinal de β(controle) − β(chão) depende da escala.
- **H3c:** a razão β_dist / β_chão não muda (0,64 contra 0,60; IC da diferença: −0,15 a 0,25). Mesmo assim, o bloco em pé acrescenta mais nas decisões (ΔAUC de 0,096, contra 0,045 no agregado).

**Sensibilidade:** nas 14 variantes, a fração do erro nas decisões fica entre 0,56 e 0,64.

## 7. Conclusão

**A hipótese foi refutada na forma forte.** O grappling encolhe nas decisões, mas não se dissolve. A parte que some é a da mecânica do fim da luta: sem as tentativas de finalização, as decisões empatam com o modelo agregado. Nas decisões, o grappling diz mais sobre o que os juízes premiam do que sobre o que vence lutas.

As principais limitações são três:

- **Viés de seleção:** condicionar na decisão é condicionar em um evento posterior ao tratamento.
- **Circularidade dos juízes:** eles pontuam com as mesmas estatísticas usadas como preditoras.
- **Medida exploratória:** a fração do erro foi definida depois de ver os dados.

**Próximos passos:**

- análise round a round com `round_stats`;
- métricas fixadas antes da análise, com margem de equivalência;
- modelo multinomial do desfecho.
