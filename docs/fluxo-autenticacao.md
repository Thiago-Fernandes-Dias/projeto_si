# Fluxo de Cadastro, Enrollment, Login e Troca de Senha

Documentação detalhada da comunicação, persistência e validações entre **Frontend (`projeto_si_frontend`)**, **Backend .NET (`projeto-si-backend`)** e **ML API (`projeto_si_ml_api`)**.

O desenho atual usa **treinamento one-class** (somente amostras genuínas), cadastro unificado no backend, features mascaradas como vetor posicional e o classificador **`M2005_IDB`** na ML API. A ML API não cadastra usuários, não lê nem verifica senhas e não escreve novas linhas em `amostras`.

---

## Visão Geral dos Módulos

| Módulo | Responsabilidades no Fluxo | Acesso ao Banco (`projeto_si`) |
|---|---|---|
| `projeto_si_frontend` (Angular 18) | UI (`/register`, `/enrollment`, `/change-password`, `/login`, `/session`, `/blocked`), captura de teclas (`KEY_DOWN`/`KEY_UP`) com `performance.now()`, cálculo local de `H`/`DD`/`UD` (`computeSample`) e roteamento de sessão. | Nenhum (envia requisições `/api/*` para o Backend .NET). |
| `projeto-si-backend` (.NET 10) | Ponto de entrada `/api/*`, cadastro, troca de senha, hash PBKDF2, bloqueio por tentativas, validação das `features` contra a senha, mascaramento posicional, persistência em `usuarios`/`amostras`, JWT e chamadas à ML API. | Leitura/escrita em `usuarios` e `amostras`; leitura dos `modelos` retornados pela ML API. |
| `projeto_si_ml_api` (Django + DRF) | Treinamento inicial (`M2005_IDB` one-class), inferência biométrica, calibração de limiar, sinalização `shouldRecordForRetraining`, retreinamento incremental/periódico e rollback de modelos. Não lê nem verifica senhas e não expõe `/api/register`. | Leitura de `usuarios` e `amostras`; escrita apenas em `modelos` (+ `.joblib`/SHA-256 em disco). |

### Algoritmo de decisão

- **`M2005_IDB`** (`AdaptiveM2005Alg`, `variant="idb"`): média aritmética de um submodelo deslizante (janela de 50 amostras genuínas) e de um submodelo crescente incremental (Welford, memória `O(1)`).
- **Limiar por usuário:** calibrado com `min`/`min_non_outlier` sobre os scores da galeria genuína, limitado ao teto `0.7`, com recalibração dinâmica após retreinamento.
- **Resultado experimental:** CMU `BACC = 0.8909`, `FRR = 15.24%`, `FAR = 6.58%`; KeyRecs `BACC = 0.7980`, `FRR = 9.03%`, `FAR = 31.37%` (52 experimentos em `projeto_si_experimentos`).

---

## 1. Cadastro e Enrollment

### 1.1. Pré-checagem de username no frontend

- **Arquivo:** `projeto_si_frontend/src/app/features/register/register.component.ts` e `src/app/core/services/biometric-api.service.ts`.
- O `RegisterComponent` valida localmente:
  - username normalizado (`trim` + minúsculas) com 3–50 caracteres e apenas `a-z`, `0-9`, `.`, `_`, `@`, `+`, `-`;
  - senha com 8–64 caracteres, sem letras acentuadas nem teclas mortas (`´`, `` ` ``, `~`, `^`, `¨`);
  - confirmação igual à senha.
- Em seguida chama `GET /api/username-availability?username=...`.
  - `200 { username, available }`: se `available=true`, o usuário e a senha-alvo ficam em memória no `UserSessionService` e a navegação segue para `/enrollment`.
  - Se indisponível, o formulário permanece com mensagem de erro.
- A disponibilidade é apenas uma pré-checagem: a unicidade é garantida novamente por `POST /api/register` (`409` em corrida).

### 1.2. Coleta das 5 amostras no frontend

- **Arquivos:** `src/app/shared/components/keystroke-input/keystroke-input.component.ts`, `src/app/core/services/keystroke-capture.service.ts`, `src/app/features/enrollment/enrollment.component.ts`.
- A captura usa `performance.now()` e registra `{ key, code, timestamp, eventType: 'KEY_DOWN' | 'KEY_UP' }`.
- O texto é registrado apenas no `KEY_DOWN`. O par `KEY_DOWN`/`KEY_UP` é associado pela tecla física (`code`), não pelo caractere, para que soltar `Shift` antes da letra não invalide a amostra.
- São recusados/reiniciados: `Backspace`, `Delete`, setas/Home/End, colagem, teclas mortas (`Dead`), `keydown` repetido e `KEY_UP` órfão.
- `computeSample(repetition, targetPassword)`:
  - exige texto não vazio, igual à senha-alvo e exatamente `2 × quantidade de caracteres` eventos;
  - valida a alternância `KEY_DOWN → KEY_UP → KEY_DOWN` (rejeita sobreposição);
  - calcula `H` (hold), `DD` (down-down) e `UD` (up-down) em segundos com 4 casas;
  - gera chaves com sufixo sequencial anti-colisão (`H.t.0`, `DD.t.e.0`, `UD.t.e.0`, ...);
  - recusa localmente `H`/`DD` fora de `(0, 5]` s;
  - retorna `{ repetition, features, rawTimings, textEntered, totalDurationMs }`.
- A tela exige exatamente 5 repetições válidas. Ao concluir a 5ª, bloqueia o campo e envia o lote assim que possível; depois de sucesso, o usuário clica para ir ao login.

### 1.3. Cadastro unificado no backend

- **Arquivo:** `projeto-si-backend/src/ProjetoSi.Api/Features/Cadastro/CadastrarUsuario.cs`.
- Request: `{ username, password, samples[5] }`.
- Validações:
  - username normalizado e regex `^[a-z0-9._@+-]{3,50}$` (`400` se inválido);
  - senha de 8 a 64 caracteres (`400`);
  - exatamente `Biometria:AmostrasCadastro = 5` amostras (`400`);
  - todas as amostras com o mesmo `textEntered` e exatamente igual a `password` (`400`);
  - `totalDurationMs` finito e `>= 0` (`400`);
  - `VetorCaracteristicas.TentarExtrair(features, senha)` valida chaves, ordem cronológica e limites (`400` se falhar):
    - `H`/`DD` ∈ `(0, 5]` s;
    - `UD` ∈ `[-5, 5]` s;
    - quantidade de features `3n - 2`.
- Persistência:
  - hash da senha via `PasswordHasher<Usuario>` (PBKDF2-HMAC-SHA512, salt por usuário);
  - `usuarios` com `data_mudanca_senha = agora` e `biometria_cadastrada_em = agora`;
  - `amostras` com `origem = Cadastro`, `vetor = double[]` posicional mascado (`H.0, DD.0.1, UD.0.1, ...`), `repeticao`, `duracao_total_ms` e `coletada_em`;
  - nunca grava caracteres da senha, `textEntered` nem `rawTimings`.
- Depois de salvar, chama a ML API:

```json
POST /api/enrollment
{
  "username": "thiago_dias",
  "userId": "0199f3a0-6f2b-7c3e-8a41-123456789abc",
  "vectors": [[...], [...], [...], [...], [...]]
}
```

- Se o treinamento falhar, o backend remove amostras e usuário (`ReverterCadastroAsync`) e propaga o erro; nenhum cadastro parcial permanece.
- Sucesso: `201 Created` com `{ success, message, samplesRegistered: 5, calculatedAverageHoldMs }`.

### 1.4. Enrollment na ML API

- **Arquivos:** `projeto_si_ml_api/biometrics/serializers.py`, `views.py`, `services.py`, `ml/features.py`, `ml/training.py`, `ml/registry.py`.
- `EnrollmentRequestSerializer` exige `username` **ou** `userId` e `vectors` não vazio.
- `enroll_user`:
  - busca o usuário existente (senão `404`);
  - rejeita enrollment duplicado para a senha vigente (`criado_em >= data_mudanca_senha`, `409`);
  - exige exatamente 5 vetores (`ENROLLMENT_SAMPLES`);
  - valida cada vetor posicional: `len >= 4`, `(len + 2) % 3 == 0`, finitos, limites por slot e mesma dimensão;
  - treina **apenas com os vetores genuínos** o `AdaptiveM2005Alg` configurado (`M2005_IDB` por padrão, `window_size=50`, `threshold=0.7`, `threshold_method=min_non_outlier`, `dynamic_threshold=True`);
  - persiste `.joblib` com SHA-256, `.meta.json`, `active_version.json` e o registro `v1` em `modelos`;
  - **não** escreve em `amostras` e **não** altera `usuarios`.
- Resposta `201`: `{ userId, username, modelId, modelVersion, featureCount, samplesRegistered, calculatedAverageHoldMs, success, message }`.

---

## 2. Troca de Senha

### 2.1. Frontend

- **Rota:** `/change-password` (`ChangePasswordComponent`).
- Campos: usuário, senha atual, nova senha e confirmação. Validações: usuário no padrão do cadastro, nova senha 8–64 sem acentos, confirmação igual e nova senha diferente da atual.
- Após a validação, o `UserSessionService.startPasswordChange(...)` guarda usuário, senha atual e nova senha-alvo em memória e navega para `/enrollment`.
- A coleta captura **5 repetições da nova senha** e envia `POST /api/change-password` em vez de `POST /api/register`.

### 2.2. Backend

- **Arquivo:** `projeto-si-backend/src/ProjetoSi.Api/Features/Senha/AlterarSenha.cs`.
- Request: `{ username, currentPassword, newPassword, samples[5] }`.
- Validações:
  - `currentPassword` obrigatória (`400`);
  - `newPassword` e `samples` com as mesmas regras do cadastro (`400`);
  - `newPassword !== currentPassword` (`400`);
  - conta bloqueada ou senha atual incorreta → `401` (contabilizando falha);
  - usuário inexistente ou senha atual inválida → `401` sem revelar qual campo falhou.
- Transação:
  1. salva o hash anterior, `data_mudanca_senha` e `biometria_cadastrada_em` para compensação;
  2. `AlterarSenha(novoHash, agora)` atualiza hash, `data_mudanca_senha` e zera `biometria_cadastrada_em`;
  3. grava as 5 amostras da nova senha (`origem = Cadastro`);
  4. chama a ML API (`POST /api/enrollment`) para treinar o novo modelo;
  5. em sucesso, conclui a biometria e retorna `200` com `{ success, message, samplesRegistered: 5, calculatedAverageHoldMs }`.
- Em falha do treinamento:
  1. chama `POST /api/models/rollback` com `createdAfter = nova data de mudança`;
  2. remove as amostras novas;
  3. restaura hash, `data_mudanca_senha` e `biometria_cadastrada_em` anteriores;
  4. retorna `502` informando que a senha anterior foi restaurada.
- Concorrência: `DbUpdateConcurrencyException` retorna `409`.
- JWT: a claim `pwd` é comparada com `data_mudanca_senha` em `GET /api/session`; tokens emitidos antes da troca passam a ser recusados (`401`).

### 2.3. ML API e isolamento de modelos

- **Arquivo:** `projeto_si_ml_api/biometrics/services.py`.
- `get_active_model` considera apenas versões com `criado_em >= usuarios.data_mudanca_senha`; não há fallback para modelos antigos.
- `activate_model_version` rejeita versões anteriores à troca, e `retrain_user` não reutiliza a galeria de um modelo pré-troca.
- `POST /api/models/rollback` remove modelos criados após o timestamp informado, restaura `active_version.json` para a versão anterior e limpa o cache de modelos.

---

## 3. Login e Retreinamento

### 3.1. Frontend

- **Arquivo:** `src/app/features/login/login.component.ts`.
- O usuário informa username e digita a senha; `computeSample(1)` gera a amostra da tentativa (sem exigir senha-alvo).
- `POST /api/authenticate` com `{ username, passwordAttempt, sample }`.
- Decisões:
  - `GENUINE`: salva a sessão (incluindo `accessToken`) em `localStorage` e navega para `/session`;
  - `IMPOSTOR`: guarda a tentativa em memória e navega para `/blocked`;
  - `INVALID_CREDENTIALS`: exibe a mensagem da API e permanece no login.
- `authGuard` valida `GET /api/session` com `Authorization: Bearer <token>` antes de abrir `/session`; token ausente/expirado ou emitido antes de uma troca de senha encerra a sessão local e redireciona para `/login`.

### 3.2. Backend

- **Arquivo:** `projeto-si-backend/src/ProjetoSi.Api/Features/Login/Autenticar.cs`.
- Validações:
  - `username` e `passwordAttempt` obrigatórios (`400`);
  - `sample` presente com `totalDurationMs` finito e `>= 0` (`400`);
  - se `sample.textEntered` estiver preenchido, precisa ser igual a `passwordAttempt` (`400`);
  - features exatas para a senha, na ordem correta e com limites de tempo (`400`).
- Segurança:
  - conta bloqueada (`5` falhas / `15` min) → `200` com `INVALID_CREDENTIALS`;
  - senha textual inválida → contabiliza falha e retorna `200` com `INVALID_CREDENTIALS`;
  - usuário sem biometria → `200` com `INVALID_CREDENTIALS`;
  - concorrência → `409`.
- Chamada à ML API e persistência:
  - `POST /api/authenticate` com `{ username, userId, vector }`;
  - se `shouldRecordForRetraining == true`, persiste a amostra de login (`origem = Login`, vetor posicional, decisão, confiança, modelo);
  - `GENUINE` → zera falhas, gera JWT e retorna `accessToken`/`accessTokenExpiresAt`;
  - `IMPOSTOR` → contabiliza falha e retorna sem token.
- Resposta `200`: `decision`, `confidenceScore`, `thresholdScore`, `inferenceTimeMs`, `message`, `userAverageHoldTimeMs`, `attemptAverageHoldTimeMs`, `accessToken`, `accessTokenExpiresAt`.

### 3.3. ML API

- **Arquivos:** `biometrics/services.py`, `ml/inference.py`, `ml/registry.py`, `ml/features.py`.
- `AuthenticateRequestSerializer` exige `username` ou `userId` e `vector`.
- `authenticate_user`:
  - busca o usuário (`404`) e o modelo ativo pós-troca de senha (`409`);
  - carrega o `.joblib` validando SHA-256 (erro de integridade → `500`);
  - valida o vetor posicional `3n - 2` e a dimensão esperada (`400`);
  - classifica com `M2005_IDB`, usando o limiar calibrado do modelo;
  - calcula `label_for(confidence, threshold)` com margem `0.15`;
  - retorna `shouldRecordForRetraining = (decision == GENUINE and label == GENUINE)`;
  - **não grava nada em `amostras`**.
- Rate limit: `ScopedRateThrottle` (`120/minute` por padrão; configurável por `BIOMETRICS_AUTH_THROTTLE_RATE`).

### 3.4. Retreinamento

- **Arquivos:** `biometrics/management/commands/run_scheduler.py`, `retrain_models.py`, `biometrics/services.py`.
- O scheduler executa a cada `RETRAIN_INTERVAL_MINUTES` (`60` por padrão); também é possível rodar manualmente:
  ```bash
  python manage.py retrain_models
  python manage.py retrain_models --username thiago_dias --force
  ```
- `retrain_user`:
  - adquire `select_for_update(skip_locked=True)`;
  - exige, sem `--force`, pelo menos `RETRAIN_MIN_NEW_SAMPLES = 5` amostras `Login` `Genuine` com `confianca >= threshold + 0.15` posteriores ao modelo ativo;
  - usa apenas `Cadastro` e `Login` `Genuine` de alta confiança; ignora `Impostor` e `UNCERTAIN`;
  - se o modelo ativo é `AdaptiveM2005Alg` da mesma variante, faz **adaptação incremental** (novas amostras + Welford + janela deslizante);
  - caso contrário, reconstrói o modelo a partir da janela de até `RETRAIN_GENUINE_WINDOW = 50` amostras genuínas;
  - persiste a próxima versão em `modelos` + `.joblib`/SHA-256 + `active_version.json`.
- A ML API apenas **lê** `amostras`; a gravação de novas linhas no login é exclusiva do Backend .NET.

---

## 4. Persistência e Integridade dos Modelos

- Diretório: `models_data/<user_id>/v<version>.joblib`; cada versão tem `.meta.json` e hash SHA-256 registrado em `modelos`.
- A versão ativa é a maior `versao` com `criado_em >= data_mudanca_senha`, respeitando `active_version.json`.
- `activate_model <username> <version>` permite rollback/ativação manual:
  ```bash
  python manage.py activate_model thiago_dias 1
  ```
- O endpoint interno `POST /api/models/rollback` existe para compensar falha na troca de senha; ele assume a rede Docker privada (ver "Melhorias Futuras" em `projeto_si_ml_api/requisitos.md`).
- A gravação do arquivo é atômica e a falha de banco dispara compensação dos arquivos, evitando modelos órfãos.

---

## 5. Pontos de Atenção no Contrato Entre Módulos

- **`rawTimings`:** o frontend inclui `rawTimings` em `KeystrokeSample`, mas o backend o ignora (`AmostraDigitacaoRequest` usa apenas `Repetition`, `Features`, `TextEntered` e `TotalDurationMs`). Nada de `rawTimings` é persistido; a ML API recebe somente vetores posicionais.
- **Sobreposição de teclas (`UD < 0`):** o frontend exige `KEY_DOWN → KEY_UP → KEY_DOWN` e recusa sobreposição; o backend e a ML API aceitam matematicamente `UD ∈ [-5, 5]` s. A restrição mais forte é a do frontend.
- **Senha:** o frontend e o backend aplicam as mesmas regras (8–64 caracteres, sem acentos); a mensagem de erro do backend é exibida na UI.
- **Username:** a pré-checagem é apenas informativa; a unicidade real é garantida por `POST /api/register` (`409`).
- **Credenciais inválidas no login:** o backend retorna `200` com `decision: "INVALID_CREDENTIALS"` em vez de `401`, para não revelar se o usuário existe.
- **JWT:** só é emitido em decisão `GENUINE`; após troca de senha, tokens antigos são recusados por `GET /api/session`.
- **ML API sem cadastro:** `POST /api/register` na ML API responde `404`; o cadastro é exclusividade do backend.
- **Portas:** no Docker da raiz, o backend é publicado em `http://localhost:5001` (interno `backend:5000`), a ML API em `8000` e o frontend em `4200`. No desenvolvimento local do backend, a API escuta em `5000` (alvo padrão do `proxy.conf.json`).
