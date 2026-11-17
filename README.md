# Projeto SI — Autenticação por Dinâmica de Digitação

Sistema completo de autenticação biométrica comportamental baseada em **dinâmica de digitação** (*Keystroke Dynamics*). A aplicação combina verificação tradicional de usuário/senha com classificação biométrica adaptativa individual por usuário.

---

## Arquitetura e Submódulos

O repositório principal orquestra os seguintes submódulos:

| Diretório | Stack | Responsabilidade |
|---|---|---|
| `projeto_si_frontend` | Angular 18 (Standalone + Signals) + Nginx | Interface web de cadastro, coleta de amostras (5 repetições), troca de senha e login biométrico. Captura local de `KEY_DOWN` / `KEY_UP` com `performance.now()` e extração das features `H`, `DD` e `UD` via `computeSample`. |
| `projeto-si-backend` | .NET 10 (Minimal API) + EF Core + PostgreSQL | API de entrada (`/api/*`), gestão de usuários, hashing de senha (PBKDF2-HMAC-SHA512), verificação de disponibilidade de username, validação de `features` e ordem esperada contra a senha, mascaramento posicional (`H.0, DD.0.1, UD.0.1, ...`, tamanho `3n - 2`), persistência seletiva em `amostras`, JWT e delegação biométrica para a ML API. |
| `projeto_si_ml_api` | Python 3.12 + Django 5.2 LTS + DRF + scikit-learn | Treinamento inicial, inferência biométrica com **`M2005_IDB`** (*Improved Double Parallel*, janela deslizante de 50 amostras + estatísticas incrementais de Welford), calibração de limiar por usuário, isolamento de modelos após troca de senha, sinalização `shouldRecordForRetraining` e retreinamento contínuo (`ml-scheduler`). |
| `projeto_si_experimentos` | Python + scikit-learn | Infraestrutura de experimentos e avaliação comparativa dos algoritmos biométricos (CMU e KeyRecs) que embasaram a escolha do `M2005_IDB`. |

### Fluxo de Dados e Banco Compartilhado

1. O **Frontend** calcula localmente as `features` (em ordem cronológica via `computeSample`) e envia ao **Backend .NET** (`/api`). Antes de iniciar a coleta, consulta `GET /api/username-availability`.
2. No cadastro, o **Backend .NET** recebe username, senha e as 5 amostras em `POST /api/register`, valida credenciais e features, persiste usuário + amostras e chama a **ML API** (`POST /api/enrollment`) para treinar o modelo inicial. Se o treinamento falhar, o cadastro é revertido.
3. No login, o **Backend .NET** valida a senha textual, confere se as chaves e a ordem das `features` correspondem exatamente à senha, converte/mascara para o vetor posicional (`double[]`) e delega a inferência à **ML API** (`POST /api/authenticate`).
4. A **ML API** cria/consulta `modelos` + artefatos `.joblib` (SHA-256), apenas lê `amostras` (sem criar linhas) e retorna `decision`, `confidence`, `threshold`, `inferenceTimeMs` e `shouldRecordForRetraining`. O **Backend .NET** usa essa flag para persistir a amostra de login no **PostgreSQL compartilhado** apenas quando ela é genuína e confiável.
5. Na troca de senha (`POST /api/change-password`), o Backend invalida a biometria anterior (`data_mudanca_senha`), coleta 5 amostras da nova senha, treina o novo modelo de forma síncrona e, em caso de falha, restaura senha/credenciais e remove os modelos criados após a troca.

---

## Algoritmo de Machine Learning Utilizado

O classificador de produção é o **`M2005_IDB`** — *Improved Double Parallel* (Pisani, Lorena & Carvalho, IJCNN 2015). A implementação fica em `projeto_si_ml_api/biometrics/ml/m2005.py` (classe `AdaptiveM2005Alg`) e é configurada por `BIOMETRICS_ALGORITHM` (padrão `M2005_IDB`).

### Como o `M2005_IDB` decide

- **Submodelo deslizante (`window_size = 50`):** mantém uma fila FIFO com as 50 amostras genuínas mais recentes e calcula média, mediana e desvio-padrão por dígrafo (`DigraphsMetrics`).
- **Submodelo crescente incremental (Welford):** acumula média e desvio-padrão em memória constante `O(1)` sobre todo o histórico genuíno desde o cadastro, evitando deriva e esquecimento catastrófico.
- **Escore final:** média aritmética dos escores dos dois submodelos. Cada atributo recebe *hit* quando cai no intervalo de tolerância construído a partir de média/mediana/desvio-padrão do dígrafo.
- **Limiar por usuário:** calibrado a partir dos scores da galeria genuína com `min` ou `min_non_outlier` (descarta scores abaixo de `média - 2.5 × desvio-padrão`), sem teto fixo. A recalibração ocorre a cada retreinamento (`dynamic_threshold=True`).
- **Treinamento One-Class:** utiliza exclusivamente amostras genuínas do próprio usuário (*anomaly detector*). Amostras de impostores não entram no retreinamento.

### Resultados que embasaram a escolha

A escolha foi feita a partir dos 52 experimentos do `projeto_si_experimentos` (consolidados localmente em `results/exp_metric_averages.csv` por `generate_exp_metric_averages.py`), priorizando BACC e equilíbrio entre FRR/FAR:

| Algoritmo | BACC CMU | FRR CMU | FAR CMU | BACC KeyRecs | FRR KeyRecs | FAR KeyRecs | BACC médio |
|---|---|---|---|---|---|---|---|
| **`M2005_IDB`** | **0.8909** | **15.24%** | **6.58%** | **0.7980** | **9.03%** | **31.37%** | **0.8445** |
| `M2005_DB` | 0.8907 | 14.64% | 7.22% | 0.7822 | 8.68% | 34.87% | 0.8365 |
| `SelfDet_Sliding` | 0.8655 | 23.41% | 3.49% | 0.8068 | 33.25% | 5.38% | 0.8362 |
| `M2005_Pisani2024_Sliding_Dynamic` | 0.8659 | 7.84% | 18.98% | 0.7219 | 5.28% | 50.33% | 0.7939 |

O `M2005_IDB` reduz a FAR do CMU em aproximadamente **3×** em relação ao `M2005_Pisani2024_Sliding_Dynamic` (`6.58%` vs `18.98%`) e melhora o BACC no KeyRecs em **+7,61 p.p.**, mantendo FRR controlada. Para compatibilidade e experimentos, a ML API também aceita `M2005_DB` (`variant="db"`) e o `M2005Alg` legado (`M2005_Pisani2024_Sliding_Dynamic`) via configuração.

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
- `backend`: API .NET 10 na porta interna `5000`.
- `ml-api`: API Django + Gunicorn na porta interna `8000` (executa `migrate --fake-initial` automaticamente).
- `ml-scheduler`: Worker APScheduler para retreinamento periódico dos modelos.
- `frontend`: Aplicação Angular servida via Nginx na porta `4200` (o Nginx faz proxy de `/api/` para `backend:5000`).

### URLs de Acesso (Docker)

- **Frontend:** http://localhost:4200
- **Backend API (.NET):** http://localhost:5001 (porta publicada pelo Compose; healthcheck em `/api/health`)
- **ML API (Django):** http://localhost:8000 (healthcheck em `/api/health`)
- **PostgreSQL:** `localhost:5432` (banco `projeto_si`, usuário `projeto_si`, senha padrão `projeto_si_dev`)

> O Backend escuta na porta `5000` dentro da rede Docker. O Compose publica essa porta no host como `5001` por padrão (`BACKEND_PORT`), enquanto o Nginx do frontend continua falando com `backend:5000` internamente.

### Variáveis de Ambiente Opcionais (Docker)

Você pode sobrescrever os valores padrão definindo variáveis no ambiente ou em um arquivo `.env` na raiz:

| Variável | Padrão |
|---|---|
| `POSTGRES_DB` | `projeto_si` |
| `POSTGRES_USER` | `projeto_si` |
| `POSTGRES_PASSWORD` | `projeto_si_dev` |
| `POSTGRES_PORT` | `5432` |
| `BACKEND_PORT` | `5001` |
| `ML_API_PORT` | `8000` |
| `FRONTEND_PORT` | `4200` |
| `ASPNETCORE_ENVIRONMENT` | `Development` |
| `JWT_SECRET_KEY` | `chave-apenas-para-desenvolvimento-nao-usar-em-producao-0123456789` |
| `DJANGO_SECRET_KEY` | `django-insecure-docker-secret-key` |
| `DJANGO_DEBUG` | `False` |
| `DJANGO_ALLOWED_HOSTS` | `localhost,127.0.0.1,api,ml-api` |
| `DEBUG_LOGS` | `true` (padrão no Compose; `false` em produção) |
| `BIOMETRICS_DEBUG_LOGS` | `${DEBUG_LOGS:-true}` (sobrepõe `DEBUG_LOGS` na ML API e no `ml-scheduler`) |
| `BIOMETRICS_ALGORITHM` | `M2005_IDB` |
| `BIOMETRICS_M2005_VARIANT` | `idb` |
| `BIOMETRICS_THRESHOLD_METHOD` | `min_non_outlier` |
| `BIOMETRICS_THRESHOLD_DYNAMIC` | `True` |

### Comandos Úteis (Docker)

```bash
# Acompanhar logs (backend, ml-api e ml-scheduler em JSON estruturado no stdout; frontend via console do navegador)
docker compose logs -f

# Subir com debug logs desativados (modo produção)
DEBUG_LOGS=false ASPNETCORE_ENVIRONMENT=Production docker compose up --build

# Executar retreinamento manual pela ML API
docker compose exec ml-api python manage.py retrain_models

# Retreinar um usuário específico, forçando mesmo sem novas amostras
docker compose exec ml-api python manage.py retrain_models --username thiago_dias --force

# Ativar/rollback de uma versão específica de modelo
docker compose exec ml-api python manage.py activate_model thiago_dias 1

# Parar os containers mantendo os volumes (banco e modelos salvos)
docker compose down

# Parar e remover também os volumes de banco e modelos
docker compose down -v
```

---

## Executando sem Docker (Desenvolvimento Local)

### Pré-requisitos

| Ferramenta | Versão mínima/recomendada |
|---|---|
| PostgreSQL | 17 |
| .NET SDK | 10.0 |
| Python | 3.12+ (recomendado `venv` ou `uv`) |
| Node.js | 22+ (Angular 18 também aceita `18.19+` e `20.11+`; o Docker usa `node:22-alpine`) |
| npm | 9+ |
| Google Chrome/Chromium | Necessário apenas para os testes Karma do frontend |

### 1. Configurar o PostgreSQL compartilhado

Crie o usuário e o banco correspondentes às configurações padrão de desenvolvimento (`projeto-si-backend/src/ProjetoSi.Api/appsettings.Development.json` e `projeto_si_ml_api/.env.example`):

```bash
psql -U postgres -c "CREATE USER projeto_si WITH PASSWORD 'projeto_si_dev';"
psql -U postgres -c "CREATE DATABASE projeto_si OWNER projeto_si;"
```

Opcionalmente, aplique o schema inicial manualmente (caso contrário, o Backend .NET aplica as migrações na inicialização em `Development`):

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
- A conexão padrão usa `localhost:5432`, banco `projeto_si`, usuário `projeto_si` e senha `projeto_si_dev`.

### 3. Configurar e Iniciar a ML API (Django)

Em um segundo terminal:

```bash
cd projeto_si_ml_api

# Copie o arquivo de variáveis de ambiente
cp .env.example .env
# No Windows PowerShell:
# Copy-Item .env.example .env

# Crie o ambiente virtual e instale as dependências
python3 -m venv .venv
source .venv/bin/activate
# No Windows PowerShell:
# .\.venv\Scripts\Activate.ps1

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
python manage.py retrain_models --username thiago_dias --force
python manage.py activate_model thiago_dias 1
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
- Se estiver usando o backend via Docker Compose (porta publicada `5001`), ajuste o `proxy.conf.json` para `http://127.0.0.1:5001` **ou** execute o backend localmente na porta `5000`. A forma mais simples de rodar tudo junto é usar o Docker Compose da raiz.

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
# Windows PowerShell:
# .\.venv\Scripts\pytest.exe
# .\.venv\Scripts\ruff.exe check .
```

### Frontend (Angular)

```bash
cd projeto_si_frontend
npm test -- --watch=false --browsers=ChromeHeadless
```

### Experimentos (Python)

```bash
cd projeto_si_experimentos
pytest -v
```

---

## Documentação Adicional

- [docs/fluxo-autenticacao.md](docs/fluxo-autenticacao.md): fluxo detalhado de cadastro, enrollment, login, troca de senha, retreinamento e rollback entre Frontend, Backend e ML API.
- [`projeto-si-backend/README.md`](projeto-si-backend/README.md): execução, configuração, endpoints e decisões de modelagem do Backend .NET.
- [`projeto_si_ml_api/requisitos.md`](projeto_si_ml_api/requisitos.md): especificação funcional e não funcional da ML API, incluindo a formulação do `M2005_IDB`.
- [`projeto_si_frontend/requisitos.md`](projeto_si_frontend/requisitos.md): requisitos funcionais do frontend, captura de features e contratos HTTP.
- [`projeto_si_experimentos/README.md`](projeto_si_experimentos/README.md): algoritmos avaliados, protocolos experimentais e comandos de reprodução dos benchmarks.
