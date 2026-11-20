# Relatório de validação

Coleta: `2026-09-21` · resultado: **PASSOU**

| tabela | linhas |
| --- | --- |
| events | 789 |
| fights | 8897 |
| fight_stats | 17752 |
| round_stats | 41792 |
| fighters | 4618 |

## integridade

| checagem | status | detalhes |
| --- | --- | --- |
| `check_events_have_html` | ✅ |  |
| `check_fights_cover_cards` | ✅ |  |
| `check_fighters_exist` | ✅ |  |
| `check_unique_pks` | ✅ |  |

## consistência

| checagem | status | detalhes |
| --- | --- | --- |
| `check_landed_le_attempted` | ✅ |  |
| `check_sig_le_total` | ✅ |  |
| `check_sig_breakdown_sums` | ✅ |  |
| `check_round_sums` | ✅ |  |
| `check_rounds_vs_fight` | ✅ |  |
| `check_ctrl_le_duration` | ✅ |  |
| `check_winner_and_result` | ✅ |  |
| `check_decision_flag` | ✅ |  |
| `check_stats_fighters_belong_to_fight` | ✅ |  |
