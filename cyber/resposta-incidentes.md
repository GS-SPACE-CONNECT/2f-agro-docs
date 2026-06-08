# 2F-AGRO — Plano de Resposta a Incidentes de Segurança

> **Global Solution · FIAP · 1º semestre 2026 · 3ES — Engenharia de Software**
>
> Matéria: **Cybersecurity** · Entrega: Plano de Resposta a Incidentes (3 pts)
>
> Responsável: [@jota0802](https://github.com/jota0802)
>
> Última atualização: junho 2026

---

## Sumário

| # | Seção |
|---|---|
| 1 | [Objetivo e escopo](#1-objetivo-e-escopo) |
| 2 | [Classificação de incidentes](#2-classificação-de-incidentes) |
| 3 | [Equipe de resposta (CSIRT)](#3-equipe-de-resposta-csirt) |
| 4 | [Detecção e triagem](#4-detecção-e-triagem) |
| 5 | [Fase 1 — Contenção (0–30 min)](#5-fase-1--contenção-030-min) |
| 6 | [Fase 2 — Erradicação (30 min – 4 h)](#6-fase-2--erradicação-30-min--4-h) |
| 7 | [Fase 3 — Recuperação (4 h – 48 h)](#7-fase-3--recuperação-4-h--48-h) |
| 8 | [Playbooks por vetor de ataque](#8-playbooks-por-vetor-de-ataque) |
| 9 | [Comunicação e escalação](#9-comunicação-e-escalação) |
| 10 | [Métricas de resposta](#10-métricas-de-resposta) |
| 11 | [Template de registro de incidente](#11-template-de-registro-de-incidente) |
| 12 | [Referências](#12-referências) |

---

## 1. Objetivo e escopo

Este documento define o procedimento de resposta a incidentes de segurança da informação da plataforma **2F-AGRO**. O plano é acionado sempre que um evento de segurança é detectado — seja por alerta automatizado do SIEM (Grafana + Loki), pelo WAF (Cloudflare), por reporte de usuário ou por análise manual.

**Escopo:** toda a superfície de ataque do 2F-AGRO — app mobile (React Native/Expo), API Gateway (C# .NET 8), microsserviços SOA (Java/Spring Boot), pipeline ML (Python/scikit-learn), estações edge IoT (Raspberry Pi 4) e bases de dados (PostgreSQL + PostGIS).

**Integração com outros documentos de Cybersecurity:**

| Documento | Relação com este plano |
|---|---|
| [Threat Model](threat-model.md) | Vetores VET-01, VET-02, VET-03 — base dos playbooks da seção 8 |
| [Arquitetura de Segurança](arquitetura-seguranca.md) | Controles técnicos (JWT, RBAC, SIEM, WAF, Vault) — ferramentas usadas na resposta |
| [Governança ISO 27001 + LGPD](governanca-iso-lgpd.md) | SGSI (papéis, KPIs), LGPD (DPO, notificação ANPD) — obrigações legais e organizacionais |

**Premissa operacional:** o 2F-AGRO atende agricultores familiares cujo sustento depende dos alertas agroclimáticos da plataforma. Um incidente de segurança não é apenas um problema técnico — pode significar perda de safra, prejuízo financeiro e risco físico para o produtor. A resposta deve ser rápida e priorizar a proteção do agricultor.

---

## 2. Classificação de incidentes

Cada incidente recebe uma severidade que determina os tempos de resposta e a cadeia de escalação.

| Severidade | Descrição | Exemplos | Tempo de resposta | Escalação |
|---|---|---|---|---|
| **SEV-1 (Crítico)** | Comprometimento confirmado com impacto em dados pessoais ou integridade do sistema | Vazamento de PII (VET-03); envenenamento confirmado do modelo ML (VET-02); ransomware; acesso admin não autorizado | Contenção em **15 min**; CSIRT completo acionado | Imediata: Gestor SI + DPO + toda equipe |
| **SEV-2 (Alto)** | Comprometimento parcial ou tentativa bem-sucedida sem vazamento confirmado | MITM em telemetria (VET-01); brute force com login bem-sucedido; estação IoT comprometida; indisponibilidade prolongada da API | Contenção em **30 min** | Gestor SI + responsável técnico da camada afetada |
| **SEV-3 (Médio)** | Tentativa de ataque detectada e bloqueada, sem comprometimento | Brute force bloqueado pelo rate limiter; anomalia de telemetria isolada (quarentenada); scan de vulnerabilidade detectado pelo WAF | Análise em **4 h** | Gestor SI notificado; ação no próximo turno |
| **SEV-4 (Baixo)** | Evento informativo sem impacto operacional | Scan de portas externo; SQLi bloqueado pelo WAF; bot tentando endpoints inexistentes | Registro e revisão em **24 h** | Log para revisão na próxima auditoria |

**Regra de ouro:** na dúvida entre dois níveis de severidade, **escalar para o mais alto**. É melhor mobilizar a equipe desnecessariamente do que subestimar um incidente.

---

## 3. Equipe de resposta (CSIRT)

O CSIRT (Computer Security Incident Response Team) do 2F-AGRO é composto por todos os membros da equipe, com papéis específicos durante um incidente.

| Papel no incidente | Responsável | Atribuições durante o incidente |
|---|---|---|
| **Coordenador de incidente** | jota ([@jota0802](https://github.com/jota0802) — Gestor de SI) | Declarar o incidente; classificar severidade; coordenar ações; manter timeline; decidir escalar ou desescalar; convocar post-mortem |
| **DPO (Encarregado de Dados)** | roji ([@roji-menez](https://github.com/roji-menez)) | Avaliar se houve violação de dados pessoais (LGPD); preparar notificação à ANPD (72 h); comunicar titulares afetados; documentar impacto em PII |
| **Resposta Backend / API** | brunão ([@brnleao](https://github.com/brnleao)) + ruan ([@DevRuanVieira](https://github.com/DevRuanVieira)) | Isolar microsserviços; revogar tokens JWT; ativar modo degradado; aplicar patches emergenciais; rotacionar credenciais no Vault |
| **Resposta ML / IoT** | lucca ([@lucksza](https://github.com/lucksza)) | Desconectar estações comprometidas; validar integridade do modelo ML; executar rollback de modelo; quarentenar dados anômalos |
| **Apoio geral** | Todos | Preservar evidências; atualizar timeline no canal de comunicação; executar tarefas delegadas pelo coordenador |

**Canais de comunicação durante o incidente:**

| Canal | Uso |
|---|---|
| **Grupo de emergência (WhatsApp/Telegram)** | Notificação inicial e coordenação rápida |
| **Canal #incidentes (Discord do time)** | Timeline detalhada, links de evidências, atualizações de status |
| **Chamada de voz (Discord/Meet)** | War room para SEV-1 e SEV-2 — todos online até contenção confirmada |
| **Issue no GitHub** | Registro formal do incidente (label `incidente-segurança`); post-mortem anexado |

---

## 4. Detecção e triagem

### 4.1 Fontes de detecção

O 2F-AGRO utiliza múltiplas fontes para detectar incidentes, configuradas na Arquitetura de Segurança:

| Fonte | O que detecta | Severidade típica |
|---|---|---|
| **SIEM — Grafana + Loki** (alertas automáticos) | Brute force (>10 falhas/5 min), tokens inválidos em massa (>50/min), telemetria anômala, acesso admin fora de horário (00h–06h), erros 5xx em cascata (>20/2 min), pico de requests (>500/min) | SEV-2 a SEV-4 |
| **WAF — Cloudflare** | SQLi, XSS, LFI/RFI, command injection, payloads de telemetria fora de range, User-Agents de ferramentas de ataque (sqlmap, nikto) | SEV-3 a SEV-4 |
| **Correlação cruzada de estações IoT** | Divergência >30% entre estações vizinhas (<20 km) — possível adulteração | SEV-2 a SEV-3 |
| **Validação estatística de ingestão** | Leituras fora da faixa esperada para região/época (ex.: 10°C no semiárido em dezembro) | SEV-3 |
| **Sensor de abertura de gabinete (estação edge)** | Violação física da estação IoT | SEV-2 |
| **Monitoramento de anomalia de rede** | Estação silenciosa por >5 min ou jitter anormal | SEV-3 |
| **Reporte de usuário / cooperativa** | Comportamento inesperado no app, alertas errados, dados inconsistentes | SEV-2 a SEV-4 |
| **GitHub push protection** | Tentativa de commit com secret/credencial | SEV-3 |

### 4.2 Processo de triagem

Quando um alerta é recebido, o membro da equipe que o identifica primeiro executa a triagem:

```
┌──────────────────────────────────────────────────────────────┐
│  ALERTA RECEBIDO (SIEM / WAF / reporte / sensor)             │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  1. Confirmar que não é falso positivo                        │
│     • Verificar logs no Grafana (query LogQL)                │
│     • Correlacionar com outros alertas recentes              │
│     • Verificar se há manutenção programada em andamento     │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  2. Classificar severidade (tabela da seção 2)               │
│     • Houve acesso a dados pessoais? → mínimo SEV-2          │
│     • Há comprometimento confirmado? → SEV-1                 │
│     • Impacto no agricultor (alertas / safra)? → escalar     │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  3. Notificar o Coordenador de incidente (jota)              │
│     • Mensagem no grupo de emergência com:                   │
│       — Descrição do alerta                                  │
│       — Severidade proposta                                  │
│       — Evidência inicial (link do dashboard / screenshot)   │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  4. Coordenador confirma severidade e declara incidente      │
│     • Cria issue no GitHub (label: incidente-segurança)      │
│     • Aciona os membros do CSIRT conforme a severidade       │
│     • Inicia o cronômetro de resposta                        │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Fase 1 — Contenção (0–30 min)

**Objetivo:** limitar o impacto do incidente e impedir que o atacante avance. Cada ação abaixo tem um responsável e um procedimento técnico concreto.

### 5.1 Isolar serviço comprometido

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão / ruan (backend) ou lucca (IoT/ML) — conforme a camada afetada |
| **Como** | Em ambiente Kubernetes: aplicar `NetworkPolicy` de quarentena que bloqueia todo tráfego de entrada/saída do pod comprometido, exceto a porta do SIEM (para continuar coletando logs forenses). Em ambiente não-containerizado: remover o serviço do load balancer e bloquear via firewall (iptables/NSG). |
| **Verificação** | Confirmar isolamento com `kubectl get networkpolicy` ou tentativa de request ao serviço isolado → deve retornar timeout |

**NetworkPolicy de quarentena (Kubernetes):**

```yaml
# k8s/networkpolicy-quarentena.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarentena-incidente
  namespace: 2f-agro
spec:
  podSelector:
    matchLabels:
      quarentena: "true"     # label aplicado no pod comprometido
  policyTypes:
    - Ingress
    - Egress
  ingress: []                # bloqueia toda entrada
  egress:
    - to:                    # permite apenas saída para o SIEM
        - namespaceSelector:
            matchLabels:
              app: loki
      ports:
        - port: 3100
          protocol: TCP
```

**Procedimento:**

```bash
# 1. Adicionar label de quarentena ao pod comprometido
kubectl label pod <nome-do-pod> quarentena=true -n 2f-agro

# 2. Aplicar a NetworkPolicy
kubectl apply -f k8s/networkpolicy-quarentena.yaml

# 3. Verificar que o pod está isolado
kubectl exec -n 2f-agro <outro-pod> -- curl -s --max-time 5 http://<servico-isolado>:8080/health
# Esperado: timeout (sem resposta)
```

### 5.2 Revogar tokens JWT

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão / ruan |
| **Quando** | Sempre que houver suspeita de comprometimento de credenciais, sessão hijacking ou acesso não autorizado |
| **Como** | Adicionar o `jti` (ID único do token) à lista de revogação no Redis. O middleware de autenticação do API Gateway consulta essa lista a cada request. Para revogação em massa: rotacionar a chave privada RS256 no Vault (invalida todos os tokens existentes). |

**Revogação individual (por `jti`):**

```bash
# Conectar ao Redis e adicionar o jti à blocklist
# TTL = tempo restante até expiração do token (máx 3600s para access_token)
redis-cli SET "jwt:revogado:<jti-do-token>" "1" EX 3600
```

**Revogação em massa (rotação de chave — SEV-1):**

```bash
# 1. Gerar nova chave RS256 no Vault
vault write secret/2f-agro/auth/jwt-private-key \
  key="$(openssl genrsa 2048 2>/dev/null)"

# 2. Reiniciar o Serviço Auth para carregar a nova chave
kubectl rollout restart deployment/auth-service -n 2f-agro

# 3. Todos os tokens existentes tornam-se inválidos
#    (assinados com a chave anterior)
# 4. Usuários precisam refazer login — impacto aceitável em SEV-1
```

**Validação no API Gateway (C# .NET 8) — já implementado:**

```csharp
// Middleware que consulta a blocklist Redis antes de aceitar o JWT
public class JwtRevogacaoMiddleware(RequestDelegate next, IConnectionMultiplexer redis)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var jti = context.User?.FindFirst("jti")?.Value;

        if (jti != null)
        {
            var db = redis.GetDatabase();
            var revogado = await db.KeyExistsAsync($"jwt:revogado:{jti}");

            if (revogado)
            {
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                return;
            }
        }

        await next(context);
    }
}
```

### 5.3 Cortar uplink suspeito

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | lucca (IoT/ML) |
| **Quando** | Estação IoT comprometida (sensor de abertura acionado, dados manipulados, ou correlação cruzada positiva) |
| **Como** | Desativar o gateway LoRaWAN que atende a estação comprometida; ou bloquear o IP/certificado mTLS da estação no API Gateway |

**Procedimento:**

```bash
# Opção 1: Revogar certificado mTLS da estação no Vault
vault write secret/2f-agro/edge/<station-id>/status revogado=true

# Opção 2: Bloquear IP da estação no firewall do API Gateway
# (via Cloudflare WAF — regra de IP block)
curl -X POST "https://api.cloudflare.com/client/v4/zones/<zone>/firewall/rules" \
  -H "Authorization: Bearer <token>" \
  -d '{"filter":{"expression":"ip.src eq <ip-estacao>"},"action":"block"}'

# Opção 3: Desativar gateway LoRaWAN via console de gerenciamento
# (procedimento específico do hardware — ChirpStack/TTN)
```

**Dados em trânsito da estação:** após o corte, os dados ficam armazenados localmente no buffer do Raspberry Pi (filesystem F2FS) e serão sincronizados após a estação ser revalidada — design offline-first da camada edge.

### 5.4 Modo degradado (read-only)

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão / ruan |
| **Quando** | Comprometimento da API que exige interromper operações de escrita sem derrubar a plataforma inteira |
| **Por quê** | Agricultores continuam consultando alertas existentes e dados da propriedade (leitura); apenas escritas são bloqueadas para impedir que o atacante persista ou modifique dados |

**Implementação via feature flag:**

```csharp
// Middleware que bloqueia operações de escrita quando o modo degradado está ativo
public class ModoDegradadoMiddleware(RequestDelegate next, IConfiguration config)
{
    private static readonly HashSet<string> MetodosLeitura = ["GET", "HEAD", "OPTIONS"];

    public async Task InvokeAsync(HttpContext context)
    {
        var modoDegradado = config.GetValue<bool>("Seguranca:ModoDegradado");

        if (modoDegradado && !MetodosLeitura.Contains(context.Request.Method))
        {
            context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
            await context.Response.WriteAsJsonAsync(new
            {
                erro = "Sistema em modo de manutenção. Apenas consultas estão disponíveis.",
                codigo = "MODO_DEGRADADO"
            });
            return;
        }

        await next(context);
    }
}
```

**Ativação:**

```bash
# Via variável de ambiente (Kubernetes ConfigMap)
kubectl set env deployment/api-gateway -n 2f-agro \
  Seguranca__ModoDegradado=true

# Ou via Vault (se configuração centralizada)
vault write secret/2f-agro/config modo_degradado=true
```

**Mensagem no app mobile:** quando o modo degradado está ativo, o app exibe um banner:

```
⚠ O 2F-AGRO está em manutenção de segurança.
Seus alertas e dados estão disponíveis para consulta.
Novas ações estarão disponíveis em breve.
```

### 5.5 Notificar DPO

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | Coordenador de incidente (jota) |
| **Quando** | **Sempre** em SEV-1 e SEV-2. Em SEV-3, apenas se houver suspeita de exposição de dados pessoais |
| **Como** | Mensagem imediata no grupo de emergência + e-mail formal para dpo@2f-agro.com.br |

**Conteúdo mínimo da notificação ao DPO:**

```
NOTIFICAÇÃO DE INCIDENTE DE SEGURANÇA

Data/hora de detecção: ____
Severidade: SEV-__
Descrição resumida: ____
Dados pessoais possivelmente afetados: [ ] Sim  [ ] Não
  Se sim, quais categorias: ____
  Estimativa de titulares afetados: ____
Ações de contenção já executadas: ____
Coordenador do incidente: ____
```

**Obrigação LGPD (art. 48):** se confirmado o comprometimento de dados pessoais, o DPO tem **72 horas** (prazo recomendado pela ANPD, alinhado com o GDPR) para comunicar a ANPD e os titulares afetados. O modelo de notificação está na seção 9.

---

## 6. Fase 2 — Erradicação (30 min – 4 h)

**Objetivo:** eliminar a causa raiz do incidente e garantir que o atacante não mantém acesso residual ao sistema.

### 6.1 Análise forense (Grafana + Loki)

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | Coordenador (jota) + responsável técnico da camada afetada |
| **Ferramenta** | Grafana com data source Loki — dashboard "Segurança 2F-AGRO" |

**Queries LogQL para investigação:**

```logql
# 1. Requests do IP suspeito nas últimas 24h
{app="api-gateway"} |= "<ip-suspeito>"
  | json
  | line_format "{{.timestamp}} {{.method}} {{.path}} {{.status}} {{.userId}}"

# 2. Todas as autenticações do usuário comprometido
{app="auth-service"} |= "<user-id>"
  | json
  | status_code >= 200
  | line_format "{{.timestamp}} {{.action}} {{.ip}} {{.userAgent}}"

# 3. Telemetria recebida de estação suspeita
{app="ingestao-service"} |= "<station-id>"
  | json
  | line_format "{{.timestamp}} temp={{.temperatura}} umid={{.umidade}} vento={{.vento}}"

# 4. Erros 401/403 concentrados (possível tentativa de privilege escalation)
{app="api-gateway"} | json | status_code=~"401|403"
  | rate({app="api-gateway"} | json | status_code=~"401|403" [5m]) > 10

# 5. Requests com user-agents de ferramentas de ataque
{app="api-gateway"} |~ "sqlmap|nikto|dirbuster|gobuster|hydra"
```

**Preservação de evidências:**

- **Não reiniciar** pods/serviços até que os logs sejam exportados
- Exportar logs do período do incidente para bucket S3 com write-once (retenção legal)
- Capturar estado do pod comprometido: `kubectl logs <pod>` + `kubectl describe pod <pod>`
- Se estação IoT: clonar o cartão SD antes de qualquer alteração (imagem forense com `dd`)

### 6.2 Identificar IoC (Indicators of Compromise)

Ao longo da análise forense, documentar todos os IoCs encontrados:

| Tipo de IoC | Exemplos | Onde procurar |
|---|---|---|
| **IPs maliciosos** | IPs de origem dos requests suspeitos | Logs do API Gateway, WAF |
| **User-Agents anômalos** | Ferramentas de ataque, bots, scripts | Logs do API Gateway |
| **Tokens comprometidos** | JTIs usados de IPs diferentes simultaneamente | Logs do Auth Service + Redis |
| **Payloads maliciosos** | SQLi, XSS, command injection, dados de telemetria forjados | WAF logs, logs de ingestão |
| **Hashes de arquivos alterados** | Modelo ML com hash diferente do esperado | Comparação com hash SHA-256 do MLflow |
| **Certificados inválidos** | Certificado mTLS da estação com assinatura diferente | Logs do TLS termination |
| **Contas comprometidas** | Contas com login de IPs/horários incomuns | Logs de autenticação |

**Registro dos IoCs:**

Cada IoC é documentado na issue do GitHub com o formato:

```markdown
### IoC-001
- **Tipo:** IP malicioso
- **Valor:** 203.0.113.42
- **Primeira ocorrência:** 2026-06-01T14:23:00Z
- **Última ocorrência:** 2026-06-01T15:47:00Z
- **Ação tomada:** IP bloqueado no WAF (Cloudflare)
```

### 6.3 Patch da vulnerabilidade

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão / ruan (backend) ou lucca (IoT/ML) |
| **Procedimento** | Identificar a vulnerabilidade explorada → desenvolver fix → code review emergencial (mínimo 1 reviewer) → deploy em ambiente de homologação → teste de regressão rápido → deploy em produção |

**Fluxo de deploy emergencial:**

```
1. Branch: hotfix/inc-<número>
2. Commit: fix: corrigir <descrição> (Refs #<issue-incidente>)
3. PR com review mínimo de 1 pessoa (excepcionalmente, em SEV-1, o coordenador pode aprovar)
4. CI roda (se passar): merge + deploy automático
5. CI falha: avaliar se o fix é crítico o suficiente para deploy manual
```

**Categorias de patches comuns:**

| Vulnerabilidade | Ação de patch |
|---|---|
| Endpoint sem `[Authorize]` | Adicionar atributo + teste de integração que previna regressão |
| BOLA/IDOR | Implementar owner check no handler de autorização |
| SSH com senha padrão na estação | Desabilitar login por senha; forçar chave RSA-4096 |
| Dependência vulnerável (CVE) | Atualizar via Dependabot ou manualmente; rodar SAST |
| Validação de telemetria insuficiente | Adicionar regras de sanidade na faixa esperada por região/época |

### 6.4 Re-treinar modelo se envenenado

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | lucca (ML/IoT) |
| **Quando** | Quando a análise forense confirma que dados fraudulentos entraram no pipeline de treinamento (VET-02) |

**Procedimento:**

```
1. IDENTIFICAR o período de contaminação
   • Usar IoCs (estação comprometida, timestamps dos dados forjados)
   • Marcar registros contaminados na tabela de telemetria

2. ROLLBACK do modelo
   • Restaurar a última versão limpa do modelo via artefato versionado
     (MLflow ou repositório de modelos com hash SHA-256)
   • Verificar acurácia do modelo restaurado em validação cruzada

3. QUARENTENAR dados
   • Mover registros contaminados para tabela de quarentena
   • Não deletar — preservar como evidência forense

4. RE-TREINAR com dados limpos
   • Executar pipeline de treinamento excluindo dados quarentenados
   • Validar acurácia ≥ baseline esperado (degradação < 5%)
   • Gerar novo artefato versionado com hash SHA-256

5. DEPLOY do modelo corrigido
   • Substituir modelo em produção pelo re-treinado
   • Monitorar métricas de inferência por 24h
```

### 6.5 Rotação de chaves

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão / ruan (com acesso ao Vault) |
| **Quando** | Sempre em SEV-1. Em SEV-2, quando houver suspeita de comprometimento de credenciais |

**Chaves a rotacionar (por prioridade):**

| Prioridade | Secret | Caminho no Vault | Procedimento |
|---|---|---|---|
| 1 | Chave privada RS256 (JWT) | `secret/2f-agro/auth/jwt-private-key` | Gerar nova chave → reiniciar Auth Service → todos os tokens existentes invalidados |
| 2 | Connection string PostgreSQL | `secret/2f-agro/db/postgres` | Alterar senha do usuário no PostgreSQL → atualizar no Vault → restart dos serviços que usam o banco |
| 3 | Certificado mTLS da estação comprometida | `secret/2f-agro/edge/<station-id>/mtls-cert` | Revogar certificado antigo → gerar novo → reprovisionar a estação |
| 4 | API keys externas (NASA POWER, CPTEC) | `secret/2f-agro/external/*` | Rotacionar apenas se houver evidência de exfiltração |
| 5 | Chave AES-256 de backups | `secret/2f-agro/backup/aes-key` | Gerar nova chave → próximos backups usam a nova; backups antigos continuam legíveis com a chave anterior (armazenada separadamente) |

**Verificação pós-rotação:**

```bash
# Confirmar que os serviços reiniciaram com as novas credenciais
kubectl get pods -n 2f-agro -o wide | grep -v Running

# Testar autenticação com as novas credenciais
curl -s https://api.2f-agro.com.br/health
# Esperado: 200 OK

# Verificar que tokens antigos são rejeitados
curl -s -H "Authorization: Bearer <token-antigo>" \
  https://api.2f-agro.com.br/api/v1/alertas
# Esperado: 401 Unauthorized
```

---

## 7. Fase 3 — Recuperação (4 h – 48 h)

**Objetivo:** restaurar a operação normal do sistema com segurança verificada, comunicar stakeholders e aprender com o incidente.

### 7.1 Restore de backup limpo

| Aspecto | Detalhe |
|---|---|
| **Quem executa** | brunão (responsável por backups) |
| **Quando** | Quando dados foram alterados/corrompidos pelo atacante e não é possível reverter cirurgicamente |
| **RPO alvo** | ≤ 24 horas (backup diário criptografado — AES-256-GCM) |
| **RTO alvo** | ≤ 4 horas (tempo máximo para restaurar a operação) |

**Procedimento de restore:**

```bash
# 1. Identificar o backup mais recente ANTERIOR ao comprometimento
#    (usar timestamp da primeira atividade maliciosa como corte)
aws s3 ls s3://2f-agro-backups/ --recursive | sort -k1,2

# 2. Baixar e descriptografar o backup
aws s3 cp s3://2f-agro-backups/<backup-limpo>.sql.enc ./
openssl enc -d -aes-256-gcm -in <backup-limpo>.sql.enc \
  -out backup-limpo.sql -pass file:<chave-do-vault>

# 3. Restaurar em banco temporário primeiro (validação)
createdb 2f_agro_restore_temp
pg_restore -d 2f_agro_restore_temp backup-limpo.sql

# 4. Validar integridade (seção 7.2)

# 5. Swap: promover banco restaurado para produção
# (procedimento específico conforme infraestrutura — failover de réplica)
```

### 7.2 Validação de integridade

Antes de promover o backup restaurado para produção, validar:

| Verificação | Comando / Método | Critério de aceite |
|---|---|---|
| **Contagem de registros** | `SELECT COUNT(*) FROM propriedades, alertas, usuarios` | Valores consistentes com o período do backup |
| **Hash do modelo ML** | `sha256sum modelo_producao.pkl` | Hash igual ao artefato versionado limpo |
| **Integridade de coordenadas** | `SELECT * FROM propriedades WHERE latitude IS NULL OR longitude IS NULL` | Zero registros com coordenadas nulas (exceto titulares que exerceram DELETE /me) |
| **Consentimentos válidos** | `SELECT COUNT(*) FROM consentimentos WHERE concedido = true` | Compatível com base de usuários ativos |
| **Certificados de estações** | Verificar cada certificado mTLS ativo contra a CA interna | Nenhum certificado não reconhecido |
| **Configurações de segurança** | RBAC, rate limiting, WAF rules | Conferir contra baseline documentado na Arquitetura de Segurança |

### 7.3 Comunicação a stakeholders + ANPD

#### 7.3.1 Comunicação interna (imediata)

Após a contenção, o coordenador envia um resumo para toda a equipe:

```
ATUALIZAÇÃO DE INCIDENTE — INC-<número>

Status: [Contido / Em erradicação / Em recuperação / Resolvido]
Severidade: SEV-__
Resumo: ____
Impacto no agricultor: ____
Próximos passos: ____
ETA para operação normal: ____
```

#### 7.3.2 Notificação à ANPD (até 72 h — se houver violação de PII)

Conforme art. 48 da LGPD, o DPO (roji) prepara e envia notificação à ANPD:

| Campo da notificação | Conteúdo |
|---|---|
| **Natureza dos dados afetados** | Categorias: identificação, localização, produtivos, financeiros (conforme aplicável) |
| **Titulares afetados** | Número estimado de agricultores cujos dados foram expostos/comprometidos |
| **Período da violação** | Data/hora de início e de detecção |
| **Consequências prováveis** | Risco de exposição de localização (roubo de safra), fraude de identidade, etc. |
| **Medidas adotadas** | Contenção, erradicação e recuperação executadas (resumo técnico) |
| **Medidas futuras** | Ações preventivas planejadas (do post-mortem) |
| **Contato do DPO** | Roji Menez — dpo@2f-agro.com.br |

#### 7.3.3 Comunicação aos titulares (se dados pessoais foram comprometidos)

Os agricultores afetados são notificados via:

| Canal | Formato |
|---|---|
| **Push notification no app** | Mensagem clara e em linguagem acessível: "Identificamos um problema de segurança que pode ter afetado seus dados. Já resolvemos. Para sua segurança, troque sua senha." |
| **Cooperativa parceira** | Aviso ao gestor da cooperativa para comunicação presencial aos agricultores sem acesso digital |
| **E-mail / SMS** | Para titulares que forneceram esses canais |

### 7.4 Post-mortem

Até **5 dias úteis** após o encerramento do incidente, o coordenador conduz uma reunião de post-mortem com toda a equipe.

**Formato: blameless (sem culpabilização)**

O post-mortem foca em **sistemas e processos**, não em indivíduos. A pergunta não é "quem errou?" mas "o que no nosso sistema permitiu que isso acontecesse?".

**Estrutura do documento de post-mortem:**

```markdown
# Post-Mortem — INC-<número>

## Resumo
- **Data do incidente:** ____
- **Duração total:** ____
- **Severidade:** SEV-__
- **Coordenador:** ____
- **Impacto:** ____

## Timeline
| Horário | Evento |
|---|---|
| HH:MM | Primeiro alerta detectado |
| HH:MM | Incidente declarado |
| HH:MM | Contenção iniciada |
| HH:MM | Contenção confirmada |
| HH:MM | Erradicação concluída |
| HH:MM | Recuperação concluída |
| HH:MM | Incidente encerrado |

## Causa raiz (5 Whys)
1. Por que o incidente aconteceu? → ____
2. Por que essa condição existia? → ____
3. Por que não foi detectado antes? → ____
4. Por que o controle preventivo falhou? → ____
5. Por que esse gap existia no processo? → ____

## O que funcionou bem
- ____

## O que pode melhorar
- ____

## Ações corretivas (action items)
| # | Ação | Responsável | Prazo | Issue |
|---|---|---|---|---|
| 1 | ____ | ____ | ____ | #__ |
| 2 | ____ | ____ | ____ | #__ |
```

### 7.5 Atualização do threat model

Após o post-mortem, o threat model (`cyber/threat-model.md`) é atualizado para refletir:

| Atualização | Detalhe |
|---|---|
| **Novo vetor de ataque** | Se o incidente revelou um vetor não mapeado, adicionar à seção 4 do threat model com classificação STRIDE, probabilidade, impacto e mitigações |
| **Reavaliação de probabilidade** | Se um vetor considerado "Baixa probabilidade" se concretizou, reclassificar para "Média" ou "Alta" |
| **Novas mitigações** | Controles identificados no post-mortem que foram ou serão implementados |
| **Matriz de risco** | Recalcular o risco residual com os novos controles e a nova probabilidade |
| **Governança** | Atualizar a matriz de riscos do SGSI (`governanca-iso-lgpd.md` § 3.3) com os novos achados |

---

## 8. Playbooks por vetor de ataque

Para cada vetor de ataque do Threat Model, existe um playbook específico que detalha a resposta.

### 8.1 Playbook VET-01 — MITM na telemetria edge

**Cenário:** interceptação ou manipulação de dados entre a estação IoT e o serviço de ingestão via LoRaWAN/4G.

| Fase | Ação | Responsável |
|---|---|---|
| **Detecção** | Alerta de anomalia de rede (estação silenciosa >5 min) ou correlação cruzada (divergência >30% entre estações vizinhas) | SIEM automático |
| **Contenção** | Cortar uplink da estação suspeita (revogar certificado mTLS ou bloquear no gateway LoRaWAN); quarentenar dados recebidos no período suspeito | lucca |
| **Erradicação** | Análise forense do Raspberry Pi (clonar SD, verificar integridade do firmware, auditar processos); verificar se o gateway LoRaWAN foi comprometido; inspecionar logs de rede | lucca + jota |
| **Recuperação** | Reprovisionar a estação com firmware limpo; gerar novo certificado mTLS; reativar uplink; monitorar por 48h | lucca |
| **Pós-incidente** | Avaliar necessidade de proteção física adicional (gabinete reforçado, câmera); atualizar threat model | jota |

### 8.2 Playbook VET-02 — Envenenamento do modelo ML

**Cenário:** dados de telemetria fraudulentos alimentam o pipeline de treinamento, degradando a acurácia do modelo.

| Fase | Ação | Responsável |
|---|---|---|
| **Detecção** | Validação estatística rejeita leituras fora de range; degradação de acurácia >5% em validação cruzada periódica; correlação cruzada entre estações | SIEM + pipeline ML |
| **Contenção** | Isolar estação comprometida; ativar modo degradado para o serviço ML (usar último modelo validado); quarentenar dados anômalos | lucca + brunão |
| **Erradicação** | Identificar período e fonte da contaminação; marcar dados fraudulentos; rollback do modelo para versão limpa (artefato versionado com hash SHA-256); hardening da estação (SSH por chave, firewall, boot seguro) | lucca |
| **Recuperação** | Re-treinar modelo com dados limpos; validar acurácia; deploy do modelo corrigido; monitorar inferências por 48h; comunicar cooperativas afetadas sobre possíveis alertas incorretos no período | lucca + roji |
| **Pós-incidente** | Avaliar implementação de assinatura digital ECDSA nos pacotes de telemetria (M2.2); reforçar quarentena automática; atualizar threat model | jota + lucca |

### 8.3 Playbook VET-03 — Vazamento massivo de PII via API

**Cenário:** exploração de BOLA/IDOR ou endpoint sem autenticação expõe dados pessoais de agricultores.

| Fase | Ação | Responsável |
|---|---|---|
| **Detecção** | SIEM detecta volume anormal de requests de listagem (>100 registros em 1h); WAF detecta padrão de scraping (paginação sequencial completa); reporte externo (pesquisador/CERT) | SIEM + WAF |
| **Contenção** | Bloquear IP do atacante no WAF; revogar tokens JWT suspeitos; ativar modo degradado (read-only); notificar DPO **imediata­mente** (dados pessoais envolvidos — SEV-1 automático) | brunão + ruan + jota |
| **Erradicação** | Identificar endpoint vulnerável; aplicar patch (`[Authorize]` + owner check); adicionar teste de integração para prevenir regressão; auditar todos os endpoints com `GET` para verificar cobertura de autorização | brunão + ruan |
| **Recuperação** | Verificar extensão do vazamento (quais dados, quantos titulares); DPO notifica ANPD em até 72h; notificar titulares afetados (push notification + cooperativa); recomendar troca de senha; se CPF vazou: orientar agricultores sobre monitoramento de fraude | roji + jota |
| **Pós-incidente** | Implementar SAST obrigatório no CI para detectar endpoints sem `[Authorize]`; adicionar rate limiting mais restritivo em endpoints de listagem; atualizar threat model e RIPD | jota |

### 8.4 Playbook VET-04 — DDoS contra a API Gateway em período de safra

**Cenário:** ataque volumétrico (L3/L4) ou de aplicação (L7 — HTTP flood, slowloris) derruba a API Gateway durante período crítico de safra, impedindo que agricultores recebam alertas agroclimáticos.

| Fase | Ação | Responsável |
|---|---|---|
| **Detecção** | SIEM detecta latência p99 > 2 s ou erros 5xx > 5% por 2 min; WAF (Cloudflare) reporta pico anormal de tráfego; agricultores reportam app "fora do ar" | SIEM + WAF automático |
| **Contenção** | Ativar modo "Under Attack" no Cloudflare (challenge em todo tráfego L7); ativar modo degradado read-only na API (alertas e dados cacheados permanecem acessíveis); escalar pods via HPA se carga for absorvível; bloquear faixas de IP de origem predominante no WAF | brunão + ruan |
| **Erradicação** | Analisar padrão do ataque (volumétrico vs. aplicação) nos logs do WAF; identificar e bloquear IPs/ASNs de origem da botnet; ajustar regras WAF (rate limiting mais agressivo, validação de headers, JS challenge); confirmar que tráfego legítimo está fluindo normalmente | brunão + ruan + jota |
| **Recuperação** | Desativar modo "Under Attack" gradualmente (manter rate limiting reforçado por 24 h); desativar modo degradado; verificar que o pipeline de alertas (ingestão IoT + consulta NASA POWER/CPTEC) está operacional; enviar push notification aos agricultores: "O 2F-AGRO está de volta. Confira seus alertas atualizados." | brunão + ruan + roji |
| **Pós-incidente** | Avaliar se o plano de capacidade é adequado (dimensionamento dos pods, tier do Cloudflare); considerar redundância geográfica (multi-region); documentar padrão do ataque para enriquecer regras do WAF; atualizar threat model | jota |

---

## 9. Comunicação e escalação

### 9.1 Matriz de escalação

```
                    SEV-4          SEV-3          SEV-2          SEV-1
                   (Baixo)        (Médio)        (Alto)        (Crítico)
──────────────────────────────────────────────────────────────────────────
Coordenador (jota)  Notificado     Notificado     Acionado       Acionado
                    em 24h         em 4h          em 30 min      em 15 min

DPO (roji)          —              Se PII         Acionado       Acionado
                                   envolvido      em 30 min      em 15 min

Backend             —              Se             Acionado       Acionado
(brunão/ruan)                      aplicável

ML/IoT (lucca)      —              Se             Acionado       Acionado
                                   aplicável

ANPD                —              —              Se PII         Notificada
                                                  confirmado     em até 72h

Stakeholders        —              —              Informados     Informados
(cooperativas)                                    após           após
                                                  contenção      contenção
```

### 9.2 Regras de escalação

1. **Qualquer membro** da equipe pode declarar um incidente SEV-3 ou SEV-4.
2. **Apenas o coordenador** declara SEV-1 ou SEV-2 (ou qualquer membro em sua ausência, com revisão posterior).
3. Se o coordenador não responder em **15 minutos**, o próximo na cadeia (brunão → ruan → lucca → roji) assume a coordenação.
4. **Desescalação** (reduzir severidade) requer confirmação do coordenador com justificativa documentada.
5. Um incidente só é **encerrado** quando todas as ações de contenção e erradicação estão concluídas e verificadas.

---

## 10. Métricas de resposta

O desempenho do processo de resposta a incidentes é medido pelos seguintes indicadores, alinhados com os KPIs do SGSI (Governança ISO 27001 § 4.5):

| Métrica | Meta | Fórmula |
|---|---|---|
| **MTTD (Mean Time to Detect)** | ≤ 15 min para SEV-1/SEV-2 | Tempo entre início da atividade maliciosa e primeiro alerta |
| **MTTC (Mean Time to Contain)** | ≤ 30 min para SEV-1/SEV-2 | Tempo entre declaração do incidente e contenção confirmada |
| **MTTR (Mean Time to Resolve)** | ≤ 4 h para SEV-1/SEV-2 | Tempo entre declaração e encerramento do incidente |
| **Taxa de post-mortem** | 100% para SEV-1/SEV-2 | Post-mortems realizados / incidentes SEV-1+SEV-2 |
| **Reincidência** | 0% | Incidentes com mesma causa raiz nos 6 meses seguintes |
| **Conformidade LGPD** | 100% | Notificações à ANPD dentro do prazo de 72h / total de incidentes com PII |
| **Ações corretivas implementadas** | ≥ 90% em 30 dias | Action items do post-mortem concluídos no prazo |

**Revisão:** métricas consolidadas e apresentadas na auditoria semestral do SGSI.

---

## 11. Template de registro de incidente

Todo incidente SEV-1 a SEV-3 deve ser registrado usando o template abaixo (issue no GitHub com label `incidente-segurança`):

```markdown
# INC-<AAAA-NNN> — <Título descritivo>

## Classificação
- **Severidade:** SEV-__
- **Vetor relacionado:** VET-__ (ou "novo — não mapeado")
- **Dados pessoais afetados:** [ ] Sim  [ ] Não
- **Coordenador:** ____

## Timeline
| Data/Hora (UTC) | Evento |
|---|---|
| ____ | Primeiro sinal / alerta |
| ____ | Triagem concluída |
| ____ | Incidente declarado (SEV-__) |
| ____ | DPO notificado |
| ____ | Contenção iniciada |
| ____ | Contenção confirmada |
| ____ | Erradicação iniciada |
| ____ | Causa raiz identificada |
| ____ | Patch aplicado |
| ____ | Erradicação concluída |
| ____ | Recuperação iniciada |
| ____ | Validação de integridade OK |
| ____ | Operação normal restaurada |
| ____ | Incidente encerrado |
| ____ | Post-mortem realizado |
| ____ | ANPD notificada (se aplicável) |

## Indicadores de Comprometimento (IoCs)
| Tipo | Valor | Observação |
|---|---|---|
| ____ | ____ | ____ |

## Ações executadas

### Contenção
- [ ] Serviço isolado: ____
- [ ] Tokens JWT revogados: [ ] individual [ ] em massa
- [ ] Uplink cortado: ____
- [ ] Modo degradado ativado: [ ] Sim [ ] Não
- [ ] DPO notificado: ____

### Erradicação
- [ ] Análise forense concluída
- [ ] IoCs documentados
- [ ] Vulnerabilidade identificada: ____
- [ ] Patch aplicado: PR #____
- [ ] Modelo ML re-treinado: [ ] Sim [ ] N/A
- [ ] Chaves rotacionadas: ____

### Recuperação
- [ ] Backup restaurado: [ ] Sim [ ] N/A — Data do backup: ____
- [ ] Integridade validada
- [ ] Stakeholders comunicados
- [ ] ANPD notificada (se PII): [ ] Sim [ ] N/A — Data: ____
- [ ] Titulares notificados (se PII): [ ] Sim [ ] N/A — Qtd: ____

## Causa raiz
____

## Ações corretivas (do post-mortem)
| # | Ação | Responsável | Prazo | Status |
|---|---|---|---|---|
| 1 | ____ | ____ | ____ | [ ] |
| 2 | ____ | ____ | ____ | [ ] |

## Lições aprendidas
____
```

---

## 12. Referências

- **NIST SP 800-61 Rev. 2** — Computer Security Incident Handling Guide — <https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final>
- **ISO/IEC 27035:2023** — Information Security Incident Management
- **LGPD — Lei nº 13.709/2018** (arts. 46, 48, 49) — <https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm>
- **ANPD** — Comunicação de Incidentes de Segurança — <https://www.gov.br/anpd/pt-br>
- **GDPR Art. 33** — Notification of a personal data breach (referência para prazo de 72h)
- **OWASP Incident Response Project** — <https://owasp.org/www-project-incident-response/>
- Threat Model do 2F-AGRO — `cyber/threat-model.md`
- Arquitetura de Segurança do 2F-AGRO — `cyber/arquitetura-seguranca.md`
- Governança ISO 27001 + LGPD do 2F-AGRO — `cyber/governanca-iso-lgpd.md`
- International Charter Space and Major Disasters — <https://disasterscharter.org/>

---

*Documento gerado em junho/2026 como parte da entrega de Cybersecurity (Global Solution — FIAP 3ES).*
