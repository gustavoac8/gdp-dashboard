# Estrutura de Pastas do Projeto

## Árvore de Diretórios Proposta

```
gdp-dashboard/
│
├── app/                          # Aplicação principal
│   ├── __init__.py
│   ├── main.py                   # Entry point do FastAPI
│   │
│   ├── api/                      # Camada de API
│   │   ├── __init__.py
│   │   ├── deps.py               # Dependências compartilhadas (DB connection)
│   │   └── v1/                   # API versão 1
│   │       ├── __init__.py
│   │       ├── router.py         # Router principal da v1
│   │       └── endpoints/        # Endpoints organizados por domínio
│   │           ├── __init__.py
│   │           ├── datasets.py   # Upload, list, delete datasets
│   │           ├── preview.py    # Preview e schema
│   │           ├── profiling.py  # Estatísticas e profiling
│   │           └── query.py      # Consultas customizadas
│   │
│   ├── core/                     # Configurações e utilitários core
│   │   ├── __init__.py
│   │   ├── config.py             # Configurações (settings)
│   │   ├── logging.py            # Configuração de logs
│   │   └── exceptions.py         # Exceções customizadas
│   │
│   ├── models/                   # Modelos Pydantic (schemas)
│   │   ├── __init__.py
│   │   ├── dataset.py            # Schemas de dataset
│   │   ├── query.py              # Schemas de query
│   │   └── response.py           # Schemas de resposta
│   │
│   ├── services/                 # Lógica de negócio
│   │   ├── __init__.py
│   │   ├── dataset_service.py    # Serviço de datasets
│   │   ├── profiling_service.py  # Serviço de profiling
│   │   └── query_service.py      # Serviço de queries
│   │
│   ├── repositories/             # Camada de acesso a dados
│   │   ├── __init__.py
│   │   ├── base.py               # Repository base
│   │   ├── duckdb_repository.py  # Abstração DuckDB
│   │   └── dataset_repository.py # Repository de datasets
│   │
│   └── utils/                    # Utilitários gerais
│       ├── __init__.py
│       ├── file_utils.py         # Manipulação de arquivos
│       ├── validation.py         # Validações customizadas
│       └── csv_utils.py          # Utilitários para CSV
│
├── tests/                        # Testes
│   ├── __init__.py
│   ├── conftest.py               # Fixtures do pytest
│   ├── unit/                     # Testes unitários
│   │   ├── __init__.py
│   │   ├── test_services/
│   │   ├── test_repositories/
│   │   └── test_utils/
│   └── integration/              # Testes de integração
│       ├── __init__.py
│       └── test_api/
│
├── storage/                      # Armazenamento local
│   ├── duckdb/                   # Arquivos do banco DuckDB
│   │   └── .gitkeep
│   └── uploads/                  # Upload temporário de CSVs
│       └── .gitkeep
│
├── data/                         # Dados de exemplo (existente)
│   └── gdp_data.csv
│
├── docs/                         # Documentação adicional
│   ├── api.md                    # Documentação da API
│   └── examples.md               # Exemplos de uso
│
├── scripts/                      # Scripts utilitários
│   ├── setup_db.py               # Setup inicial do DuckDB
│   └── sample_data.py            # Gerar dados de exemplo
│
├── .env.example                  # Exemplo de variáveis de ambiente
├── .gitignore                    # Git ignore (atualizado)
├── ARCHITECTURE.md               # Documento de arquitetura
├── PROJECT_STRUCTURE.md          # Este arquivo
├── README.md                     # README atualizado
├── requirements.txt              # Dependências Python
├── requirements-dev.txt          # Dependências de desenvolvimento
└── streamlit_app.py              # App Streamlit original (mantido)
```

## Descrição dos Diretórios

### `/app` - Aplicação Principal
Contém todo o código da aplicação FastAPI organizado em camadas.

### `/app/api` - Camada de API
- **deps.py**: Injeção de dependências (ex: obter conexão DuckDB)
- **v1/**: Versionamento da API para facilitar futuras mudanças
- **endpoints/**: Endpoints organizados por funcionalidade

### `/app/core` - Core da Aplicação
- **config.py**: Configurações centralizadas usando Pydantic Settings
- **logging.py**: Setup de logging estruturado
- **exceptions.py**: Exceções HTTP customizadas

### `/app/models` - Schemas Pydantic
Modelos para validação de request/response:
- Schemas de entrada (requests)
- Schemas de saída (responses)
- DTOs (Data Transfer Objects)

### `/app/services` - Camada de Serviço
Lógica de negócio isolada dos endpoints:
- Orquestração de múltiplos repositories
- Regras de negócio
- Transformações de dados

### `/app/repositories` - Camada de Dados
Abstração do acesso aos dados:
- Queries ao DuckDB
- Gerenciamento de conexões
- Mapeamento de dados

### `/app/utils` - Utilitários
Funções auxiliares reutilizáveis:
- Manipulação de arquivos
- Validações
- Helpers para CSV

### `/tests` - Testes Automatizados
- **unit/**: Testes unitários (isolados, mockados)
- **integration/**: Testes de integração (com DuckDB real)

### `/storage` - Armazenamento Local
- **duckdb/**: Arquivos `.duckdb` (banco de dados)
- **uploads/**: CSVs temporários durante upload

### `/docs` - Documentação
Documentação adicional além do README e ARCHITECTURE:
- Guias de uso da API
- Exemplos práticos
- Tutoriais

### `/scripts` - Scripts Utilitários
Scripts para setup, manutenção e testes:
- Inicialização do banco
- Geração de dados de exemplo
- Migração de dados

## Padrões de Nomenclatura

### Arquivos Python
- **snake_case** para módulos e pacotes: `dataset_service.py`
- **PascalCase** para classes: `DatasetService`
- **snake_case** para funções e variáveis: `get_dataset_by_id()`

### Diretórios
- **snake_case** para todos os diretórios: `app/`, `test_services/`
- Plural para coleções: `models/`, `services/`, `repositories/`

### Constantes
- **UPPER_SNAKE_CASE**: `MAX_FILE_SIZE`, `DEFAULT_LIMIT`

## Convenções de Importação

```python
# Imports padrão do Python
import os
from pathlib import Path

# Imports de terceiros
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

# Imports locais (do próprio projeto)
from app.core.config import settings
from app.services.dataset_service import DatasetService
```

## Arquivos de Configuração

### `.env.example`
```env
# Configurações da aplicação
APP_NAME=CSV Analyzer MVP
APP_VERSION=1.0.0
DEBUG=True

# DuckDB
DUCKDB_PATH=./storage/duckdb/analytics.duckdb
DUCKDB_MEMORY_LIMIT=4GB

# Upload
MAX_UPLOAD_SIZE=1GB
UPLOAD_DIR=./storage/uploads

# API
API_V1_PREFIX=/api/v1
```

### `requirements.txt`
Dependências de produção (ver arquivo separado)

### `requirements-dev.txt`
```
# Dependências de desenvolvimento
pytest>=7.4.0
pytest-asyncio>=0.21.0
pytest-cov>=4.1.0
httpx>=0.24.0        # Cliente HTTP para testes
black>=23.0.0        # Formatação
flake8>=6.0.0        # Linting
mypy>=1.4.0          # Type checking
```

## Observações Importantes

### 1. Separação de Responsabilidades
Cada camada tem uma responsabilidade clara:
- **API**: Receber requests, validar, retornar responses
- **Services**: Lógica de negócio
- **Repositories**: Acesso a dados

### 2. Testabilidade
A estrutura facilita testes porque:
- Cada camada pode ser testada isoladamente
- Dependências podem ser mockadas facilmente
- Separação clara entre lógica e I/O

### 3. Escalabilidade
Fácil adicionar novos recursos:
- Novo endpoint? Adicione em `endpoints/`
- Nova regra de negócio? Adicione em `services/`
- Nova fonte de dados? Adicione em `repositories/`

### 4. Manutenibilidade
- Código organizado e previsível
- Fácil localizar funcionalidades
- Convenções consistentes

## Próximos Passos

Após criar a estrutura:
1. Criar os arquivos `__init__.py` em todos os pacotes
2. Implementar `app/core/config.py` com settings
3. Implementar `app/main.py` com FastAPI app
4. Criar endpoints básicos em `app/api/v1/endpoints/`
5. Implementar services e repositories
6. Adicionar testes
