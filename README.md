# 3rd IMDC - XGBSillas

Pipeline final para previsao probabilistica semanal de dengue e chikungunya no
3rd Infodengue-Mosqlimate Dengue Challenge.

## Equipe

Time: XGBSillas

Integrante principal: Sillas Rocha da Costa, EMAp/FGV.

## Estrutura

```text
src/config.py                desafios, rodadas, paths, quantis e modelos finais
src/download.py              baixa os arquivos oficiais do FTP
src/validate_raw_data.py     valida colunas minimas dos dados brutos
src/mapbiomas.py             baixa/processa MapBiomas anual
src/spatial.py               pesos e vizinhanca espacial de UFs
src/build_dataset.py         monta os paineis semanais por desafio
src/features.py              cria a matriz supervisionada sem vazamento temporal
src/models.py                baseline historico e modelos XGBoost quantilicos
src/metrics.py               WIS, MAE, cobertura e ordenacao de quantis
src/backtest.py              validacoes retrospectivas 2022-2025
src/forecast.py              previsao final 2026-2027
src/submission.py            CSVs por localidade/rodada no formato do sprint
src/plots.py                 graficos simples gerados pelo backtest

scripts/plot_final_validation_models.py
                            figuras do relatorio e comparacao entre modelos
scripts/fetch_external_predictions.py
                            opcional: baixa previsoes externas para comparacao

submission_workflow.ipynb    notebook operacional para validacao, plots e envio
final_report.md              relatorio tecnico e cientifico
data/submissions/            unico diretorio de dados versionavel
```

Arquivos gerados pela pipeline ficam em `data/raw`, `data/interim`,
`data/processed`, `data/results` e `data/submissions`. Apenas
`data/submissions` entra no Git. As demais pastas sao cache local e podem
ser apagadas e regeneradas.

## O que e Necessario

Para a submissao, o caminho necessario e:

```text
src.build_dataset -> src.backtest -> src.forecast -> src.submission
```

Esses modulos puxam internamente os auxiliares realmente usados:

- `config.py`: fonte unica dos desafios, rodadas, paths e modelos finais;
- `download.py`, `validate_raw_data.py`, `mapbiomas.py`, `spatial.py`: usados por
  `build_dataset.py`;
- `features.py`, `models.py`, `metrics.py`, `plots.py`: usados por backtest,
  forecast e validacao dos CSVs;
- `utils.py`: logging, quantis e helpers pequenos.

Arquivos opcionais:

- `scripts/fetch_external_predictions.py`: baixa previsoes de outro modelo para
  comparacao visual. Nao e usado para gerar a submissao.
- `scripts/plot_final_validation_models.py`: gera figuras agregadas e analises
  do relatorio. O `backtest.py` chama esse script ao final.
- `src/diagnostics.py`: diagnosticos exploratorios de zeros, covariaveis,
  extremos e espaco. Pode ser removido sem quebrar a pipeline de submissao se o
  objetivo for manter apenas o codigo final.
- `submission_workflow.ipynb`: notebook de operacao e envio; util para revisar e
  submeter, mas nao e dependencia dos modulos `src`.

Arquivos gerados e ignorados:

- `data/raw`, `data/interim`, `data/processed`, `data/results`;
- `plot_submissions/`;
- caches Python/Jupyter e logs locais.

## Dados e Variaveis

O projeto usa os dados oficiais do FTP do IMDC: casos provaveis de dengue e
chikungunya, clima observado, populacao Datasus, oscilacoes oceanicas, regioes
de saude e malhas territoriais. Tambem usa MapBiomas como fonte aberta de
variaveis anuais de uso e cobertura da terra.

O processamento esta em `src/build_dataset.py`. As variaveis finais sao criadas
em `src/features.py`: defasagens de casos, medias moveis, quantis historicos por
semana epidemiologica, indicadores sazonais, pressao regional, populacao e
variaveis ambientais MapBiomas.

## Dependencias

O ambiente e definido em `pyproject.toml` e `uv.lock`. As bibliotecas principais
sao: `pandas`, `numpy`, `pyarrow`, `epiweeks`, `xgboost`, `scikit-learn`,
`geopandas`, `libpysal`, `esda`, `matplotlib`, `rasterio`, `statsmodels`,
`mosqlient` e `python-dotenv`.

## Modelos e Incerteza

O backtest treina quatro componentes: `baseline`, `xgb_quantile`,
`xgb_quantile_log1p` e `xgb_residual`. A submissao usa um modelo final por
desafio, definido em `src/config.py`:

- dengue UF: `xgb_residual`;
- dengue cidade: `xgb_residual`;
- chikungunya UF: `xgb_quantile_log1p`;
- chikungunya cidade: `xgb_quantile`.

Os modelos XGBoost estimam diretamente os quantis 0.025, 0.05, 0.10, 0.25,
0.50, 0.75, 0.90, 0.95 e 0.975. Esses quantis geram a mediana e os intervalos
centrais de 50%, 80%, 90% e 95%. A ordem dos quantis e valores nao negativos
sao garantidos antes da escrita dos CSVs.

## Restricao Temporal

Cada rodada usa apenas dados disponiveis ate a semana epidemiologica 25 do ano
de origem. A temporada prevista comeca na EW41 e termina na EW40 do ano
seguinte. Essa regra fica centralizada em `src/features.py` e e usada tanto no
backtest quanto na previsao final.

## Rodar Pipeline Final

Pipeline minima para gerar os CSVs finais:

```bash
uv sync
uv run python -m src.build_dataset
MODEL_DEVICE=cuda uv run python -m src.backtest
MODEL_DEVICE=cuda uv run python -m src.forecast
uv run python -m src.submission
```

Para rodar sem GPU:

```bash
MODEL_DEVICE=cpu uv run python -m src.backtest
MODEL_DEVICE=cpu uv run python -m src.forecast
```

Para gerar comparacoes com previsoes externas no relatorio:

```bash
export MOSQLIMATE_API_KEY="sua-chave"
uv run python scripts/fetch_external_predictions.py
MODEL_DEVICE=cuda uv run python -m src.backtest
```

## Saidas Principais

```text
data/results/predictions/
data/results/metrics/
data/results/models/
data/results/figures/model_comparison/
data/submissions/
```

Os CSVs finais para envio ficam em `data/submissions/{challenge}/{model}/{round}`.
Cada arquivo tem as colunas exigidas pelo sprint: `date`, `pred`, `lower_50`,
`upper_50`, `lower_80`, `upper_80`, `lower_90`, `upper_90`, `lower_95`,
`upper_95`.

A estrutura versionavel de submissao fica limitada aos quatro modelos finais:

```text
data/submissions/dengue_uf/xgb_residual/{validation_1..4,final}/
data/submissions/dengue_city/xgb_residual/{validation_1..4,final}/
data/submissions/chikungunya_uf/xgb_quantile_log1p/{validation_1..4,final}/
data/submissions/chikungunya_city/xgb_quantile/{validation_1..4,final}/
```

## Simplificacoes Possiveis

Para uma versao ainda mais enxuta do repositorio:

- remover `src/diagnostics.py`, se os diagnosticos exploratorios nao forem mais
  usados;
- deixar `scripts/fetch_external_predictions.py` fora da execucao padrao, pois
  ele depende da API e serve apenas para comparacao;
- manter `scripts/plot_final_validation_models.py` enquanto `final_report.md`
  depender das figuras agregadas;
- manter `submission_workflow.ipynb` como guia de envio, mas tratar
  `plot_submissions/` como saida local, nao como artefato de Git.

## Referencias

O metodo combina regressao quantilica com XGBoost e baselines sazonais
historicos. Detalhes, formulas, diagnosticos e resultados de validacao estao em
`final_report.md`.
