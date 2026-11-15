# Projeto SI — Autenticação por Dinâmica de Digitação

Sistema completo de autenticação biométrica comportamental baseada em **dinâmica de digitação** (*Keystroke Dynamics*). A aplicação combina verificação tradicional de usuário/senha com classificação biométrica adaptativa individual por usuário.

---

## Arquitetura e Submódulos

O repositório principal orquestra os seguintes submódulos:

| Diretório | Stack | Responsabilidade |
|---|---|---|
| `projeto_si_frontend` | Angular 18 + Nginx | Interface web de cadastro, coleta de amostras (5 repetições) e login biométrico com captura e extração local de `features` (`KEY_DOWN` / `KEY_UP` via `computeSample`). |
| `projeto-si-backend` | .NET 10 (Minimal API) + EF Core | API de entrada (`/api/*`), gestão de usuários, hashing de senha (PBKDF2-HMAC-SHA512), validação de `features` e ordem esperada contra a senha do usuário, mascaramento posicional (`H.0, DD.0.1, UD.0.1, ...`, tamanho `3n - 2`), persistência seletiva em `amostras` e delegação biométrica para a ML API. |
| `projeto_si_ml_api` | Python 3.12 + Django + DRF + scikit-learn | Treinamento inicial, inferência biométrica com **`M2005_Pisani2024_Sliding_Dynamic`** (janela deslizante de 50 amostras e limiar dinâmico), sinalização de gravação para retreinamento (`shouldRecordForRetraining`) e agendamento de retreinamento contínuo (`ml-scheduler`). |
| `projeto_si_experimentos` | Python | Experimentos e avaliação comparativa dos algoritmos biométricos que embasaram a escolha do classificador. |

### Fluxo de Dados e Banco Compartilhado

1. O **Frontend** calcula localmente as `features` (em ordem cronológica via `computeSample`) e envia ao **Backend .NET** (`/api`).
2. O **Backend .NET** valida credenciais, confere se as chaves e a ordem de `features` correspondem exatamente à senha do usuário, converte/mascara para o vetor posicional CMU (`double[]`) sem gravar caracteres da senha e delega à **ML API** via HTTP.
3. A **ML API** cria/consulta `modelos` + artefatos `.joblib` (SHA-256), apenas lê `amostras` (sem criar linhas) e retorna na autenticação `shouldRecordForRetraining` indicando ao **Backend .NET** se a amostra deve ser salva no **PostgreSQL compartilhado** para retreinamento futuro.

---

## Clonando o Repositório

Como o projeto utiliza submódulos Git, clone com `--recurse-submodules`:

```bash
git clone --recurse-submodules <url-do-repositorio>
cd projeto_si
```

Caso já tenha clonado sem os submódulos:

```bash
git submodule update --init --recursive
```

---

## Executando com Docker

### Pré-requisitos
- [Docker](https://docs.docker.com/get-docker/) e Docker Compose (`docker compose`).

### Subindo toda a stack

Na raiz do repositório (`projeto_si/`):

```bash
docker compose up --build
```

Ou em segundo plano (`detached`):

```bash
docker compose up --build -d
```

O `docker-compose.yml` sobe 5 serviços na rede `keystroke-network`:
- `postgres`: PostgreSQL 17 compartilhado (inicializado automaticamente com `projeto-si-backend/schema.sql`).
- `backend`: API .NET 10 na porta `5000`.
- `ml-api`: API Django + Gunicorn na porta `8000` (executa `migrate --fake-initial` automaticamente).
- `ml-scheduler`: Worker APScheduler para retreinamento periódico dos modelos.
- `frontend`: Aplicação Angular servida via Nginx na porta `4200` (com proxy de `/api/` para `backend:5000`).

### URLs de Acesso (Docker)
- **Frontend:** http://localhost:4200
- **Backend API (.NET):** http://localhost:5000 (Healthcheck: http://localhost:5000/api/health)
- **ML API (Django):** http://localhost:8000 (Healthcheck: http://localhost:8000/api/health)
- **PostgreSQL:** `localhost:5432` (banco `projeto_si`, usuário `projeto_si`, senha padrão `projeto_si_dev`)

### Variáveis de Ambiente Opcionais (Docker)

Você pode sobrescrever os valores padrão definindo variáveis no ambiente ou em um arquivo `.env` na raiz:

| Variável | Padrão |
|---|---|
| `POSTGRES_DB` | `projeto_si` |
| `POSTGRES_USER` | `projeto_si` |
| `POSTGRES_PASSWORD` | `projeto_si_dev` |
| `POSTGRES_PORT` | `5432` |
| `BACKEND_PORT` | `5000` |
| `ML_API_PORT` | `8000` |
| `FRONTEND_PORT` | `4200` |
| `JWT_SECRET_KEY` | `chave-apenas-para-desenvolvimento-nao-usar-em-producao-0123456789` |
| `DJANGO_SECRET_KEY` | `django-insecure-docker-secret-key` |
| `DJANGO_DEBUG` | `False` |
| `DEBUG_LOGS` | `true` (padrão no Compose; `false` em builds/configurações de produção) |
| `BIOMETRICS_DEBUG_LOGS` | `${DEBUG_LOGS:-true}` (sobrepõe `DEBUG_LOGS` na ML API e no `ml-scheduler`) |

### Comandos Úteis (Docker)

```bash
# Acompanhar logs (backend, ml-api e ml-scheduler em JSON estruturado no stdout; frontend via console do navegador)
docker compose logs -f

# Subir com debug logs desativados (modo produção)
DEBUG_LOGS=false ASPNETCORE_ENVIRONMENT=Production docker compose up --build

# Executar retreinamento manual pela ML API
docker compose exec ml-api python manage.py retrain_models

# Parar os containers mantendo os volumes (banco e modelos salvos)
docker compose down

# Parar e remover também os volumes de banco e modelos
docker compose down -v
```

---

## Executando sem Docker (Desenvolvimento Local)

### Pré-requisitos
- **PostgreSQL 17** (instalado e em execução localmente)
- **.NET SDK 10.0**
- **Python 3.12+** (recomendado `venv` ou `uv`)
- **Node.js 22+** e **npm**

### 1. Configurar o PostgreSQL compartilhado

Crie o usuário e o banco de dados correspondentes às configurações padrão de desenvolvimento (`projeto-si-backend/src/ProjetoSi.Api/appsettings.Development.json` e `projeto_si_ml_api/.env.example`):

```bash
psql -U postgres -c "CREATE USER projeto_si WITH PASSWORD 'projeto_si_dev';"
psql -U postgres -c "CREATE DATABASE projeto_si OWNER projeto_si;"
```

Opcionalmente, aplique o schema inicial manualmente (caso contrário, o Backend .NET aplica na inicialização em `Development`):

```bash
psql -U projeto_si -d projeto_si -f projeto-si-backend/schema.sql
```

### 2. Iniciar a Backend API (.NET 10)

Em um primeiro terminal:

```bash
cd projeto-si-backend
dotnet run --project src/ProjetoSi.Api
```

- A API ficará disponível em `http://localhost:5000`.
- Em ambiente `Development`, as migrações do EF Core são aplicadas automaticamente (`Banco:AplicarMigracoesNaInicializacao: true`) e o OpenAPI fica acessível em `http://localhost:5000/openapi/v1.json`.

### 3. Configurar e Iniciar a ML API (Django)

Em um segundo terminal:

```bash
cd projeto_si_ml_api

# Copie o arquivo de variáveis de ambiente
cp .env.example .env

# Crie o ambiente virtual e instale as dependências
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -e ".[dev]"
```

> **Importante:** Certifique-se de que as tabelas (`usuarios`, `modelos`, `amostras`) já foram criadas pelo `schema.sql` ou pela inicialização do Backend .NET antes do comando de migração do Django abaixo.

```bash
# Marca a migração inicial do Django sobre as tabelas já existentes
python manage.py migrate --fake-initial

# Inicia a ML API em http://localhost:8000
python manage.py runserver 0.0.0.0:8000
```

*(Opcional)* Para habilitar o retreinamento agendado em segundo plano localmente, abra outro terminal na pasta `projeto_si_ml_api`, ative o `.venv` e execute:

```bash
python manage.py run_scheduler
```

Ou dispare manualmente quando desejar:

```bash
python manage.py retrain_models
```

### 4. Iniciar o Frontend (Angular)

Em um quarto terminal:

```bash
cd projeto_si_frontend
npm install
npm start
```

- A aplicação estará disponível em `http://localhost:4200`.
- O `proxy.conf.json` redireciona automaticamente todas as requisições `/api` para `http://127.0.0.1:5000`.

---

## Testes Automatizados

### Backend (.NET)

```bash
cd projeto-si-backend
dotnet test
```

### ML API (Python / Django)

```bash
cd projeto_si_ml_api
.venv/bin/pytest
.venv/bin/ruff check .
```

### Frontend (Angular)

```bash
cd projeto_si_frontend
npm test
```
