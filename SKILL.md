---
name: f1-dev-standards
description: Regras e fluxo de desenvolvimento Python (engenharia de dados) do repositório F1 (pacote `f1` em `src/f1`, dados da API OpenF1). Use sempre que for criar ou alterar código, pipelines, testes, dependências, configuração ou documentação deste repositório, ou rodar lint, formatação, testes e cobertura, mesmo que a pessoa não cite Poetry, Ruff, pytest ou MkDocs.
---

# Padrões de desenvolvimento do repositório F1

## 1. Estado atual (verificado no repositório)

- Layout `src`: pacote `src/f1/` e testes em `tests/`. Ambos só têm `__init__.py` vazio; `README.md` vazio.
- `pyproject.toml`: `[project]` (PEP 621) com `name = "f1"`, `requires-python = ">=3.13"`, `dependencies = []`; `[tool.poetry] packages = [{include = "f1", from = "src"}]`; build backend `poetry-core>=2.0.0,<3.0.0`.
- `doc_api.md` (raiz): resumo da API OpenF1. Leia antes de escrever código que a consuma e não invente endpoints ou campos.
- Git: branch `main`, remote `origin` em `github.com/thiagogarciiav/f1-dashboard`, commits convencionais (`docs: ...`).
- Arquivos em CRLF (Windows). Não reescreva finais de linha de arquivos inteiros.
- **Não existem ainda**: `.python-version`, `poetry.lock`, grupos de dependência (dev/docs), configuração de Ruff/pytest, `mkdocs.yml`, `docs/`, `.gitignore`, `.env.example`. As seções abaixo explicam como criar cada um.

## 2. Divergências e como alinhar

| Requisito | Situação atual | Como alinhar |
| --- | --- | --- |
| Layout `src/project` | O pacote é `src/f1` (importa-se `f1`) | Mantenha `f1` e use `src/f1` em cobertura e docs. Se o time exigir outro nome: `git mv src/f1 src/<nome>` e atualize `[tool.poetry] packages`, `--cov` e os imports |
| Ferramentas de dev (Ruff, pytest, cobertura) | Não declaradas | Seções 5 e 6 |
| Documentação MkDocs Material | Sem `mkdocs.yml` e `docs/`; API documentada em `doc_api.md` | Seção 7 |
| Versão local do Python | Sem `.python-version` | Seção 3 |
| Reprodutibilidade | Sem `poetry.lock` | `poetry install` gera; versione o arquivo |
| Segredos e config | Sem `.gitignore` nem `.env.example` | Seção 8 |

## 3. Ambiente (pyenv + Poetry)

Ambiente do autor: Windows com pyenv-win; os comandos funcionam no cmd e no PowerShell.

```
pyenv versions
pyenv install <3.13.x>
pyenv local <3.13.x>
pyenv which python
poetry env use <caminho impresso por "pyenv which python">
poetry install
poetry env info
```

- `pyenv local` grava `.python-version` na raiz; escolha uma versão que atenda `>=3.13` e commite o arquivo.
- `pyenv install` só é necessário se a versão não aparecer em `pyenv versions`.
- Execute tudo dentro do ambiente do Poetry: `poetry run <comando>`.
- **Nunca use `pip install`** (nem `poetry run pip install`). Dependências entram só com `poetry add`.

```
poetry add <pacote>
poetry add --group dev <pacote>
poetry add --group docs <pacote>
poetry remove <pacote>
poetry show --tree
```

- Não edite `poetry.lock` à mão e commite-o junto com o `pyproject.toml`.
- Ferramentas iniciais (ainda não declaradas):

```
poetry add --group dev ruff pytest pytest-cov
poetry add --group docs mkdocs-material
```

- Dependências de runtime ainda não existem. O exemplo de `doc_api.md` usa `requests` e `pandas`; adicione-as com `poetry add requests pandas` quando o código precisar.

## 4. Estrutura e modularidade

Mantenha o layout `src` e separe lógica de negócio de I/O. Estrutura proposta (ainda não existe):

```
src/f1/
  config.py        # leitura de configuração (variáveis de ambiente)
  io/              # leitura e escrita: API OpenF1, arquivos, bancos
  transform/       # lógica pura: entra dado, sai dado, sem I/O
  pipelines/       # orquestração: io -> transform -> io
tests/             # espelha src/f1 (tests/transform/test_laps.py ...)
docs/
```

Regras:

- Funções de `transform/` são puras: sem rede, disco, `print`, relógio (`datetime.now()`) ou variáveis globais. Recebem tudo por parâmetro.
- `io/` não contém regra de negócio: só busca, lê e grava.
- `pipelines/` apenas liga as camadas. Dependências (caminhos, URL, cliente) chegam por parâmetro, o que permite testar sem rede.
- Um módulo, uma responsabilidade. Funções pequenas, nomes claros, sem imports circulares.

Exemplo das três camadas (ilustrativo; exige `pandas`):

```python
# src/f1/transform/laps.py — lógica pura
import pandas as pd

REQUIRED_COLUMNS = {"driver_number", "lap_number", "lap_duration"}


def clean_laps(laps: pd.DataFrame) -> pd.DataFrame:
    missing = REQUIRED_COLUMNS - set(laps.columns)
    if missing:
        raise ValueError(f"Colunas ausentes em laps: {sorted(missing)}")
    return (
        laps.dropna(subset=["lap_duration"])
        .drop_duplicates(subset=["driver_number", "lap_number"])
        .sort_values(["driver_number", "lap_number"])
        .reset_index(drop=True)
    )
```

```python
# src/f1/pipelines/laps.py — orquestração
import logging
from pathlib import Path

import pandas as pd

from f1.io.openf1 import fetch_json  # I/O (a criar)
from f1.transform.laps import clean_laps

logger = logging.getLogger(__name__)


def run(session_key: int, out_dir: Path) -> Path:
    laps = clean_laps(pd.DataFrame(fetch_json("laps", f"session_key={session_key}")))
    out_dir.mkdir(parents=True, exist_ok=True)
    target = out_dir / f"laps_session_{session_key}.csv"
    tmp = target.with_name(target.name + ".tmp")
    laps.to_csv(tmp, index=False)
    tmp.replace(target)  # escrita atômica; reexecutar sobrescreve, sem duplicar
    logger.info("laps gravadas: session_key=%s linhas=%d", session_key, len(laps))
    return target
```

## 5. Qualidade de código (Ruff)

Adicione ao `pyproject.toml` (proposta; ainda não existe):

```toml
[tool.ruff]
line-length = 88
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "S", "T20"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]
```

- `I` organiza imports; `UP` moderniza sintaxe; `S` detecta senhas e segredos no código; `T20` proíbe `print` (use logging).

```
poetry run ruff check .
poetry run ruff check . --fix
poetry run ruff format .
poetry run ruff format --check .
```

- `check` faz o lint; `--fix` corrige o que é seguro (inclui ordenar imports); `format` formata; `format --check` só verifica.
- Respeite a configuração do projeto. Use `# noqa: <regra>` só com justificativa em comentário.

## 6. Testes e cobertura (pytest)

Adicione ao `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

Comando que roda os testes e falha se a cobertura de `src/f1` ficar abaixo de 50%:

```
poetry run pytest --cov=src/f1 --cov-report=term-missing --cov-fail-under=50
```

- Retorna código diferente de zero se algum teste falhar **ou** se a cobertura for menor que 50%. Não reduza o limite.
- Sem nenhum `test_*.py`, o pytest termina com "no tests ran" (código 5). Escreva o primeiro teste antes de rodar o gate.
- Teste isolado, sem cobertura: `poetry run pytest tests/<arquivo>.py -k <nome>`.
- Testes unitários não acessam rede nem o disco real: use `tmp_path`, `monkeypatch` e dados pequenos em memória.
- Use `@pytest.mark.parametrize` para casos de borda. Todo código novo vem com testes.

## 7. Documentação (MkDocs Material)

Ainda não há `mkdocs.yml` nem `docs/`. Para configurar:

```
mkdir docs
git mv doc_api.md docs/api.md
```

Crie `docs/index.md` (visão geral) e `mkdocs.yml` na raiz:

```yaml
site_name: F1
theme:
  name: material
nav:
  - Início: index.md
  - API OpenF1: api.md
```

```
poetry run mkdocs serve
poetry run mkdocs build --strict
```

- `serve` publica em `http://127.0.0.1:8000` e recarrega ao salvar.
- `build --strict` gera a versão de produção em `site/` e falha com avisos (links quebrados, páginas fora do `nav`).
- Mudou comportamento, pipeline ou configuração? Atualize `docs/` na mesma alteração.

## 8. Boas práticas de Python e engenharia de dados

**Tipagem**
- Anote parâmetros e retornos de toda função pública. Use sintaxe moderna do Python 3.13: `list[str]`, `str | None`.
- Verificador de tipos é opcional: `poetry add --group dev mypy` e `poetry run mypy src`.

**Erros**
- Capture exceções específicas; nunca `except:` nem `except Exception: pass`. Preserve a causa com `raise NovoErro(...) from exc`.
- Chamadas HTTP: sempre `timeout`, `raise_for_status()` e retentativas limitadas apenas para falhas transitórias.
- Falhe cedo com mensagem clara quando faltar coluna, config ou variável obrigatória.

**Logs**
- `logger = logging.getLogger(__name__)` em cada módulo; nada de `print`.
- Configure o logging uma única vez, no ponto de entrada. Use formatação preguiçosa (`logger.info("x=%s", x)`) e nunca registre segredos ou dados pessoais.

**Configuração externa**
- URL base, caminhos, limites e credenciais vêm de variáveis de ambiente, lidas em `config.py`. Nada fixo no código.
- Obrigatórias: `os.environ["NOME"]` (falha cedo). Opcionais: `os.getenv("NOME", "padrão")`.
- Versione um `.env.example` sem valores reais. Para carregar `.env` localmente: `poetry add python-dotenv`.

**Segredos**
- Nunca em código, commits, logs, testes ou docs. O `.env` real fica no `.gitignore`. Credencial exposta deve ser rotacionada.
- Se usar dados em tempo real (assinatura paga na OpenF1), trate a credencial como segredo.
- Crie o `.gitignore` (não existe) com pelo menos:

```
.venv/
.env
__pycache__/
.pytest_cache/
.ruff_cache/
.coverage
htmlcov/
site/
```

**Validação de dados**
- Valide na fronteira (entrada e saída do pipeline): colunas esperadas, tipos, nulos, duplicatas e faixas de valores.
- Para esquemas declarativos, `pydantic` (registros) ou `pandera` (DataFrames) são opções; só adicione com `poetry add` com aprovação do time.

**Idempotência e reprodutibilidade**
- Reexecutar o pipeline com a mesma entrada produz o mesmo resultado, sem duplicar: sobrescreva por partição (ex.: por `session_key`) ou faça upsert por chave.
- Escreva de forma atômica (arquivo temporário + `replace`), ordene antes de gravar e não use estado global.
- Mantenha os dados brutos (raw) imutáveis; transforme em etapa separada. `poetry.lock` e `.python-version` versionados.
- Injete datas e aleatoriedade (com seed) por parâmetro; não chame `datetime.now()` dentro de transformações.

**Casos de borda a testar (vindos da API OpenF1, ver `doc_api.md`)**
- Resposta vazia (`[]`) e campos `null`.
- `interval` e `gap_to_leader` podem ser `null` (líder) ou texto como `+1 LAP`; trate antes de converter para número.
- Em `session_result` de qualifying, `duration` e `gap_to_leader` são arrays (Q1, Q2, Q3).
- Datas em UTC (ISO 8601); cuidado com fuso ao comparar.
- Campos deprecated (`country_code` em `/drivers`, `pit_duration` em `/pit`) e dados que só existem em certas sessões (`intervals`, `overtakes`, `championship_*` só em corrida).

## 9. Fluxo antes de concluir uma alteração

1. Leia o código e a documentação existentes (`doc_api.md` ou `docs/`) antes de mudar algo.
2. Implemente em `src/f1/`, mantendo lógica pura separada de I/O, com tipos, logs e erros explícitos.
3. Adicione ou atualize testes, incluindo casos de borda.
4. Rode o checklist completo:

```
poetry run ruff check . --fix
poetry run ruff format .
poetry run ruff check . && poetry run ruff format --check . && poetry run pytest --cov=src/f1 --cov-report=term-missing --cov-fail-under=50 && poetry run mkdocs build --strict
```

5. Atualize `docs/` e `.env.example` se o comportamento ou a configuração mudou.
6. Confirme que não há segredos no diff (`git diff`) e que `pyproject.toml` e `poetry.lock` estão consistentes.
7. Commit no padrão convencional, em minúsculas: `feat: ...`, `fix: ...`, `docs: ...`, `test: ...`, `chore: ...`.

## 10. Se algo falhar

- Ferramenta ou módulo ausente: `poetry install`; se ainda faltar, `poetry add --group dev <pacote>`. Nunca `pip install`.
- Python errado: confira `.python-version`, `pyenv which python` e `poetry env info`; refaça `poetry env use`.
- Cobertura abaixo de 50%: escreva testes para as linhas listadas em `Missing`; não reduza o limite.