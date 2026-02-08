# Fluxo Detalhado: Upload → Import → Análise

Este documento detalha o fluxo completo de processamento de um arquivo CSV no MVP.

## 📤 Fase 1: Upload de Arquivo

### Endpoint
```
POST /api/v1/datasets/upload
Content-Type: multipart/form-data
```

### Request
```
FormData:
  - file: arquivo.csv (binary)
  - name: "Vendas 2024" (opcional, string)
  - description: "Dados de vendas do ano" (opcional, string)
```

### Processamento

1. **Recebimento do Arquivo**
   ```python
   async def upload_dataset(file: UploadFile):
       # FastAPI recebe o arquivo como stream
       # Não carrega tudo em memória de uma vez
   ```

2. **Validações**
   - ✅ Extensão do arquivo (.csv)
   - ✅ Tamanho máximo (configurável, padrão: 1GB)
   - ✅ Tipo MIME (text/csv)
   - ✅ Arquivo não vazio

3. **Salvamento Temporário**
   ```python
   # Salva em chunks para não sobrecarregar memória
   temp_path = f"./storage/uploads/{unique_id}.csv"
   
   with open(temp_path, "wb") as buffer:
       for chunk in file.chunks(chunk_size=1024*1024):  # 1MB chunks
           buffer.write(chunk)
   ```

4. **Análise Preliminar**
   - Detecta delimitador (vírgula, ponto-e-vírgula, tab)
   - Lê primeiras linhas para inferir colunas
   - Estima número de linhas (opcional)

### Response
```json
{
  "status": "uploaded",
  "dataset_id": "ds_abc123",
  "filename": "arquivo.csv",
  "size_bytes": 52428800,
  "temp_path": "./storage/uploads/ds_abc123.csv"
}
```

---

## 📥 Fase 2: Import para DuckDB

### Processamento Automático

Após o upload, o sistema automaticamente:

1. **Cria Tabela no DuckDB**
   ```sql
   CREATE TABLE dataset_ds_abc123 AS 
   SELECT * FROM read_csv_auto(
       './storage/uploads/ds_abc123.csv',
       header=true,
       delim=',',
       sample_size=10000
   );
   ```

2. **Inferência Automática de Schema**
   
   DuckDB analisa o CSV e:
   - Detecta tipos de dados (INTEGER, VARCHAR, DOUBLE, DATE, etc.)
   - Identifica colunas com valores nulos
   - Otimiza tipos para economizar espaço
   
   Exemplo de schema detectado:
   ```
   id: INTEGER
   nome: VARCHAR
   preco: DOUBLE
   data_venda: DATE
   quantidade: INTEGER
   categoria: VARCHAR
   ```

3. **Leitura Lazy e Otimizada**
   
   DuckDB NÃO carrega todo o CSV em memória:
   - Lê em chunks otimizados
   - Processa em paralelo (múltiplas threads)
   - Comprime dados automaticamente
   - Cria estruturas de índice

4. **Salvamento dos Metadados**
   ```python
   metadata = {
       "dataset_id": "ds_abc123",
       "name": "Vendas 2024",
       "description": "Dados de vendas do ano",
       "filename": "arquivo.csv",
       "table_name": "dataset_ds_abc123",
       "size_bytes": 52428800,
       "row_count": 1000000,
       "column_count": 6,
       "columns": [...],
       "created_at": "2024-02-08T10:30:00Z",
       "status": "ready"
   }
   ```

5. **Limpeza**
   ```python
   # Remove arquivo temporário (dados já estão no DuckDB)
   os.remove(temp_path)
   ```

### Vantagens do DuckDB

- **Zero-Copy**: Não duplica dados na memória
- **Compressão**: Reduz tamanho em disco (até 10x)
- **Indexação**: Queries rápidas mesmo em bilhões de linhas
- **Vetorização**: Processa dados em lotes (SIMD)

---

## 🔍 Fase 3: Análise dos Dados

Agora que os dados estão no DuckDB, podemos analisá-los de várias formas:

### 3.1 Preview dos Dados

**Endpoint**: `GET /api/v1/datasets/{dataset_id}/preview?limit=10`

**Query SQL**:
```sql
SELECT * FROM dataset_ds_abc123
LIMIT 10;
```

**Response**:
```json
{
  "dataset_id": "ds_abc123",
  "rows": [
    {"id": 1, "nome": "Produto A", "preco": 29.90, ...},
    {"id": 2, "nome": "Produto B", "preco": 49.90, ...},
    ...
  ],
  "total_rows": 1000000,
  "preview_rows": 10
}
```

### 3.2 Obter Schema

**Endpoint**: `GET /api/v1/datasets/{dataset_id}/schema`

**Query SQL**:
```sql
DESCRIBE dataset_ds_abc123;
```

**Response**:
```json
{
  "columns": [
    {
      "name": "id",
      "type": "INTEGER",
      "nullable": false
    },
    {
      "name": "nome",
      "type": "VARCHAR",
      "nullable": true
    },
    {
      "name": "preco",
      "type": "DOUBLE",
      "nullable": true
    },
    ...
  ]
}
```

### 3.3 Estatísticas Básicas

**Endpoint**: `GET /api/v1/datasets/{dataset_id}/stats`

**Query SQL**:
```sql
SELECT
    COUNT(*) as total_rows,
    COUNT(DISTINCT id) as unique_ids,
    MIN(preco) as min_preco,
    MAX(preco) as max_preco,
    AVG(preco) as avg_preco,
    SUM(quantidade) as total_quantidade
FROM dataset_ds_abc123;
```

**Response**:
```json
{
  "total_rows": 1000000,
  "unique_ids": 1000000,
  "min_preco": 9.90,
  "max_preco": 999.90,
  "avg_preco": 125.45,
  "total_quantidade": 5432100
}
```

### 3.4 Profiling Completo

**Endpoint**: `GET /api/v1/datasets/{dataset_id}/profile`

Gera um perfil completo de cada coluna:

**Para colunas numéricas**:
```sql
SELECT
    'preco' as column_name,
    COUNT(*) as count,
    COUNT(DISTINCT preco) as unique_count,
    SUM(CASE WHEN preco IS NULL THEN 1 ELSE 0 END) as null_count,
    MIN(preco) as min_value,
    MAX(preco) as max_value,
    AVG(preco) as mean,
    MEDIAN(preco) as median,
    STDDEV(preco) as std_dev,
    percentile_cont(0.25) WITHIN GROUP (ORDER BY preco) as q25,
    percentile_cont(0.75) WITHIN GROUP (ORDER BY preco) as q75
FROM dataset_ds_abc123;
```

**Para colunas categóricas**:
```sql
SELECT
    categoria,
    COUNT(*) as frequency,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () as percentage
FROM dataset_ds_abc123
GROUP BY categoria
ORDER BY frequency DESC
LIMIT 20;
```

**Response**:
```json
{
  "dataset_id": "ds_abc123",
  "columns": {
    "preco": {
      "type": "numeric",
      "count": 1000000,
      "unique_count": 15432,
      "null_count": 120,
      "null_percentage": 0.012,
      "min": 9.90,
      "max": 999.90,
      "mean": 125.45,
      "median": 99.90,
      "std_dev": 78.23,
      "q25": 49.90,
      "q75": 199.90
    },
    "categoria": {
      "type": "categorical",
      "count": 1000000,
      "unique_count": 5,
      "null_count": 0,
      "top_values": [
        {"value": "Eletrônicos", "frequency": 450000, "percentage": 45.0},
        {"value": "Vestuário", "frequency": 300000, "percentage": 30.0},
        ...
      ]
    }
  }
}
```

### 3.5 Consultas Agregadas

**Endpoint**: `POST /api/v1/datasets/{dataset_id}/query`

**Request**:
```json
{
  "query": "SELECT categoria, SUM(quantidade) as total FROM dataset_ds_abc123 GROUP BY categoria ORDER BY total DESC"
}
```

**Validações**:
- ✅ Apenas queries SELECT permitidas
- ✅ Timeout configurável (ex: 30 segundos)
- ✅ Limite de resultados (ex: 10.000 linhas)
- ✅ Proteção contra SQL injection

**Response**:
```json
{
  "query": "SELECT categoria, SUM(quantidade)...",
  "rows": [
    {"categoria": "Eletrônicos", "total": 2500000},
    {"categoria": "Vestuário", "total": 1800000},
    ...
  ],
  "row_count": 5,
  "execution_time_ms": 42
}
```

### 3.6 Distribuição de Valores

**Endpoint**: `GET /api/v1/datasets/{dataset_id}/columns/preco/distribution?bins=10`

Gera histograma da coluna:

**Query SQL**:
```sql
SELECT
    width_bucket(preco, 9.90, 999.90, 10) as bin,
    COUNT(*) as frequency
FROM dataset_ds_abc123
WHERE preco IS NOT NULL
GROUP BY bin
ORDER BY bin;
```

**Response**:
```json
{
  "column": "preco",
  "bins": [
    {"range": "9.90-108.89", "frequency": 150000},
    {"range": "108.89-207.88", "frequency": 200000},
    ...
  ]
}
```

---

## ⚡ Performance

### Por que é Rápido?

1. **DuckDB é otimizado para análise**
   - Processa colunas em vez de linhas (columnar storage)
   - Usa vetorização SIMD
   - Executa queries em paralelo

2. **Sem transferência de dados**
   - Dados ficam em disco
   - Queries executam diretamente no DuckDB
   - Zero overhead de rede

3. **Compressão inteligente**
   - Reduz I/O de disco
   - Mais dados cabem em memória cache

### Benchmarks Estimados

Para um CSV de 1GB (10 milhões de linhas):

| Operação | Tempo Estimado |
|----------|----------------|
| Upload | 5-10 segundos |
| Import DuckDB | 10-20 segundos |
| Preview (10 linhas) | < 100ms |
| Stats básicas | 1-2 segundos |
| Profiling completo | 5-10 segundos |
| Query agregada simples | 1-3 segundos |
| Query agregada complexa | 3-10 segundos |

*Tempos variam conforme hardware e complexidade dos dados*

---

## 🔄 Fluxo Completo - Diagrama

```
┌─────────────┐
│   Cliente   │
└──────┬──────┘
       │
       │ 1. POST /upload (CSV file)
       ▼
┌─────────────────────────────┐
│     FastAPI Endpoint        │
│  - Valida arquivo           │
│  - Salva temporariamente    │
│  - Retorna dataset_id       │
└──────┬──────────────────────┘
       │
       │ 2. Trigger import
       ▼
┌─────────────────────────────┐
│    Dataset Service          │
│  - Cria dataset_id único    │
│  - Chama DuckDB Repository  │
└──────┬──────────────────────┘
       │
       │ 3. COPY ... FROM CSV
       ▼
┌─────────────────────────────┐
│   DuckDB Repository         │
│  - CREATE TABLE AS SELECT   │
│  - read_csv_auto()          │
│  - Inferência de schema     │
└──────┬──────────────────────┘
       │
       │ 4. Processa CSV em chunks
       ▼
┌─────────────────────────────┐
│        DuckDB               │
│  - Leitura paralela         │
│  - Compressão               │
│  - Indexação                │
│  - Salva no .duckdb file    │
└──────┬──────────────────────┘
       │
       │ 5. Import completo
       ▼
┌─────────────────────────────┐
│    Dataset Service          │
│  - Salva metadados          │
│  - Remove arquivo temp      │
│  - Retorna sucesso          │
└──────┬──────────────────────┘
       │
       │ 6. Response
       ▼
┌─────────────┐
│   Cliente   │ ← {"dataset_id": "ds_abc123", "status": "ready"}
└──────┬──────┘
       │
       │ 7. GET /datasets/ds_abc123/preview
       ▼
┌─────────────────────────────┐
│     FastAPI Endpoint        │
└──────┬──────────────────────┘
       │
       │ 8. Query DuckDB
       ▼
┌─────────────────────────────┐
│        DuckDB               │
│  - SELECT * LIMIT 10        │
│  - Execução vetorizada      │
└──────┬──────────────────────┘
       │
       │ 9. Resultados
       ▼
┌─────────────┐
│   Cliente   │ ← Preview data (10 rows)
└─────────────┘
```

---

## 🎯 Casos de Uso

### Caso 1: Análise Exploratória
1. Upload do CSV
2. Preview dos dados
3. Obter schema e tipos
4. Gerar perfil completo
5. Identificar problemas de qualidade

### Caso 2: Queries Ad-hoc
1. Upload do CSV
2. Executar queries SQL customizadas
3. Exportar resultados (futuro)

### Caso 3: Dashboard Simples
1. Upload do CSV
2. APIs de agregação predefinidas
3. Frontend consome os dados
4. Visualizações dinâmicas

---

## 📊 Exemplo Prático Completo

### 1. Upload
```bash
curl -X POST http://localhost:8000/api/v1/datasets/upload \
  -F "file=@vendas_2024.csv" \
  -F "name=Vendas 2024" \
  -F "description=Dados completos de vendas"
```

**Response**:
```json
{
  "dataset_id": "ds_abc123",
  "status": "ready",
  "row_count": 1000000,
  "column_count": 6
}
```

### 2. Preview
```bash
curl http://localhost:8000/api/v1/datasets/ds_abc123/preview?limit=5
```

### 3. Estatísticas
```bash
curl http://localhost:8000/api/v1/datasets/ds_abc123/stats
```

### 4. Query Customizada
```bash
curl -X POST http://localhost:8000/api/v1/datasets/ds_abc123/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT categoria, COUNT(*) as vendas, SUM(preco * quantidade) as receita FROM dataset_ds_abc123 GROUP BY categoria ORDER BY receita DESC"
  }'
```

**Response**:
```json
{
  "rows": [
    {"categoria": "Eletrônicos", "vendas": 450000, "receita": 45000000.00},
    {"categoria": "Vestuário", "vendas": 300000, "receita": 12000000.00},
    ...
  ],
  "execution_time_ms": 1250
}
```

---

## 🔐 Segurança

### Proteções Implementadas

1. **Upload**
   - Limite de tamanho
   - Validação de extensão
   - Sanitização de nomes

2. **Queries**
   - Apenas SELECT permitido
   - Queries parametrizadas
   - Timeout configurável
   - Limite de resultados

3. **Recursos**
   - Limite de memória DuckDB
   - Rate limiting (futuro)
   - Autenticação (futuro)

---

## 🚀 Próximos Passos

Após entender o fluxo, o próximo passo é implementar:

1. **Estrutura básica** (pastas, configs)
2. **FastAPI app** (main.py)
3. **Endpoints** (um por vez)
4. **Services** (lógica de negócio)
5. **Repositories** (acesso ao DuckDB)
6. **Testes** (unitários e integração)
