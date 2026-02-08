# Arquitetura do MVP - Análise de CSV Grandes com FastAPI + DuckDB

## 1. Visão Geral

Este documento descreve a arquitetura proposta para um MVP local que permite analisar arquivos CSV grandes sem carregar tudo em memória, utilizando FastAPI como framework web e DuckDB como banco de dados analítico.

## 2. Objetivos

- ✅ Processar CSV grandes sem carregar tudo em memória
- ✅ Persistir dados em DuckDB para consultas rápidas
- ✅ Expor endpoints RESTful para preview, profiling e consultas agregadas
- ✅ Backend apenas (frontend será desenvolvido posteriormente)
- ✅ Execução totalmente local

## 3. Decisões Técnicas

### 3.1 FastAPI
**Por que FastAPI?**
- Performance alta (baseado em Starlette e Pydantic)
- Documentação automática (Swagger/OpenAPI)
- Suporte nativo para async/await
- Validação de dados automática
- Tipagem com Python type hints

### 3.2 DuckDB
**Por que DuckDB?**
- Banco de dados analítico in-process (sem servidor separado)
- Otimizado para consultas OLAP (Online Analytical Processing)
- Leitura eficiente de CSV sem carregar tudo em memória
- Suporta consultas SQL padrão
- Zero configuração - arquivo único
- Excelente para análise de dados

### 3.3 Chunked Processing
- Upload de arquivos em chunks para evitar timeout
- Importação incremental para DuckDB
- Processamento assíncrono quando possível

## 4. Arquitetura Proposta

### 4.1 Estrutura de Camadas

```
┌─────────────────────────────────────┐
│         API Layer (FastAPI)         │
│  - Endpoints REST                   │
│  - Validação de requests            │
│  - Documentação automática          │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│       Service Layer (Business)      │
│  - Lógica de negócio                │
│  - Orquestração de operações        │
│  - Tratamento de erros              │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│     Repository Layer (Data Access)  │
│  - Queries DuckDB                   │
│  - Gerenciamento de conexões        │
│  - Mapeamento de dados              │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│          DuckDB Database            │
│  - Persistência de dados            │
│  - Execução de queries SQL          │
└─────────────────────────────────────┘
```

### 4.2 Componentes Principais

#### API Endpoints
1. **Upload & Import**
   - `POST /api/v1/datasets/upload` - Upload de arquivo CSV
   - `GET /api/v1/datasets` - Listar datasets importados
   - `DELETE /api/v1/datasets/{dataset_id}` - Remover dataset

2. **Preview & Exploration**
   - `GET /api/v1/datasets/{dataset_id}/preview` - Preview das primeiras linhas
   - `GET /api/v1/datasets/{dataset_id}/schema` - Esquema das colunas
   - `GET /api/v1/datasets/{dataset_id}/stats` - Estatísticas básicas

3. **Profiling**
   - `GET /api/v1/datasets/{dataset_id}/profile` - Perfil completo do dataset
   - `GET /api/v1/datasets/{dataset_id}/columns/{column_name}/distribution` - Distribuição de uma coluna

4. **Queries Agregadas**
   - `POST /api/v1/datasets/{dataset_id}/query` - Executar query SQL customizada
   - `GET /api/v1/datasets/{dataset_id}/aggregate` - Agregações predefinidas

#### Services
- **DatasetService**: Gerencia operações de datasets
- **ProfilingService**: Gera perfis e estatísticas
- **QueryService**: Executa consultas customizadas

#### Repositories
- **DuckDBRepository**: Abstração para operações DuckDB
- **DatasetRepository**: CRUD de datasets

## 5. Fluxo de Dados

### 5.1 Upload → Import → Análise

```
┌──────────┐
│ Cliente  │
└────┬─────┘
     │
     │ 1. POST /api/v1/datasets/upload
     │    (Multipart form-data com CSV)
     ▼
┌─────────────────┐
│  API Endpoint   │
│  - Valida file  │
│  - Salva temp   │
└────┬────────────┘
     │
     │ 2. Chama DatasetService.import_csv()
     ▼
┌──────────────────────────────┐
│     DatasetService           │
│  - Gera dataset_id único     │
│  - Cria tabela no DuckDB     │
│  - Importa CSV usando:       │
│    COPY table FROM 'file.csv'│
└────┬─────────────────────────┘
     │
     │ 3. DuckDB lê CSV em chunks
     ▼
┌──────────────────┐
│     DuckDB       │
│  - Leitura lazy  │
│  - Compressão    │
│  - Indexação     │
└────┬─────────────┘
     │
     │ 4. Retorna dataset_id
     ▼
┌──────────┐
│ Cliente  │ ← dataset_id
└──────────┘

     │
     │ 5. GET /api/v1/datasets/{dataset_id}/preview
     ▼
┌─────────────────┐
│  API Endpoint   │
└────┬────────────┘
     │
     │ 6. Chama QueryService
     ▼
┌──────────────────────────────┐
│     QueryService             │
│  - Executa: SELECT * LIMIT 10│
└────┬─────────────────────────┘
     │
     │ 7. Retorna resultados
     ▼
┌──────────┐
│ Cliente  │ ← Preview data
└──────────┘
```

### 5.2 Detalhamento do Fluxo

**Fase 1: Upload**
1. Cliente envia arquivo CSV via multipart/form-data
2. FastAPI recebe o arquivo em chunks (não carrega tudo em memória)
3. Arquivo é salvo temporariamente em disco
4. Validações básicas (tamanho, formato)

**Fase 2: Import**
1. DuckDB cria uma nova tabela com nome único
2. Comando `COPY` importa o CSV diretamente
3. DuckDB faz leitura lazy e inferência de schema
4. Dados são comprimidos e indexados automaticamente
5. Arquivo temporário é removido
6. Metadados do dataset são salvos (nome, tamanho, data, schema)

**Fase 3: Análise**
1. Cliente pode fazer preview (SELECT * LIMIT N)
2. Obter estatísticas (COUNT, AVG, MIN, MAX, etc.)
3. Profiling completo (distribuições, valores únicos, nulls)
4. Queries agregadas customizadas
5. Todas as operações são executadas pelo DuckDB de forma otimizada

## 6. Vantagens da Arquitetura

### 6.1 Performance
- DuckDB processa CSV sem carregar tudo em memória
- Queries vetorizadas e paralelas
- Compressão automática de dados

### 6.2 Escalabilidade
- Suporta CSV de vários GB
- Processamento incremental
- Conexões de pool (se necessário no futuro)

### 6.3 Manutenibilidade
- Separação clara de responsabilidades (camadas)
- Código type-safe com Pydantic
- Testes unitários facilitados
- Documentação automática da API

### 6.4 Simplicidade
- Zero configuração de banco de dados
- Execução local sem dependências externas
- Deploy simples (apenas Python + bibliotecas)

## 7. Limitações e Considerações

### 7.1 Limitações Conhecidas
- Não suporta múltiplos usuários simultâneos escrevendo (DuckDB é single-writer)
- Arquivo DuckDB cresce com os dados (precisa de espaço em disco)
- Não é adequado para OLTP (transações online)

### 7.2 Futuras Melhorias
- Adicionar autenticação/autorização
- Implementar cache de queries frequentes
- Suportar outros formatos (Parquet, JSON)
- Interface web (frontend)
- Export de resultados
- Scheduled jobs para análises recorrentes

## 8. Segurança

### 8.1 Medidas de Segurança
- Validação de tamanho máximo de arquivo
- Sanitização de nomes de tabelas/colunas
- Queries parametrizadas (proteção contra SQL injection)
- Validação de tipos com Pydantic
- Rate limiting (futuro)

## 9. Conclusão

Esta arquitetura proporciona uma solução robusta, performática e simples para análise de CSV grandes localmente. A combinação de FastAPI + DuckDB oferece o melhor dos dois mundos: APIs modernas e rápidas com capacidades analíticas poderosas, sem a complexidade de configurar infraestrutura adicional.
