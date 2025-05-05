# Previsão de Produtividade Agrícola (Sprint 2)

Eduardo Carvalho RM559438 Jhonatan Salles RM554190


## 1. Preparação dos Dados

### 1.1 Fontes
- **Produtividade histórica:** `produtividade_historica.csv` (colunas: `date`, `yield`)
- **Imagens NDVI:** Arquivos GeoTIFF nomeados `ndvi_YYYYMMDD.tif` na pasta `ndvi_images/`

### 1.2 Tratamento
- Conversão de datas com `pandas`
- Leitura e média de NDVI com `rasterio` e `numpy`
- Junção das tabelas com NDVI médio e produtividade
- Criação da variável `day_of_year`

python
df_yield = pd.read_csv('produtividade_historica.csv')
df_yield['date'] = pd.to_datetime(df_yield['date'], dayfirst=True)

def load_ndvi_mean(path):
    with rasterio.open(path) as src:
        arr = src.read(1)
        arr = np.where(arr == src.nodata, np.nan, arr)
        return np.nanmean(arr)

ndvi_means = []
for d in df_yield['date'].unique():
    path = f"ndvi_images/ndvi_{d.strftime('%Y%m%d')}.tif"
    ndvi_means.append({'date': d, 'ndvi_mean': load_ndvi_mean(path)})

ndvi_df = pd.DataFrame(ndvi_means)
df = df_yield.merge(ndvi_df, on='date').dropna()
df['day_of_year'] = df['date'].dt.dayofyear
2. Extração de Informações Relevantes
2.1 Variáveis-Chave
Variável	Descrição	Justificativa
ndvi_mean	Média de NDVI por área	Indicador da saúde da vegetação, correlaciona com produtividade
day_of_year	Dia do ano (1–365)	Captura padrões sazonais
rolling_ndvi	Média móvel de NDVI	Modela tendências de curto/médio prazo

### 2.2 Visualizações
Scatter: NDVI vs Yield

python
Copy
Edit
plt.scatter(df['ndvi_mean'], df['yield'], alpha=0.7)
plt.xlabel('NDVI Mean')
plt.ylabel('Yield')
plt.title('NDVI vs Yield')
plt.show()
2.3 NDVI por Parcelas
Uso de geopandas e rasterio.mask para extrair médias por polígono:

python
Copy
Edit
parcels = gpd.read_file('shapefile_cultivos.shp')
ndvi_agg = []
for d in df['date'].unique():
    with rasterio.open(f"ndvi_images/ndvi_{d.strftime('%Y%m%d')}.tif") as src:
        for _, row in parcels.iterrows():
            geom = [row.geometry]
            out_img, _ = mask(src, geom, crop=True)
            arr = np.where(out_img[0] == src.nodata, np.nan, out_img[0])
            ndvi_agg.append({
                'parcel_id': row.id,
                'date': d,
                'ndvi_parcel_mean': np.nanmean(arr)
            })

parcel_ndvi_df = pd.DataFrame(ndvi_agg)
### 3. Modelo e Lógica Preditiva
Modelo: RandomForestRegressor
Captura relações não-lineares

Robusto a outliers

Pipeline:

Split treino/teste (80/20)

Escalonamento (StandardScaler)

GridSearchCV (parâmetros: n_estimators=[100,200], max_depth=[5,10])

Lógica:
Combinação de múltiplas árvores para melhorar a generalização

### 4. Análises Exploratórias
NDVI vs Yield: Correlação positiva clara

Observed vs Predicted: Curvas quase sobrepostas no conjunto de teste

### 5. Métricas e Resultados
Métrica	Valor	Interpretação
MSE	24.19	Erro médio baixo
R²	0.94	Explica 94% da variância da produtividade

Conclusão:
O modelo capturou bem a sazonalidade e a variabilidade espacial via NDVI.

Pode ser melhorado com variáveis adicionais (chuva, temperatura, solo).

Próximos Passos:
Validação com k-fold (ex: k=5)

Inclusão de variáveis climáticas

Deploy automatizado da pipeline

css
Copy
Edit
