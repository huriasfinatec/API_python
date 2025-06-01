```
seu_projeto_poluicao_df/
├── backend_api/
│   ├── main.py
│   ├── database.py
│   ├── crud.py
│   ├── models.py
│   ├── .env
│   └── data_ingestion/
│       ├── __init__.py
│       ├── ingest_manual_stations.py
│       ├── ingest_automatic_api.py
│       ├── calculate_vehicle_emissions.py
│       ├── ingest_calculated_emissions.py
│       ├── veiculos_km.csv
│       ├── manual_stations_example.csv
│       └── veiculos_emissoes_calculadas.csv
│
└── wordpress_frontend/
    └── wp-content/
        └── plugins/
            └── poluicao-df-dashboard/
                ├── poluicao-df-dashboard.php
                ├── assets/
                │   ├── css/
                │   │   └── poluicao-df-dashboard.css
                │   └── js/
                │       ├── google-maps-integrator.js
                │       ├── regions-dashboard.js
                │       ├── automatic-stations-dashboard.js
                │       ├── manual-stations-dashboard.js
                │       ├── low-cost-stations-dashboard.js
                │       └── comparison-dashboard.js
                └── readme.txt
```

