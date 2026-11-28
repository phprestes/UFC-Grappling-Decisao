# Dados UFC Stats — coleta de 2026-09-21

Dados extraídos do [UFC Stats](http://ufcstats.com) por
[ufc-stats-scraper](https://github.com/JuanHPassos/ufc-stats-scraper) (o código está no git;
aqui estão só os dados). Tudo é um retrato do site em **2026-09-21** — recoletar depois dá
números diferentes, então a análise deve fixar esta coleta.

## O que tem aqui

- `*.parquet` — as 5 tabelas, com tipos já corretos (inteiros nulos, datas, booleanos);
  carregue com `pd.read_parquet("fights.parquet")`. A coleta original também gera CSV e
  SQLite com o mesmo conteúdo, mas neste repositório só entram os arquivos Parquet.
- `relatorios/validation_report.md` — as 13 checagens de qualidade. **Todas passaram.**
- `relatorios/coverage_report.md` — a partir de que ano cada campo é confiável, % de nulos
  por coluna e ano, lutas por método e ano.
- `DICIONARIO-DE-DADOS.md` — descrição de todas as colunas das 5 tabelas e os avisos
  importantes para a análise (é o README do repositório).

## Números

| tabela | linhas | o que é |
| --- | --- | --- |
| `events` | 789 | um evento por linha (1994–2026) |
| `fights` | 8 897 | uma luta por linha |
| `fight_stats` | 17 752 | um lutador por luta (totais) |
| `round_stats` | 41 792 | um lutador por round |
| `fighters` | 4 618 | um lutador por linha |

## Antes de começar a análise, leia isto

1. **Não use `wins/losses/slpm/td_avg`… de `fighters` como feature pré-luta.** São o cartel
   e as estatísticas de carreira **na data da coleta**, e incluem lutas posteriores a
   qualquer luta específica — usar isso é vazamento de dados. Features pré-luta saem de
   `fights`/`fight_stats`, filtrando por data.
2. **A partir de que ano cada campo é confiável** (detalhe em `coverage_report.md`):
   estatísticas por round e alvo/posição desde **1999**, tempo de controle desde **2000**,
   placares dos juízes e **ordem A/B** desde **2010**. Para a hipótese do grappling
   (controle e quedas), o recorte seguro é **2000 em diante**: 8 673 das 8 897 lutas
   (97,5 %); se a análise usar a ordem A/B ou placares, o recorte é **2010** (7 245 lutas).
3. **Nulo não é zero.** Lutas antigas sem estatísticas simplesmente não têm linha em
   `fight_stats`. Zero significa que aconteceu zero vezes.
4. **`fighter_a`/`fighter_b` só refletem os cantos a partir de 2010.** Até 2009 o site
   lista o **vencedor primeiro**: `winner_id == fighter_a_id` em 1 250/1 250 lutas de
   1994–2009. De 2010 em diante a taxa cai para 54–62 % (vantagem do canto vermelho).
   **Usar A/B como se fosse canto em lutas pré-2010 é vazamento de dados perfeito** —
   filtre por ano ou simetrize. Veja `pct_winner_is_a` por ano em `coverage_report.md`.
   O scraper não sorteia: qualquer simetrização é decisão da análise.
5. **Percentuais das lutas foram descartados de propósito**: recalcule de
   `landed`/`attempted`, que é mais preciso.

O `DICIONARIO-DE-DADOS.md` tem a lista completa de avisos e a descrição coluna a coluna.
