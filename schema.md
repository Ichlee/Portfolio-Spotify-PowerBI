# Esquema: Spotify Artist Streaming

## Visão geral

- **Arquivo de origem:** `spotify_artist_streaming_2020_2025.csv`
- **Granularidade:** uma linha por faixa, artista e contexto de lançamento/streaming.
- **Registros:** 50.000
- **Colunas:** 33
- **Período de lançamento:** 2020 a 2025
- **Tabela recomendada no Power BI:** `FactStreaming`
- **Chave candidata:** `track_id` (UUID). Validar unicidade durante a carga.
- **Separador:** vírgula (`,`)
- **Codificação esperada:** UTF-8

## Dicionário de dados

| Coluna | Tipo Power BI | Papel | Descrição / domínio |
|---|---|---|---|
| `track_id` | Texto | Chave | Identificador UUID da faixa. |
| `track_name` | Texto | Dimensão | Nome da faixa. |
| `artist_name` | Texto | Dimensão | Nome do artista. |
| `album_name` | Texto | Dimensão | Nome do álbum. |
| `release_date` | Data | Dimensão | Data de lançamento da faixa (`AAAA-MM-DD`). |
| `genre` | Texto | Dimensão | Gênero musical, por exemplo `Pop`, `R&B`, `K-Pop`, `Latin`, `Classical`, `Punk` e `Trap`. |
| `duration_ms` | Número inteiro | Métrica | Duração da faixa em milissegundos. |
| `popularity` | Número inteiro | Métrica | Popularidade da faixa, escala esperada de 0 a 100. |
| `danceability` | Número decimal | Métrica | Grau de adequação para dança, normalmente de 0 a 1. |
| `energy` | Número decimal | Métrica | Intensidade e atividade percebida, normalmente de 0 a 1. |
| `key` | Número inteiro | Dimensão musical | Classe da tonalidade em notação Pitch Class, de 0 a 11. |
| `loudness` | Número decimal | Métrica | Volume médio da faixa em decibéis (dB), normalmente negativo. |
| `mode` | Número inteiro | Dimensão musical | Modalidade: `1` = maior, `0` = menor. |
| `instrumentalness` | Número decimal | Métrica | Probabilidade de a faixa não conter vocais, normalmente de 0 a 1. |
| `tempo` | Número decimal | Métrica | Tempo estimado em batidas por minuto (BPM). |
| `stream_count` | Número inteiro | Métrica | Quantidade de reproduções. Usar soma como agregação padrão. |
| `country` | Texto | Dimensão | Código de país com duas letras, por exemplo `US`, `ES`, `ZA`, `CO`, `IN`, `AU`, `PT` e `NG`. |
| `explicit` | Booleano | Dimensão | Indica se a faixa possui conteúdo explícito. |
| `label` | Texto | Dimensão | Gravadora ou selo responsável pelo lançamento. |
| `release_year` | Número inteiro | Dimensão | Ano derivado de `release_date`. |
| `release_month` | Número inteiro | Dimensão | Mês derivado de `release_date`, de 1 a 12. |
| `release_day_of_week` | Texto | Dimensão | Dia da semana derivado de `release_date`, por exemplo `Monday` e `Saturday`. |
| `duration_minutes` | Número decimal | Métrica derivada | Duração em minutos, calculada como `duration_ms / 60000`. |
| `popularity_category` | Texto | Dimensão derivada | Faixa categórica de popularidade: `Low`, `Medium` ou `High`. |
| `loudness_category` | Texto | Dimensão derivada | Categoria de volume, como `Quiet`, `Moderate` ou `Loud`. |
| `key_name` | Texto | Dimensão derivada | Nome da tonalidade, como `A`, `C#`, `G#` ou `D#`. |
| `mode_name` | Texto | Dimensão derivada | Nome da modalidade: `Major` ou `Minor`. |
| `is_explicit_bool` | Booleano | Dimensão derivada | Versão booleana normalizada de `explicit`. |
| `release_quarter` | Texto | Dimensão derivada | Trimestre do lançamento: `Q1`, `Q2`, `Q3` ou `Q4`. |
| `is_weekend_release` | Booleano | Dimensão derivada | Indica lançamento no sábado ou domingo. |
| `log_stream_count` | Número decimal | Métrica derivada | Transformação logarítmica da contagem de streams, útil para reduzir assimetria. |
| `upbeat_score` | Número decimal | Métrica derivada | Índice derivado que combina características de ritmo/energia. Validar a fórmula na origem antes de usá-lo como KPI oficial. |
| `artist_track_count` | Número inteiro | Métrica derivada | Quantidade de faixas associadas ao artista na base. |

## Regras de modelagem

1. Importar `release_date` como **Data**, e não como texto.
2. Usar `pt-BR` apenas para formatação; os números decimais do arquivo usam ponto (`.`).
3. Definir `stream_count`, `duration_ms`, `release_year`, `release_month`, `key`, `mode` e `artist_track_count` como números inteiros.
4. Definir `danceability`, `energy`, `instrumentalness`, `tempo`, `loudness`, `duration_minutes`, `log_stream_count` e `upbeat_score` como números decimais.
5. Não somar `popularity`, `danceability`, `energy`, `tempo` ou `artist_track_count`; usar média, mínimo ou máximo conforme a análise.
6. Classificar `release_month` por número e `release_day_of_week` por uma coluna de ordem de 1 a 7, caso seja usado em visuais.
7. Manter `track_id` como texto para preservar o UUID e evitar conversões numéricas.
8. Ocultar as colunas derivadas caso a mesma informação já seja exposta por uma dimensão de calendário ou por medidas.

## Medidas DAX iniciais

```DAX
Total Streams = SUM(FactStreaming[stream_count])

Total Tracks = DISTINCTCOUNT(FactStreaming[track_id])

Total Artists = DISTINCTCOUNT(FactStreaming[artist_name])

Average Popularity = AVERAGE(FactStreaming[popularity])

Average Track Duration (min) = AVERAGE(FactStreaming[duration_minutes])

Explicit Track % =
DIVIDE(
	CALCULATE([Total Tracks], FactStreaming[is_explicit_bool] = TRUE()),
	[Total Tracks]
)

Streams per Track = DIVIDE([Total Streams], [Total Tracks])
```

## Validações recomendadas após a importação

- Confirmar que `track_id` não possui duplicatas indevidas.
- Confirmar que não existem valores nulos em `track_id`, `release_date`, `stream_count` e `artist_name`.
- Verificar `0 <= popularity <= 100`.
- Verificar `0 <= danceability, energy, instrumentalness <= 1`.
- Verificar a consistência: `duration_minutes = duration_ms / 60000`.
- Verificar a consistência entre `explicit` e `is_explicit_bool`.
- Verificar a consistência entre `release_date` e `release_year`, `release_month`, `release_quarter` e `is_weekend_release`.
- Validar se `artist_track_count` representa a contagem global do artista ou apenas a contagem dentro do recorte de 2020-2025.
