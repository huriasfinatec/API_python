# Milestone: Base de Conexão Frontend-Backend e Mapa Inicial
## Issue #1: Configurar Ambiente Python e Dependências

#### Descrição: Configure o ambiente de desenvolvimento Python, incluindo um ambiente virtual, e instale todas as bibliotecas necessárias para o backend.

Comandos no Terminal:

```

# 1. Crie e ative um ambiente virtual (recomendado)
python -m venv venv
# No Windows:
# .\venv\Scripts\activate
# No macOS/Linux:
# source venv/bin/activate

# 2. Instale as bibliotecas necessárias
pip install fastapi uvicorn "sqlalchemy[psycopg2]" pandas python-dotenv # 'psycopg2' para PostgreSQL, use 'mysqlclient' para MySQL, ou 'sqlite3' já vem com Python.

```

Critérios de Aceitação:

Ambiente virtual criado e ativado.
Todas as dependências Python listadas instaladas com sucesso.
Você pode rodar pip freeze e ver as bibliotecas instaladas.


## Issue #2: Inicializar o Banco de Dados (Estrutura Vazia)
#### Descrição: Crie o arquivo do banco de dados (SQLite) ou conecte-se ao PostgreSQL/MySQL e gere apenas a estrutura das tabelas vazias.

Código Relacionado:

Crie o arquivo backend_api/database.py com o seguinte conteúdo:

``` 
Python

from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, Enum
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime
import enum
import os
from dotenv import load_dotenv

# Carrega variáveis de ambiente do arquivo .env
load_dotenv()

# Use SQLite para este exemplo rápido. Para PostgreSQL/MySQL, mude a string de conexão.
# DATABASE_URL é lida do arquivo .env ou usa SQLite como padrão.
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./air_quality.db")

engine = create_engine(DATABASE_URL)
Base = declarative_base()
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# --- Modelos ORM (SQLAlchemy) ---

class StationType(enum.Enum):
    AUTOMATIC = "automatic"
    MANUAL = "manual"
    LOW_COST = "low_cost" # Adicionado tipo de estação de baixo custo

class Station(Base):
    __tablename__ = "stations"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    latitude = Column(Float, nullable=False)
    longitude = Column(Float, nullable=False)
    station_type = Column(Enum(StationType), nullable=False)

class PollutantReading(Base):
    __tablename__ = "pollutant_readings"
    id = Column(Integer, primary_key=True, index=True)
    station_id = Column(Integer, index=True) # Poderia ser ForeignKey para Station
    pollutant_type = Column(String, index=True) # Ex: 'PM2.5', 'O3', 'CO', 'NO2'
    value = Column(Float, nullable=False)
    unit = Column(String) # Ex: 'µg/m³', 'ppm'
    timestamp = Column(DateTime, default=datetime.utcnow)

class MobilityData(Base):
    __tablename__ = "mobility_data"
    id = Column(Integer, primary_key=True, index=True)
    data_type = Column(String, index=True) # Ex: 'CarDensity', 'BikeUsage', 'Emissao_CO2'
    value = Column(Float, nullable=False)
    latitude = Column(Float) # Pode ser nulo para dados gerais de área
    longitude = Column(Float) # Pode ser nulo para dados gerais de área
    area_name = Column(String) # Ex: 'Plano Piloto', 'Guará', 'Distrito Federal (Total)'
    timestamp = Column(DateTime, default=datetime.utcnow)
    unit = Column(String) # Adicionado para dados de emissão

class GreenhouseGasReading(Base):
    __tablename__ = "greenhouse_gas_readings"
    id = Column(Integer, primary_key=True, index=True)
    station_id = Column(Integer, index=True) # Pode ser nulo se for emissão veicular agregada (sem estação específica)
    gas_type = Column(String, index=True) # Ex: 'CO2', 'CH4', 'N2O'
    value = Column(Float, nullable=False)
    unit = Column(String)
    timestamp = Column(DateTime, default=datetime.utcnow)

# Função para criar as tabelas no banco de dados (rode uma vez para inicializar)
def create_db_tables():
    Base.metadata.create_all(bind=engine)

# Dependência para obter uma sessão de banco de dados por requisição
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Exemplo de uso (execute uma vez para criar o DB):
if __name__ == "__main__":
    create_db_tables()
    print("Tabelas do banco de dados criadas ou já existentes.")

```

Crie o arquivo .env na raiz da pasta backend_api/ com o seguinte conteúdo:

```
DATABASE_URL="sqlite:///./air_quality.db"
# Se for PostgreSQL, use:
# DATABASE_URL="postgresql://user:password@host:port/dbname"

```
Comando no Terminal:

```
python backend_api/database.py
```

Critérios de Aceitação:

Arquivo air_quality.db criado na pasta backend_api/ (se usar SQLite).
Mensagem "Tabelas do banco de dados criadas ou já existentes." no terminal.
Nenhum erro reportado durante a execução.


## Issue #3: Configurar o Servidor FastAPI e CORS
#### Descrição: Inicie a aplicação FastAPI e configure as políticas de CORS (Cross-Origin Resource Sharing) para permitir requisições do frontend WordPress.

Código Relacionado:

Crie o arquivo backend_api/models.py com o seguinte conteúdo:
Python

```

from pydantic import BaseModel
from datetime import datetime
from typing import Optional, List
import enum

class StationTypeEnum(str, enum.Enum):
    AUTOMATIC = "automatic"
    MANUAL = "manual"
    LOW_COST = "low_cost"

class StationBase(BaseModel):
    name: str
    latitude: float
    longitude: float
    station_type: StationTypeEnum

class StationCreate(StationBase):
    pass

class Station(StationBase):
    id: int
    class Config:
        orm_mode = True

class PollutantReadingBase(BaseModel):
    station_id: int
    pollutant_type: str
    value: float
    unit: Optional[str] = None
    timestamp: datetime = datetime.utcnow()

class PollutantReadingCreate(PollutantReadingBase):
    pass

class PollutantReading(PollutantReadingBase):
    id: int
    class Config:
        orm_mode = True

class MobilityDataBase(BaseModel):
    data_type: str
    value: float
    latitude: Optional[float] = None
    longitude: Optional[float] = None
    area_name: Optional[str] = None
    timestamp: datetime = datetime.utcnow()
    unit: Optional[str] = None

class MobilityDataCreate(MobilityDataBase):
    pass

class MobilityData(MobilityDataBase):
    id: int
    class Config:
        orm_mode = True

class GreenhouseGasReadingBase(BaseModel):
    station_id: Optional[int] = None
    gas_type: str
    value: float
    unit: Optional[str] = None
    timestamp: datetime = datetime.utcnow()

class GreenhouseGasReadingCreate(GreenhouseGasReadingBase):
    pass

class GreenhouseGasReading(GreenhouseGasReadingBase):
    id: int
    class Config:
        orm_mode = True

# Modelo para retorno de dados combinados no mapa
class MapData(BaseModel):
    stations: List[Station]
    pollutant_readings: List[PollutantReading]
    mobility_data: List[MobilityData]
    greenhouse_gas_readings: List[GreenhouseGasReading]

```

Crie o arquivo backend_api/crud.py com o seguinte conteúdo:

```
Python

from sqlalchemy.orm import Session
from . import database, models as db_models
from .models import StationCreate, PollutantReadingCreate, MobilityDataCreate, GreenhouseGasReadingCreate, StationTypeEnum
from typing import List, Optional
from datetime import datetime

# --- Funções CRUD para Stations ---
def get_station(db: Session, station_id: int):
    return db.query(db_models.Station).filter(db_models.Station.id == station_id).first()

def get_stations(db: Session, skip: int = 0, limit: int = 100, station_type: Optional[StationTypeEnum] = None):
    query = db.query(db_models.Station)
    if station_type:
        query = query.filter(db_models.Station.station_type == station_type)
    return query.offset(skip).limit(limit).all()

def create_station(db: Session, station: StationCreate):
    db_station = db_models.Station(**station.dict())
    db.add(db_station)
    db.commit()
    db.refresh(db_station)
    return db_station

# --- Funções CRUD para PollutantReadings ---
def get_pollutant_readings(db: Session, station_id: Optional[int] = None, pollutant_type: Optional[str] = None,
                           start_time: Optional[datetime] = None, end_time: Optional[datetime] = None,
                           station_type: Optional[StationTypeEnum] = None, # Filtrar por tipo de estação
                           skip: int = 0, limit: int = 100):
    query = db.query(db_models.PollutantReading)
    if station_id:
        query = query.filter(db_models.PollutantReading.station_id == station_id)
    if pollutant_type:
        query = query.filter(db_models.PollutantReading.pollutant_type == pollutant_type)
    if start_time:
        query = query.filter(db_models.PollutantReading.timestamp >= start_time)
    if end_time:
        query = query.filter(db_models.PollutantReading.timestamp <= end_time)
    if station_type:
        # Fazer um join com a tabela de estações para filtrar por tipo
        query = query.join(db_models.Station).filter(db_models.Station.station_type == station_type)

    return query.order_by(db_models.PollutantReading.timestamp.desc()).offset(skip).limit(limit).all()

def create_pollutant_reading(db: Session, reading: PollutantReadingCreate):
    db_reading = db_models.PollutantReading(**reading.dict())
    db.add(db_reading)
    db.commit()
    db.refresh(db_reading)
    return db_reading

# --- Funções CRUD para MobilityData ---
def get_mobility_data(db: Session, data_type: Optional[str] = None, area_name: Optional[str] = None,
                      start_time: Optional[datetime] = None, end_time: Optional[datetime] = None,
                      skip: int = 0, limit: int = 100, data_type_startswith: Optional[str] = None): # Adicionado para filtro like
    query = db.query(db_models.MobilityData)
    if data_type:
        query = query.filter(db_models.MobilityData.data_type == data_type)
    if data_type_startswith:
        query = query.filter(db_models.MobilityData.data_type.startswith(data_type_startswith))
    if area_name:
        query = query.filter(db_models.MobilityData.area_name == area_name)
    if start_time:
        query = query.filter(db_models.MobilityData.timestamp >= start_time)
    if end_time:
        query = query.filter(db_models.MobilityData.timestamp <= end_time)
    return query.order_by(db_models.MobilityData.timestamp.desc()).offset(skip).limit(limit).all()

def create_mobility_data(db: Session, data: MobilityDataCreate):
    db_data = db_models.MobilityData(**data.dict())
    db.add(db_data)
    db.commit()
    db.refresh(db_data)
    return db_data

# --- Funções CRUD para GreenhouseGasReadings ---
def get_greenhouse_gas_readings(db: Session, station_id: Optional[int] = None, gas_type: Optional[str] = None,
                                start_time: Optional[datetime] = None, end_time: Optional[datetime] = None,
                                skip: int = 0, limit: int = 100):
    query = db.query(db_models.GreenhouseGasReading)
    if station_id:
        query = query.filter(db_models.GreenhouseGasReading.station_id == station_id)
    if gas_type:
        query = query.filter(db_models.GreenhouseGasReading.gas_type == gas_type)
    if start_time:
        query = query.filter(db_models.GreenhouseGasReading.timestamp >= start_time)
    if end_time:
        query = query.filter(db_models.GreenhouseGasReading.timestamp <= end_time)
    return query.order_by(db_models.GreenhouseGasReading.timestamp.desc()).offset(skip).limit(limit).all()

def create_greenhouse_gas_reading(db: Session, reading: GreenhouseGasReadingCreate):
    db_reading = db_models.GreenhouseGasReading(**reading.dict())
    db.add(db_reading)
    db.commit()
    db.refresh(db_reading)
    return db_reading

```

Crie o arquivo backend_api/main.py com o seguinte conteúdo. Atenção: A linha origins deve ser ajustada para o seu domínio WordPress.

```
Python

from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List, Optional
from datetime import datetime

from . import crud, models, database
from .database import SessionLocal, engine, get_db
from fastapi.middleware.cors import CORSMiddleware # Importa CORSMiddleware

# Cria as tabelas do banco de dados na inicialização se ainda não existirem
database.create_db_tables()

app = FastAPI(
    title="API de Qualidade do Ar e Mobilidade do DF",
    description="API para consultar dados de estações de poluentes, gases de efeito estufa e mobilidade urbana no Distrito Federal.",
    version="1.0.0"
)

# Configuração do CORS para permitir requisições do seu frontend WordPress
origins = [
    "http://localhost", # Se você estiver testando localmente
    "http://localhost:8000", # Se o seu frontend estiver em outra porta local
    "http://seu-dominio-wordpress.com", # <--- SUBSTITUA PELO DOMÍNIO REAL DO SEU WORDPRESS
    "https://seu-dominio-wordpress.com", # <--- SUBSTITUA PELO DOMÍNIO REAL DO SEU WORDPRESS
    # Adicione outros domínios ou IPs se necessário
    # Ex: "http://192.168.1.100"
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"], # Permite todos os métodos (GET, POST, etc.)
    allow_headers=["*"], # Permite todos os cabeçalhos
)

# --- Endpoints (apenas placeholders mínimos para iniciar o servidor) ---
@app.post("/api/stations/", response_model=models.Station, status_code=status.HTTP_201_CREATED)
def create_station(station: models.StationCreate, db: Session = Depends(get_db)):
    return [] # Placeholder

@app.get("/api/stations/", response_model=List[models.Station])
def read_stations(skip: int = 0, limit: int = 100, station_type: Optional[models.StationTypeEnum] = None, db: Session = Depends(get_db)):
    return []

@app.get("/api/stations/{station_id}", response_model=models.Station)
def read_station(station_id: int, db: Session = Depends(get_db)):
    raise HTTPException(status_code=404, detail="Station not found") # Placeholder

@app.post("/api/pollutant_readings/", response_model=models.PollutantReading, status_code=status.HTTP_201_CREATED)
def create_pollutant_reading(reading: models.PollutantReadingCreate, db: Session = Depends(get_db)):
    return [] # Placeholder

@app.get("/api/pollutant_readings/", response_model=List[models.PollutantReading])
def read_pollutant_readings(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return []

@app.post("/api/mobility_data/", response_model=models.MobilityData, status_code=status.HTTP_201_CREATED)
def create_mobility_data(data: models.MobilityDataCreate, db: Session = Depends(get_db)):
    return [] # Placeholder

@app.get("/api/mobility_data/", response_model=List[models.MobilityData])
def read_mobility_data(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return []

@app.post("/api/greenhouse_gas_readings/", response_model=models.GreenhouseGasReading, status_code=status.HTTP_201_CREATED)
def create_greenhouse_gas_reading(reading: models.GreenhouseGasReadingCreate, db: Session = Depends(get_db)):
    return [] # Placeholder

@app.get("/api/greenhouse_gas_readings/", response_model=List[models.GreenhouseGasReading])
def read_greenhouse_gas_readings(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return []

# --- Endpoint Mínimo Mockado para o Mapa (Será preenchido na Issue #4) ---
@app.get("/api/map_data/", response_model=models.MapData)
def get_map_data(db: Session = Depends(get_db)):
    # Dados mockados mínimos, a serem substituídos na Issue #4
    return models.MapData(
        stations=[],
        pollutant_readings=[],
        mobility_data=[],
        greenhouse_gas_readings=[]
    )

# --- Endpoint para Índice de Qualidade do Ar (Placeholder) ---
@app.get("/api/air_quality_index/", response_model=dict)
def get_air_quality_index(region_name: Optional[str] = None, db: Session = Depends(get_db)):
    return {"status": "implementar cálculo do IQA aqui", "details": "Dados agregados por região ou geral"}

# --- Endpoint para Dados Agregados de Regiões (Placeholder) ---
@app.get("/api/regions_summary/", response_model=dict)
def get_regions_summary(db: Session = Depends(get_db)):
    return {"status": "implementar agregação de dados por região aqui", "data": {}}
Comando no Terminal:

Bash

uvicorn backend_api.main:app --reload
Critérios de Aceitação:

Servidor FastAPI rodando em http://127.0.0.1:8000.
Acessar http://127.0.0.1:8000/docs e ver a documentação interativa.
Acessar /api/stations/, /api/pollutant_readings/, /api/mobility_data/, /api/greenhouse_gas_readings/ e /api/map_data/ e ver que todos retornam listas vazias ([]) ou um MapData vazio.
Nenhum erro de CORS deve aparecer no console do navegador (isso será testado em issues futuras, mas a configuração já deve estar lá).
Issue #4: Implementar Endpoint Mínimo da API para Dados do Mapa
Descrição: Crie o endpoint /api/map_data/ na API Python que retorne uma estrutura JSON mínima e fixa (mockada) para estações, leituras de poluentes e dados de mobilidade. O objetivo é apenas ter algo para o frontend consumir inicialmente.

Código Relacionado:

Modifique o arquivo backend_api/main.py. Substitua a função get_map_data que retorna dados vazios pelo seguinte conteúdo:
Python

# backend_api/main.py
# ... (código existente da importação de FastAPI, CORS, etc.)
from datetime import datetime # Certifique-se de que datetime é importado

# ... (todos os outros endpoints permanecem como estão)

# --- Endpoint Mínimo Mockado para o Mapa ---
@app.get("/api/map_data/", response_model=models.MapData)
def get_map_data(db: Session = Depends(get_db)): # db é necessário, mesmo que não usado com dados mockados
    # Dados mockados mínimos para testar o frontend
    mock_stations = [
        models.Station(id=1, name="Estacao Mock Auto", latitude=-15.78, longitude=-47.88, station_type=models.StationTypeEnum.AUTOMATIC),
        models.Station(id=2, name="Estacao Mock Manual", latitude=-15.80, longitude=-47.90, station_type=models.StationTypeEnum.MANUAL)
    ]
    mock_pollutant_readings = [
        models.PollutantReading(id=1, station_id=1, pollutant_type="PM2.5", value=35.5, unit="ug/m3", timestamp=datetime.utcnow()),
        models.PollutantReading(id=2, station_id=2, pollutant_type="CO", value=1.2, unit="ppm", timestamp=datetime.utcnow())
    ]
    mock_mobility_data = [
        models.MobilityData(id=1, data_type="CarDensity", value=500.0, latitude=-15.79, longitude=-47.89, area_name="Asa Sul", timestamp=datetime.utcnow(), unit="carros/km2"),
        models.MobilityData(id=2, data_type="Emissao_CO2", value=1000000.0, latitude=-15.7801, longitude=-47.9292, area_name="Distrito Federal (Total)", timestamp=datetime.utcnow(), unit="g")
    ]
    mock_greenhouse_gas_readings = [
        models.GreenhouseGasReading(id=1, station_id=1, gas_type="CO2", value=450.0, unit="ppm", timestamp=datetime.utcnow())
    ]

    return models.MapData(
        stations=mock_stations,
        pollutant_readings=mock_pollutant_readings,
        mobility_data=mock_mobility_data,
        greenhouse_gas_readings=mock_greenhouse_gas_readings
    )

```

Critérios de Aceitação:

Servidor FastAPI reiniciado (se estiver usando --reload, já deve ter reiniciado).
Acessar http://127.0.0.1:8000/api/map_data/ e observar um JSON contendo os dados mockados definidos acima. A estrutura do JSON deve ser compatível com os modelos MapData.


## Issue #5: Obter Chave da API do Google Maps
#### Descrição: Crie um projeto no Google Cloud Console, habilite a "Maps JavaScript API" e gere uma chave de API para uso no frontend.

Critérios de Aceitação:

Você deve ter uma chave de API do Google Maps em mãos (começa com AIza...).
(Opcional, mas recomendado) Restrições de API configuradas para permitir apenas o seu domínio WordPress e a Maps JavaScript API.
Issue #6: Configurar Plugin WordPress e Carregar Google Maps API
Descrição: Instale o plugin WordPress e configure o arquivo principal (poluicao-df-dashboard.php) para carregar a Google Maps API, substituindo a chave de API e o callback.

Código Relacionado:
```
Crie a pasta poluicao-df-dashboard/ dentro de wp-content/plugins/ do seu WordPress.
Dentro dela, crie o arquivo poluicao-df-dashboard.php com o seguinte conteúdo. Lembre-se de substituir SUA_CHAVE_API_GOOGLE e a URL da API.
PHP

<?php
/**
 * Plugin Name: Poluição DF Dashboard
 * Description: Integra dados de qualidade do ar e mobilidade do DF de uma API Python para visualização no WordPress.
 * Version: 1.0
 * Author: Seu Nome
 */

// Define a URL base da sua API Python
define('POLUICAO_DF_API_BASE_URL', 'http://127.0.0.1:8000/api/'); // <--- Mude para o endereço REAL da sua API em produção!

// --- Funções para Enfileirar Scripts e Estilos ---
function poluicao_df_dashboard_enqueue_scripts() {
    // Carrega o Google Maps API (substitua 'SUA_CHAVE_API_GOOGLE' pela sua chave real)
    // Recomenda-se carregar de forma assíncrona e adiar a execução
    wp_enqueue_script(
        'google-maps-api',
        'https://maps.googleapis.com/maps/api/js?key=SUA_CHAVE_API_GOOGLE&callback=initMap', // <--- SUBSTITUA A CHAVE AQUI!
        [],
        null,
        true // Carrega no rodapé
    );

    // Carrega a biblioteca Chart.js para gráficos (necessária para dashboards futuros)
    wp_enqueue_script(
        'chart-js',
        'https://cdn.jsdelivr.net/npm/chart.js@4.4.2/dist/chart.umd.min.js',
        [],
        '4.4.2',
        true
    );

    // Scripts para todas as janelas que precisam da URL da API
    $dashboard_scripts = [
        'google-maps-integrator',
        // 'regions-dashboard', # Serão ativados em issues futuras
        // 'automatic-stations-dashboard',
        // 'manual-stations-dashboard',
        // 'low-cost-stations-dashboard',
        // 'comparison-dashboard',
    ];

    foreach ($dashboard_scripts as $handle) {
        // Prepend 'poluicao-df-' para o handle completo
        $full_handle = 'poluicao-df-' . $handle;
        wp_register_script(
            $full_handle,
            plugins_url('assets/js/' . $handle . '.js', __FILE__),
            ['google-maps-api', 'chart-js'], // Dependências comuns
            '1.0',
            true
        );
        wp_localize_script(
            $full_handle,
            'poluicaoDfApiSettings',
            array(
                'apiUrl' => POLUICAO_DF_API_BASE_URL,
            )
        );
    }

    // Carrega o CSS do plugin
    wp_enqueue_style(
        'poluicao-df-dashboard-style',
        plugins_url('assets/css/poluicao-df-dashboard.css', __FILE__),
        [],
        '1.0'
    );
}
add_action('wp_enqueue_scripts', 'poluicao_df_dashboard_enqueue_scripts');

// --- Funções para Shortcodes (apenas o do mapa por enquanto) ---

/**
 * Shortcode para exibir o mapa do Google com os balões de poluentes e mobilidade.
 * Uso: [poluicao_df_mapa]
 */
function poluicao_df_mapa_shortcode() {
    // Enfileira o script específico do mapa apenas quando o shortcode é usado
    wp_enqueue_script('poluicao-df-google-maps-integrator');

    ob_start(); // Inicia o buffer de saída
    ?>
    <div id="map" style="height: 600px; width: 100%;"></div>
    <div id="map-legend" style="margin-top: 10px;">
        <h3>Legenda</h3>
        <p><span style="color: green; font-weight: bold;">●</span> Qualidade Boa / Baixas Emissões</p>
        <p><span style="color: orange; font-weight: bold;">●</span> Qualidade Moderada / Médias Emissões</p>
        <p><span style="color: red; font-weight: bold;">●</span> Qualidade Ruim / Altas Emissões</p>
        <p><span style="background-color: lightblue; padding: 0 5px; border-radius: 3px;">Mob.</span> Dados de Mobilidade/Emissões Veiculares (marcador azul)</p>
        <p><span style="background-color: lightgray; padding: 0 5px; border-radius: 3px;">Est.</span> Estações de Poluentes (marcador cinza)</p>
    </div>
    <?php
    return ob_get_clean(); // Retorna o conteúdo do buffer
}
add_shortcode('poluicao_df_mapa', 'poluicao_df_mapa_shortcode');

// Função de callback para o Google Maps
// Essa função precisa ser global para ser acessada pelo script do Google Maps
function initMap() {
    // Esta função será preenchida pelo JavaScript do seu plugin (google-maps-integrator.js)
    // O WordPress garante que o google-maps-integrator.js será carregado.
    // O JavaScript precisa definir 'window.initMap = function() { ... }'
}
Dentro da pasta poluicao-df-dashboard/, crie as subpastas assets/css/ e assets/js/.
Crie o arquivo wordpress_frontend/wp-content/plugins/poluicao-df-dashboard/assets/css/poluicao-df-dashboard.css com o seguinte conteúdo:
CSS

/* /wp-content/plugins/poluicao-df-dashboard/assets/css/poluicao-df-dashboard.css */

#map {
    height: 600px;
    width: 100%;
    border: 1px solid #ccc;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

#map-legend {
    background-color: #f9f9f9;
    border: 1px solid #ddd;
    padding: 15px;
    margin-top: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

#map-legend h3 {
    color: #333;
    border-bottom: 1px solid #eee;
    padding-bottom: 8px;
    margin-top: 0;
    margin-bottom: 10px;
}

#map-legend p {
    margin: 5px 0;
    font-size: 0.9em;
    color: #555;
}

#map-legend span {
    font-size: 1.2em;
    vertical-align: middle;
    margin-right: 8px;
    display: inline-block; /* Para alinhar melhor */
    width: 1.2em; /* Garante largura para o círculo */
    text-align: center;
}

/* Classes gerais para dashboards futuros */
.poluicao-df-dashboard-section {
    background-color: #f9f9f9;
    border: 1px solid #ddd;
    padding: 20px;
    margin-bottom: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.poluicao-df-dashboard-section h2 {
    color: #333;
    border-bottom: 2px solid #eee;
    padding-bottom: 10px;
    margin-top: 0;
}

.poluicao-df-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 15px;
}

.poluicao-df-table th, .poluicao-df-table td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: left;
}

.poluicao-df-table th {
    background-color: #eef;
    font-weight: bold;
}

.poluicao-df-table tbody tr:nth-child(even) {
    background-color: #f5f5f5;
}

.poluicao-df-table tbody tr:hover {
    background-color: #e0e0e0;
}

/* Estilos para os controles de filtro */
.controls {
    margin-bottom: 20px;
    display: flex;
    gap: 15px;
    align-items: center;
    flex-wrap: wrap;
}

.controls label {
    font-weight: bold;
}

.controls select, .controls button {
    padding: 8px 12px;
    border: 1px solid #ccc;
    border-radius: 4px;
}

.controls button {
    background-color: #0073aa;
    color: white;
    cursor: pointer;
    border-color: #0073aa;
}

.controls button:hover {
    background-color: #005177;
}

```

Critérios de Aceitação:

Plugin ativado no painel de administração do WordPress.
A URL da sua API no poluicao-df-dashboard.php está correta.
A chave da API do Google Maps foi inserida no local correto.

## Issue #7: Criar Página WordPress e Exibir Mapa Básico
#### Descrição: Crie uma nova página no WordPress e use o shortcode do plugin para exibir um mapa do Google centralizado no Distrito Federal.

Código Relacionado:

Não há código para criar/modificar nesta issue. Você usará o shortcode já definido na Issue #6.
Passos no WordPress:

No painel de administração do WordPress, vá em Páginas > Adicionar Nova.
Dê um título à página (ex: "Mapa de Qualidade do Ar DF").
No editor de conteúdo, insira o shortcode: [poluicao_df_mapa]
Publique a página.
Critérios de Aceitação:

Uma nova página WordPress está publicada.
Ao visitar a página no navegador, o mapa do Google é exibido, centralizado no DF, sem marcadores personalizados ainda.
Não há erros de JavaScript ou CORS no console do navegador (F12).
Issue #8: Conectar Frontend ao Endpoint de Dados do Mapa e Exibir Marcadores Mockados
Descrição: Modifique o JavaScript do frontend para chamar o endpoint /api/map_data/ da API Python e usar os dados mockados para renderizar os marcadores (estações, mobilidade) e os balões (regiões) no mapa.

Código Relacionado:

Crie o arquivo wordpress_frontend/wp-content/plugins/poluicao-df-dashboard/assets/js/google-maps-integrator.js com o seguinte conteúdo:

```
JavaScript

// /wp-content/plugins/poluicao-df-dashboard/assets/js/google-maps-integrator.js

// Garante que o objeto global para a API esteja disponível
// (poluicaoDfApiSettings é injetado pelo wp_localize_script no PHP)
if (typeof poluicaoDfApiSettings === 'undefined') {
    console.error('poluicaoDfApiSettings não está definido. Verifique a função wp_localize_script no seu plugin PHP.');
}

const API_BASE_URL = poluicaoDfApiSettings.apiUrl;
let map;

// Callback obrigatório para o Google Maps API. Precisa ser uma função global.
window.initMap = async function() {
    console.log('Google Maps API carregado. Inicializando mapa...');
    const dfCoords = { lat: -15.7801, lng: -47.9292 }; // Centro do Distrito Federal

    map = new google.maps.Map(document.getElementById('map'), {
        zoom: 10,
        center: dfCoords,
    });

    // Chama a API para obter os dados do mapa
    const mapData = await getMapData();
    if (mapData) {
        displayMarkersAndCircles(mapData);
    } else {
        console.warn('Não foi possível carregar dados para o mapa.');
    }
};

// Função para buscar dados da sua API Python
async function getMapData() {
    try {
        console.log(`Buscando dados do mapa em: ${API_BASE_URL}map_data/`);
        const response = await fetch(`${API_BASE_URL}map_data/`); // Seu endpoint combinado
        if (!response.ok) {
            throw new Error(`Erro HTTP: ${response.status}`);
        }
        const data = await response.json();
        console.log('Dados do mapa recebidos:', data);
        return data;
    } catch (error) {
        console.error("Erro ao buscar dados para o mapa:", error);
        // Pode exibir uma mensagem de erro na página ou um fallback
        return null;
    }
}

// --- Lógica para exibir Marcadores e Círculos ---
function displayMarkersAndCircles(mapData) {
    // Dados de regiões mockados para os balões coloridos.
    // Em uma fase futura, esses dados viriam de um endpoint específico da API.
    const dfRegions = [
        { name: 'Plano Piloto', coords: { lat: -15.795, lng: -47.882 }, currentAQI: 'Good', totalEmissions: 1000 },
        { name: 'Taguatinga', coords: { lat: -15.833, lng: -48.058 }, currentAQI: 'Moderate', totalEmissions: 2500 },
        { name: 'Ceilândia', coords: { lat: -15.811, lng: -48.109 }, currentAQI: 'Bad', totalEmissions: 4000 },
    ];

    // Crie os "balões" por região (Circle Overlays)
    dfRegions.forEach(region => {
        let fillColor;
        if (region.currentAQI === 'Good') {
            fillColor = 'green';
        } else if (region.currentAQI === 'Moderate') {
            fillColor = 'orange';
        } else {
            fillColor = 'red';
        }

        const cityCircle = new google.maps.Circle({
            strokeColor: fillColor,
            strokeOpacity: 0.8,
            strokeWeight: 2,
            fillColor: fillColor,
            fillOpacity: 0.35,
            map,
            center: region.coords,
            radius: 5000, // Raio em metros (ajuste conforme a área da região)
        });

        // Adiciona um listener para mostrar info ao clicar no círculo
        cityCircle.addListener('click', () => {
            const infoWindow = new google.maps.InfoWindow({
                content: `
                    <div>
                        <h4>${region.name}</h4>
                        <p>Qualidade do Ar: <strong>${region.currentAQI}</strong></p>
                        <p>Emissões Veiculares Estimadas: ${region.totalEmissions} kg CO2e</p>
                        <p>Clique para mais detalhes na página de regiões.</p>
                    </div>
                `,
            });
            infoWindow.open(map, cityCircle);
        });

        // Opcional: Adicione um marcador de texto para o nome da região
        new google.maps.Marker({
            position: region.coords,
            map,
            label: {
                text: region.name,
                color: 'black',
                fontSize: '12px',
                fontWeight: 'bold'
            },
            icon: { // Ícone transparente para que apenas o texto apareça
                path: google.maps.SymbolPath.CIRCLE,
                scale: 0,
            }
        });
    });

    // Marcadores para Estações de Poluentes (cinza)
    mapData.stations.forEach(station => {
        const marker = new google.maps.Marker({
            position: { lat: station.latitude, lng: station.longitude },
            map,
            icon: {
                path: google.maps.SymbolPath.CIRCLE,
                fillColor: 'lightgray', // Cor para estações
                fillOpacity: 0.8,
                strokeWeight: 0,
                scale: 8,
            },
            title: `Estação: ${station.name} (${station.station_type})`
        });

        // Exibe as últimas leituras de poluentes daquela estação
        const stationReadings = mapData.pollutant_readings.filter(r => r.station_id === station.id)
                                                    .sort((a,b) => new Date(b.timestamp) - new Date(a.timestamp)); // Mais recentes primeiro
        const latestReadings = {}; // Para mostrar apenas a leitura mais recente de cada tipo de poluente
        stationReadings.forEach(reading => {
            if (!latestReadings[reading.pollutant_type]) {
                latestReadings[reading.pollutant_type] = reading;
            }
        });

        let infoContent = `<h4>Estação: ${station.name}</h4><p>Tipo: ${station.station_type}</p>`;
        const latestReadingValues = Object.values(latestReadings);
        if (latestReadingValues.length > 0) {
            infoContent += '<h5>Últimas Leituras:</h5><ul>';
            latestReadingValues.forEach(reading => {
                infoContent += `<li>${reading.pollutant_type}: ${reading.value} ${reading.unit} (${new Date(reading.timestamp).toLocaleString()})</li>`;
            });
            infoContent += '</ul>';
        } else {
            infoContent += '<p>Nenhuma leitura recente disponível.</p>';
        }

        const infoWindow = new google.maps.InfoWindow({ content: infoContent });
        marker.addListener('click', () => {
            infoWindow.open(map, marker);
        });
    });

    // Marcadores para Dados de Mobilidade/Emissões Veiculares (azul)
    mapData.mobility_data.forEach(mobility => {
        // Apenas para dados com coordenadas e que não sejam dados de emissão agregada total do DF
        if (mobility.latitude && mobility.longitude && !mobility.data_type.startsWith('Emissao_')) {
            const marker = new google.maps.Marker({
                position: { lat: mobility.latitude, lng: mobility.longitude },
                map,
                icon: {
                    path: google.maps.SymbolPath.CIRCLE,
                    fillColor: 'lightblue', // Cor para mobilidade
                    fillOpacity: 0.8,
                    strokeWeight: 0,
                    scale: 8,
                },
                title: `Dados de Mobilidade: ${mobility.area_name || 'Desconhecido'}`
            });

            const infoWindow = new google.maps.InfoWindow({
                content: `
                    <h4>Dados de Mobilidade</h4>
                    <p>Tipo: ${mobility.data_type}</p>
                    <p>Valor: ${mobility.value} ${mobility.unit || ''}</p>
                    <p>Área: ${mobility.area_name || 'N/A'}</p>
                    <p>Data: ${new Date(mobility.timestamp).toLocaleString()}</p>
                `,
            });
            marker.addListener('click', () => {
                infoWindow.open(map, marker);
            });
        }
    });

    // Adicionar marcador para emissões veiculares totais do DF (se presente e tiver lat/lng)
    const dfTotalEmissions = mapData.mobility_data.find(d => d.data_type === 'Emissao_CO2' && d.area_name === 'Distrito Federal (Total)');
    if (dfTotalEmissions && dfTotalEmissions.latitude && dfTotalEmissions.longitude) {
        const emissionMarker = new google.maps.Marker({
            position: { lat: dfTotalEmissions.latitude, lng: dfTotalEmissions.longitude },
            map,
            label: {
                text: `CO2 Total: ${(dfTotalEmissions.value / 1000000).toFixed(2)} ton`, // Converte g para toneladas
                color: 'purple',
                fontSize: '14px',
                fontWeight: 'bold'
            },
            icon: {
                path: google.maps.SymbolPath.FORWARD_CLOSED_ARROW, // Ou um ícone de fumaça
                fillColor: 'purple',
                fillOpacity: 0.8,
                strokeWeight: 0,
                scale: 10,
            },
            title: 'Emissões Totais de Veículos no DF'
        });
        const infoWindow = new google.maps.InfoWindow({
            content: `
                <h4>Emissões Veiculares Totais (DF)</h4>
                <p>CO2: ${(dfTotalEmissions.value / 1000000).toFixed(2)} toneladas</p>
                <p>Estimativa baseada em dados de KM.</p>
            `,
        });
        emissionMarker.addListener('click', () => {
            infoWindow.open(map, emissionMarker);
        });
    }
}
```


Critérios de Aceitação:

Ao carregar a página do mapa no WordPress, os marcadores e balões (baseados nos dados mockados da API) aparecem no mapa.
Ao clicar nos marcadores e balões, as InfoWindows exibem os detalhes dos dados mockados.
Não há erros no console do navegador (F12).