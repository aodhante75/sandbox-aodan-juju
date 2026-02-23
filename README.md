# ETL de órdenes (PySpark + API + MSSQL opcional)

Pipeline de ETL: lee órdenes de uns API(JSON), lee usuarios y productos (desde MSSQL o desde CSV) y genera salida en raw (JSON), realiza la union de los datasets mencionados al principio y son exportados en curated (Parquet). Se puede usar Docker para tener MSSQL o correr todo solo con los datos de ejemplo.

## Qué necesitas

- Python 3.9 o superior
- Java (JRE o JDK 8, 11, 17 o 21 para PySpark; probado con Java 8 y 21)
- Opcional: Docker y Docker Compose si quieres usar MSSQL

## Cómo ejecutarlo

### 1. Entorno virtual e instalar dependencias

Desde la **raíz del proyecto**:

**Windows (PowerShell):**
```powershell
python -m venv orderetl
.\orderetl\Scripts\activate
pip install -r requirements.txt
```

**Mac/Linux:**
```bash
python -m venv orderetl
source orderetl/bin/activate
pip install -r requirements.txt
```

### 2. Levantar la base de datos con Docker (opcional)

Si se quiere usar MSSQL en lugar de los CSV de ejemplo:

1. **Crea un archivo `.env`**:

```env
MSSQL_SA_PASSWORD=Password123
MSSQL_HOST=localhost
MSSQL_PORT=1433
MSSQL_DB=etl_db
MSSQL_USER=sa
```

2. **Levanta el contenedor de SQL SERVER**:

```bash
docker-compose --env-file .env up -d
```

- Arranca SQL Server y espera a que este listo.
- El servicio **mssql-init** corre solo y ejecuta `sql/init.sql` No se tiene que hacer más nada.

**Espera** a que el contenedor `mssql_init` termine (principalmente la primera vez). Para ver si ya acabo:

```bash
docker logs mssql_init
```

Tambien se puede verificar conecntandose desde un gestor de base datos DBEAVER, MS Management

### 3. Ejecutar el ETL

Con el venv activado,  s i se quiere usar MSSQL el Docker debe estar levantado:

**Para procesar todas las ordenes**
```bash
python -m src.etl_job
```

Si la base está disponible, el job lee usuarios y productos desde MSSQL; si no, usa los CSV de `sample_data/`.

**Para procesar solo órdenes desde una fecha:**

```bash
python -m src.etl_job --since 2025-08-21
```

Dónde se guarda todo:

- **Raw:** `output/raw/ingest_ts=YYYYMMDDHHMMSS/orders.json` — los datos como vienen de la API.
- **Curated:** `output/curated/event_date=YYYY-MM-DD/` — Parquet particionado por fecha, listo para usar.

### 4. Tests

Desde la raíz del proyecto y con el mismo venv activado:

```bash
python -m pytest tests/ -v
```

## Estructura del proyecto

```
├── README.md
├── requirements.txt
├── sample_data/
│   ├── api_orders.json
│   ├── users.csv
│   └── products.csv
├── src/
│   ├── etl_job.py
│   ├── transforms.py
│   ├── api_client.py
│   ├── db.py
│   ├── write_curated.py
│   └── retry.py
├── sql/
│   ├── redshift-ddl.sql
│   └── init.sql
├── tests/
│   ├── test_transforms.py
│   └── test_api_client.py
├── output/          # se genera: raw/ y curated/
└── docs/
    └── design_notes.md
```

## Credenciales

- **Todas las credenciales van en `.env`.** No hay contraseñas en el código. En `docker-compose.yml` se usan variables (por ejemplo `${MSSQL_SA_PASSWORD}`).



## Tiempo y limtaciones 
 Me tomo cerca de 7 horas. Tuve porblemas con la idempotencia en el motor de spark por eso opte por pyarrow. 
