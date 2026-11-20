# ufc-stats-scraper

Scraper do [UFC Stats](http://ufcstats.com/statistics/events/completed) para um
trabalho de ciência de dados em grupo. Baixa o HTML bruto de todos os eventos,
lutas e lutadores, e gera cinco tabelas relacionais limpas (`events`, `fights`,
`fight_stats`, `round_stats`, `fighters`). O scraper não faz análise.

- Especificação: [.specify/spec.md](.specify/spec.md)
- Plano de execução (partes sequenciais, status e critérios): [.specify/plan.md](.specify/plan.md)

## Instalação

Requer Python 3.12+ e [uv](https://docs.astral.sh/uv/).

```bash
uv sync
uv run pytest -q
```

## Uso

```bash
uv run python -m ufc_scraper --help
uv run python -m ufc_scraper crawl    --date 2026-09-21 [--limit 3] [--mode full|incremental]
uv run python -m ufc_scraper parse    --date 2026-09-21 [--table events]
uv run python -m ufc_scraper validate --date 2026-09-21
uv run python -m ufc_scraper export   --date 2026-09-21 --format parquet
```

A pasta de coleta é `data/raw/<date>/` e a saída processada fica em
`data/processed/<date>/` (ambas ignoradas pelo git). Para retomar uma coleta
interrompida, rode o mesmo comando **com a mesma `--date`**.

## Fluxo completo

```bash
uv run python -m ufc_scraper crawl    --date 2026-09-21                    # ~13 mil páginas, algumas horas
uv run python -m ufc_scraper parse    --date 2026-09-21                    # 5 tabelas em CSV
uv run python -m ufc_scraper validate --date 2026-09-21                    # critérios de aceite (exit 1 se falhar)
uv run python -m ufc_scraper export   --date 2026-09-21 --format parquet   # ou sqlite
```

Saídas em `data/processed/<date>/`: `events.csv`, `fights.csv`, `fight_stats.csv`,
`round_stats.csv`, `fighters.csv`, `validation_report.md`, `coverage_report.md`,
`*.parquet` / `ufc_stats.sqlite`, `scraper.log`.

## Atualização incremental

Para atualizar uma coleta antiga sem baixar tudo de novo:

```bash
uv run python -m ufc_scraper crawl    --mode incremental --date 2026-11-15 --base-date 2026-09-21
uv run python -m ufc_scraper parse    --date 2026-11-15
uv run python -m ufc_scraper validate --date 2026-11-15
uv run python -m ufc_scraper export   --date 2026-11-15 --format parquet
```

O incremental baixa só a lista de eventos, os **eventos novos** (com suas lutas),
os perfis dos **lutadores envolvidos** nesses eventos (o cartel mudou), as 26 listas
A–Z e perfis de lutadores inéditos nelas. Todo o resto é copiado da coleta-base por
hard link (instantâneo, sem duplicar espaço; a base nunca é alterada). O resultado
é uma pasta `data/raw/<nova-date>/` completa, com `manifest.json` dizendo de onde
veio e o que foi baixado. `--base-date` é opcional: o padrão é a coleta mais
recente em `data/raw/`. Também é retomável com a mesma `--date`.

## Modelo de dados

Cinco tabelas relacionais ligadas pelos IDs hexadecimais do site (16 caracteres).
Nomes de colunas em `snake_case`. Toda tabela tem `scraped_at` (data da coleta) e
`source_url` (página de origem). **Nulo nunca é zero**: campo ausente no site é nulo;
zero significa que aconteceu zero vezes.

### `events` — um evento por linha

| coluna | tipo | observação |
| --- | --- | --- |
| `event_id` | texto (PK) | |
| `name` | texto | |
| `date` | data | ISO `YYYY-MM-DD` |
| `location` | texto | cidade, estado, país |

### `fights` — uma luta por linha

| coluna | tipo | observação |
| --- | --- | --- |
| `fight_id` | texto (PK) | |
| `event_id` | texto (FK) | |
| `card_position` | inteiro | 1 = main event (ordem do card) |
| `fighter_a_id`, `fighter_b_id` | texto (FK) | **ordem da página da luta (cantos)**; ver avisos |
| `fighter_order_source` | texto | `fight_page` (normal) ou `card` (página da luta ausente) |
| `winner_id` | texto, nulo | nulo em empate e no contest |
| `result_type` | categórico | `win`, `draw`, `no_contest` |
| `weight_class` | categórico | conjunto fixo (Strawweight … Heavyweight, Catch Weight, Open Weight) |
| `weight_class_raw` | texto | original, ex.: `UFC Interim Heavyweight Title Bout`, `Road to UFC 4 … Tournament Title Bout` |
| `is_title_fight` | booleano | inclui interinos e títulos de torneio (TUF, Road to UFC); use `weight_class_raw` para separar |
| `is_women` | booleano | |
| `bonuses` | JSON (lista) | `fight_of_the_night`, `performance_of_the_night`, `ko_of_the_night`, `submission_of_the_night` |
| `method` | categórico | `KO/TKO`, `SUB`, `U-DEC`, `S-DEC`, `M-DEC`, `DQ`, `OVERTURNED`, `CNC`, `OTHER` |
| `method_raw` | texto | original do card (`U-DEC`, `KO/TKO`…) |
| `method_detail` | texto, nulo | golpe/finalização (`Punches`, `Rear Naked Choke`, `Doctor's Stoppage`) |
| `went_to_decision` | booleano | `method` ∈ {U-DEC, S-DEC, M-DEC} |
| `end_round` | inteiro | |
| `end_time_seconds` | inteiro | tempo no round final |
| `total_fight_seconds` | inteiro | rounds anteriores (pelo formato) + tempo final |
| `scheduled_rounds` | inteiro, nulo | do formato; nulo em `No Time Limit` |
| `time_format_raw` | texto | `3 Rnd (5-5-5)`, `5 Rnd (5-5-5-5-5)`, `1 Rnd + OT (12-3)`… |
| `referee` | texto, nulo | |
| `judges_scores` | JSON (lista), nulo | `[{judge, score_1, score_2, score_a, score_b}]` — ver avisos |
| `details_raw` | texto, nulo | texto original de "Details" (técnica, motivo do NC, placares) |

### `fight_stats` — um lutador por luta (totais) · `round_stats` — um lutador por round

PK: `fight_id, fighter_id` (e `round` em `round_stats`). Lutas sem estatísticas no site
(era antiga) **não têm linha** nestas tabelas.

| coluna | tipo | observação |
| --- | --- | --- |
| `kd` | inteiro | knockdowns |
| `sig_str_landed`, `sig_str_attempted` | inteiro | golpes significativos |
| `total_str_landed`, `total_str_attempted` | inteiro | golpes totais |
| `td_landed`, `td_attempted` | inteiro | quedas |
| `sub_att` | inteiro | tentativas de finalização |
| `rev` | inteiro | reversões |
| `ctrl_seconds` | inteiro, nulo | tempo de controle |
| `head_`, `body_`, `leg_` + `landed`/`attempted` | inteiro, nulo | significativos por alvo; somam `sig_str_*` |
| `distance_`, `clinch_`, `ground_` + `landed`/`attempted` | inteiro, nulo | significativos por posição; somam `sig_str_*` |

Percentuais do site são descartados: recalcule a partir de `landed`/`attempted`.

### `fighters` — um lutador por linha

| coluna | tipo | observação |
| --- | --- | --- |
| `fighter_id` | texto (PK) | |
| `name`, `nickname` | texto | apelido nulo se não há |
| `height_cm`, `weight_kg`, `reach_cm` | decimal, nulo | convertidos de pés/polegadas/libras |
| `stance` | texto, nulo | `Orthodox`, `Southpaw`, `Switch`… |
| `dob` | data, nulo | |
| `wins`, `losses`, `draws`, `nc` | inteiro | **cartel na data da coleta**; `nc` = 0 quando o site não mostra |
| `slpm`, `sapm`, `td_avg`, `sub_avg` | decimal | carreira, na data da coleta |
| `str_acc`, `str_def`, `td_acc`, `td_def` | decimal 0–1 | carreira, na data da coleta |

## Avisos para quem for analisar

Decisões já tomadas (spec §8 e implementação); não reverter sem conversa no grupo.

1. **Cartel e estatísticas de carreira de `fighters` são um retrato da data da coleta**
   e incluem lutas posteriores a qualquer luta específica. Usá-los como feature
   pré-luta é vazamento de dados. Features pré-luta devem ser calculadas a partir de
   `fights`/`fight_stats`, por data.
2. **`fighter_a`/`fighter_b` só refletem os cantos a partir de 2010.** A ordem vem da
   página da luta (o card sempre lista o vencedor primeiro), mas **até 2009 a própria
   página também listava o vencedor primeiro**: `winner_id == fighter_a_id` em
   1 250/1 250 lutas de 1994–2009. De 2010 em diante a taxa cai para 54–62 %, compatível
   com a vantagem do canto vermelho. Tratar A/B como canto em lutas pré-2010 é vazamento
   de dados perfeito — filtre por ano ou simetrize. O scraper **não sorteia**; qualquer
   simetrização é decisão da análise. `coverage_report.md` traz a coluna
   `pct_winner_is_a` por ano.
3. **Placares dos juízes**: o site não os lista na ordem dos lutadores. `score_1`/`score_2`
   são a ordem literal do site; `score_a`/`score_b` são atribuídos ao lutador pelo lado
   favorecido pela maioria dos juízes (= vencedor). Em empates e no contests
   `score_a`/`score_b` são nulos — use `score_1`/`score_2` e `details_raw`.
4. **`is_title_fight` inclui interinos e títulos de torneio** (TUF, Road to UFC). O card
   do site mostra cinturão para todos; só `weight_class_raw` distingue.
5. **Nulo ≠ zero.** Lutas antigas sem estatísticas não têm linha em `fight_stats`;
   `ctrl_seconds` e alvo/posição podem ser nulos em eras intermediárias. O
   `coverage_report.md` diz a partir de que ano cada campo é confiável.
6. **Lutas `OVERTURNED`** têm `result_type = no_contest`, `winner_id` nulo e, em geral,
   os placares originais em `judges_scores`; o motivo fica em `details_raw`.
7. **Eventos futuros** (`/upcoming`) ficam fora; o próximo evento listado na página
   "completed" é ignorado.
8. **Reprodutibilidade**: cada coleta vive em `data/raw/<date>/` e gera
   `data/processed/<date>/`. A análise deve fixar uma data.

## Validação e cobertura

`validate` roda 13 checagens (spec §7) e escreve dois relatórios em `data/processed/<date>/`:

- `validation_report.md` — integridade (todo evento tem HTML; toda luta do card está
  em `fights`; todo `fighter_id` tem perfil; PKs únicas) e consistência
  (`landed ≤ attempted`; `sig_str ≤ total_str`; alvo e posição somam `sig_str`; rounds
  somam totais; `end_round ≤ scheduled_rounds`; `ctrl ≤ duração`; vencedor é A ou B;
  `went_to_decision` ↔ `method`). Qualquer ❌ faz `validate` sair com código 1.
- `coverage_report.md` — por ano: % de lutas com `fight_stats`, `round_stats`,
  `ctrl`, alvo/posição, placares e árbitro; lutas por método; % de nulos por coluna;
  e **"confiável desde"** por campo (primeiro ano a partir do qual todos os seguintes
  têm ≥ 95 % de cobertura). Não bloqueia.

Conferência manual: `uv run python scripts/sample_fights.py --n 10 --seed 42`
sorteia 10 lutas estratificadas por era (antes de 2005, 2005–2015, 2016+) e imprime
todas as colunas para comparar com o site; `--template .specify/manual_check.md`
gera a tabela de registro.

## Estrutura do código

```
src/ufc_scraper/
├── __main__.py      CLI: crawl | parse | validate | export
├── config.py        Settings (delays, timeout, retries, User-Agent, data, modo)
├── paths.py         entidade <-> URL <-> caminho no cache
├── http.py          fetch() com cache em disco, rate limit, retry e desafio anti-bot
├── links.py         extração mínima de IDs para o crawler
├── crawl.py         coleta completa e incremental
├── normalize.py     conversões puras (X of Y, m:ss, medidas, datas, categorias, placares)
├── parse/           HTML salvo -> tabelas (events, fights, fight_stats, round_stats, fighters)
├── validate.py      critérios de aceite + relatório de cobertura
├── export.py        Parquet / SQLite com tipos explícitos
└── sample.py        sorteio para conferência manual
```

Especificação: [.specify/spec.md](.specify/spec.md) · Plano e histórico de decisões:
[.specify/plan.md](.specify/plan.md).
