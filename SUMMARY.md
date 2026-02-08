# MVP FastAPI + DuckDB - Resumo Executivo

## 📋 Visão Geral

Este documento é um resumo executivo da proposta de arquitetura para o MVP de análise de CSV grandes usando FastAPI + DuckDB.

## 🎯 Objetivo

Criar um MVP **local** que permite analisar arquivos CSV grandes (até vários GB) sem carregar tudo em memória, oferecendo:

- API RESTful para upload e análise
- Persistência eficiente em DuckDB
- Endpoints para preview, profiling e consultas agregadas
- **Backend apenas** (frontend será desenvolvido posteriormente)

## 📚 Documentação Disponível

### 1. [ARCHITECTURE.md](ARCHITECTURE.md)
**Conteúdo**: Arquitetura completa do sistema
- Decisões técnicas (por que FastAPI? por que DuckDB?)
- Estrutura de camadas (API → Service → Repository → Database)
- Componentes principais e endpoints
- Vantagens, limitações e considerações de segurança

**Leia se**: Você quer entender as decisões técnicas e a arquitetura geral

### 2. [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md)
**Conteúdo**: Estrutura de pastas e arquivos
- Árvore completa de diretórios
- Descrição de cada pasta e seu propósito
- Padrões de nomenclatura
- Convenções de código
- Arquivos de configuração

**Leia se**: Você vai implementar o código e precisa saber onde colocar cada arquivo

### 3. [FLOW.md](FLOW.md)
**Conteúdo**: Fluxo detalhado de processamento
- Fase 1: Upload de arquivo (passo a passo)
- Fase 2: Import para DuckDB (como funciona internamente)
- Fase 3: Análise dos dados (todos os tipos de queries)
- Diagramas e exemplos práticos completos
- Benchmarks de performance

**Leia se**: Você quer entender o fluxo de dados de ponta a ponta

### 4. [README.md](README.md)
**Conteúdo**: Instruções de uso
- Como instalar dependências
- Como configurar o ambiente
- Como executar a aplicação
- Endpoints principais

**Leia se**: Você quer começar a usar o sistema

## 🏗️ Arquitetura em Uma Página

```
┌─────────────────────────────────────────────────────────────┐
│                    Cliente (cURL, Postman, Frontend)        │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP/REST
┌────────────────────────────▼────────────────────────────────┐
│                        FastAPI                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  API Endpoints (v1)                                  │   │
│  │  • POST /datasets/upload                             │   │
│  │  • GET  /datasets/{id}/preview                       │   │
│  │  • GET  /datasets/{id}/profile                       │   │
│  │  • POST /datasets/{id}/query                         │   │
│  └────────────────────┬─────────────────────────────────┘   │
└─────────────────────────┼─────────────────────────────────-─┘
                          │
┌─────────────────────────▼─────────────────────────────────┐
│                    Services Layer                         │
│  • DatasetService: Upload, import, metadata              │
│  • ProfilingService: Stats, distributions                │
│  • QueryService: SQL queries customizadas                │
└─────────────────────────┬─────────────────────────────────┘
                          │
┌─────────────────────────▼─────────────────────────────────┐
│                  Repositories Layer                       │
│  • DuckDBRepository: Conexão e queries DuckDB            │
│  • DatasetRepository: CRUD de datasets                    │
└─────────────────────────┬─────────────────────────────────┘
                          │
┌─────────────────────────▼─────────────────────────────────┐
│                       DuckDB                              │
│  • Arquivo: storage/duckdb/analytics.duckdb              │
│  • Leitura lazy de CSV (não carrega tudo em memória)    │
│  • Compressão e indexação automática                     │
│  • Queries vetorizadas e paralelas                       │
└───────────────────────────────────────────────────────────┘
```

## 🛠️ Stack Tecnológica

| Componente | Tecnologia | Versão Mínima | Por quê? |
|------------|------------|---------------|----------|
| **Framework Web** | FastAPI | 0.104.0 | Performance, validação automática, docs |
| **Banco de Dados** | DuckDB | 0.9.0 | Analítico, in-process, leitura lazy de CSV |
| **Servidor ASGI** | Uvicorn | 0.24.0 | Async, compatível com FastAPI |
| **Validação** | Pydantic | 2.4.0 | Type-safe, schemas automáticos |
| **Upload** | python-multipart | 0.0.6 | Suporte a multipart/form-data |
| **Config** | pydantic-settings | 2.0.0 | Gerenciamento de configurações |
| **Env Vars** | python-dotenv | 1.0.0 | Carregar variáveis de .env |

## 🗂️ Estrutura de Pastas (Resumida)

```
gdp-dashboard/
├── app/                    # Aplicação FastAPI
│   ├── api/v1/            # Endpoints REST
│   ├── services/          # Lógica de negócio
│   ├── repositories/      # Acesso a dados
│   ├── models/            # Schemas Pydantic
│   └── core/              # Config, logging, exceptions
│
├── storage/               # Dados persistidos
│   ├── duckdb/           # Arquivos .duckdb
│   └── uploads/          # CSV temporários
│
├── tests/                # Testes
│   ├── unit/
│   └── integration/
│
├── docs/                 # Docs adicionais
├── scripts/              # Scripts utilitários
│
├── .env.example          # Exemplo de configuração
├── requirements.txt      # Dependências
└── requirements-dev.txt  # Dependências de dev
```

## 🔄 Fluxo Simplificado

### Upload → Import → Análise

1. **Upload** (Cliente → FastAPI)
   ```
   POST /api/v1/datasets/upload
   Body: multipart/form-data com CSV
   ```
   - FastAPI recebe arquivo em chunks
   - Salva temporariamente em `storage/uploads/`
   - Retorna `dataset_id`

2. **Import** (FastAPI → DuckDB)
   ```sql
   CREATE TABLE dataset_xyz AS 
   SELECT * FROM read_csv_auto('arquivo.csv');
   ```
   - DuckDB lê CSV sem carregar tudo em memória
   - Infere schema automaticamente
   - Comprime e indexa dados
   - Salva em `storage/duckdb/analytics.duckdb`

3. **Análise** (Cliente → FastAPI → DuckDB)
   ```
   GET /api/v1/datasets/{id}/preview
   GET /api/v1/datasets/{id}/profile
   POST /api/v1/datasets/{id}/query
   ```
   - Queries SQL executadas diretamente no DuckDB
   - Resultados retornados em JSON
   - Performance alta (vetorização, paralelismo)

## ✅ Checklist de Implementação

### Fase 1: Setup Inicial
- [ ] Criar estrutura de pastas
- [ ] Configurar `.env` baseado em `.env.example`
- [ ] Instalar dependências (`pip install -r requirements.txt`)
- [ ] Criar `app/main.py` básico

### Fase 2: Core & Config
- [ ] Implementar `app/core/config.py` (Settings)
- [ ] Implementar `app/core/exceptions.py`
- [ ] Implementar `app/core/logging.py`

### Fase 3: Models (Schemas)
- [ ] Criar schemas de Dataset
- [ ] Criar schemas de Query
- [ ] Criar schemas de Response

### Fase 4: Repositories
- [ ] Implementar `DuckDBRepository` (conexão, queries básicas)
- [ ] Implementar `DatasetRepository` (CRUD)

### Fase 5: Services
- [ ] Implementar `DatasetService` (upload, import)
- [ ] Implementar `QueryService` (preview, stats)
- [ ] Implementar `ProfilingService` (profiling completo)

### Fase 6: API Endpoints
- [ ] Endpoint de upload
- [ ] Endpoints de preview e schema
- [ ] Endpoints de profiling
- [ ] Endpoint de query customizada

### Fase 7: Testes
- [ ] Testes unitários de services
- [ ] Testes de integração de endpoints
- [ ] Testes com CSV de exemplo

### Fase 8: Documentação & Refinamento
- [ ] Melhorar docstrings
- [ ] Adicionar exemplos na docs do Swagger
- [ ] README com exemplos práticos

## 🎬 Como Começar?

### 1. Clone e Instale
```bash
git clone <repo-url>
cd gdp-dashboard
pip install -r requirements.txt
```

### 2. Configure
```bash
cp .env.example .env
# Edite .env se necessário
```

### 3. Crie a Estrutura
```bash
# A estrutura de pastas proposta está em PROJECT_STRUCTURE.md
mkdir -p app/api/v1/endpoints
mkdir -p app/core app/models app/services app/repositories app/utils
mkdir -p tests/unit tests/integration
# ... etc
```

### 4. Implemente
Siga o checklist acima, implementando um componente por vez:
- Comece pelos arquivos de configuração (`core/config.py`)
- Depois os repositories (acesso ao DuckDB)
- Depois os services (lógica de negócio)
- Por fim os endpoints (API REST)

### 5. Teste
```bash
# Teste manualmente
uvicorn app.main:app --reload

# Acesse a documentação
# http://localhost:8000/docs

# Testes automatizados
pytest tests/
```

## 📊 Exemplo de Uso (Quando Implementado)

```bash
# 1. Upload de CSV
curl -X POST http://localhost:8000/api/v1/datasets/upload \
  -F "file=@dados.csv" \
  -F "name=Meu Dataset"

# Resposta: {"dataset_id": "ds_abc123", "status": "ready"}

# 2. Preview
curl http://localhost:8000/api/v1/datasets/ds_abc123/preview?limit=5

# 3. Profiling
curl http://localhost:8000/api/v1/datasets/ds_abc123/profile

# 4. Query Customizada
curl -X POST http://localhost:8000/api/v1/datasets/ds_abc123/query \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT coluna, COUNT(*) FROM dataset_ds_abc123 GROUP BY coluna"}'
```

## 🚀 Próximos Passos

1. **Ler a documentação completa**
   - ARCHITECTURE.md para entender o design
   - PROJECT_STRUCTURE.md para saber onde colocar código
   - FLOW.md para entender o fluxo de dados

2. **Implementar progressivamente**
   - Não tente fazer tudo de uma vez
   - Comece pelo mínimo viável (upload + preview)
   - Adicione features incrementalmente
   - Teste cada componente antes de avançar

3. **Ajustar conforme necessário**
   - A arquitetura proposta é flexível
   - Adapte às suas necessidades específicas
   - Mantenha a separação de responsabilidades

## 💡 Dicas Importantes

1. **DuckDB é a chave**
   - Ele faz a "mágica" de processar CSV sem carregar tudo em memória
   - Use `read_csv_auto()` para inferência automática de schema
   - Aproveite as capacidades analíticas (funções de agregação, window functions, etc.)

2. **FastAPI facilita muito**
   - Documentação automática (Swagger) é gerada
   - Validação automática com Pydantic
   - Async/await para operações I/O

3. **Mantenha simples**
   - Comece com o básico que funciona
   - Adicione complexidade apenas quando necessário
   - Documente decisões importantes

4. **Teste cedo e frequentemente**
   - Testes unitários para lógica de negócio
   - Testes de integração para fluxos completos
   - Teste com CSVs de diferentes tamanhos e formatos

## 📞 Suporte

Para dúvidas sobre a arquitetura proposta:
1. Consulte os documentos específicos (ARCHITECTURE.md, FLOW.md, etc.)
2. Revise os exemplos práticos no FLOW.md
3. Verifique a estrutura de pastas no PROJECT_STRUCTURE.md

---

**Última atualização**: 2024-02-08

**Status**: Documentação completa ✅ | Implementação pendente ⏳
