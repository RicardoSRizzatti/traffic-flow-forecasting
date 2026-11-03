# CLAUDE.md

Contexto para o Claude Code neste repositório. O README tem a descrição completa do desafio; aqui ficam o estado atual, as decisões e o que não pode ser quebrado.

## O projeto

Resolver o **TrafficFlowBench** (IEEE Big Data Cup 2026, Kaggle): dez corredores de rodovia da Califórnia, dados de detectores a cada 5 minutos e quatro tarefas.

| Tarefa | Saída | Peso |
|---|---|---|
| 1. Reconstrução do estado | velocidade e fluxo em detectores/instantes retirados | 0,35 |
| 2. Previsão de filas | trecho congestionado ou não nos próximos 30 min (binário) | 0,30 |
| 3. Consistência física | reconstrução coerente com o diagrama fundamental e conservação de fluxo | 0,15 |
| 4. ODME | fluxos não negativos por caminho | 0,20 |

- **Competição:** <https://www.kaggle.com/competitions/2026-ieee-big-data-traffic-flow-bench>. A definição exata de cada métrica está na aba *Evaluation*; confira antes de implementar o score local.
- **Dados:** `make data` usa a CLI do Kaggle (via `uv tool run`) e descompacta em `data/raw/trafficflowbench`. Exige aceitar as regras no site e o token em `~/.kaggle/kaggle.json`. São ~12 GB e não podem ser redistribuídos: `data/` nunca vai para o git.
- **Prazos:** relatório final e solução da IEEE Big Data Cup até 15/11/2026; confira a *Timeline* no Kaggle.
- **Leaderboard:** o público avalia um mês; a classificação final usa outro mês.

## Estado atual

O repositório foi criado a partir do template `datascience_template` e **o pipeline ainda roda o exemplo do Iris**. O que é específico do Iris e precisa ser trocado:

- `etl/extract.py` (carrega o Iris do scikit-learn), `etl/transform.py`
- `configs/config.yaml` (`project`, `data.table`, `data.target`, `data.features`, `display_names`, `split`, `model.params`)
- `ml/train.py` → `build_pipeline()` (hoje `StandardScaler` + `LogisticRegression`)
- `models/model.onnx` e `models/metadata.json` (regerar com `make all`)
- `tests/test_api.py`, `tests/test_etl.py` e o payload do smoke test em `.github/workflows/ci.yml`
- `pyproject.toml` ainda usa o nome de pacote `datascience-template`. Para renomear, troque `name` e rode `uv lock` (a CI usa `uv sync --locked` e falha se o lock estiver desatualizado).

O checklist do projeto está na seção **Roteiro** do README: marque os itens conforme forem concluídos.

## Decisões de projeto

- **O pipeline do template (ETL → ONNX → API) serve a tarefa 2**, o classificador de fila a 30 minutos. É ele que fica em `models/model.onnx` e no app.
- **As tarefas 1, 3 e 4 geram arquivos de submissão**, não modelos servidos. Coloque esse código em `ml/` (por exemplo `ml/submission.py`), lendo os dados processados, e grave as saídas em `data/` (fora do git). A submissão final junta as quatro tarefas no formato da competição (mais de 6 milhões de linhas; conferir o `sample_submission`).
- **Validação por blocos de dias**, nunca split aleatório por linha: linhas vizinhas no tempo vazam informação. O `ml/split.py` do template faz `train_test_split` estratificado e precisa ser trocado por um corte temporal.
- **Volume:** ler os arquivos em partes (pyarrow/parquet, filtrando colunas) e evitar carregar os 12 GB de uma vez. Se o pandas não der conta, considerar `polars` ou `duckdb` num grupo de dependências próprio.
- **Física:** calibrar um diagrama fundamental por trecho (velocidade de fluxo livre, capacidade, densidade crítica e de congestionamento) e usar para corrigir a reconstrução da tarefa 1.
- **ODME:** mínimos quadrados não negativos ajustando os fluxos por caminho às contagens dos detectores.

## Regras do template (verificadas em `tests/test_project_rules.py` e na CI)

1. `etl`, `ml` e `app` importam de `tools`; `tools` não importa nenhum deles; `app` não importa `etl`, `ml`, scikit-learn nem pandas.
2. Modelos só em ONNX, salvos por `tools.model_io.save_onnx_model` (valida e compara com o modelo original). Nada de `joblib`/`pickle` em `models/`. LightGBM/XGBoost convertem via `onnxmltools`.
3. Pré-processamento que o modelo precisa na previsão fica dentro do `Pipeline`.
4. `ml/evaluate.py` mede o próprio `model.onnx`.
5. `data/` fora do git; `models/` versionado (acima de 50 MB, usar Git LFS).

## Comandos

```bash
make setup   # dependências Python (uv) e do front (npm)
make data    # baixa os dados da competição para data/raw/trafficflowbench
make all     # ETL + treino + avaliação → models/model.onnx + metadata.json
make app     # API + front em http://localhost:8000
make check   # lint + testes (o mesmo que a CI)
make help    # todos os comandos
```

## Convenções

- Documentação, comentários e docstrings em português, como no resto do template.
- Antes de comitar: `make check`. Ao trocar o exemplo do Iris, rodar `make all` e comitar `models/` junto.
- A CI roda `make all` sem acesso aos dados do Kaggle: quando o extract passar a ler `data/raw`, prever um modo de amostra pequena (fixture em `tests/`) para a CI continuar verde.
- Notebooks de exploração em `notebooks/`; o que for definitivo vai para `etl/` ou `ml/`.
