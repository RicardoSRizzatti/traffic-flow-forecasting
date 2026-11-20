# Reconstrução e previsão do estado do tráfego em rodovias

Reconstruir velocidade e fluxo onde os sensores não medem, prever congestionamentos nos próximos 30 minutos e estimar a demanda que gera esse tráfego, a partir de dados de detectores de rodovias da Califórnia. O problema é o do **TrafficFlowBench**, competição da **IEEE Big Data Cup 2026** hospedada no Kaggle.

> 🚧 Projeto em andamento. O roteiro e o status de cada etapa estão na seção [Roteiro](#roteiro).

---

## Contexto

Agências de trânsito monitoram rodovias com detectores instalados no pavimento, que medem a cada poucos minutos quantos veículos passam (fluxo), a que velocidade e com que ocupação da pista. Essa rede é esparsa e falha com frequência: há trechos sem sensor, detectores desligados e janelas inteiras sem dado. Mesmo assim, operar a rodovia exige saber o estado do tráfego em todo o corredor, antecipar onde vai formar fila e entender de onde vem a demanda.

O TrafficFlowBench reúne esses três problemas num único benchmark. Os dados são **sintéticos**: um modelo de fluxo de tráfego, calibrado com detectores reais desses mesmos corredores, gerou os registros a cada 5 minutos. A calibração cobriu velocidade de fluxo livre, capacidade por faixa, densidade de congestionamento, demanda por dia da semana e horário, ruído de medição e disponibilidade dos detectores. Nenhum registro real aparece na base, e o calendário (2030–2031) não corresponde a nenhum período real. A rede viária vem em parte do OpenStreetMap.

Por trás de todas as tarefas está a teoria de fluxo de tráfego. Velocidade, densidade e fluxo de cada trecho seguem um **diagrama fundamental**, e os veículos se conservam de um trecho para o seguinte. Uma boa solução precisa respeitar essa física, não só acertar números.

## O desafio

São dez corredores direcionais (cinco rodovias da região de Los Angeles e de Orange County, nos dois sentidos), com dados a cada 5 minutos e quatro tarefas avaliadas sobre os mesmos dados:

| Tarefa | O que prever | Peso no score |
|---|---|---|
| 1. Reconstrução do estado | Velocidade e fluxo nas células (detector × intervalo de 5 min) apagadas da base: 20%, 30% ou 50% das células, conforme o dia | 35% |
| 2. Previsão de filas | Em quais trechos haverá fila nos próximos 30 minutos, a partir da última hora observada (sim ou não, por trecho e intervalo) | 30% |
| 3. Consistência física | Se a reconstrução da tarefa 1 respeita o diagrama fundamental e a conservação de veículos. Não tem arquivo próprio: é calculada sobre a submissão da tarefa 1 | 15% |
| 4. Estimativa de demanda (ODME) | Fluxos não negativos por caminho, coerentes com as contagens nos detectores | 20% |

O que torna o problema difícil:

- **Dados esparsos e com lacunas**, que exigem combinar informação no tempo (o mesmo detector em outros instantes) e no espaço (detectores vizinhos no corredor).
- **Detectores imperfeitos**: só cerca de 76% das células têm medição suficiente para contar no score.
- **Eventos raros**: a formação de fila é a minoria das observações e o que mais importa prever.
- **Restrições físicas**: uma previsão estatisticamente boa pode violar a relação entre velocidade, densidade e fluxo e perder pontos na tarefa 3.
- **Volume**: a base tem 12,4 GB, e a submissão completa tem quase 7 milhões de linhas (6,7 milhões só da tarefa 1).
- **Generalização no tempo**: o leaderboard público avalia março de 2031 e a classificação final, abril. Os dois meses foram gerados de forma independente, com incidentes diferentes.

## Objetivo e métrica

**Objetivo:** construir uma solução para as quatro tarefas que supere com folga os baselines da competição, com atenção especial à reconstrução (tarefa 1) e à previsão de filas (tarefa 2), e servir o modelo de previsão de filas por uma API.

**Métrica:** o score final é a soma ponderada dos scores das tarefas, cada um entre 0 e 1:

```
Score = 0,35·S_estado + 0,30·S_fila + 0,15·S_física + 0,20·S_ODME
```

Cada tarefa é avaliada por corredor. Os sentidos de uma rodovia entram por média simples, e as rodovias também, então um corredor pequeno pesa tanto quanto um grande. Só contam as células em que o detector mediu pelo menos 75% do intervalo (`pct_observed ≥ 75`).

| Tarefa | Como é medida |
|---|---|
| 1. Estado | Em cada nível de lacuna (R1 = 20%, R2 = 30%, R3 = 50%): `0,54·max(0, 1 − RMSE_velocidade/25) + 0,46·max(0, 1 − RMSE_fluxo_por_faixa/600)`, com velocidade em km/h e fluxo em veículos/h por faixa. O score é a média simples dos três níveis |
| 2. Fila | IoU entre as células (trecho × intervalo) previstas com fila e as reais. Há fila quando a velocidade cai a 60% da velocidade de fluxo livre ou menos. Janelas de início de fila e de fila em andamento têm o mesmo peso |
| 3. Física | `⅓·S_FD + ⅔·S_LWR`, calculado sobre a submissão da tarefa 1: aderência ao diagrama fundamental e resíduo da conservação de veículos entre trechos (modelo LWR) |
| 4. ODME | `0,45·S_od + 0,25·S_link + 0,15·S_dev + 0,15·S_attr`: erro dos fluxos por caminho, reprodução das contagens nos links, desvio em relação à demanda a priori e distribuição dos destinos |

A especificação completa está no [SCORING_SPEC](https://github.com/jacky850/trafficflowbench-public/blob/main/docs/SCORING_SPEC.md) do repositório oficial.

**Referência a superar:** baselines oficiais na validação, média dos dez corredores.

| Tarefa (baseline) | Baseline | Resposta perfeita |
|---|---:|---:|
| Estado (perfil histórico dia da semana × horário) | 0,6929 | 1,0000 |
| Fila (repete o último estado observado) | 0,3017 | 1,0000 |
| Física (vem da submissão da tarefa 1) | 0,3549 | 0,9636 |
| ODME (mínimos quadrados não negativos com prior) | 0,8359 | 1,0000 |
| **Total** | **0,5534** | **0,9945** |

**Avaliação local:** só a tarefa 1 pode ser medida localmente com a fórmula oficial, nos meses de treino, que trazem os dados completos ao lado da versão com lacunas. As tarefas 2 e 3 dependem de gabaritos que os organizadores não publicam, e a tarefa 4 só pode ser conferida em parte.

## Dados

| | |
|---|---|
| **Competição (dados e regras)** | <https://www.kaggle.com/competitions/2026-ieee-big-data-traffic-flow-bench> |
| **Repositório oficial (avaliadores e baselines)** | <https://github.com/jacky850/trafficflowbench-public> |
| **IEEE Big Data Cup 2026** | <https://bigdataieee.org/BigData2026/cup/> |
| **Origem** | Registros sintéticos gerados por um modelo de fluxo de tráfego calibrado com detectores reais dos mesmos corredores; rede viária derivada em parte do OpenStreetMap |
| **Corredores** | Distrito 7 (Los Angeles): I-10 leste/oeste, I-210 leste/oeste, I-405 norte/sul. Distrito 12 (Orange County): I-5 norte/sul, I-405 norte/sul |
| **Período** | Treino: 01/06/2030 a 28/02/2031 (273 dias, com e sem lacunas). Validação (leaderboard público): 03/2031. Privado (classificação final): 04/2031 |
| **Conteúdo** | Estados das pistas principais (velocidade, fluxo, ocupação), contagens das rampas de acesso e saída, rede (links, diagrama fundamental por link, topologia, caminhos origem-destino) e modelos de submissão por tarefa |
| **Tamanho** | 9.688 arquivos (Parquet e CSV), 12,4 GB descompactados (7,8 GB compactados) |
| **Uso** | Os dados seguem as regras da competição e não podem ser redistribuídos |

Os dados **não são versionados** neste repositório. Para reproduzir, aceite as regras na página da competição e baixe com a CLI do Kaggle (token da API, criado em kaggle.com/settings, salvo em `~/.kaggle/access_token`):

```bash
make data   # usa a CLI do Kaggle via uv e descompacta em data/raw/trafficflowbench
```

**Prazos:** pela página da IEEE Big Data Cup 2026, a submissão final no Kaggle foi até 06/11/2026, e os resultados são apresentados na IEEE BigData 2026 (Phoenix, 14 a 17/12/2026). Este projeto continua depois do prazo, como estudo e portfólio, com submissões tardias no Kaggle (que recebem score, mas não entram no ranking).

## Sobre o projeto

| | |
|---|---|
| **Problema** | Reconstruir e prever o estado do tráfego em corredores de rodovia e estimar a demanda origem-destino. |
| **Dados** | TrafficFlowBench (IEEE Big Data Cup 2026): dez corredores sintéticos, detectores a cada 5 minutos. |
| **Abordagem (planejada)** | Baselines por perfil histórico (dia da semana × horário) e interpolação espaço-temporal; gradient boosting com defasagens e detectores vizinhos; parâmetros publicados do diagrama fundamental por trecho para manter a reconstrução fisicamente coerente; mínimos quadrados não negativos para a demanda. Validação por blocos de dias. |
| **Métrica principal** | Score ponderado da competição (0,35 / 0,30 / 0,15 / 0,20). |
| **Resultado** | A preencher: ver `models/metadata.json` ou a seção *Qualidade do modelo* no app. |

## Roteiro

- [x] Baixar os dados para `data/raw/trafficflowbench`
- [ ] Adaptar `etl/extract.py` aos arquivos da competição
- [ ] Análise exploratória em `notebooks/`: cobertura dos detectores, padrões diários e semanais, episódios de fila, diagrama fundamental empírico por trecho (tarefa 1: [`01_eda_tarefa1.ipynb`](notebooks/01_eda_tarefa1.ipynb))
- [ ] Baselines das quatro tarefas e primeira submissão completa
- [ ] Tarefa 1: modelo de reconstrução com features temporais e espaciais, validado por blocos de dias
- [ ] Tarefa 2: classificador de fila para 30 minutos à frente, exportado em ONNX e servido pela API
- [ ] Tarefa 3: reconstrução coerente com o diagrama fundamental e a conservação de veículos (avaliada sobre a saída da tarefa 1)
- [ ] Tarefa 4: estimativa de fluxos por caminho a partir das contagens
- [ ] Relatório com resultados por tarefa e por corredor

## Referências

- Repositório oficial do benchmark (dados, regras de avaliação e baselines): <https://github.com/jacky850/trafficflowbench-public>
- Especificação das métricas: <https://github.com/jacky850/trafficflowbench-public/blob/main/docs/SCORING_SPEC.md>
- Caltrans Performance Measurement System (PeMS), sistema de detectores reais dos distritos 7 e 12: <https://pems.dot.ca.gov/>

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
3. **Features prontas no ETL.** O ETL limpa, aplica as regras de negócio e calcula as features; o modelo (e o ONNX) recebe os dados prontos, sem pré-processamento embutido.
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
