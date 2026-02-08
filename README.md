# :earth_americas: GDP Dashboard & CSV Analyzer MVP

Este repositório contém dois projetos:

1. **GDP Dashboard** - Um dashboard Streamlit simples mostrando o PIB de diferentes países
2. **CSV Analyzer MVP** - Um MVP local para análise de arquivos CSV grandes usando FastAPI + DuckDB (NOVO)

---

## 📊 CSV Analyzer MVP - FastAPI + DuckDB

MVP local para analisar arquivos CSV grandes sem carregar tudo em memória.

### Características

- ✅ Processar CSV grandes sem carregar tudo em memória
- ✅ Persistir dados em DuckDB para consultas rápidas
- ✅ API RESTful com FastAPI
- ✅ Endpoints para preview, profiling e consultas agregadas
- ✅ Documentação automática (Swagger/OpenAPI)
- ✅ Execução totalmente local

### Documentação

- [📐 Arquitetura](ARCHITECTURE.md) - Decisões técnicas e arquitetura do sistema
- [📁 Estrutura do Projeto](PROJECT_STRUCTURE.md) - Organização de pastas e arquivos

### Como executar (FastAPI MVP)

#### 1. Instale as dependências

```bash
pip install -r requirements.txt
```

Para desenvolvimento (inclui ferramentas de teste e linting):

```bash
pip install -r requirements.txt -r requirements-dev.txt
```

#### 2. Configure as variáveis de ambiente

Copie o arquivo de exemplo e ajuste conforme necessário:

```bash
cp .env.example .env
```

#### 3. Execute a aplicação

```bash
uvicorn app.main:app --reload
```

A API estará disponível em: http://localhost:8000

Documentação interativa (Swagger): http://localhost:8000/docs

### Endpoints Principais

- `POST /api/v1/datasets/upload` - Upload de arquivo CSV
- `GET /api/v1/datasets` - Listar datasets importados
- `GET /api/v1/datasets/{dataset_id}/preview` - Preview das primeiras linhas
- `GET /api/v1/datasets/{dataset_id}/profile` - Perfil completo do dataset
- `POST /api/v1/datasets/{dataset_id}/query` - Executar query SQL customizada

### Fluxo de Uso

1. **Upload**: Envie um arquivo CSV via endpoint de upload
2. **Import**: O sistema importa para DuckDB automaticamente
3. **Análise**: Use os endpoints para explorar e analisar os dados

### Tecnologias

- **FastAPI** - Framework web moderno e rápido
- **DuckDB** - Banco de dados analítico in-process
- **Pydantic** - Validação de dados
- **Uvicorn** - Servidor ASGI

---

## 🌍 GDP Dashboard (Original)

A simple Streamlit app showing the GDP of different countries in the world.

[![Open in Streamlit](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://gdp-dashboard-template.streamlit.app/)

### How to run it on your own machine

1. Install the requirements

   ```
   $ pip install -r requirements.txt
   ```

2. Run the app

   ```
   $ streamlit run streamlit_app.py
   ```

---

## 📝 Licença

Ver arquivo [LICENSE](LICENSE)
