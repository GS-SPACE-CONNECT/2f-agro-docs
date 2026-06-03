# Arquitetura de Segurança — 2F-AGRO

> **Matéria:** Cybersecurity (3 pts — Arquitetura de Segurança)
> **Projeto:** 2F-AGRO · Global Solution · FIAP 3ES · 2026.1
> **Issue:** [#11](https://github.com/GS-SPACE-CONNECT/2f-agro-docs/issues/11)
> **Autor:** @jota0802
> **Última atualização:** junho 2026

---

## Sumário

| # | Seção | Pts |
|---|---|---|
| 1 | [Controles de Acesso](#1-controles-de-acesso) | 1 |
| 2 | [Proteção de Dados](#2-proteção-de-dados) | 1 |
| 3 | [Segurança da Infraestrutura](#3-segurança-da-infraestrutura) | 1 |
| — | [Referências](#referências) | — |

---

## Contexto

O 2F-AGRO é uma plataforma de tecnologia espacial acessível para o pequeno agricultor familiar brasileiro. A superfície de ataque abrange:

- **App mobile** (React Native / Expo) — celulares Android básicos, zonas rurais com conectividade instável
- **API Gateway + microsserviços** (C# .NET 8) — autenticação, alertas, notificações, ingestão de dados
- **Serviço SOA** (Java / Spring Boot) — integração com NASA POWER e CPTEC-INPE via REST e SOAP
- **Serviço ML** (Python / FastAPI / scikit-learn) — inferência de risco agronômico
- **Estação edge IoT** (Raspberry Pi 4 + Raspbian) — coleta de sensores em fazenda remota, solar, sem internet fixa
- **Banco de dados** (PostgreSQL + PostGIS) — dados de propriedades, geolocalização, produção, alertas

Os controles abaixo protegem **três perfis de usuário** com necessidades distintas: agricultor familiar (baixa literacia digital), cooperativa (gestão regional) e órgão público (EMATER/MAPA).

```
┌────────────────────────────────────────────────────────────────────┐
│                        PERÍMETRO DE SEGURANÇA                      │
│                                                                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│  │ 🧑‍🌾 App   │    │ 👥 Web   │    │ 📡 Edge  │   ← canais de      │
│  │ Mobile   │    │ Dashboard│    │ Estação  │     entrada          │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘                     │
│       │               │               │                            │
│  ─────┼───────────────┼───────────────┼──── TLS 1.3 ──────────    │
│       │               │               │                            │
│  ┌────▼───────────────▼───────────────▼────┐                      │
│  │         WAF + Rate Limiter              │                      │
│  └────────────────┬────────────────────────┘                      │
│                   │                                                │
│  ┌────────────────▼────────────────────────┐                      │
│  │   API GATEWAY (C# .NET 8)               │                      │
│  │   • JWT validation + RBAC enforcement   │                      │
│  │   • Request logging (coords truncadas)  │                      │
│  └────────────────┬────────────────────────┘                      │
│                   │                                                │
│  ┌────────┬───────┼───────┬───────────┐                           │
│  │        │       │       │           │                            │
│  ▼        ▼       ▼       ▼           ▼                            │
│ Auth    Alertas  ML    Ingestão    SOA (Java)                     │
│ (C#)   (C#)   (Python) (C#)     (Spring Boot)                   │
│                   │                                                │
│  ┌────────────────▼────────────────────────┐                      │
│  │   PostgreSQL + PostGIS                  │                      │
│  │   AES-256 em repouso (pgcrypto + LUKS)  │                      │
│  │   Backups criptografados                │                      │
│  └─────────────────────────────────────────┘                      │
│                                                                    │
│  Secrets em Vault │ SIEM (Grafana+Loki) │ Zero Trust              │
└────────────────────────────────────────────────────────────────────┘
```

---

## 1. Controles de Acesso

### 1.1 Autenticação Multifator (MFA)

| Aspecto | Detalhe |
|---|---|
| **Quem exige MFA** | Cooperativas e administradores (perfis com acesso a dados agregados de múltiplas propriedades) |
| **Agricultores** | MFA **opcional** — o agricultor familiar pode não ter segundo dispositivo ou e-mail; forçar MFA seria barreira de acesso |
| **Método primário** | TOTP (Time-based One-Time Password) — compatível com Google Authenticator, Authy |
| **Método secundário** | SMS OTP como fallback para regiões sem internet estável (degradação planejada) |
| **Implementação** | ASP.NET Identity (`UserManager.SetTwoFactorEnabledAsync`) integrado ao Serviço Auth |

**Fluxo de login com MFA:**

```
Usuário                    API Gateway              Serviço Auth
  │                            │                        │
  ├─── POST /auth/login ──────►│                        │
  │    {email, senha}          ├── valida credencial ──►│
  │                            │◄── 200 + mfa_required ─┤
  │◄── 200 {mfa_required} ────┤                        │
  │                            │                        │
  ├─── POST /auth/mfa ────────►│                        │
  │    {token_temporário, otp} ├── valida TOTP ────────►│
  │                            │◄── JWT assinado ───────┤
  │◄── 200 {access_token, ────┤                        │
  │         refresh_token}     │                        │
```

### 1.2 JWT com Claims

Os tokens JWT carregam claims que identificam o papel e o escopo do usuário, permitindo decisões de autorização sem consulta ao banco a cada request.

**Estrutura do payload JWT:**

```json
{
  "sub": "a1b2c3d4-uuid-do-usuário",
  "name": "João Silva",
  "role": "agricultor",
  "propriedade_id": "f5e6d7c8-uuid",
  "cooperativa_id": null,
  "iss": "2f-agro-auth",
  "aud": "2f-agro-api",
  "iat": 1717372800,
  "exp": 1717376400,
  "jti": "token-único-uuid"
}
```

| Claim | Descrição | Valores possíveis |
|---|---|---|
| `role` | Papel principal do usuário | `agricultor`, `cooperativa`, `emater`, `admin` |
| `propriedade_id` | UUID da propriedade vinculada (agricultor) | UUID ou `null` |
| `cooperativa_id` | UUID da cooperativa (gestores) | UUID ou `null` |
| `iss` | Emissor do token | `2f-agro-auth` |
| `aud` | Audiência destinatária | `2f-agro-api` |
| `jti` | ID único do token (para revogação) | UUID |

**Configuração de segurança do token:**

- **Algoritmo de assinatura:** RS256 (RSA 2048 bits) — chave privada no Vault, chave pública distribuída aos microsserviços
- **Tempo de vida (access_token):** 1 hora
- **Tempo de vida (refresh_token):** 7 dias, rotação a cada uso (um refresh_token usado é invalidado imediatamente)
- **Revogação:** lista de `jti` revogados em cache Redis com TTL igual ao `exp` do token

### 1.3 RBAC (Role-Based Access Control)

O acesso é restritivo por padrão — cada role tem apenas as permissões mínimas necessárias.

**Matriz de permissões:**

| Recurso | Agricultor | Cooperativa | EMATER | Admin |
|---|---|---|---|---|
| `GET /propriedades/{id}` | Apenas a própria | Apenas da sua cooperativa | Região atribuída | Todas |
| `POST /propriedades` | Criar a própria | — | — | Sim |
| `PUT /propriedades/{id}` | Apenas a própria | — | — | Sim |
| `DELETE /propriedades/{id}` | — | — | — | Sim |
| `GET /alertas` | Seus alertas | Alertas da cooperativa | Alertas da região | Todos |
| `POST /diagnosticos` | Sim (sua lavoura) | — | — | Sim |
| `GET /lavouras` | Suas lavouras | Lavouras da cooperativa | Lavouras da região | Todas |
| `GET /cooperativa/mapa` | — | Seu mapa regional | Mapa da região | Todos |
| `GET /admin/*` | — | — | — | Sim |
| `DELETE /me` | Sim | Sim | Sim | — |

**Implementação no API Gateway (C# .NET 8):**

```csharp
// Middleware de autorização por policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ApenasProprietario", policy =>
        policy.RequireRole("agricultor", "admin"));

    options.AddPolicy("CooperativaOuSuperior", policy =>
        policy.RequireRole("cooperativa", "emater", "admin"));

    options.AddPolicy("ApenasAdmin", policy =>
        policy.RequireRole("admin"));
});

// Exemplo de endpoint protegido
[Authorize(Policy = "ApenasProprietario")]
[HttpGet("propriedades/{id}")]
public async Task<IActionResult> ObterPropriedade(Guid id)
{
    var usuarioId = User.FindFirst("sub")?.Value;
    var propriedadeId = User.FindFirst("propriedade_id")?.Value;

    // Privilégio mínimo: agricultor só acessa a própria propriedade
    if (User.IsInRole("agricultor") && propriedadeId != id.ToString())
        return Forbid();

    // ...
}
```

### 1.4 Princípio do Privilégio Mínimo

O privilégio mínimo é aplicado em todas as camadas:

| Camada | Aplicação |
|---|---|
| **Banco de dados** | Cada microsserviço tem seu próprio usuário PostgreSQL com `GRANT` restrito às tabelas que opera (ex.: serviço ML tem `SELECT` apenas em `telemetria` e `alertas`, sem `DELETE` ou `UPDATE`) |
| **Microsserviços** | Comunicação interna via tokens de serviço com escopo limitado (service-to-service JWT com `scope: ["alertas:read", "telemetria:write"]`) |
| **App mobile** | Token do agricultor não carrega claim `cooperativa_id`; token da cooperativa não inclui claim `admin` |
| **Estação edge** | Certificado mTLS com permissão restrita a `POST /ingestao/telemetria` — não pode ler nem alterar dados de outros endpoints |
| **CI/CD** | GitHub Actions usa tokens `GITHUB_TOKEN` com `permissions: contents: read` por padrão; só jobs de deploy recebem escopo `write` |

---

## 2. Proteção de Dados

### 2.1 TLS 1.3 em Trânsito

Toda comunicação entre componentes do 2F-AGRO é criptografada com TLS 1.3.

| Trecho | Protocolo | Detalhe |
|---|---|---|
| App mobile ↔ API Gateway | TLS 1.3 | HTTPS obrigatório; HSTS habilitado; cipher suites: `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` |
| Estação edge ↔ API Gateway | mTLS 1.3 | Certificado de cliente embarcado no Raspberry Pi (provisionado no primeiro boot) + certificado do servidor |
| Microsserviço ↔ Microsserviço | TLS 1.3 | Comunicação interna via HTTPS; em ambiente Kubernetes, service mesh (Istio) gerencia mTLS automaticamente |
| API Gateway ↔ PostgreSQL | TLS 1.3 | `sslmode=verify-full` na connection string; certificado do servidor validado |
| Serviço SOA ↔ APIs externas (NASA/CPTEC) | TLS 1.2+ | Dependente do servidor externo; mínimo TLS 1.2 exigido na configuração do `RestTemplate` |

**Configuração no API Gateway (C# .NET 8):**

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(https =>
    {
        https.SslProtocols = SslProtocols.Tls13;
    });
});

// HSTS — impedir downgrade pra HTTP
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});
```

**Por que TLS 1.3 (e não 1.2):**

- **Handshake mais rápido** (1-RTT vs. 2-RTT) — relevante para zonas rurais com alta latência
- **Forward secrecy obrigatório** — chaves efêmeras em todo handshake; comprometer a chave do servidor não decifra tráfego passado
- **Cipher suites simplificados** — removidos algoritmos legados vulneráveis (RC4, CBC, SHA-1)

### 2.2 AES-256 em Repouso

| Dado | Mecanismo | Detalhe |
|---|---|---|
| **Banco PostgreSQL** | Criptografia de coluna via `pgcrypto` (campos sensíveis) + criptografia full-disk (LUKS/dm-crypt) | Dados em disco ilegíveis sem a chave mestra; chave mestra armazenada no Vault |
| **Backups** | AES-256-GCM antes do upload para S3 | `pg_dump` → criptografia com chave rotacionada mensalmente → upload; chave no Vault |
| **Modelos ML (.pkl)** | AES-256 no armazenamento (S3 server-side encryption) | Protege propriedade intelectual do modelo treinado contra exfiltração |
| **Logs arquivados** | AES-256 em repouso no bucket de long-term storage | Compliance — logs de auditoria intactos por 12 meses |
| **Dados locais no app** | Expo SecureStore (Keychain no iOS, Keystore no Android) | Tokens JWT e dados sensíveis do agricultor protegidos no device |

**Algoritmo padrão:** AES-256-GCM (Galois/Counter Mode) — oferece confidencialidade **e** integridade (authentication tag) em uma única operação.

### 2.3 bcrypt em Senhas

Senhas nunca são armazenadas em texto plano ou com hashing fraco. O 2F-AGRO utiliza **bcrypt** com custo adaptativo.

| Parâmetro | Valor | Justificativa |
|---|---|---|
| **Algoritmo** | bcrypt | Resistente a ataques de GPU/ASIC por design (CPU-intensivo e resistente a paralelismo em GPU); amplamente auditado |
| **Work factor (custo)** | 12 (4.096 iterações) | Equilíbrio entre segurança e tempo de resposta aceitável (~250ms no servidor) |
| **Salt** | 128 bits, gerado automaticamente pelo bcrypt | Impede ataques de rainbow table |
| **Migração futura** | Argon2id preparado como fallback | ASP.NET Identity suporta troca transparente de hasher; migração sob demanda no próximo login |

**Implementação no Serviço Auth (BCrypt.Net-Next + ASP.NET Identity):**

```csharp
// Hasher customizado substituindo o PBKDF2 padrão do Identity por bcrypt
public class BcryptPasswordHasher<TUser> : IPasswordHasher<TUser>
    where TUser : class
{
    private const int WorkFactor = 12;

    public string HashPassword(TUser user, string password)
        => BCrypt.Net.BCrypt.HashPassword(password, workFactor: WorkFactor);

    public PasswordVerificationResult VerifyHashedPassword(
        TUser user, string hashedPassword, string providedPassword)
        => BCrypt.Net.BCrypt.Verify(providedPassword, hashedPassword)
            ? PasswordVerificationResult.Success
            : PasswordVerificationResult.Failed;
}

// Registro no Program.cs
builder.Services.AddScoped<IPasswordHasher<ApplicationUser>,
    BcryptPasswordHasher<ApplicationUser>>();
```

**Política de senhas:**

- Mínimo 8 caracteres (simplicidade para o público rural)
- Verificação contra lista de senhas vazadas (HaveIBeenPwned API, offline via k-anonymity)
- Bloqueio temporário (15 min) após 5 tentativas falhas consecutivas
- Sem exigência de caracteres especiais (pesquisas mostram que comprimento supera complexidade)

### 2.4 Anonimização de Coordenadas em Logs

A geolocalização das propriedades é dado sensível sob a LGPD (pode identificar o agricultor e expor sua propriedade a riscos físicos). Em logs operacionais, as coordenadas são **truncadas a 2 casas decimais**, resultando em precisão de ~1,1 km — suficiente para diagnóstico técnico, insuficiente para localização exata.

**Antes (dado real):**

```
latitude: -8.047562, longitude: -34.876543
```

**Depois (em log — truncado em direção ao zero):**

```
latitude: -8.04, longitude: -34.87
```

**Implementação no middleware de logging (C# .NET 8):**

```csharp
public class AnonimizadorCoordenadas
{
    /// <summary>
    /// Trunca coordenada para 2 casas decimais (~1 km de imprecisão).
    /// Usado exclusivamente em logs — dados reais preservados no banco.
    /// </summary>
    public static double Truncar(double coordenada)
    {
        return Math.Round(coordenada, 2, MidpointRounding.ToZero);
    }
}

// No middleware de request logging (simplificado)
public class RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        // Extrai coordenadas do body, se presentes (ex.: POST /propriedades)
        var (latitude, longitude) = await ExtrairCoordenadasDoBody(context.Request);

        var logEntry = new
        {
            Timestamp = DateTime.UtcNow,
            Path = context.Request.Path,
            UserId = context.User?.FindFirst("sub")?.Value,
            // Coordenadas anonimizadas — nunca logamos precisão total
            Latitude = latitude.HasValue
                ? AnonimizadorCoordenadas.Truncar(latitude.Value) : (double?)null,
            Longitude = longitude.HasValue
                ? AnonimizadorCoordenadas.Truncar(longitude.Value) : (double?)null
        };

        logger.LogInformation("Request: {@LogEntry}", logEntry);
        await next(context);
    }

    // Implementação real parseia o body JSON e busca campos lat/lng
    private static Task<(double?, double?)> ExtrairCoordenadasDoBody(
        HttpRequest request) => /* ... */ Task.FromResult<(double?, double?)>((null, null));
}
```

**Camadas de proteção de coordenadas:**

| Camada | Precisão | Quem vê |
|---|---|---|
| Banco de dados (PostGIS) | Total (~10 cm) | Microsserviço Alertas (para cálculos agronômicos) |
| API de resposta ao agricultor | Total (sua própria propriedade) | Apenas o dono |
| API de resposta à cooperativa | Truncada a 3 casas (~100 m) | Gestores da cooperativa (visão de mapa regional) |
| Logs operacionais | Truncada a 2 casas (~1 km) | Equipe de operações |
| Logs exportados para SIEM | Truncada a 2 casas (~1 km) | Equipe de segurança |
| Dados analíticos / relatórios | Agregados por município | Público / EMATER |

---

## 3. Segurança da Infraestrutura

### 3.1 Arquitetura Zero Trust

O 2F-AGRO adota o princípio de **"nunca confiar, sempre verificar"** — mesmo dentro da rede interna, cada request é autenticado e autorizado.

**Pilares do Zero Trust aplicados:**

| Pilar | Implementação no 2F-AGRO |
|---|---|
| **Verificar explicitamente** | Todo request ao API Gateway valida o JWT (assinatura, expiração, claims). Microsserviços internos validam tokens de serviço (service-to-service JWT) |
| **Menor privilégio** | RBAC com escopo mínimo (seção 1.3–1.4); tokens de curta duração (1h) |
| **Assumir violação** | Segmentação de rede por microsserviço; SIEM monitora anomalias; resposta automatizada (rate limit, bloqueio de IP) |

**Microsserviços isolados — sem confiança implícita:**

```
┌────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                         │
│  ┌─────────────┐    NetworkPolicy    ┌──────────────┐  │
│  │  Serviço    │◄═══════════════════►│  Serviço     │  │
│  │  Alertas    │  JWT serviço +      │  ML          │  │
│  │  (C#)       │  mTLS obrigatório   │  (Python)    │  │
│  └──────┬──────┘                     └──────┬───────┘  │
│         │ NetworkPolicy: só porta 5432       │          │
│         ▼                                    ▼          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  PostgreSQL (acesso restrito por IP + user)      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Kubernetes NetworkPolicy bloqueia TODO tráfego por     │
│  padrão; apenas rotas explícitas são permitidas.        │
└────────────────────────────────────────────────────────┘
```

**Verificação em cada camada:**

1. **Edge → Cloud:** certificado mTLS da estação IoT é validado antes de aceitar telemetria
2. **Mobile → Gateway:** JWT validado em todo request (sem sessão stateful)
3. **Gateway → Microsserviço:** token de serviço propagado com escopo reduzido
4. **Microsserviço → Banco:** usuário PostgreSQL específico por serviço, com `GRANT` mínimo

### 3.2 WAF (Web Application Firewall)

O WAF é a primeira linha de defesa, posicionado entre a internet e o API Gateway.

| Aspecto | Configuração |
|---|---|
| **Provedor** | Cloudflare WAF (produção) / Azure WAF (alternativa) |
| **Regras OWASP** | Core Rule Set (CRS) habilitado — proteção contra SQLi, XSS, LFI, RFI, command injection |
| **Regras customizadas** | Bloqueio de payloads com padrões de envenenamento de telemetria (ex.: valores de temperatura fora de -50°C a 70°C) |
| **Geo-blocking** | Tráfego permitido apenas do Brasil (requisito de negócio — plataforma nacional) |
| **Bot protection** | Challenge (Turnstile) em endpoints públicos (`/auth/login`, `/auth/register`); APIs autenticadas não exigem challenge |
| **DDoS** | Mitigação automática na camada de rede (L3/L4) e aplicação (L7) |

**Regras específicas para o 2F-AGRO:**

```
# Regra 1: Bloquear injeção SQL em parâmetros de busca de propriedades
IF (URI contains "/propriedades" AND query_string matches "(\b(UNION|SELECT|INSERT|UPDATE|DELETE|DROP)\b)")
THEN block

# Regra 2: Limitar tamanho de payload de telemetria (estação envia < 1 KB)
IF (URI contains "/ingestao/telemetria" AND body_size > 10KB)
THEN block

# Regra 3: Bloquear User-Agents suspeitos em endpoints de autenticação
IF (URI contains "/auth/" AND user_agent matches "(sqlmap|nikto|dirbuster|gobuster)")
THEN block
```

### 3.3 SIEM (Security Information and Event Management)

A stack de observabilidade de segurança utiliza **Grafana + Loki + Promtail** — open-source, leve, e com custo acessível para o projeto.

**Arquitetura do SIEM:**

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API Gateway  │  │ Microsserviço│  │ Estação Edge │
│ (logs JSON)  │  │ (logs JSON)  │  │ (syslog)     │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌────────────────────────────────────────────────────┐
│                    Promtail                          │
│  (agente coletor — um por nó/pod)                   │
│  • Scrape de logs de containers (stdout/stderr)     │
│  • Labels: app, env, severity                       │
│  • Filtro: coordenadas anonimizadas antes de enviar │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│                      Loki                           │
│  (armazenamento de logs indexados por label)        │
│  • Retenção: 90 dias (hot) + 12 meses (S3 cold)   │
│  • Sem full-text index — custo baixo               │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│                    Grafana                           │
│  (dashboards + alertas)                             │
│  • Dashboard "Segurança 2F-AGRO"                   │
│  • Alertas automáticos via Grafana Alerting         │
└────────────────────────────────────────────────────┘
```

**Alertas configurados:**

| Alerta | Condição | Ação |
|---|---|---|
| **Brute force** | > 10 falhas de login do mesmo IP em 5 min | Notificar equipe + bloquear IP temporariamente (15 min) |
| **Token inválido em massa** | > 50 requests com JWT expirado/inválido em 1 min | Investigar possível ataque de replay |
| **Telemetria anômala** | Estação reportando dados fora do range físico (ex.: umidade > 100%) | Alerta de possível manipulação; quarentenar dados |
| **Acesso admin fora de horário** | Login com role `admin` entre 00h-06h | Notificação imediata ao DPO |
| **Pico de requests** | > 500 req/min em um único endpoint | Possível DDoS; WAF auto-escala |
| **Erro 5xx em cascata** | > 20 erros 500 em 2 min no mesmo serviço | Alerta de incidente operacional |

### 3.4 Secrets em Vault

Nenhum segredo é armazenado em código, variáveis de ambiente estáticas ou arquivos de configuração. Todos os secrets residem no **HashiCorp Vault**.

| Secret | Caminho no Vault | Rotação |
|---|---|---|
| Chave privada RS256 (JWT) | `secret/2f-agro/auth/jwt-private-key` | Trimestral (com grace period de 24h para tokens existentes) |
| Connection string PostgreSQL | `secret/2f-agro/db/postgres` | Mensal (credential lease do Vault Database Engine) |
| Chave AES-256 para backups | `secret/2f-agro/backup/aes-key` | Mensal |
| API key NASA POWER | `secret/2f-agro/external/nasa-power` | Anual (ou se comprometida) |
| API key CPTEC-INPE | `secret/2f-agro/external/cptec-inpe` | Anual |
| Expo push notification token | `secret/2f-agro/mobile/expo-push` | Semestral |
| Certificado mTLS da estação edge | `secret/2f-agro/edge/{station-id}/mtls-cert` | Anual (com reprovisão remota) |

**Políticas de acesso ao Vault:**

```hcl
# Política para o serviço de autenticação
path "secret/data/2f-agro/auth/*" {
  capabilities = ["read"]
}

# Política para o serviço de ingestão (só lê credenciais externas)
path "secret/data/2f-agro/external/*" {
  capabilities = ["read"]
}

# Política para o serviço de backup
path "secret/data/2f-agro/backup/*" {
  capabilities = ["read"]
}

# Nenhum serviço tem 'update' ou 'delete' — apenas operadores humanos
```

**Fluxo de provisionamento (CI/CD):**

1. GitHub Actions autentica no Vault via OIDC (sem secrets estáticos no CI)
2. Vault emite credenciais temporárias (lease de 1h) para o deploy
3. Aplicação inicia e busca secrets via Vault Agent Sidecar (Kubernetes)
4. Secrets nunca tocam disco — injetados em memória via volume `tmpfs`

### 3.5 Certificate Pinning no App Mobile

O app mobile implementa **certificate pinning** para impedir ataques Man-in-the-Middle, mesmo que o atacante comprometa uma CA (Certificate Authority) ou instale um certificado malicioso no device.

| Aspecto | Detalhe |
|---|---|
| **O que é fixado** | Hash SHA-256 do **Subject Public Key Info (SPKI)** do certificado do servidor (pin-sha256) |
| **Backup pin** | Hash de um certificado de contingência (rotação sem deploy emergencial) |
| **Implementação** | Configuração via `react-native-ssl-pinning` (Config Plugin Expo) + interceptor Axios customizado |
| **Fallback** | Se o pin falhar, request é **recusado** (fail-closed) — app exibe mensagem de erro de conectividade |

**Configuração no app React Native (Expo):**

```typescript
// src/config/seguranca.ts
export const CERTIFICATE_PINS = {
  // Pin primário (certificado atual do servidor)
  principal: 'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
  // Pin de backup (certificado de contingência, gerado e armazenado offline)
  backup: 'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',
};

// Interceptor Axios com validação de pin
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'https://api.2f-agro.com.br',
  timeout: 30000,
});

// Em produção, o certificate pinning é aplicado na camada nativa
// via plugin Expo ou react-native-ssl-pinning
```

**Rotação de certificados (sem quebrar o app):**

1. Novo certificado gerado com antecedência (30 dias antes da expiração)
2. Hash do novo certificado adicionado como `backup` no app (deploy via OTA update do Expo)
3. Após migração do servidor, o novo hash vira `principal` e o antigo é removido

### 3.6 Rate Limiting

O rate limit protege contra abuso, DDoS na camada de aplicação, e tentativas de brute force.

**Configuração por endpoint:**

| Endpoint / Grupo | Limite | Janela | Justificativa |
|---|---|---|---|
| **Global (por IP)** | 100 req/min | 1 minuto | Proteção geral contra abuso |
| `POST /auth/login` | 5 req/min | 1 minuto | Anti brute-force |
| `POST /auth/register` | 3 req/hora | 1 hora | Anti criação massiva de contas |
| `POST /auth/mfa` | 5 req/min | 1 minuto | Anti brute-force de OTP |
| `POST /ingestao/telemetria` | 60 req/min | 1 minuto | Estação envia a cada ~30s; margem para retries |
| `POST /diagnosticos` | 10 req/min | 1 minuto | Uso normal: 1-2 fotos por sessão |
| `GET /alertas` | 30 req/min | 1 minuto | Polling do app; margem confortável |

**Implementação (C# .NET 8 — middleware nativo):**

```csharp
// Program.cs — Rate Limiting com fixed window
builder.Services.AddRateLimiter(options =>
{
    // Limite global: 100 req/min por IP
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        context => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "desconhecido",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0
            }));

    // Limite específico para login: 5 req/min
    options.AddFixedWindowLimiter("login", limiter =>
    {
        limiter.PermitLimit = 5;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });

    // Resposta quando rate limit é atingido
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

// No endpoint
[EnableRateLimiting("login")]
[HttpPost("auth/login")]
public async Task<IActionResult> Login([FromBody] LoginRequest request) { /* ... */ }
```

**Headers de resposta (transparência para o cliente):**

```
HTTP/1.1 429 Too Many Requests
Retry-After: 42
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1717373000
```

---

## Referências

- OWASP Top 10 (2021) — <https://owasp.org/Top10/>
- NIST SP 800-207 — Zero Trust Architecture — <https://csrc.nist.gov/publications/detail/sp/800-207/final>
- NIST SP 800-63B — Digital Identity Guidelines (Authentication) — <https://pages.nist.gov/800-63-3/sp800-63b.html>
- RFC 7519 — JSON Web Token (JWT) — <https://datatracker.ietf.org/doc/html/rfc7519>
- RFC 8446 — TLS 1.3 — <https://datatracker.ietf.org/doc/html/rfc8446>
- bcrypt (Provos & Mazières, 1999) — <https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf>
- LGPD — Lei nº 13.709/2018 — <https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm>
- HashiCorp Vault — <https://www.vaultproject.io/>
- Grafana Loki — <https://grafana.com/oss/loki/>
- International Charter Space and Major Disasters — <https://disasterscharter.org/>
