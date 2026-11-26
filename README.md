# O grappling vence lutas ou convence juízes?

**Reanálise dos indicadores de desempenho no UFC estratificada por método de vitória**

**Disciplina:** SME0828 - Introdução à Ciência de Dados  
**Instituição:** ICMC-USP  
**Docente:** Prof. Francisco A. Rodrigues  
**Alunos:**
- Pedro Henrique de Sousa Prestes (15507819)
- Laura Pazini Medeiros (15468452)
- `<Nome>` (`<nº USP>`)

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

`<Preencher: n por estrato, figuras principais (forest plot de Cohen's d por estrato e ΔAUC do bloco de grappling), resultado dos testes de interação.>`

## 7. Conclusão

`<Preencher: a hipótese foi confirmada ou refutada, principais limitações (viés de seleção ao condicionar na decisão) e próximos passos.>`
