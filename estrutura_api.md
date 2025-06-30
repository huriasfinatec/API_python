

# Estrutura da API: Backend em Python (FastAPI)
Sua API será construída com FastAPI e se organizará em torno de endpoints RESTful, que são URLs específicas que seu frontend acessará para obter ou enviar dados. O banco de dados (PostgreSQL ou SQLite) será o armazenamento central, e o SQLAlchemy será usado para interagir com ele.

### 1. Organização dos Arquivos no Backend
Seu projeto Python (na pasta backend_api/) terá a seguinte estrutura principal:

```
backend_api/
├── main.py                     # Onde sua aplicação FastAPI é inicializada e os endpoints são definidos.
├── database.py                 # Contém a configuração da conexão com o banco de dados e as definições dos modelos ORM (SQLAlchemy) para suas tabelas.
├── crud.py                     # Funções para interagir com o banco de dados (Criar, Ler, Atualizar, Deletar - CRUD). É a "camada de serviço" que a API usa para buscar/salvar dados.
├── models.py                   # Define os modelos de dados (Pydantic) que a API usará para validar requisições e formatar respostas.
├── .env                        # Arquivo para variáveis de ambiente (ex: URL de conexão com o banco de dados, chaves de API, etc.).
└── data_ingestion/             # Pasta que contém os scripts para coletar e processar seus dados brutos antes de inseri-los no banco.
    ├── __init__.py             # (Arquivo vazio para indicar que é um pacote Python)
    ├── ingest_manual_stations.py   # Script para ler CSVs de estações manuais e inseri-los no DB.
    ├── ingest_automatic_api.py     # Script para coletar dados de APIs externas de estações automáticas e inseri-los no DB.
    ├── calculate_vehicle_emissions.py # Script para calcular emissões a partir de dados de veículos (CSV).
    └── ingest_calculated_emissions.py # Script para inserir os dados de emissões calculadas no DB.

```
### 2. Modelos de Dados (Pydantic e SQLAlchemy ORM)

Você precisará de modelos para representar as entidades no seu banco de dados e os dados que a API envia/recebe.


- database.py (SQLAlchemy ORM Models): Definem a estrutura das tabelas no seu banco de dados.

- Station: Para armazenar informações sobre cada estação de monitoramento (ID, nome, latitude, longitude, tipo - automática, manual, baixo custo).

- PollutantReading: Para armazenar leituras de poluentes específicas de uma estação (ID, ID da estação, tipo de poluente (e.g., PM2.5, O3), valor, unidade, timestamp).

- MobilityData: Para dados relacionados à mobilidade e emissões veiculares (ID, tipo de dado (e.g., "CarDensity", "Emissao_CO2"), valor, latitude, longitude, nome da área, timestamp, unidade).

- GreenhouseGasReading: Para leituras de gases de efeito estufa, que podem estar associadas a estações ou serem dados agregados de outras fontes (ID, ID da estação (opcional), tipo de gás (e.g., CO2, CH4), valor, unidade, timestamp).

- models.py (Pydantic Models): São usados pelo FastAPI para validação automática de dados de entrada e para formatar as respostas JSON da sua API. Cada modelo ORM terá um modelo Pydantic correspondente (Base, Create, e o modelo principal).

- Station, PollutantReading, MobilityData, GreenhouseGasReading: Versões Pydantic dos modelos ORM.

- MapData: Um modelo combinado para a resposta do endpoint do mapa, contendo listas de estações, leituras de poluentes, dados de mobilidade e leituras de GEE.



### 3. Endpoints da API (URLs)

Seus endpoints serão acessados pelo frontend WordPress. Eles devem ser claros e seguir convenções RESTful.

#### Dados de Estações:

- GET /api/stations/: Lista todas as estações de monitoramento.

- Filtros: station_type (e.g., ?station_type=automatic), skip, limit.

- GET /api/stations/{station_id}: Retorna os detalhes de uma estação específica.

- POST /api/stations/: Cria uma nova estação (geralmente usada pelos scripts de ingestão).

Leituras de Poluentes:

- GET /api/pollutant_readings/: Retorna leituras de poluentes.

- Filtros: station_id, pollutant_type, start_time, end_time, station_type, skip, limit.

- POST /api/pollutant_readings/: Adiciona uma nova leitura de poluente.

#### Dados de Mobilidade:

- GET /api/mobility_data/: Retorna dados de mobilidade e emissões veiculares.

- Filtros: data_type (e.g., ?data_type=CarDensity ou ?data_type_startswith=Emissao_), area_name, start_time, end_time, skip, limit.

- POST /api/mobility_data/: Adiciona um novo dado de mobilidade.

Leituras de Gases de Efeito Estufa (GEE):

- GET /api/greenhouse_gas_readings/: Retorna leituras de gases de efeito estufa.

- Filtros: station_id, gas_type, start_time, end_time, skip, limit.

- POST /api/greenhouse_gas_readings/: Adiciona uma nova leitura de GEE.

#### Endpoints para Dashboards Específicos:

- GET /api/map_data/: Endpoint principal para o mapa. Retorna um conjunto combinado de todas as estações, leituras recentes de poluentes, dados de mobilidade/emissões e GEE para visualização no mapa.

- Filtros: last_hours (para buscar dados do último X horas para desempenho).

- GET /api/regions_summary/: Retorna dados agregados por região do Distrito Federal, incluindo emissões veiculares calculadas, (simulados) dados de população e o Índice de Qualidade do Ar (IQA) por região.

- GET /api/air_quality_index/: Calcula e retorna o Índice de Qualidade do Ar (IQA) para uma estação específica, uma região ou para o Distrito Federal em geral.

- Filtros: station_id, region_name.

### 4. Considerações de Implementação

- CORS (Cross-Origin Resource Sharing): Essencial para permitir que seu frontend WordPress (que estará em um domínio diferente da sua API) acesse os endpoints. Isso é configurado no main.py do FastAPI.

- Autenticação/Autorização: Para uma API em produção, você deve considerar adicionar autenticação (e.g., tokens JWT) para proteger seus endpoints, especialmente os de POST. Para o seu caso, se a API for pública e apenas para leitura, pode ser menos crítica inicialmente.

- Paginação e Filtros: Implemente skip (offset) e limit nos endpoints para controlar a quantidade de dados retornados e start_time/end_time para filtros por período, otimizando a performance.

- Tratamento de Erros: Use as HTTPException do FastAPI para retornar mensagens de erro claras (e.g., 404 Not Found, 400 Bad Request).