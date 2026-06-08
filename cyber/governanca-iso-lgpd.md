# 2F-AGRO — Governança ISO 27001 + LGPD

> **Global Solution · FIAP · 1º semestre 2026 · 3ES — Engenharia de Software**
>
> Matéria: **Cybersecurity** · Entrega: Governança e Compliance (2 pts)
>
> Responsável: [@jota0802](https://github.com/jota0802)

---

## Sumário

1. [Escopo](#1-escopo)
2. [SGSI — Sistema de Gestão de Segurança da Informação](#2-sgsi--sistema-de-gestão-de-segurança-da-informação)
3. [Matriz de riscos](#3-matriz-de-riscos)
4. [Plano de auditoria semestral](#4-plano-de-auditoria-semestral)
5. [LGPD — Conformidade com a Lei Geral de Proteção de Dados](#5-lgpd--conformidade-com-a-lei-geral-de-proteção-de-dados)
6. [RIPD — Relatório de Impacto à Proteção de Dados Pessoais](#6-ripd--relatório-de-impacto-à-proteção-de-dados-pessoais)
7. [Referências](#7-referências)

---

## 1. Escopo

Este documento estabelece a governança de segurança da informação e privacidade de dados da plataforma **2F-AGRO**, cobrindo dois eixos:

- **ISO 27001** — Sistema de Gestão de Segurança da Informação (SGSI), com política formal, matriz de riscos e plano de auditoria semestral.
- **LGPD (Lei nº 13.709/2018)** — Política de proteção de dados pessoais, consentimento, direitos do titular, DPO designado e Relatório de Impacto (RIPD).

**Sistemas cobertos:** app mobile (React Native/Expo), API Gateway (C# .NET 8), microsserviços SOA (Java/Spring Boot), pipeline ML (Python/scikit-learn), estação edge IoT (Raspberry Pi 4) e todas as bases de dados associadas (PostgreSQL + PostGIS).

**Público-alvo:** equipe de desenvolvimento, cooperativas parceiras, auditor externo e Autoridade Nacional de Proteção de Dados (ANPD).

---

## 2. SGSI — Sistema de Gestão de Segurança da Informação

### 2.1 Declaração da política de segurança da informação

A 2F-AGRO compromete-se a proteger a **confidencialidade**, **integridade** e **disponibilidade** das informações sob sua custódia — em especial os dados pessoais e produtivos dos agricultores familiares que utilizam a plataforma. Esta política aplica-se a todos os colaboradores, prestadores de serviço e sistemas que processam, armazenam ou transmitem informações da organização.

**Princípios:**

1. **Necessidade de conhecer (need-to-know):** acesso a informações concedido apenas quando indispensável à função.
2. **Privilégio mínimo:** cada identidade (humana ou de serviço) recebe o menor conjunto de permissões necessário.
3. **Defesa em profundidade:** múltiplas camadas de controle (rede, aplicação, dados, físico).
4. **Melhoria contínua:** ciclo PDCA (Plan-Do-Check-Act) aplicado semestralmente.

### 2.2 Escopo do SGSI

| Dimensão | Cobertura |
|---|---|
| **Organizacional** | Equipe 2F-AGRO (5 membros), cooperativas parceiras, EMATER regional |
| **Tecnológica** | App mobile, API Gateway, microsserviços, banco de dados, pipeline ML, estação edge IoT |
| **Geográfica** | Infraestrutura cloud (Azure/AWS), estações de campo no semiárido nordestino |
| **Dados** | Dados pessoais (nome, CPF, localização), dados produtivos (safra, área), telemetria IoT, modelos ML |

### 2.3 Papéis e responsabilidades

| Papel | Responsável | Atribuições |
|---|---|---|
| **Gestor de Segurança da Informação** | jota ([@jota0802](https://github.com/jota0802)) | Definição de políticas, acompanhamento de riscos, coordenação de auditorias, resposta a incidentes |
| **DPO (Encarregado de Dados)** | roji ([@roji-menez](https://github.com/roji-menez)) | Canal com a ANPD, orientação sobre LGPD, revisão de consentimentos, acompanhamento de solicitações de titulares |
| **Desenvolvedor Seguro** | brunão + ruan | Implementação de controles técnicos (autenticação, criptografia, logging), revisão de código com foco em segurança |
| **Responsável ML/IoT** | lucca ([@lucksza](https://github.com/lucksza)) | Integridade dos modelos ML, hardening das estações edge, monitoramento de telemetria |
| **Alta Direção (para fins do SGSI)** | Equipe completa (5 membros) | Aprovação de políticas, alocação de recursos, análise crítica semestral |

### 2.4 Classificação da informação

| Nível | Definição | Exemplos no 2F-AGRO | Controles mínimos |
|---|---|---|---|
| **Público** | Informação de acesso livre | Dados climáticos agregados por região, alertas públicos, documentação da API pública | Sem restrição de acesso |
| **Interno** | Uso restrito à equipe e parceiros | Código-fonte, documentação técnica, logs de sistema, métricas de uso | Autenticação obrigatória; repositórios privados |
| **Confidencial** | Acesso restrito por função | Dados pessoais (nome, CPF, e-mail), dados produtivos individuais, credenciais de serviço | Criptografia em repouso (AES-256) e em trânsito (TLS 1.3); RBAC; log de acesso |
| **Proteção reforçada** | Dado pessoal (LGPD art. 5º, I) com proteção reforçada pelo risco contextual (perfilamento + risco à integridade física — art. 12, §2º c/c art. 11) | Geolocalização precisa das propriedades, dados financeiros (renda, crédito Pronaf) | Tudo de "Confidencial" + consentimento qualificado; acesso auditado; anonimização em logs |

### 2.5 Política de controle de acesso

| Controle | Implementação |
|---|---|
| **Autenticação** | JWT com expiração de 1h (access token) + 7d (refresh token); MFA obrigatório para cooperativas e admins (TOTP via app autenticador) |
| **Autorização (RBAC)** | 4 roles: `agricultor`, `cooperativa`, `emater`, `admin`. Cada role tem escopo de acesso definido por `IAuthorizationHandler` customizado |
| **Owner check** | Agricultor só acessa seus próprios dados (`propriedadeId == claims.userId`); cooperativa acessa apenas membros vinculados |
| **Segregação de ambientes** | Ambientes de desenvolvimento, homologação e produção com credenciais separadas e sem acesso cruzado |
| **Gestão de credenciais** | Secrets armazenados em Vault (HashiCorp); rotação trimestral; credenciais padrão proibidas; push protection ativo no GitHub |
| **Política de senhas** | Mínimo 8 caracteres (acessível ao público rural — NIST SP 800-63B recomenda comprimento sobre complexidade); verificação contra lista de senhas vazadas (HaveIBeenPwned, k-anonymity); bloqueio após 5 tentativas (lockout 15 min); sem exigência de caracteres especiais |

### 2.6 Controles técnicos de segurança (Anexo A — ISO/IEC 27001:2013)

| Domínio ISO 27001 | Controle | Status |
|---|---|---|
| A.5 — Políticas de SI | Política documentada neste SGSI | Implementado |
| A.6 — Organização da SI | Papéis definidos (§ 2.3); gestor e DPO designados | Implementado |
| A.7 — Segurança em RH | Termos de confidencialidade para membros da equipe | Planejado |
| A.8 — Gestão de ativos | Inventário de 7 ativos críticos (ver Threat Model); classificação de dados (§ 2.4) | Implementado |
| A.9 — Controle de acesso | RBAC + JWT + MFA + owner check (§ 2.5) | Implementado |
| A.10 — Criptografia | AES-256 em repouso (pgcrypto em colunas sensíveis + LUKS full-disk); TLS 1.3 em trânsito; bcrypt para senhas; AES-256-GCM para telemetria edge | Implementado |
| A.12 — Segurança operacional | Logging centralizado (Grafana + Loki); monitoramento 24/7; backups diários criptografados | Implementado |
| A.13 — Segurança de comunicações | TLS 1.3 em todas as comunicações; mTLS no fallback 4G das estações; certificate pinning no app mobile | Implementado |
| A.14 — Aquisição e desenvolvimento | Code review obrigatório (PR + 1 reviewer); SAST no CI; testes de autorização automatizados | Implementado |
| A.16 — Gestão de incidentes | Plano de resposta a incidentes documentado (ver `cyber/resposta-incidentes.md`) | Documentado |
| A.18 — Conformidade | LGPD (seção 5 deste documento); RIPD (seção 6) | Implementado |

---

## 3. Matriz de riscos

### 3.1 Metodologia

A avaliação de riscos segue a abordagem qualitativa da ISO 27005, com classificação em duas dimensões:

- **Probabilidade** — chance de o evento ocorrer (Muito Baixa, Baixa, Média, Alta, Muito Alta)
- **Impacto** — consequência para a organização se o evento ocorrer (Insignificante, Baixo, Moderado, Alto, Crítico)

O **nível de risco** é o cruzamento das duas dimensões na matriz abaixo.

### 3.2 Matriz probabilidade × impacto

```
Probabilidade ↓    Insignificante   Baixo       Moderado     Alto         Crítico
─────────────────────────────────────────────────────────────────────────────────
Muito Alta          Baixo           Médio       Alto         Crítico      Crítico
Alta                Baixo           Médio       Alto         Crítico      Crítico
Média               Baixo           Baixo       Médio        Alto         Crítico
Baixa               Baixo           Baixo       Baixo        Médio        Alto
Muito Baixa         Baixo           Baixo       Baixo        Baixo        Médio
```

**Legenda de cores/prioridade:**

| Nível | Ação requerida |
|---|---|
| **Crítico** | Tratamento imediato; escalar para alta direção; prazo máximo 7 dias |
| **Alto** | Plano de mitigação em até 30 dias; acompanhamento semanal |
| **Médio** | Plano de mitigação em até 90 dias; acompanhamento mensal |
| **Baixo** | Aceitar ou monitorar; revisão na próxima auditoria semestral |

### 3.3 Inventário de riscos do 2F-AGRO

| ID | Risco | Ativo(s) afetado(s) | Probabilidade | Impacto | Nível | Tratamento | Responsável |
|---|---|---|---|---|---|---|---|
| R01 | Interceptação de telemetria edge via MITM (LoRaWAN/4G) | Estação IoT, Geolocalização | Média | Alto | **Alto** | Mitigar — criptografia E2E (AES-256-GCM) + mTLS no 4G + anonimização de coordenadas em trânsito | lucca / jota |
| R02 | Envenenamento de dados de telemetria → degradação do modelo ML | Modelo ML, Banco de alertas, Estação IoT | Média | Crítico | **Crítico** | Mitigar — validação estatística de ingestão + assinatura digital ECDSA + quarentena de dados anômalos + versionamento de modelo | lucca |
| R03 | Vazamento massivo de PII via API (BOLA/IDOR) | Geolocalização, Dados financeiros, API Gateway | Alta | Crítico | **Crítico** | Mitigar — RBAC + owner check + filtragem de campos por role + rate limiting + testes automatizados de autorização | brunão / ruan |
| R04 | Acesso físico não autorizado à estação edge | Estação IoT | Alta | Moderado | **Alto** | Mitigar — gabinete com lacre anti-violação + sensor de abertura + hardening do Raspberry Pi (SSH por chave RSA-4096, firewall iptables, boot seguro) | lucca / jota |
| R05 | Indisponibilidade da API em período crítico de safra (inclui DDoS — VET-04) | API Gateway, Links de dados espaciais (A7) | Baixa | Crítico | **Alto** | Mitigar — CDN/WAF com mitigação DDoS (Cloudflare) + rate limiting + auto-scaling horizontal + modo degradado read-only + cache offline no app + réplica de banco | brunão / ruan |
| R06 | Vazamento de credenciais/secrets no repositório | Todos os sistemas | Baixa | Alto | **Médio** | Mitigar — push protection no GitHub + secrets em Vault + rotação trimestral + CI com detecção de secrets | jota |
| R07 | Perda de dados por falha de backup | Banco de dados, Banco de alertas | Muito Baixa | Crítico | **Médio** | Mitigar — backups diários criptografados + teste de restore mensal + retenção de 90 dias + réplica geográfica | brunão |
| R08 | Phishing contra membros da equipe/cooperativa | Controle de acesso | Média | Moderado | **Médio** | Mitigar — MFA obrigatório para cooperativas e admins + treinamento semestral de conscientização | roji |
| R09 | Falha no consentimento/conformidade LGPD | Dados pessoais dos agricultores | Baixa | Alto | **Médio** | Mitigar — fluxo de consentimento explícito no app + endpoint DELETE /me + RIPD documentado + DPO designado | roji (DPO) |
| R10 | Injeção SQL/NoSQL nos microsserviços | Banco de dados, API Gateway | Baixa | Alto | **Médio** | Mitigar — ORM parametrizado (EF Core / JPA) + SAST no CI + code review obrigatório + WAF com regras OWASP | brunão / ruan |

### 3.4 Mapa de calor consolidado

```
                    Insignificante   Baixo     Moderado     Alto         Crítico
                   ──────────────────────────────────────────────────────────────
Muito Alta         │              │           │            │            │         │
Alta               │              │           │   R04      │            │  R03    │
Média              │              │           │   R08      │   R01      │  R02    │
Baixa              │              │           │            │ R06,R09,R10│  R05    │
Muito Baixa        │              │           │            │            │  R07    │
                   ──────────────────────────────────────────────────────────────
```

### 3.5 Risco residual

Após implementação dos controles listados, espera-se que todos os riscos **Críticos** (R02, R03) sejam reduzidos para **Alto** ou inferior, e que os riscos **Altos** (R01, R04, R05) sejam reduzidos para **Médio**. A reavaliação ocorre na auditoria semestral (seção 4).

---

## 4. Plano de auditoria semestral

### 4.1 Objetivo

Verificar a eficácia dos controles de segurança, a conformidade com a ISO 27001 e a aderência à LGPD, em ciclo PDCA contínuo.

### 4.2 Frequência e calendário

| Auditoria | Período | Tipo | Auditor |
|---|---|---|---|
| **1ª Auditoria** | Janeiro/2027 | Interna | Gestor de SI (jota) + DPO (roji) |
| **2ª Auditoria** | Julho/2027 | Interna + revisão externa (cooperativa parceira) | Gestor de SI + auditor convidado |
| Ciclo subsequente | A cada 6 meses | Alternância interna/externa | Conforme designação |

### 4.3 Escopo da auditoria

| Área | Itens verificados |
|---|---|
| **Controle de acesso** | Revisão de contas ativas vs. membros atuais; verificação de MFA; revisão de roles RBAC; contas de serviço com permissões excessivas |
| **Gestão de vulnerabilidades** | Dependabot ativo e PRs processados; resultado do último SAST/DAST; CVEs conhecidos não corrigidos |
| **Criptografia** | Certificados TLS válidos e não expirados; chaves de criptografia rotacionadas; conformidade com política de senhas |
| **Logs e monitoramento** | Logs de acesso a PII íntegros e retidos por 5 anos; alertas de SIEM configurados e testados; tempo de resposta a incidentes simulados |
| **Backup e recuperação** | Último backup verificado com sucesso; teste de restore executado; RPO/RTO dentro do alvo (RPO ≤ 24h, RTO ≤ 4h) |
| **Conformidade LGPD** | Consentimentos válidos e atualizados; solicitações de titulares atendidas no prazo (15 dias); RIPD revisado; DPO acessível |
| **Segurança física (edge)** | Estado dos lacres das estações IoT; registros do sensor de abertura; firmware atualizado |
| **Gestão de riscos** | Matriz de riscos reavaliada; riscos residuais dentro do apetite; novos riscos identificados |

### 4.4 Processo de auditoria

```
┌──────────────────────────────────────────────────────────────────┐
│  FASE 1 — PLANEJAMENTO (semana 1)                                │
│  • Definir escopo e critérios da auditoria                       │
│  • Notificar responsáveis dos controles                          │
│  • Preparar checklist baseado nos itens da § 4.3                 │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  FASE 2 — EXECUÇÃO (semanas 2-3)                                 │
│  • Coleta de evidências (screenshots, logs, configs)             │
│  • Entrevistas com responsáveis                                  │
│  • Testes técnicos (scan de vulnerabilidades, teste de restore)  │
│  • Verificação de conformidade LGPD (consentimentos, prazos)     │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  FASE 3 — RELATÓRIO (semana 4)                                   │
│  • Classificar achados: Conformidade / Não-conformidade menor /  │
│    Não-conformidade maior / Oportunidade de melhoria             │
│  • Atribuir responsável e prazo para cada não-conformidade       │
│  • Apresentar à alta direção (análise crítica)                   │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  FASE 4 — ACOMPANHAMENTO (contínuo)                              │
│  • Verificar implementação das ações corretivas                  │
│  • Atualizar matriz de riscos com novos achados                  │
│  • Alimentar backlog com melhorias identificadas                 │
│  • Documentar lições aprendidas para próximo ciclo               │
└──────────────────────────────────────────────────────────────────┘
```

### 4.5 Indicadores-chave de segurança (KPIs)

| KPI | Meta | Frequência de medição |
|---|---|---|
| Tempo médio de correção de vulnerabilidade crítica | ≤ 7 dias | Contínuo |
| Cobertura de MFA em contas privilegiadas | 100% | Mensal |
| Taxa de sucesso de testes de restore de backup | ≥ 95% | Mensal |
| Solicitações LGPD de titulares atendidas no prazo | 100% | Contínuo |
| Não-conformidades maiores abertas | 0 | Semestral (pós-auditoria) |
| Incidentes de segurança com vazamento de PII | 0 | Contínuo |

---

## 5. LGPD — Conformidade com a Lei Geral de Proteção de Dados

### 5.1 Bases legais para tratamento de dados

| Dado pessoal | Base legal (LGPD art. 7º) | Justificativa |
|---|---|---|
| Nome completo, CPF, e-mail, telefone | **Consentimento (art. 7º, I)** | Cadastro voluntário no app; consentimento granular coletado na tela de primeiro acesso |
| Geolocalização precisa da propriedade | **Consentimento qualificado (art. 7º, I)** | Dado pessoal com proteção reforçada pelo risco contextual; consentimento destacado e específico com explicação clara da finalidade |
| Dados produtivos (safra, área cultivada) | **Execução de contrato (art. 7º, V)** | Necessário para prestação do serviço de alertas agroclimáticos |
| Dados financeiros (renda, crédito Pronaf) | **Consentimento (art. 7º, I)** | Dado opcional; coletado apenas se o agricultor optar por funcionalidades financeiras |
| Telemetria IoT (temperatura, umidade, vento) | **Legítimo interesse (art. 7º, IX)** | Dados técnicos sem identificação direta; necessários para funcionamento do sistema; teste de proporcionalidade documentado no RIPD |

### 5.2 Consentimento explícito

O 2F-AGRO implementa consentimento conforme arts. 7º, 8º e 11 da LGPD:

**Fluxo no app mobile:**

```
┌────────────────────────────────────────────────────┐
│            BEM-VINDO AO 2F-AGRO                    │
│                                                    │
│  Para funcionar, o app precisa coletar alguns      │
│  dados seus. Você escolhe o que compartilhar:      │
│                                                    │
│  ☑ Nome e CPF (obrigatório para cadastro)          │
│  ☑ Localização da propriedade (obrigatório para    │
│    alertas — dado protegido por criptografia        │
│    e anonimização em logs)                         │
│  ☐ Dados financeiros (opcional — usado para        │
│    sugestões de crédito Pronaf)                    │
│                                                    │
│  📄 Ler Política de Privacidade completa           │
│                                                    │
│  [ACEITAR E CONTINUAR]    [RECUSAR]                │
│                                                    │
│  Você pode alterar essas escolhas a qualquer       │
│  momento em Configurações > Privacidade.           │
└────────────────────────────────────────────────────┘
```

**Requisitos técnicos do consentimento:**

| Requisito LGPD | Implementação |
|---|---|
| **Livre** | Botão "Recusar" disponível sem consequência punitiva; funcionalidades básicas acessíveis sem dados opcionais |
| **Informado** | Texto claro em linguagem acessível (nível de leitura compatível com público-alvo); link para política completa |
| **Inequívoco** | Ação afirmativa (toque no botão); checkboxes não pré-marcados para dados opcionais |
| **Específico** | Consentimento granular por categoria de dado (cadastro, localização, financeiro) |
| **Destacado para dados sensíveis (art. 11)** | Localização possui consentimento em tela separada com explicação detalhada da finalidade e dos riscos |
| **Revogável (art. 8º, § 5º)** | Tela de Configurações > Privacidade permite revogar qualquer consentimento; revogação processada em até 24h |

**Armazenamento do consentimento:**

```
Tabela: consentimentos
─────────────────────────────────────────────────
| id (UUID)       | titular_id  | categoria        | concedido | data_consentimento   | ip_origem    | versao_termos |
| a1b2c3...       | usr_001     | dados_cadastrais | true      | 2026-06-01T10:30:00Z | 189.x.x.x   | v2.1          |
| d4e5f6...       | usr_001     | geolocalizacao   | true      | 2026-06-01T10:30:15Z | 189.x.x.x   | v2.1          |
| g7h8i9...       | usr_001     | dados_financeiros| false     | 2026-06-01T10:30:15Z | 189.x.x.x   | v2.1          |
```

### 5.3 Direito ao esquecimento (DELETE /me)

O 2F-AGRO implementa o direito à eliminação de dados pessoais conforme art. 18, VI da LGPD.

**Endpoint:**

```
DELETE /api/v1/me
Authorization: Bearer <jwt_token>
```

**Fluxo de exclusão:**

```
Agricultor solicita          Backend processa           Resultado
exclusão no app              a requisição
─────────────                ──────────────             ─────────

  [Excluir                   1. Valida JWT
   minha conta] ──────────►  2. Marca conta como
                                "exclusão_pendente"
  Tela de        ◄──────────  3. Envia e-mail de
  confirmação                   confirmação (72h
  (dupla                        para cancelar)
  verificação)
                             4. Após 72h sem
  "Seus dados   ◄──────────     cancelamento:
   foram                        a) Anonimiza PII
   removidos"                      (nome→"REMOVIDO",
                                    CPF→hash, coords
                                    →null)
                                b) Remove registros de
                                   consentimento
                                c) Mantém dados
                                   agregados/anônimos
                                   (LGPD art. 16, IV)
                                d) Log de auditoria
                                   da exclusão (sem
                                   PII do titular)
```

**Dados retidos após exclusão (permitido pela LGPD):**

| Dado | Justificativa legal | Forma de retenção |
|---|---|---|
| Dados agregados de telemetria (sem vínculo ao titular) | Art. 16, IV — pesquisa e estatística | Anonimizados irreversivelmente |
| Log de auditoria da exclusão (sem PII) | Art. 16, I — cumprimento de obrigação legal | Apenas registro: "titular X solicitou exclusão em data Y" (sem nome/CPF) |
| Dados necessários para exercício de direitos em processo | Art. 16, II — exercício regular de direitos | Retidos apenas se houver litígio em curso |

### 5.4 DPO designado

| Campo | Detalhe |
|---|---|
| **Nome** | Roji Menez |
| **GitHub** | [@roji-menez](https://github.com/roji-menez) |
| **Papel** | Encarregado pelo Tratamento de Dados Pessoais (DPO) conforme art. 41 da LGPD |
| **Canal de contato** | dpo@2f-agro.com.br (a ser ativado em produção) |
| **Responsabilidades** | (i) Aceitar reclamações e comunicações dos titulares e da ANPD; (ii) Orientar a equipe sobre práticas de proteção de dados; (iii) Acompanhar solicitações de titulares e garantir atendimento no prazo legal (15 dias); (iv) Manter o RIPD atualizado |
| **Publicidade** | Nome e contato do DPO divulgados na Política de Privacidade do app e no site institucional, conforme art. 41, § 1º |

### 5.5 Minimização de coleta

Conforme art. 6º, III da LGPD (princípio da necessidade), o 2F-AGRO coleta apenas os dados estritamente necessários para cada funcionalidade:

| Funcionalidade | Dados coletados | Dados **não** coletados (minimização) |
|---|---|---|
| Cadastro básico | Nome, CPF, e-mail, telefone, senha (hash) | Endereço residencial, RG, data de nascimento, foto |
| Alertas agroclimáticos | Coordenadas da propriedade, cultura plantada | Área total da fazenda (usa apenas raio de alerta) |
| Detecção de pragas | Foto da folha (processada e descartada) | Metadados EXIF da foto (removidos antes do envio) |
| Dashboard financeiro (opcional) | Renda estimada, área cultivada | Dados bancários, número de conta, extrato |
| Telemetria IoT | Temperatura, umidade, vento, chuva | Identificação do operador da estação |

**Regras técnicas de minimização:**

1. **Endpoints retornam apenas campos necessários** — respostas de listagem (`GET /api/propriedades`) nunca incluem CPF; coordenadas são truncadas a 2 casas decimais para roles que não sejam owner.
2. **Retenção limitada** — fotos enviadas para detecção de pragas são processadas pelo modelo YOLOv8 e descartadas em 24h; apenas o resultado (tipo de praga + confiança) é retido.
3. **Metadados removidos** — o app mobile remove dados EXIF (localização embutida, modelo do dispositivo) das fotos antes do upload.
4. **Logs sem PII** — logs de aplicação usam IDs internos (UUID), nunca nome ou CPF do agricultor; coordenadas em logs são truncadas a 2 casas decimais (~1,1 km de imprecisão).

---

## 6. RIPD — Relatório de Impacto à Proteção de Dados Pessoais

*Conforme art. 38 da LGPD e recomendações da ANPD.*

### 6.1 Identificação

| Campo | Valor |
|---|---|
| **Controlador** | 2F-AGRO (projeto acadêmico FIAP — GS 2026) |
| **Encarregado (DPO)** | Roji Menez ([@roji-menez](https://github.com/roji-menez)) |
| **Data de elaboração** | Junho/2026 |
| **Versão** | 1.0 |
| **Próxima revisão** | Janeiro/2027 (auditoria semestral) |

### 6.2 Descrição do tratamento

| Aspecto | Detalhe |
|---|---|
| **Natureza** | Coleta, armazenamento, processamento e compartilhamento de dados pessoais de agricultores familiares brasileiros via plataforma digital (app mobile + backend cloud + estações IoT de campo) |
| **Finalidade** | Fornecer alertas agroclimáticos personalizados, detecção de pragas por visão computacional e sugestões de manejo baseadas em dados de satélite e sensores locais — visando reduzir perdas de safra e aumentar a resiliência do pequeno produtor |
| **Escopo** | Dados pessoais de agricultores cadastrados (estimativa inicial: 500-5.000 titulares); dados coletados via app mobile e estações IoT; processamento em cloud (Azure/AWS, região Brasil) |
| **Contexto** | Público-alvo de baixa literacia digital; regiões remotas com conectividade limitada; dados de localização especialmente sensíveis por revelar propriedades agrícolas vulneráveis a roubo |

### 6.3 Inventário de dados pessoais tratados

| Categoria | Dados específicos | Base legal | Finalidade | Compartilhamento |
|---|---|---|---|---|
| **Identificação** | Nome completo, CPF, e-mail, telefone | Consentimento | Cadastro e comunicação | Cooperativa do titular (se vinculado) |
| **Localização** | Coordenadas GPS da propriedade (lat/long) | Consentimento explícito (sensível) | Alertas regionalizados | Cooperativa (truncadas a 2 casas); EMATER (agregadas) |
| **Produtivos** | Cultura plantada, área cultivada, safra estimada | Execução de contrato | Personalização de alertas e sugestões | Cooperativa (se vinculado); EMATER (agregados) |
| **Financeiros** (opcional) | Renda estimada, linhas de crédito Pronaf | Consentimento | Sugestões de crédito | Não compartilhado |
| **Imagens** | Fotos de folhas para detecção de pragas | Consentimento | Diagnóstico por IA | Não compartilhado; descartadas em 24h |
| **Telemetria** | Temperatura, umidade, vento, chuva | Legítimo interesse | Funcionamento do sistema de alertas | EMATER (agregados); NASA/CPTEC (apenas recebe, não envia) |

### 6.4 Análise de necessidade e proporcionalidade

| Princípio (LGPD art. 6º) | Análise |
|---|---|
| **Finalidade (I)** | Cada dado coletado tem finalidade específica documentada na tabela acima; não há coleta para fins indeterminados |
| **Adequação (II)** | Os dados coletados são compatíveis com a finalidade — coordenadas GPS são essenciais para alertas regionalizados; dados produtivos permitem personalizar sugestões |
| **Necessidade (III)** | Minimização aplicada: não coletamos endereço residencial, RG, data de nascimento, dados bancários; fotos são descartadas após processamento; metadados EXIF são removidos |
| **Livre acesso (IV)** | Titular consulta seus dados a qualquer momento no app (tela "Meus Dados"); exportação em JSON disponível |
| **Qualidade dos dados (V)** | Titular pode corrigir dados via app (tela "Editar Perfil"); dados de telemetria validados estatisticamente |
| **Transparência (VI)** | Política de privacidade em linguagem acessível; tela de consentimento granular; DPO publicado |
| **Segurança (VII)** | Controles técnicos documentados no SGSI (seção 2); criptografia em repouso e trânsito; RBAC |
| **Prevenção (VIII)** | Threat model com 3 vetores de ataque e mitigações; plano de resposta a incidentes; auditorias semestrais |
| **Não discriminação (IX)** | Sistema não utiliza dados para profiling discriminatório; modelo ML avalia risco agroclimático, não perfil pessoal |
| **Responsabilização (X)** | Este RIPD documenta as medidas adotadas; DPO designado; logs de auditoria; conformidade verificada semestralmente |

### 6.5 Riscos aos titulares e medidas de mitigação

| Risco ao titular | Probabilidade | Gravidade | Medida de mitigação |
|---|---|---|---|
| **Exposição de localização** → roubo de safra, invasão de propriedade, violência | Média | Crítica | Criptografia AES-256 em repouso; truncamento de coordenadas em trânsito e em logs; RBAC com owner check; consentimento explícito com aviso de risco |
| **Vazamento de CPF/dados financeiros** → fraude, clonagem de identidade | Baixa | Alta | Autenticação obrigatória em todos os endpoints; field-level security (CPF nunca em listagens); criptografia em repouso |
| **Perfilamento indevido** → discriminação em acesso a crédito | Muito Baixa | Alta | Dados financeiros são opcionais; modelo ML não gera score de crédito; dados não compartilhados com instituições financeiras |
| **Uso secundário não autorizado** → marketing, venda de dados | Muito Baixa | Alta | Dados não são compartilhados com terceiros comerciais; cooperativa e EMATER recebem apenas dados agregados/truncados; política contratual de não-repasse |
| **Impossibilidade de exercer direitos** → titular sem acesso digital | Baixa | Moderada | Direitos exercíveis via app e via cooperativa (atendimento presencial); DPO acessível por e-mail e telefone; prazo de 15 dias para atendimento |

### 6.6 Parecer do DPO

> Com base na análise conduzida neste RIPD, o tratamento de dados pessoais pela plataforma 2F-AGRO é **necessário e proporcional** às finalidades declaradas. Os riscos identificados são **mitigáveis** pelos controles técnicos e organizacionais implementados. Recomenda-se:
>
> 1. Manter o ciclo de auditorias semestrais para reavaliação contínua.
> 2. Monitorar o volume de solicitações de titulares e ajustar recursos conforme demanda.
> 3. Revisar este RIPD sempre que houver alteração significativa no tratamento de dados.
>
> **Parecer: FAVORÁVEL ao tratamento, condicionado à manutenção dos controles descritos.**

---

## 7. Referências

- **ISO/IEC 27001:2013** — Sistemas de gestão de segurança da informação — Requisitos (Anexo A com 14 domínios de controle)
- **ISO/IEC 27005:2022** — Gestão de riscos de segurança da informação
- **LGPD — Lei nº 13.709/2018** (arts. 5º, 6º, 7º, 8º, 11, 16, 18, 37, 38, 41, 42, 46)
- **ANPD** — Guia Orientativo para Definições dos Agentes de Tratamento e do Encarregado (2021)
- **ANPD** — Guia de Boas Práticas para Relatório de Impacto à Proteção de Dados Pessoais (2023)
- **OWASP API Security Top 10 (2023)** — https://owasp.org/API-Security/
- **NIST SP 800-37 Rev. 2** — Risk Management Framework for Information Systems
- Threat Model do 2F-AGRO — `cyber/threat-model.md`
- Spec técnico 2F-AGRO § 4.6 — Cybersecurity

---

*Documento gerado em junho/2026 como parte da entrega de Cybersecurity (Global Solution — FIAP 3ES).*
