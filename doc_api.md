# OpenF1 API — Documentação

Fonte: https://openf1.org/docs/ (lido em 21/10/2026)

## 1. Visão geral

- API open-source com dados de Fórmula 1: telemetria, tempos de volta, sessões, clima, rádio, etc.
- Formatos de resposta: **JSON** (padrão) ou **CSV** (`csv=true`).
- Base URL: `https://api.openf1.org/v1/<endpoint>`
- Dados **históricos (a partir de 2023)**: grátis, **sem autenticação**.
- Dados em **tempo real**: exige assinatura paga.
- Projeto não oficial, sem ligação com a Formula 1 (F1 e marcas relacionadas pertencem à Formula One Licensing B.V.).
- Todos os exemplos da documentação usam requisições `GET` com filtros na query string.

## 2. Chaves principais

| Chave | O que identifica |
| --- | --- |
| `meeting_key` | Um fim de semana de GP ou de testes (contém várias sessões) |
| `session_key` | Uma sessão (treino, classificação, sprint, corrida) |
| `driver_number` | Número do piloto na temporada |

- `meeting_key=latest` e `session_key=latest` retornam a reunião/sessão mais recente ou em andamento.

### Fluxo típico

1. `/meetings?year=2026&country_name=Singapore` → obtém o `meeting_key`
2. `/sessions?meeting_key=...&session_name=Race` → obtém o `session_key`
3. Demais endpoints com `session_key` (+ `driver_number`, `lap_number`, etc.)

## 3. Endpoints

| Endpoint | Descrição | Campos principais | Observações |
| --- | --- | --- | --- |
| `/car_data` | Dados do carro, ~3,7 Hz | `date`, `driver_number`, `speed` (km/h), `rpm`, `n_gear` (0–8), `throttle` (%), `brake` (0 ou 100), `drs` | DRS: 0/1 = off, 8 = elegível, 10/12/14 = on |
| `/championship_drivers` (beta) | Classificação de pilotos | `driver_number`, `points_start`, `points_current`, `position_start`, `position_current` | Só sessões de corrida |
| `/championship_teams` (beta) | Classificação de equipes | `team_name`, `points_start`, `points_current`, `position_start`, `position_current` | Só sessões de corrida |
| `/drivers` | Pilotos de uma sessão | `driver_number`, `full_name`, `first_name`, `last_name`, `broadcast_name`, `name_acronym`, `team_name`, `team_colour` (hex RRGGBB), `headshot_url` | `country_code` está deprecated (removido no fim de 2026) |
| `/intervals` | Gap para o líder e para o carro da frente | `date`, `driver_number`, `gap_to_leader`, `interval` | Só em corridas, atualiza a cada ~4 s; `null` para o líder; `+1 LAP` se retardatário |
| `/laps` | Dados por volta | `lap_number`, `date_start`, `lap_duration`, `duration_sector_1/2/3`, `i1_speed`, `i2_speed`, `st_speed`, `is_pit_out_lap`, `segments_sector_1/2/3` | `segments_*` não existem em corridas; 2048 = amarelo, 2049 = verde, 2051 = roxo, 2064 = pit lane |
| `/location` | Posição 3D aproximada do carro, ~3,7 Hz | `date`, `x`, `y`, `z` | Origem (0,0,0) arbitrária; sem info de posição lateral na pista |
| `/meetings` | GPs e testes | `meeting_key`, `meeting_name`, `meeting_official_name`, `year`, `circuit_key`, `circuit_short_name`, `circuit_type`, `circuit_image`, `circuit_info_url`, `country_name`, `country_code`, `country_flag`, `location`, `date_start`, `date_end`, `gmt_offset`, `is_cancelled` | Atualizado diariamente à meia-noite UTC |
| `/overtakes` | Ultrapassagens | `date`, `overtaking_driver_number`, `overtaken_driver_number`, `position` | Só corridas; pode estar incompleto; inclui trocas por pit stop/penalidade |
| `/pit` | Passagens pelo pit lane | `date`, `driver_number`, `lap_number`, `lane_duration`, `stop_duration` | `stop_duration` só a partir do GP dos EUA 2024; `pit_duration` deprecated (use `lane_duration`) |
| `/position` | Posição do piloto ao longo da sessão | `date`, `driver_number`, `position` | Inclui posição inicial e mudanças |
| `/race_control` | Bandeiras, safety car, incidentes, status da sessão | `category`, `date`, `driver_number`, `flag`, `lap_number`, `message`, `qualifying_phase`, `scope`, `sector` | Ex.: `flag=BLACK AND WHITE`, `category=SafetyCar` |
| `/sessions` | Sessões (treinos, quali, sprint, corrida) | `session_key`, `session_name`, `session_type`, `meeting_key`, `year`, `country_name`, `circuit_short_name`, `date_start`, `date_end`, `gmt_offset`, `is_cancelled` | Atualizado diariamente à meia-noite UTC |
| `/session_result` | Resultado final da sessão | `driver_number`, `position`, `duration`, `gap_to_leader`, `number_of_laps`, `dnf`, `dns`, `dsq` | Em qualifying, `duration` e `gap_to_leader` são arrays (Q1, Q2, Q3); disponível minutos após o resultado oficial |
| `/starting_grid` | Grid de largada | `position`, `driver_number`, `lap_duration` | Disponível minutos após o resultado oficial |
| `/stints` | Períodos contínuos de pilotagem | `stint_number`, `lap_start`, `lap_end`, `compound` (SOFT/MEDIUM/HARD...), `tyre_age_at_start` | — |
| `/team_radio` | Rádios piloto–equipe | `date`, `driver_number`, `recording_url` | Seleção limitada; cobertura caiu bastante em 2026 (limitação da F1) |
| `/weather` | Clima na pista, atualizado a cada minuto | `air_temperature`, `track_temperature` (°C), `humidity` (%), `pressure` (mbar), `rainfall`, `wind_direction` (°), `wind_speed` (m/s) | — |

Campos presentes em quase todos os endpoints: `meeting_key`, `session_key`, `driver_number`, `date` (UTC, ISO 8601).

## 4. Filtros

- Qualquer atributo pode virar filtro na URL (**exceto arrays**).
- Operadores: `=`, `>`, `>=`, `<`, `<=`.
- Repetir o parâmetro filtra por vários valores (ex.: `driver_number=4&driver_number=81`).
- Filtro por tempo: `date`, `date_start`, `date_end` aceitam vários formatos (os aceitos por `dateutil.parser.parse`), como `2021-09-10`, `2021-09-10T14:30:20+00:00`, `09/10/2021`, `10 September 2021`.

Exemplos:

```
# Voltas de saída do pit do piloto 55 com duração >= 120 s
https://api.openf1.org/v1/laps?session_key=9222&driver_number=55&is_pit_out_lap=true&lap_duration>=120

# Sessões de setembro/2023
https://api.openf1.org/v1/sessions?date_start>=2023-09-01&date_end<=2023-09-30

# Car data do piloto 55 com velocidade >= 315 km/h
https://api.openf1.org/v1/car_data?driver_number=55&session_key=9159&speed>=315
```

## 5. Formato CSV

Adicione `csv=true` à URL:

```
https://api.openf1.org/v1/sessions?year=2023&csv=true
```

## 6. Exemplo em Python

```python
from urllib.parse import quote
import requests
import pandas as pd

BASE = "https://api.openf1.org/v1"


def get(endpoint: str, *filters: str) -> pd.DataFrame:
    """filters: strings como 'session_key=9159', 'speed>=315', 'driver_number=55'"""
    query = "&".join(quote(f, safe="=<>:") for f in filters)
    r = requests.get(f"{BASE}/{endpoint}?{query}", timeout=30)
    r.raise_for_status()
    return pd.DataFrame(r.json())


meetings = get("meetings", "year=2026", "country_name=Singapore")
sessions = get("sessions", f"meeting_key={meetings.loc[0, 'meeting_key']}")
print(sessions[["session_key", "session_name", "date_start"]])
```

## 7. Pontos de atenção

- Valores com espaço precisam ser codificados na URL (`BLACK AND WHITE` → `BLACK%20AND%20WHITE`); o `+` do fuso horário vira `%2B`.
- Alguns dados só existem em certos tipos de sessão (intervals, overtakes e championship → só corrida).
- Campos deprecated: `country_code` (em `/drivers`) e `pit_duration` (em `/pit`) serão removidos no fim de 2026.
- `team_radio` tem cobertura muito reduzida em 2026.
- Para tempo real há autenticação/assinatura: https://openf1.org/auth.html

## 8. Links úteis

- Documentação: https://openf1.org/docs/
- Código-fonte: https://github.com/br-g/openf1
- Dúvidas gerais: https://github.com/br-g/openf1/discussions
- Bugs: https://github.com/br-g/openf1/issues
- Tutorial de dashboard com Python: https://github.com/bordanattila/OpenF1_tutorial