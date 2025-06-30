
# Milestone: Ingestão de Dados Reais e Preenchimento da API

Nesta etapa, você irá popular o seu banco de dados com dados reais (ou mais robustos) e ajustar os endpoints da API para consultá-los. Também abordaremos a implantação básica da API em uma VM e como o WordPress se conectará a ela.

## Issue #9: Criar CSV de Dados de Veículos de Exemplo (Prontos para Ingestão)

#### Descrição: Crie um arquivo CSV de exemplo com dados de veículos (tipo, ano, quilometragem) que serão usados para calcular as emissões. Este será o arquivo de entrada para a próxima tarefa.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/veiculos_km.csv com o seguinte conteúdo. Adicione mais linhas se desejar mais dados:

```
tipo_veiculo,ano_fabricacao,quilometragem_total
carro_gasolina,2010,120000
carro_diesel,2015,80000
onibus_diesel,2005,500000
carro_flex,2018,60000
moto_gasolina,2020,30000
carro_gasolina,2012,95000
onibus_diesel,2010,400000
```

Critérios de Aceitação:

O arquivo veiculos_km.csv existe dentro da pasta backend_api/data_ingestion/.

O conteúdo do CSV está no formato esperado (cabeçalhos e dados).


## Issue #10: Implementar Cálculo de Emissões Veiculares (Geração CSV)

Descrição: Desenvolva o script que lê o CSV de veículos, aplica fatores de emissão (simulados por enquanto, mas com espaço para dados reais) e gera um novo CSV com as emissões calculadas. Este CSV de saída será o insumo para a ingestão no banco de dados.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/calculate_vehicle_emissions.py com o seguinte conteúdo:
```Python

import pandas as pd
from datetime import datetime
import os

# --- 1. FATORES DE EMISSÃO (EXEMPLO - SUBSTITUA POR DADOS REAIS E CONFIÁVEIS) ---
# Estes são dados fictícios e simplificados, em gramas por quilômetro (g/km).
# É CRUCIAL SUBSTITUIR ESTES VALORES POR DADOS DE INVENTÁRIOS DE EMISSÃO NACIONAIS/LOCAIS
# OU LITERATURA CIENTÍFICA RELEVANTE PARA O BRASIL E O DF.
EMISSION_FACTORS = {
    'carro_gasolina': {
        'CO2': {2000: 200, 2010: 180, 2015: 160, 2020: 140},
        'CH4': {2000: 0.05, 2010: 0.04, 2015: 0.03, 2020: 0.02},
        'N2O': {2000: 0.005, 2010: 0.004, 2015: 0.003, 2020: 0.002},
        'PM2.5': {2000: 0.01, 2010: 0.008, 2015: 0.005, 2020: 0.003},
        'CO': {2000: 1.5, 2010: 1.0, 2015: 0.8, 2020: 0.5},
        'NOx': {2000: 0.2, 2010: 0.15, 2015: 0.1, 2020: 0.08},
    },
    'carro_diesel': {
        'CO2': {2000: 220, 2010: 200, 2015: 180, 2020: 160},
        'CH4': {2000: 0.03, 2010: 0.02, 2015: 0.015, 2020: 0.01},
        'N2O': {2000: 0.004, 2010: 0.003, 2015: 0.002, 2020: 0.001},
        'PM2.5': {2000: 0.05, 2010: 0.03, 2015: 0.02, 2020: 0.01},
        'CO': {2000: 0.8, 2010: 0.5, 2015: 0.3, 2020: 0.2},
        'NOx': {2000: 0.5, 2010: 0.4, 2015: 0.3, 2020: 0.2},
    },
    'onibus_diesel': {
        'CO2': {2000: 800, 2010: 700, 2015: 600, 2020: 500},
        'CH4': {2000: 0.1, 2010: 0.08, 2015: 0.06, 2020: 0.05},
        'N2O': {2000: 0.01, 2010: 0.008, 2015: 0.006, 2020: 0.005},
        'PM2.5': {2000: 0.2, 2010: 0.15, 2015: 0.1, 2020: 0.05},
        'CO': {2000: 3.0, 2010: 2.0, 2015: 1.5, 2020: 1.0},
        'NOx': {2000: 1.5, 2010: 1.2, 2015: 1.0, 2020: 0.8},
    },
    'carro_flex': {
        'CO2': {2010: 170, 2015: 150, 2020: 130},
        'CH4': {2010: 0.03, 2015: 0.025, 2020: 0.015},
        'N2O': {2010: 0.004, 2015: 0.003, 2020: 0.002},
        'PM2.5': {2010: 0.007, 2015: 0.004, 2020: 0.002},
        'CO': {2010: 0.9, 2015: 0.7, 2020: 0.4},
        'NOx': {2010: 0.12, 2015: 0.09, 2020: 0.07},
    },
    'moto_gasolina': { # Exemplo de dados para moto
        'CO2': {2010: 100, 2015: 90, 2020: 80},
        'CH4': {2010: 0.02, 2015: 0.015, 2020: 0.01},
        'N2O': {2010: 0.002, 2015: 0.0015, 2020: 0.001},
        'PM2.5': {2010: 0.005, 2015: 0.003, 2020: 0.002},
        'CO': {2010: 0.5, 2015: 0.3, 2020: 0.2},
        'NOx': {2010: 0.08, 2015: 0.06, 2020: 0.05},
    }
}

# --- Função para obter o fator de emissão mais próximo para um dado ano ---
def get_emission_factor(vehicle_type, pollutant, year):
    if vehicle_type not in EMISSION_FACTORS or pollutant not in EMISSION_FACTORS[vehicle_type]:
        return 0.0

    factors_by_year = EMISSION_FACTORS[vehicle_type][pollutant]
    # Encontra o ano mais próximo (igual ou anterior) ao ano do veículo
    closest_year = max([y for y in factors_by_year if y <= year], default=min(factors_by_year, default=year))
    return factors_by_year.get(closest_year, 0.0)

# --- Caminhos dos arquivos ---
INPUT_CSV = os.path.join(os.path.dirname(__file__), "veiculos_km.csv")
OUTPUT_CSV = os.path.join(os.path.dirname(__file__), "veiculos_emissoes_calculadas.csv")

def calculate_vehicle_emissions():
    print(f"Tentando ler o arquivo de entrada: {INPUT_CSV}")
    try:
        df = pd.read_csv(INPUT_CSV)
        print("Arquivo de entrada lido com sucesso.")
    except FileNotFoundError:
        print(f"ERRO: Arquivo '{INPUT_CSV}' não encontrado.")
        print("Certifique-se de criar o arquivo 'veiculos_km.csv' no mesmo diretório.")
        return

    calculated_emissions = []

    # Coordenadas do centro do Distrito Federal para associar as emissões no mapa
    LATITUDE_DF = -15.7801
    LONGITUDE_DF = -47.9292

    for index, row in df.iterrows():
        tipo_veiculo = str(row['tipo_veiculo']).strip().lower() # Normaliza tipo de veículo
        ano_fabricacao = int(row['ano_fabricacao'])
        quilometragem_total = float(row['quilometragem_total'])

        if tipo_veiculo not in EMISSION_FACTORS:
            print(f"AVISO: Tipo de veículo '{tipo_veiculo}' não encontrado nos fatores de emissão. Ignorando esta linha.")
            continue

        for pollutant_type, _ in EMISSION_FACTORS[tipo_veiculo].items():
            factor = get_emission_factor(tipo_veiculo, pollutant_type, ano_fabricacao)
            emission_grams = factor * quilometragem_total

            calculated_emissions.append({
                'data_type': f"Emissao_{pollutant_type}", # Ex: 'Emissao_CO2', 'Emissao_PM2.5'
                'value': emission_grams, # Em gramas
                'unit': 'g',
                'latitude': LATITUDE_DF,
                'longitude': LONGITUDE_DF,
                'area_name': 'Distrito Federal (Total)', # Pode ser refinado se tiver dados de área
                'timestamp': datetime.now().isoformat()
            })

    emissions_df = pd.DataFrame(calculated_emissions)
    emissions_df.to_csv(OUTPUT_CSV, index=False)
    print(f"Cálculo de emissões concluído e salvo em '{OUTPUT_CSV}'.")
    print("Verifique este CSV para garantir que os dados estão como esperado antes de ingeri-los.")

if __name__ == "__main__":
    calculate_vehicle_emissions()

```

Comando no Terminal:
```
Bash

python backend_api/data_ingestion/calculate_vehicle_emissions.py

```
Critérios de Aceitação:

Execução do script calculate_vehicle_emissions.py sem erros.

Um novo arquivo backend_api/data_ingestion/veiculos_emissoes_calculadas.csv é gerado, contendo as emissões calculadas para cada poluente e tipo de veículo.

## Issue #11: Ingerir Emissões Veiculares Calculadas no Banco de Dados
Descrição: Desenvolva o script que lê o CSV de emissões calculadas (gerado na Issue #10) e insere esses dados na tabela MobilityData do banco de dados.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/ingest_calculated_emissions.py com o seguinte conteúdo:
```Python

import pandas as pd
import os
from sqlalchemy.orm import Session
from datetime import datetime

# Ajusta o caminho de importação para que o script possa encontrar database e crud
# Assume que este script está dentro de 'data_ingestion/' e precisa acessar '../'
import sys
sys.path.append(os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

from database import SessionLocal
from crud import create_mobility_data
from models import MobilityDataCreate

# Caminho do CSV de emissões calculadas gerado pelo script anterior
CALCULATED_EMISSIONS_CSV = os.path.join(os.path.dirname(__file__), "veiculos_emissoes_calculadas.csv")

def ingest_calculated_emissions_data():
    print(f"Tentando ler o arquivo de emissões calculadas: {CALCULATED_EMISSIONS_CSV}")
    try:
        df = pd.read_csv(CALCULATED_EMISSIONS_CSV)
        print("Arquivo de emissões calculadas lido com sucesso.")
    except FileNotFoundError:
        print(f"ERRO: Arquivo '{CALCULATED_EMISSIONS_CSV}' não encontrado.")
        print("Certifique-se de que o script 'calculate_vehicle_emissions.py' foi executado primeiro para gerá-lo.")
        return

    db: Session = SessionLocal()
    try:
        for index, row in df.iterrows():
            # Converte o timestamp para um objeto datetime
            timestamp_obj = pd.to_datetime(row['timestamp'])

            # Prepara o objeto MobilityDataCreate
            mobility_data_entry = MobilityDataCreate(
                data_type=row['data_type'], # Ex: 'Emissao_CO2', 'Emissao_PM2.5'
                value=float(row['value']),
                unit=row['unit'],
                latitude=float(row['latitude']) if pd.notna(row['latitude']) else None,
                longitude=float(row['longitude']) if pd.notna(row['longitude']) else None,
                area_name=row['area_name'] if pd.notna(row['area_name']) else None,
                timestamp=timestamp_obj
            )
            create_mobility_data(db, mobility_data_entry)
        db.commit()
        print(f"Dados de emissões calculadas do CSV '{CALCULATED_EMISSIONS_CSV}' ingeridos no banco de dados com sucesso.")
    except Exception as e:
        db.rollback()
        print(f"ERRO ao ingerir dados de emissões calculadas: {e}")
    finally:
        db.close()

if __name__ == "__main__":
    ingest_calculated_emissions_data()

```
Comando no Terminal:
```Bash

python backend_api/data_ingestion/ingest_calculated_emissions.py
```
Critérios de Aceitação:

Execução do script ingest_calculated_emissions.py sem erros.

Verifique o banco de dados (usando uma ferramenta como DB Browser for SQLite se estiver usando SQLite, ou um cliente de DB para PostgreSQL/MySQL) para confirmar que os dados de emissões veiculares foram inseridos na tabela mobility_data.

## Issue #12: Criar CSV de Dados de Estações Manuais de Exemplo (Prontos para Ingestão)
Descrição: Crie um arquivo CSV de exemplo para dados de estações de monitoramento manuais. Este será o arquivo de entrada para a ingestão de dados de estações manuais.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/manual_stations_example.csv com o seguinte conteúdo. Adicione mais dados para testar melhor:
```Snippet de código

name,latitude,longitude,pollutant_type,value,unit,timestamp
Estacao Manual A,-15.7801,-47.9292,PM2.5,25.3,ug/m3,2024-05-26 10:00:00
Estacao Manual B,-15.7900,-47.8800,O3,45.1,ppb,2024-05-26 10:05:00
Estacao Manual A,-15.7801,-47.9292,NO2,15.0,ppb,2024-05-26 10:10:00
Estacao Manual C,-15.8200,-47.9500,CO,0.8,ppm,2024-05-26 10:15:00
Estacao Manual B,-15.7900,-47.8800,PM10,30.5,ug/m3,2024-05-26 10:20:00
```
Critérios de Aceitação:

O arquivo manual_stations_example.csv existe dentro da pasta backend_api/data_ingestion/.

O conteúdo do CSV está no formato esperado.

## Issue #13: Ingerir Dados de Estações Manuais no Banco de Dados
Descrição: Desenvolva o script que lê o CSV de estações manuais (criado na Issue #12) e insere os dados de estações e leituras de poluentes no banco de dados.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/ingest_manual_stations.py com o seguinte conteúdo:
```Python

import pandas as pd
import os
from sqlalchemy.orm import Session
from datetime import datetime

# Ajusta o caminho de importação para que o script possa encontrar database e crud
import sys
sys.path.append(os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

from database import SessionLocal
from crud import create_station, create_pollutant_reading
from models import StationCreate, PollutantReadingCreate, StationTypeEnum

# Substitua pelo caminho real do seu arquivo
CSV_FILE = os.path.join(os.path.dirname(__file__), "manual_stations_example.csv")

def ingest_manual_stations_data():
    db: Session = SessionLocal()
    try:
        df = pd.read_csv(CSV_FILE)

        for index, row in df.iterrows():
            # Tenta encontrar a estação existente pelo nome, lat e long para evitar duplicatas
            existing_station = db.query(StationCreate).filter(
                StationCreate.name == row['name'],
                StationCreate.latitude == row['latitude'],
                StationType.MANUAL == row['station_type'] # Adicionado para garantir unicidade com tipo
            ).first()

            station_id = None
            if not existing_station:
                station_data = StationCreate(
                    name=row['name'],
                    latitude=float(row['latitude']),
                    longitude=float(row['longitude']),
                    station_type=StationTypeEnum.MANUAL
                )
                station = create_station(db, station_data)
                station_id = station.id
            else:
                station_id = existing_station.id

            # Cria a leitura do poluente
            timestamp_obj = pd.to_datetime(row['timestamp'])

            reading_data = PollutantReadingCreate(
                station_id=station_id,
                pollutant_type=row['pollutant_type'],
                value=float(row['value']),
                unit=row['unit'],
                timestamp=timestamp_obj
            )
            create_pollutant_reading(db, reading_data)
        db.commit()
        print(f"Dados do CSV de estações manuais '{CSV_FILE}' ingeridos com sucesso.")
    except FileNotFoundError:
        print(f"ERRO: Arquivo '{CSV_FILE}' não encontrado.")
        print("Certifique-se de criar o arquivo 'manual_stations_example.csv' no mesmo diretório.")
    except Exception as e:
        db.rollback()
        print(f"Erro ao ingerir dados do CSV manual: {e}")
    finally:
        db.close()

if __name__ == "__main__":
    ingest_manual_stations_data()

```
Comando no Terminal:
```
Bash

python backend_api/data_ingestion/ingest_manual_stations.py
```
Critérios de Aceitação:

Execução do script ingest_manual_stations.py sem erros.

Verifique o banco de dados para confirmar que as estações manuais e suas leituras foram inseridas nas tabelas stations e pollutant_readings.

## Issue #14: Implementar Ingestão de Dados de API de Estações Automáticas
Descrição: Desenvolva o script que fará requisições a uma API externa (simulada ou real) para coletar dados de estações automáticas e inseri-los no banco de dados. Este script assume uma estrutura de resposta JSON da API externa; você precisará adaptá-lo para a API real que for usar. Esta tarefa não pode ser testada completamente sem uma API externa real.

Código Relacionado:

Crie o arquivo backend_api/data_ingestion/ingest_automatic_api.py com o seguinte conteúdo:
```Python

import requests
import os
from sqlalchemy.orm import Session
from datetime import datetime

# Ajusta o caminho de importação para que o script possa encontrar database e crud
import sys
sys.path.append(os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

from database import SessionLocal
from crud import create_station, create_pollutant_reading
from models import StationCreate, PollutantReadingCreate, StationTypeEnum

# URL da API de estações automáticas (exemplo - SUBSTITUA PELA URL REAL!)
AUTOMATIC_STATION_API_URL = "https://api.example.com/air_quality_data" # <-- Altere esta URL!
# DICA: Para testes iniciais, você pode usar um serviço de mock API como Beeceptor ou Mocky.io

def fetch_automatic_station_data():
    try:
        print(f"Buscando dados da API: {AUTOMATIC_STATION_API_URL}")
        # Adapte esta parte para a forma como sua API externa requer autenticação (se houver)
        response = requests.get(AUTOMATIC_STATION_API_URL)
        response.raise_for_status() # Levanta um erro para códigos de status HTTP ruins (4xx ou 5xx)
        print("Dados da API de estações automáticas recebidos com sucesso.")
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"ERRO ao buscar dados da API de estações automáticas: {e}")
        print("Verifique a URL e sua conexão com a internet. Para testes, considere usar uma mock API.")
        return None

def ingest_automatic_stations_data():
    data = fetch_automatic_station_data()
    if not data:
        print("Nenhum dado para ingestão da API automática.")
        return

    db: Session = SessionLocal()
    try:
        # Assumindo que a API retorna uma lista de estações e suas leituras
        # Adapte esta lógica de iteração de acordo com a ESTRUTURA JSON da API REAL
        stations_list = data.get("stations", []) # ou 'data' se a API retornar diretamente a lista

        if not stations_list:
            print("Nenhuma estação encontrada nos dados da API automática.")
            return

        for item in stations_list:
            station_name = item.get("name")
            latitude = item.get("latitude")
            longitude = item.get("longitude")
            readings = item.get("readings", [])

            if not all([station_name, latitude, longitude]):
                print(f"AVISO: Estação sem nome, latitude ou longitude completos. Ignorando: {item}")
                continue

            # Cria a estação se ainda não existir
            existing_station = db.query(StationCreate).filter(
                StationCreate.name == station_name,
                StationCreate.latitude == latitude,
                StationType.AUTOMATIC == StationTypeEnum.AUTOMATIC # Adicionado para garantir unicidade com tipo
            ).first()

            station_id = None
            if not existing_station:
                station_data = StationCreate(
                    name=station_name,
                    latitude=float(latitude),
                    longitude=float(longitude),
                    station_type=StationTypeEnum.AUTOMATIC
                )
                station = create_station(db, station_data)
                station_id = station.id
            else:
                station_id = existing_station.id

            for reading in readings:
                pollutant_type = reading.get("pollutant_type")
                value = reading.get("value")
                unit = reading.get("unit")
                timestamp_str = reading.get("timestamp")

                if not all([pollutant_type, value is not None, timestamp_str]):
                    print(f"AVISO: Leitura de poluente incompleta para estação '{station_name}'. Ignorando: {reading}")
                    continue

                try:
                    # Tenta converter vários formatos de timestamp (ISO 8601, etc.)
                    timestamp = datetime.fromisoformat(timestamp_str.replace("Z", "+00:00"))
                except ValueError:
                    try:
                        # Se não for ISO, tente outros formatos comuns, ou use um parser mais robusto
                        timestamp = datetime.strptime(timestamp_str, "%Y-%m-%dT%H:%M:%S")
                    except ValueError:
                        print(f"AVISO: Formato de timestamp desconhecido '{timestamp_str}' para estação '{station_name}'. Ignorando leitura.")
                        continue

                reading_data = PollutantReadingCreate(
                    station_id=station_id,
                    pollutant_type=pollutant_type,
                    value=float(value),
                    unit=unit,
                    timestamp=timestamp
                )
                create_pollutant_reading(db, reading_data)
        db.commit()
        print("Dados da API de estações automáticas ingeridos com sucesso.")
    except Exception as e:
        db.rollback()
        print(f"ERRO ao ingerir dados da API automática: {e}")
    finally:
        db.close()

if __name__ == "__main__":
    ingest_automatic_stations_data()

```
Comando no Terminal:

```Bash

python backend_api/data_ingestion/ingest_automatic_api.py
```
Critérios de Aceitação:

Execução do script ingest_automatic_api.py sem erros (se a URL da API for válida e retornar dados).

Verifique o banco de dados para confirmar que as estações automáticas e suas leituras foram inseridas nas tabelas stations e pollutant_readings.

## Issue #15: Conectar Endpoints da API com Dados Reais do Banco de Dados
Descrição: Modifique os endpoints da API Python (/api/map_data/, /api/stations/, /api/pollutant_readings/, etc.) para que parem de retornar dados mockados e, em vez disso, consultem os dados reais do banco de dados (que você já ingeriu nas tarefas anteriores).

Código Relacionado:

Modifique o arquivo backend_api/main.py. Substitua as funções dos endpoints (read_stations, read_pollutant_readings, read_mobility_data, read_greenhouse_gas_readings, e especialmente get_map_data) para que chamem as funções do crud.py que buscam dados reais.

Observação: O código do crud.py e models.py (criados na Issue #3) já estão preparados para isso, então não há alterações nesses arquivos para esta issue.

Aqui está o conteúdo atualizado de backend_api/main.py:

```Python

from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List, Optional
from datetime import datetime, timedelta # Importar timedelta

from . import crud, models, database
from .database import SessionLocal, engine, get_db
from fastapi.middleware.cors import CORSMiddleware

# Cria as tabelas do banco de dados na inicialização se ainda não existirem
database.create_db_tables()

app = FastAPI(
    title="API de Qualidade do Ar e Mobilidade do DF",
    description="API para consultar dados de estações de poluentes, gases de efeito estufa e mobilidade urbana no Distrito Federal.",
    version="1.0.0"
)

# Configuração do CORS (conforme Issue #3)
origins = [
    "http://localhost",
    "http://localhost:8000",
    "http://seu-dominio-wordpress.com", # <--- SEU DOMÍNIO AQUI
    "https://seu-dominio-wordpress.com", # <--- SEU DOMÍNIO AQUI
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# --- Endpoints para Estações (AGORA CONSULTANDO O DB) ---
@app.post("/api/stations/", response_model=models.Station, status_code=status.HTTP_201_CREATED)
def create_station(station: models.StationCreate, db: Session = Depends(get_db)):
    return crud.create_station(db=db, station=station)

@app.get("/api/stations/", response_model=List[models.Station])
def read_stations(
    skip: int = 0,
    limit: int = 100,
    station_type: Optional[models.StationTypeEnum] = None,
    db: Session = Depends(get_db)
):
    stations = crud.get_stations(db, skip=skip, limit=limit, station_type=station_type)
    return stations

@app.get("/api/stations/{station_id}", response_model=models.Station)
def read_station(station_id: int, db: Session = Depends(get_db)):
    db_station = crud.get_station(db, station_id=station_id)
    if db_station is None:
        raise HTTPException(status_code=404, detail="Station not found")
    return db_station

# --- Endpoints para Leitura de Poluentes (AGORA CONSULTANDO O DB) ---
@app.post("/api/pollutant_readings/", response_model=models.PollutantReading, status_code=status.HTTP_201_CREATED)
def create_pollutant_reading(reading: models.PollutantReadingCreate, db: Session = Depends(get_db)):
    return crud.create_pollutant_reading(db=db, reading=reading)

@app.get("/api/pollutant_readings/", response_model=List[models.PollutantReading])
def read_pollutant_readings(
    station_id: Optional[int] = None,
    pollutant_type: Optional[str] = None,
    start_time: Optional[datetime] = None,
    end_time: Optional[datetime] = None,
    station_type: Optional[models.StationTypeEnum] = None,
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    readings = crud.get_pollutant_readings(
        db, station_id=station_id, pollutant_type=pollutant_type,
        start_time=start_time, end_time=end_time, station_type=station_type, skip=skip, limit=limit
    )
    return readings

# --- Endpoints para Dados de Mobilidade (AGORA CONSULTANDO O DB) ---
@app.post("/api/mobility_data/", response_model=models.MobilityData, status_code=status.HTTP_201_CREATED)
def create_mobility_data(data: models.MobilityDataCreate, db: Session = Depends(get_db)):
    return crud.create_mobility_data(db=db, data=data)

@app.get("/api/mobility_data/", response_model=List[models.MobilityData])
def read_mobility_data(
    data_type: Optional[str] = None,
    area_name: Optional[str] = None,
    start_time: Optional[datetime] = None,
    end_time: Optional[datetime] = None,
    data_type_startswith: Optional[str] = None,
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    data = crud.get_mobility_data(
        db, data_type=data_type, area_name=area_name,
        start_time=start_time, end_time=end_time, data_type_startswith=data_type_startswith, skip=skip, limit=limit
    )
    return data

# --- Endpoints para Gases de Efeito Estufa (AGORA CONSULTANDO O DB) ---
@app.post("/api/greenhouse_gas_readings/", response_model=models.GreenhouseGasReading, status_code=status.HTTP_201_CREATED)
def create_greenhouse_gas_reading(reading: models.GreenhouseGasReadingCreate, db: Session = Depends(get_db)):
    return crud.create_greenhouse_gas_reading(db=db, reading=reading)

@app.get("/api/greenhouse_gas_readings/", response_model=List[models.GreenhouseGasReading])
def read_greenhouse_gas_readings(
    station_id: Optional[int] = None,
    gas_type: Optional[str] = None,
    start_time: Optional[datetime] = None,
    end_time: Optional[datetime] = None,
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    readings = crud.get_greenhouse_gas_readings(
        db, station_id=station_id, gas_type=gas_type,
        start_time=start_time, end_time=end_time, skip=skip, limit=limit
    )
    return readings

# --- Endpoint Combinado para o Mapa (AGORA CONSULTANDO O DB REAL) ---
@app.get("/api/map_data/", response_model=models.MapData)
def get_map_data(db: Session = Depends(get_db), last_hours: Optional[int] = 24):
    # Para otimização, busca dados apenas do período recente
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=last_hours)

    # Buscar estações
    stations = crud.get_stations(db, limit=500) # Limitar para evitar sobrecarga no mapa

    # Buscar leituras recentes de poluentes dentro do período
    # Para um mapa, você provavelmente quer a leitura mais recente de CADA poluente de CADA estação.
    # A função crud.get_pollutant_readings já ordena por timestamp.
    pollutant_readings = crud.get_pollutant_readings(db, start_time=start_time, end_time=end_time, limit=1000)

    # Buscar dados recentes de mobilidade, incluindo emissões veiculares
    mobility_data = crud.get_mobility_data(db, start_time=start_time, end_time=end_time, limit=1000)

    # Buscar leituras recentes de gases de efeito estufa
    greenhouse_gas_readings = crud.get_greenhouse_gas_readings(db, start_time=start_time, end_time=end_time, limit=500)

    return models.MapData(
        stations=stations,
        pollutant_readings=pollutant_readings,
        mobility_data=mobility_data,
        greenhouse_gas_readings=greenhouse_gas_readings
    )

# --- Endpoint para Índice de Qualidade do Ar (Placeholder, a ser implementado depois) ---
@app.get("/api/air_quality_index/", response_model=dict)
def get_air_quality_index(region_name: Optional[str] = None, db: Session = Depends(get_db)):
    return {"status": "implementar cálculo do IQA aqui", "details": "Dados agregados por região ou geral"}

# --- Endpoint para Dados Agregados de Regiões (Placeholder, a ser implementado depois) ---
@app.get("/api/regions_summary/", response_model=dict)
def get_regions_summary(db: Session = Depends(get_db)):
    return {"status": "implementar agregação de dados por região aqui", "data": {}}

```
Critérios de Aceitação:

Servidor FastAPI reiniciado.

Ao acessar http://127.0.0.1:8000/api/map_data/, o JSON retornado contém os dados reais (estações, leituras, mobilidade/emissões) que você ingeriu nas tarefas anteriores.

Testar os outros endpoints como /api/stations/ (com ?station_type=manual ou automatic) e /api/pollutant_readings/ (com ?station_id=X) para verificar se retornam os dados corretos do DB.

Conectando a API Python (em VM) ao WordPress
Uma vez que sua API Python está funcionando localmente com dados reais, o próximo passo crucial é implantá-la em uma máquina virtual (VM) e configurar o WordPress para se comunicar com essa VM.

Aqui está um guia passo a passo sobre como fazer isso:

### Providenciar uma Máquina Virtual (VM):

Você pode usar provedores de nuvem como Google Cloud Platform (GCP), Amazon Web Services (AWS), DigitalOcean, Vultr, ou até mesmo um servidor local como Raspberry Pi.

Escolha um sistema operacional Linux (ex: Ubuntu Server).

Passos Gerais:

- Crie uma nova instância de VM.

- Configure as regras de firewall (Security Groups no AWS, Firewall rules no GCP) para permitir tráfego HTTP/HTTPS (portas 80, 443) e a porta da sua API (8000).

- Conecte-se à VM via SSH.

- Preparar a VM para a API Python:

Instalar Python e Pip:

```Bash

sudo apt update
sudo apt install python3 python3-pip python3-venv -y
Clonar seu Repositório Git:

Instale Git: sudo apt install git -y

Clone seu repositório (onde está backend_api):
git clone https://github.com/seu-usuario/seu-repositorio.git /var/www/seu_projeto_poluicao_df
```
#### Configurar Ambiente Virtual e Dependências na VM:

```Bash

cd /var/www/seu_projeto_poluicao_df/backend_api
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt # Crie um requirements.txt a partir do seu ambiente local: pip freeze > requirements.txt

```
#### Configurar Variáveis de Ambiente:

Crie o arquivo .env dentro de backend_api/ na VM. Se você usa um banco de dados remoto (PostgreSQL), ajuste a DATABASE_URL aqui:
DATABASE_URL="postgresql://user:password@seu_host_db:5432/dbname"
(Se o DB estiver na mesma VM, use postgresql://user:password@localhost:5432/dbname)

Para SQLite, o caminho deve ser absoluto ou relativo à onde a API será executada: DATABASE_URL="sqlite:////var/www/seu_projeto_poluicao_df/backend_api/air_quality.db"

##### Inicializar o Banco de Dados (na VM):

```Bash

python3 database.py
Executar Scripts de Ingestão de Dados (na VM):

Copie seus arquivos CSV de dados para backend_api/data_ingestion/ na VM.
```
##### Execute os scripts de ingestão (na ordem correta):

```Bash

python3 data_ingestion/calculate_vehicle_emissions.py
python3 data_ingestion/ingest_calculated_emissions.py
python3 data_ingestion/ingest_manual_stations.py
python3 data_ingestion/ingest_automatic_api.py
```
##### Executar a API com um Servidor de Produção (Gunicorn):

- Instale Gunicorn: pip install gunicorn (adicione ao requirements.txt)

- Teste a API: gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

- Configure um serviço (systemd no Linux) para que a API inicie com a VM e seja persistente. (Isso é mais complexo e pode ser uma issue futura: Issue #34: Preparar para Implantação)

- Configurar o WordPress para Conectar à VM:

- Obter o IP Público da VM: Após iniciar a VM, ela terá um endereço IP público (ou um nome de domínio).

- Atualizar a URL da API no Plugin WordPress:

- No seu arquivo wordpress_frontend/wp-content/plugins/poluicao-df-dashboard/poluicao-df-dashboard.php, altere a linha POLUICAO_DF_API_BASE_URL:

```PHP

define('POLUICAO_DF_API_BASE_URL', 'http://SEU_IP_PUBLICO_DA_VM:8000/api/'); // <--- ATUALIZE AQUI!
// Ou se você tiver um domínio configurado para a VM:
// define('POLUICAO_DF_API_BASE_URL', 'https://seu-dominio-da-api.com/api/');
Lembre-se de reativar o plugin no WordPress após essa alteração para que ela tenha efeito.

Configurar CORS na API (novamente): Certifique-se de que o domínio do seu WordPress (ex: http://meu-site-wordpress.com ou http://localhost) está incluído na lista origins do CORSMiddleware em backend_api/main.py na sua VM. Se a VM tiver um IP público, adicione-o também.

Testar a Conexão:

Acesse a página do mapa no seu site WordPress.

Verifique se os marcadores e balões aparecem, mas agora com os dados reais que você ingeriu na VM.

Abra o console do navegador (F12) e verifique a aba "Network" para garantir que as requisições AJAX para o IP/domínio da sua VM estão sendo feitas com sucesso (código de status 200).

```