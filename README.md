# Reconstrução e previsão do estado do tráfego em rodovias

Reconstruir velocidade e fluxo onde os sensores não medem, prever congestionamentos nos próximos 30 minutos e estimar a demanda que gera esse tráfego, a partir de dados de detectores de rodovias da Califórnia. O problema é o do **TrafficFlowBench**, competição da **IEEE Big Data Cup 2026** hospedada no Kaggle.

> 🚧 Projeto em andamento. O roteiro e o status de cada etapa estão na seção [Roteiro](#roteiro).

---

## Contexto

Agências de trânsito monitoram rodovias com detectores instalados no pavimento, que medem a cada poucos minutos quantos veículos passam (fluxo), a que velocidade e com que ocupação da pista. Essa rede é esparsa e falha com frequência: há trechos sem sensor, detectores desligados e janelas inteiras sem dado. Mesmo assim, operar a rodovia exige saber o estado do tráfego em todo o corredor, antecipar onde vai formar fila e entender de onde vem a demanda.

O TrafficFlowBench reúne esses três problemas num único benchmark, construído sobre uma rede viária extraída do OpenStreetMap (formato GMNS), observações de detectores no padrão do PeMS (Caltrans) em intervalos de 5 minutos e informações abertas de trajetórias GPS e pontos de interesse. Por trás de todas as tarefas está a teoria de fluxo de tráfego: velocidade, densidade e fluxo de cada trecho seguem um **diagrama fundamental**, e uma boa solução precisa respeitar essa física, não só acertar números.

## O desafio

São dez corredores de rodovia com dados a cada 5 minutos e quatro tarefas avaliadas sobre os mesmos dados:

| Tarefa | O que prever | Peso no score |
|---|---|---|
| 1. Reconstrução do estado | Velocidade e fluxo em detectores e instantes retirados da base | 35% |
| 2. Previsão de filas | Quais trechos estarão congestionados nos próximos 30 minutos (classificação binária) | 30% |
| 3. Consistência física | Se a reconstrução da tarefa 1 respeita o diagrama fundamental e a conservação de fluxo | 15% |
| 4. Estimativa de demanda (ODME) | Fluxos não negativos por caminho, coerentes com as contagens nos detectores | 20% |

O que torna o problema difícil:

- **Dados esparsos e com lacunas**, que exigem combinar informação no tempo (o mesmo detector em outros instantes) e no espaço (detectores vizinhos no corredor).
- **Eventos raros**: a formação de fila é a minoria das observações e o que mais importa prever.
- **Restrições físicas**: uma previsão estatisticamente boa pode violar a relação entre velocidade, densidade e fluxo e perder pontos na tarefa 3.
- **Volume**: a base tem cerca de 12 GB e a submissão final passa de 6 milhões de linhas.
- **Generalização no tempo**: o leaderboard público avalia um mês e a classificação final é decidida por outro mês, gerado à parte.

## Objetivo e métrica

**Objetivo:** construir uma solução para as quatro tarefas que supere com folga os baselines da competição, com atenção especial à reconstrução (tarefa 1) e à previsão de filas (tarefa 2), e servir o modelo de previsão de filas por uma API.

**Métrica:** o score final é a soma ponderada dos scores das tarefas:

```
Score = 0,35·S_estado + 0,30·S_fila + 0,15·S_física + 0,20·S_ODME
```

Cada `S` está entre 0 e 1. A definição exata de cada componente está na aba *Evaluation* da competição no Kaggle; a descrição do benchmark menciona erros de previsão do tipo RMSE/MAE para velocidade e fluxo.

Métricas internas para iterar: RMSE e MAE de velocidade e fluxo por corredor; precisão, recall e F1 dos eventos de fila; taxa de violação do diagrama fundamental.

## Dados

| | |
|---|---|
| **Competição (dados e regras)** | <https://www.kaggle.com/competitions/2026-ieee-big-data-traffic-flow-bench> |
| **Descrição do benchmark** | <https://bigdataieee.org/BigData2026/cup/trafficflowbench/> |
| **IEEE Big Data Cup 2026** | <https://bigdataieee.org/BigData2026/cup/> |
| **Fontes** | Rede viária do OpenStreetMap em GMNS (OSM2GMNS); detectores no padrão PeMS (velocidade, fluxo e ocupação a cada 5 min); trajetórias GPS e pontos de interesse abertos |
| **Tamanho** | Cerca de 12 GB descompactados |
| **Uso** | Os dados seguem as regras da competição e não podem ser redistribuídos |

Os dados **não são versionados** neste repositório. Para reproduzir, aceite as regras na página da competição e baixe com a CLI do Kaggle (token em `~/.kaggle/kaggle.json`):

```bash
make data   # usa a CLI do Kaggle via uv e descompacta em data/raw/trafficflowbench
```

**Prazos:** pelo cronograma da IEEE Big Data Cup 2026, as equipes entregam relatório final e solução até 15/11/2026, e os vencedores são apresentados na IEEE BigData 2026 (Phoenix, 14 a 17/12/2026). Confira as datas da competição na aba *Timeline* do Kaggle.

## Sobre o projeto

| | |
|---|---|
| **Problema** | Reconstruir e prever o estado do tráfego em corredores de rodovia e estimar a demanda origem-destino. |
| **Dados** | TrafficFlowBench (IEEE Big Data Cup 2026): dez corredores, detectores a cada 5 minutos. |
| **Abordagem (planejada)** | Baselines por perfil histórico (dia da semana × horário) e interpolação espaço-temporal; gradient boosting com defasagens e detectores vizinhos; calibração do diagrama fundamental por trecho para corrigir a reconstrução; mínimos quadrados não negativos para a demanda. Validação por blocos de dias. |
| **Métrica principal** | Score ponderado da competição (0,35 / 0,30 / 0,15 / 0,20). |
| **Resultado** | A preencher: ver `models/metadata.json` ou a seção *Qualidade do modelo* no app. |

## Roteiro

- [ ] Baixar os dados para `data/raw/trafficflowbench` e adaptar `etl/extract.py` aos arquivos da competição
- [ ] Análise exploratória em `notebooks/`: cobertura dos detectores, padrões diários e semanais, episódios de fila, diagrama fundamental empírico por trecho
- [ ] Baselines das quatro tarefas e primeira submissão completa
- [ ] Tarefa 1: modelo de reconstrução com features temporais e espaciais, validado por blocos de dias
- [ ] Tarefa 2: classificador de fila para 30 minutos à frente, exportado em ONNX e servido pela API
- [ ] Tarefa 3: ajuste da reconstrução ao diagrama fundamental calibrado
- [ ] Tarefa 4: estimativa de fluxos por caminho a partir das contagens
- [ ] Relatório com resultados por tarefa e por corredor

## Referências

- Descrição do benchmark e critérios de avaliação: <https://bigdataieee.org/BigData2026/cup/trafficflowbench/>
- Caltrans Performance Measurement System (PeMS): <https://pems.dot.ca.gov/>
- GMNS — General Modeling Network Specification: <https://github.com/zephyr-data-specs/GMNS>

---

## Início rápido

Requisitos: Linux ou WSL, [uv](https://docs.astral.sh/uv/), Node.js 22+, Docker e `make`.

```bash
make setup        # dependências Python (uv) e do front (npm)
make all          # ETL + treino + avaliação → models/model.onnx
make front-build  # build do React
make app          # http://localhost:8000  (API + front)
```

Ou, só com Docker (usa o modelo já versionado em `models/`):

```bash
make docker       # build + run em http://localhost:8000
```

Para desenvolver o front com hot reload: `make dev` (API na 8000 + Vite na 5173).

`make help` lista todos os comandos.

---

## Estrutura

```
├── configs/config.yaml   # caminhos, seed, destino do load, parâmetros do modelo
├── data/                 # NÃO versionado (só a estrutura)
│   ├── raw/              #   saída do extract: dado como veio da fonte
│   ├── interim/          #   saída do transform: limpo, ainda não publicado
│   ├── processed/        #   saída do load quando o destino é local
│   └── split/            #   train.parquet / test.parquet
├── tools/                # biblioteca própria (usada por etl, ml e app)
│   ├── config.py         #   leitura do config e caminhos
│   ├── data_io.py        #   leitura/escrita: local, BigQuery, GCS
│   ├── model_io.py       #   salvar/carregar modelos (somente ONNX)
│   ├── frames.py         #   operações genéricas com DataFrames
│   ├── git_info.py       #   commit usado no treino
│   └── log.py            #   logger padrão
├── etl/
│   ├── extract.py        # fonte → data/raw
│   ├── transform.py      # data/raw → data/interim (limpeza e regras de negócio)
│   └── load.py           # data/interim → destino (local | bigquery | gcs)
├── ml/
│   ├── split.py          # dados processados → treino/teste
│   ├── train.py          # Pipeline → models/model.onnx + metadata.json
│   └── evaluate.py       # métricas do ONNX no teste → metadata.json
├── models/               # VERSIONADO: somente .onnx + metadata.json
├── app/
│   ├── api/              # FastAPI (carrega o ONNX com onnxruntime)
│   └── frontend/         # React + TypeScript + Vite + ECharts
├── notebooks/            # análise exploratória (fora do pipeline)
├── tests/                # pytest
├── Dockerfile            # multi-stage: build do front + API + modelo
└── Makefile              # automação de todos os comandos
```

### Fluxo

```
fonte ─extract─▶ data/raw ─transform─▶ data/interim ─load─▶ processed (local | BigQuery | GCS)
                                                                   │
                                     ml/split ◀────────────────────┘
                                        │
                     data/split ─train─▶ models/model.onnx ─evaluate─▶ metadata.json (métricas)
                                                │
                                   app/api (onnxruntime) ─▶ app/frontend (React)
```

---

## Regras do repositório

Estas regras são verificadas automaticamente em `tests/test_project_rules.py` e na CI.

1. **Direção das dependências.** `etl`, `ml` e `app` importam de `tools`; `tools` não importa de nenhum deles. `app` não importa `etl`, `ml`, scikit-learn nem pandas: ele só conhece o modelo ONNX e o `metadata.json`.
2. **Modelos somente em ONNX.** A única forma de salvar um modelo é `tools.model_io.save_onnx_model`, que:
   - recusa qualquer caminho que não termine em `.onnx`;
   - valida o arquivo com `onnx.checker`;
   - compara as previsões do ONNX com as do modelo original (tolerância em `model.parity_tolerance`) e **não grava nada** se divergirem.

   Além disso, o `.gitignore` ignora qualquer outro arquivo em `models/`, e um teste falha se aparecer `.joblib`, `.pkl` etc.
3. **Pré-processamento dentro do Pipeline.** O ETL limpa e aplica regras de negócio; escala, encoding e features derivadas que o modelo precisa na previsão ficam no `Pipeline` do `ml/train.py`, para serem exportados no ONNX. Assim o app não reimplementa nada.
4. **Avaliação no artefato final.** O `evaluate.py` mede o próprio `model.onnx`, então a métrica exibida é a do modelo que vai para produção.
5. **Dados fora do git, modelos dentro.** `data/` nunca é versionado. `models/` é: o GitHub avisa acima de 50 MB e bloqueia acima de 100 MB; se um modelo crescer, use Git LFS (instruções no `.gitattributes`).

### ONNX: cuidados

- O ONNX trabalha em float32 por padrão: pequenas diferenças numéricas em relação ao scikit-learn são normais (por isso a tolerância).
- Nem todo componente tem conversor. Os nativos do scikit-learn têm; XGBoost e LightGBM convertem via `onnxmltools`; um transformador customizado exige registrar um conversor no `skl2onnx`.
- Features categóricas (texto): use `ColumnTransformer` com nomes de coluna; `convert_to_onnx` cria um input por coluna automaticamente.

---

## Configuração

Tudo fica em `configs/config.yaml`. Para usar outro arquivo: `CONFIG_PATH=configs/outro.yaml make all`.

### Destino dos dados processados

```yaml
load:
  destination: local   # local (padrão) | bigquery | gcs
```

Para BigQuery ou Cloud Storage:

```bash
make setup-gcp                             # instala o grupo gcp
gcloud auth application-default login      # autenticação
cp .env.example .env                       # e preencha GCP_PROJECT / GCS_BUCKET
```

O `ml/split.py` lê os dados processados pela mesma função (`read_processed`), então sempre encontra os dados onde o load gravou.

### Dependências (grupos do uv)

| Grupo | Para quê | Instalado por |
|---|---|---|
| (projeto) | numpy, pyyaml — comuns a todas as etapas | sempre |
| `etl` | pandas, pyarrow | `make setup` |
| `ml` | scikit-learn, skl2onnx, onnx, onnxruntime | `make setup` |
| `app` | fastapi, uvicorn, onnxruntime | `make setup` e Docker |
| `dev` | pytest, ruff, httpx, ipykernel | `make setup` |
| `gcp` | google-cloud-bigquery, google-cloud-storage | `make setup-gcp` |

A imagem Docker instala só o projeto + grupo `app`.

---

## Webapp

### API

| Rota | Descrição |
|---|---|
| `GET /api/health` | status e se o modelo foi carregado |
| `GET /api/model` | metadata: features, estatísticas, classes, métricas |
| `POST /api/predict` | `{"features": {...}}` → previsão e probabilidades |
| `POST /api/predict/batch` | `{"instances": [{...}, ...]}` → várias previsões |
| `GET /docs` | documentação interativa (Swagger) |

### Front

O front é gerado a partir do `metadata.json`: o formulário, a análise de sensibilidade e as métricas se adaptam a qualquer modelo de classificação ou regressão sem mudar código.

Gráficos padronizados ficam em `app/frontend/src/charts/`:

- `theme.ts` — **única fonte de cores**, com paleta categórica validada para daltonismo, escala sequencial e modo escuro;
- `Chart.tsx` — componente base (ciclo de vida do ECharts, redimensionamento e visão em tabela);
- `BarChart`, `LineChart` (com amostragem LTTB e zoom para séries longas) e `HeatmapChart`.

Para um novo tipo de gráfico: registre-o em `charts/echarts.ts` e crie o componente em `charts/` usando `baseOption`, `valueAxis` e `categoryAxis` do tema.

---

## Qualidade e CI

```bash
make test     # pytest
make lint     # ruff + checagem de tipos do front
make format   # formata o Python
make check    # o mesmo que a CI
```

A CI (`.github/workflows/ci.yml`) roda lint, o pipeline completo, os testes, o build do front e um smoke test da imagem Docker em cada push na `main` e em pull requests.

---

## Dúvidas comuns

- **Aviso `GetPciBusId` do onnxruntime ao iniciar** (comum em WSL e VMs): é inofensivo; o onnxruntime só não reconhece o barramento PCI virtual.
- **`make` reclama de `uv` ou `npm` não encontrados**: instale o uv (`curl -LsSf https://astral.sh/uv/install.sh | sh`) e o Node 22+ (recomendado via `nvm`).
- **Front mostra "Modelo indisponível"**: rode `make all` para gerar `models/model.onnx` e reinicie a API.
- **Mudei o modelo e o app não mudou**: a API carrega o modelo ao iniciar; reinicie (`make app`) ou refaça a imagem (`make docker`).
