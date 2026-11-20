# CLAUDE.md

Contexto para o Claude Code neste repositório. O README tem a descrição completa do desafio; aqui ficam o estado atual, as decisões e o que não pode ser quebrado.

## O projeto

Resolver o **TrafficFlowBench** (IEEE Big Data Cup 2026, Kaggle): dez corredores de rodovia da Califórnia, dados de detectores a cada 5 minutos e quatro tarefas.

| Tarefa | Saída | Peso |
|---|---|---|
| 1. Reconstrução do estado | velocidade e fluxo em detectores/instantes retirados | 0,35 |
| 2. Previsão de filas | trecho congestionado ou não nos próximos 30 min (binário) | 0,30 |
| 3. Consistência física | sem arquivo próprio: calculada sobre a submissão da tarefa 1 (diagrama fundamental + conservação LWR) | 0,15 |
| 4. ODME | fluxos não negativos por caminho | 0,20 |

- **Competição:** <https://www.kaggle.com/competitions/2026-ieee-big-data-traffic-flow-bench>. Métricas exatas, avaliadores e baselines oficiais: <https://github.com/jacky850/trafficflowbench-public> (`docs/SCORING_SPEC.md`, `docs/MASK_SPEC.md`, `src/task*/score_*.py`). O score local da tarefa 1 deve reproduzir `src/task1/score_task1.py`.
- **Dados:** `make data` usa a CLI do Kaggle (via `uv tool run`) e descompacta em `data/raw/trafficflowbench`. Exige aceitar as regras no site e o token da API (formato `KGAT_...`, criado em kaggle.com/settings) em `~/.kaggle/access_token`, com `chmod 600`. São 12,4 GB (9.688 arquivos) e não podem ser redistribuídos: `data/` nunca vai para o git. O pacote fica em `data/raw/trafficflowbench/kaggle_public/`, com `README.md` e `config/competition_contract.json` próprios. Os dados são **sintéticos**: treino de 06/2030 a 02/2031 (com e sem lacunas), validação em 03/2031 (leaderboard público), privado em 04/2031.
- **Armadilhas do pacote:**
  - os alvos da tarefa 1 vêm do template `task1/<PANEL>/<split>/sample_submission_state.csv`, não dos nulos;
  - timestamps são texto (`YYYY-MM-DDTHH:MM:SSZ`) e devem ser usados sem reformatar;
  - só contam células com `pct_observed >= 75`;
  - rampa sem observação é ausência de dado, não fluxo zero;
  - o fluxo é avaliado por faixa com as `lanes` de `network/fd_parameters.csv`, e `lwr_mainline_topology.csv` traz faixas e capacidade diferentes para o mesmo link.
- **Prazos:** a submissão final da competição encerrou em 07/11/2026 06:55 UTC (noite de 06/11 nos EUA). As regras foram aceitas depois disso e a página não mostra limite para submissões tardias, que recebem score mas não entram no ranking. O projeto segue como portfólio no GitHub.
- **Fluxo de trabalho:** toda alteração no repositório vai numa branch própria e entra na `main` por pull request revisado pelo dono do repositório; nada de commit direto na `main`.
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
- **As tarefas 1 e 4 geram arquivos de submissão**, não modelos servidos. A tarefa 3 não tem submissão: é calculada sobre as linhas da tarefa 1. Coloque esse código em `ml/` (por exemplo `ml/submission.py`), lendo os dados processados, e grave as saídas em `data/` (fora do git). A submissão final é um único `sample_submission.csv` com 6.980.503 linhas: a coluna `task` vale `state`, `queue` ou `odme`. O `submission_key.csv` diz a que célula cada `submission_id` se refere.
- **Validação por blocos de dias**, nunca split aleatório por linha: linhas vizinhas no tempo vazam informação. O `ml/split.py` do template faz `train_test_split` estratificado e precisa ser trocado por um corte temporal.
- **Volume:** ler os arquivos em partes (pyarrow/parquet, filtrando colunas) e evitar carregar os 12 GB de uma vez. Se o pandas não der conta, considerar `polars` ou `duckdb` num grupo de dependências próprio.
- **Física:** usar os parâmetros publicados do diagrama fundamental (`network/fd_parameters.csv`) e a topologia (`lwr_mainline_topology.csv`) para manter a reconstrução da tarefa 1 coerente. `S_physics` não pode ser medido localmente (faltam os fluxos de fronteira dos organizadores); a única forma de melhorá-lo é melhorar a tarefa 1.
- **ODME:** mínimos quadrados não negativos ajustando os fluxos por caminho às contagens dos detectores.

## Regras do template (verificadas em `tests/test_project_rules.py` e na CI)

1. `etl`, `ml` e `app` importam de `tools`; `tools` não importa nenhum deles; `app` não importa `etl`, `ml`, scikit-learn nem pandas.
2. Modelos só em ONNX, salvos por `tools.model_io.save_onnx_model` (valida e compara com o modelo original). Nada de `joblib`/`pickle` em `models/`. LightGBM/XGBoost convertem via `onnxmltools`.
3. Features e pré-processamento ficam no ETL, que entrega os dados prontos ao modelo; o ONNX recebe as features já calculadas. (Decisão do projeto: substitui a regra do template de pôr o pré-processamento dentro do `Pipeline`. As docstrings de `etl/transform.py` e `ml/train.py` ainda dizem o contrário e mudam quando esses arquivos forem reescritos.)
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
