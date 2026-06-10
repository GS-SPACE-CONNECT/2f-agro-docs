# 2F-AGRO — Threat Model

> **Global Solution · FIAP · 1º semestre 2026 · 3ES — Engenharia de Software**
>
> Matéria: **Cybersecurity** · Entrega: Threat Modeling (2 pts)
>
> Responsável: [@jota0802](https://github.com/jota0802)

---

## 1. Escopo

Este documento modela as ameaças à plataforma **2F-AGRO** — sistema que combina dados de satélite, sensores IoT de campo e IA para apoiar o pequeno agricultor familiar brasileiro. A análise cobre toda a superfície de ataque: desde a estação meteorológica autônoma na fazenda (edge) até a API Gateway e o app mobile no bolso do agricultor.

**Metodologia:** STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) aplicada ao fluxo de dados do 2F-AGRO.

---

## 2. Inventário de ativos críticos

| # | Ativo | Descrição | Classificação | Justificativa |
|---|---|---|---|---|
| A1 | **Geolocalização das propriedades** | Coordenadas GPS (lat/long) de cada propriedade cadastrada | **Dado pessoal (LGPD art. 5º, I) com proteção reforçada pelo risco contextual (perfilamento + risco à integridade física do titular — art. 12, §2º c/c art. 11)** | Revela localização exata do agricultor e da safra; exposição gera risco de roubo de safra, invasão de terra e violência física |
| A2 | **Dados financeiros do produtor** | Renda estimada, área cultivada, volume de produção, linhas de crédito (Pronaf) | **Confidencial** | Dados econômicos que, cruzados com localização, permitem perfilamento para fraude financeira ou extorsão |
| A3 | **Modelo ML treinado** | Modelos scikit-learn (Regressão Linear, Logística, Árvore, K-Means) serializados em produção | **Propriedade intelectual + integridade operacional** | Modelo comprometido gera previsões erradas → agricultor toma decisão equivocada → perda de safra (impacto direto na subsistência) |
| A4 | **Banco de alertas** | Registros de alertas agroclimáticos (seca, praga, geada, enchente, erosão) no PostgreSQL | **Integridade crítica** | Alertas adulterados podem suprimir avisos de seca iminente ou gerar falsos alarmes, ambos com impacto econômico real |
| A5 | **Estação edge IoT** | Raspberry Pi 4 com sensores (DHT22, BME280, anemômetro, pluviômetro) em fazenda remota do semiárido | **Infraestrutura crítica de campo** | Acesso físico em área rural sem vigilância; ponto de entrada na rede; alimenta o pipeline de dados inteiro |
| A6 | **API Gateway** | Ponto de entrada único (C# .NET 8) que roteia para todos os microsserviços internos | **Disponibilidade crítica** | Indisponibilidade em época de safra impede que agricultores recebam alertas; superfície exposta à internet |
| A7 | **Links de dados espaciais** | Integração HTTP/REST com APIs da NASA POWER (radiação solar, precipitação, evapotranspiração) e CPTEC-INPE (previsão meteorológica, alertas climáticos), consumidas pelo serviço SOA (Java/Spring Boot) | **Disponibilidade + Integridade** | Fonte primária de dados agroclimáticos que alimentam os alertas do 2F-AGRO; indisponibilidade impede emissão de alertas; adulteração gera previsões erradas → agricultor toma decisão equivocada |

---

## 3. Diagrama de fluxo de dados (DFD)

```
                         INTERNET
                            │
     ┌──────────────────────┼──────────────────────┐
     │                      │                      │
     ▼                      ▼                      ▼
┌─────────┐          ┌────────────┐          ┌──────────┐
│ 📱 App  │          │ ☕ SOA     │          │ 📡 APIs  │
│ Mobile  │          │ Java/      │          │ Externas │
│ (Expo)  │          │ Spring     │          │ NASA/    │
│         │          │ Boot       │          │ CPTEC    │
└────┬────┘          └─────┬──────┘          └────┬─────┘
     │ HTTPS/TLS           │ HTTPS/REST+SOAP      │ HTTPS
     │                     │                      │
═════╪═════════════════════╪══════════════════════╪══════════
     ▼                     ▼                      ▼
┌────────────────────────────────────────────────────────┐
│           🔐 API GATEWAY (C# .NET 8)                   │
│           JWT + Rate Limit + WAF                       │
│                                                        │
│   ┌─────────┐  ┌──────────┐  ┌──────────┐            │
│   │ Auth +  │  │ Alertas  │  │ ML Agro  │            │
│   │ Audit   │  │ Agro     │  │ (Python) │            │
│   └────┬────┘  └────┬─────┘  └────┬─────┘            │
│        └────────────┼─────────────┘                   │
│                     ▼                                 │
│        ┌─────────────────────────┐                    │
│        │ 🐘 PostgreSQL + PostGIS │                    │
│        │     (alertas + PII)     │                    │
│        └─────────────────────────┘                    │
│                                                        │
│   ┌───────────────────────┐  ┌──────────────────┐     │
│   │ Ingestão Sat/Edge     │  │ Visão Praga      │     │
│   │ (telemetria IoT)      │  │ (YOLOv8)         │     │
│   └───────────┬───────────┘  └──────────────────┘     │
└───────────────┼───────────────────────────────────────┘
                │
    ════════════╪═══════════════════════════
                │  LoRaWAN / 4G
                ▼
        ┌───────────────┐
        │ 📡 Estação    │  ← FAZENDA REMOTA (sem vigilância)
        │ Edge IoT      │
        │ (Rasp Pi 4)   │
        │ Sensores +    │
        │ Solar + LoRa  │
        └───────────────┘
```

### Fronteiras de confiança (trust boundaries)

| Fronteira | De → Para | Protocolo |
|---|---|---|
| **TB1** | App mobile → API Gateway | HTTPS/TLS 1.3 + JWT |
| **TB2** | Estação IoT → Serviço de Ingestão | LoRaWAN (AES-128) / 4G + mTLS |
| **TB3** | API Gateway → Microsserviços internos | gRPC/HTTP interno (rede privada) |
| **TB4** | Serviço ML → Modelo serializado | Filesystem local + hash SHA-256 |
| **TB5** | Serviço SOA → APIs espaciais (NASA POWER / CPTEC) | HTTPS/TLS 1.2+ (dependente do servidor externo) |

---

## 4. Vetores de ataque

### Vetor 1 — Interceptação de telemetria edge (Man-in-the-Middle)

| Aspecto | Detalhe |
|---|---|
| **ID** | VET-01 |
| **Ativo alvo** | A5 (Estação edge IoT), A1 (Geolocalização) |
| **Categoria STRIDE** | Information Disclosure + Spoofing |
| **Descrição do ataque** | Atacante posiciona-se entre a estação meteorológica (Raspberry Pi na fazenda) e o gateway LoRaWAN da cooperativa — ou entre a estação e a torre 4G — e intercepta os pacotes de telemetria em trânsito. Os pacotes contêm: leituras de sensores (temperatura, umidade, vento, chuva), timestamp e **coordenadas GPS da estação** (que correspondem à localização exata da propriedade). Na variante ativa, o atacante se passa pelo gateway LoRaWAN legítimo (rogue gateway) para capturar e reinjetar pacotes — daí a categoria Spoofing e a mitigação mTLS (M1.3). |
| **Pré-condição** | Acesso físico à área rural (baixa barreira no semiárido) ou comprometimento do gateway LoRaWAN da cooperativa |
| **Cenário narrativo** | Um concorrente desleal ou quadrilha de roubo de safra intercepta as transmissões LoRa de fazendas no entorno de Petrolina-PE. Com as coordenadas GPS interceptadas, consulta o NDVI público (Sentinel-2) daquela área e cruza com a umidade interceptada para identificar safras promissoras — e, com isso, propriedades lucrativas para roubo direcionado. |
| **Impacto** | **Alto** — exposição de dado pessoal com proteção reforçada (LGPD); risco de violência física contra o agricultor; perda patrimonial; quebra de confiança na plataforma |
| **Probabilidade** | **Média** — LoRaWAN opera em frequência aberta (915 MHz no Brasil); equipamento SDR (Software Defined Radio) custa ~R$ 200; área rural sem monitoramento |
| **Risco** | **Alto** (impacto alto × probabilidade média) |

**Mitigações propostas:**

| # | Controle | Camada |
|---|---|---|
| M1.1 | **Criptografia fim a fim (E2E):** além do AES-128 nativo do LoRaWAN, implementar camada adicional de criptografia aplicacional (AES-256-GCM) com chave pré-compartilhada por estação, para que, mesmo em caso de comprometimento do Network Server ou vazamento da AppSKey, os dados de telemetria permaneçam protegidos por uma camada independente | Dados em trânsito |
| M1.2 | **Anonimização de coordenadas em trânsito:** truncar coordenadas a 2 casas decimais (~1,1 km de imprecisão) nos pacotes de telemetria; coordenadas precisas ficam apenas no cadastro autenticado da API | Dados em trânsito |
| M1.3 | **mTLS no fallback 4G:** certificado de cliente instalado na estação + certificate pinning, impedindo MITM mesmo que o atacante controle um roteador intermediário | Autenticação |
| M1.4 | **Detecção de anomalia de rede:** se uma estação que transmite a cada 60s ficar silenciosa por >5 min ou apresentar jitter anormal, gerar alerta no SIEM (Grafana + Loki) | Monitoramento |
| M1.5 | **Proteção física:** gabinete com lacre anti-violação + sensor de abertura de tampa que envia alerta imediato via LoRa | Físico |

---

### Vetor 2 — Manipulação de telemetria / envenenamento do modelo ML

| Aspecto | Detalhe |
|---|---|
| **ID** | VET-02 |
| **Ativo alvo** | A3 (Modelo ML), A4 (Banco de alertas), A5 (Estação edge IoT) |
| **Categoria STRIDE** | Tampering + Elevation of Privilege |
| **Descrição do ataque** | Atacante compromete uma estação IoT (acesso físico ou exploração de serviço SSH exposto) e injeta dados falsos de telemetria — por exemplo, simula chuva abundante quando há seca real. Os dados fraudulentos alimentam o pipeline de ingestão e, ao longo do tempo, degradam os modelos ML (Regressão Logística, Árvore de Decisão, K-Means) via **data poisoning gradual**. O modelo passa a classificar propriedades de alto risco como baixo risco. |
| **Pré-condição** | Acesso físico à estação (rural, sem vigilância) ou credenciais SSH padrão não alteradas |
| **Cenário narrativo** | Um agente mal-intencionado (ou mesmo um funcionário insatisfeito da cooperativa) acessa a estação IoT na fazenda de "Seu João" no interior de Pernambuco. Substitui o script de coleta de sensores por um que envia temperatura fixa de 25°C e umidade de 80% — condições aparentemente ideais. O sistema deixa de emitir alertas de seca para aquela região. Duas semanas depois, Seu João perde 40% da safra de milho por não ter irrigado a tempo. |
| **Impacto** | **Crítico** — perda econômica direta para o agricultor (R$ 10-50k por safra perdida); degradação silenciosa do modelo afeta múltiplas propriedades; destruição de confiança na plataforma |
| **Probabilidade** | **Média** — requer acesso físico ou vulnerabilidade na estação, mas a estação fica em campo aberto sem vigilância |
| **Risco** | **Crítico** (impacto crítico × probabilidade média) |

**Mitigações propostas:**

| # | Controle | Camada |
|---|---|---|
| M2.1 | **Validação estatística de ingestão:** cada leitura é comparada contra a faixa esperada para a região/época (ex: temperatura no semiárido em dezembro: 28-42°C; se chegar 10°C, rejeitar e alertar). Regra de sanidade com janela deslizante de 24h | Aplicação |
| M2.2 | **Assinatura digital dos dados na estação:** cada pacote de telemetria é assinado com chave ECDSA armazenada em TPM externo (ex: Infineon SLB9670 via SPI) ou módulo de elemento seguro (Zymbit) acoplado ao Raspberry Pi; o serviço de ingestão rejeita pacotes com assinatura inválida | Integridade |
| M2.3 | **Quarentena de dados anômalos:** leituras fora de padrão não entram diretamente no pipeline de treinamento; ficam em tabela separada para revisão humana (agrônomo da cooperativa) | Processo |
| M2.4 | **Versionamento e rollback do modelo:** cada re-treino gera artefato versionado (MLflow ou similar) com hash SHA-256; se a acurácia do modelo degradar >5% em validação cruzada, rollback automático para versão anterior | ML Ops |
| M2.5 | **Correlação cruzada entre estações:** dados de uma estação são comparados com estações vizinhas (< 20 km); divergência >30% dispara alerta de possível adulteração | Detecção |
| M2.6 | **Hardening do Raspberry Pi:** desabilitar SSH por senha (apenas chave RSA-4096); firewall local (iptables) permitindo apenas porta LoRa e porta de coleta; atualizações automáticas de segurança via unattended-upgrades; boot seguro (U-Boot verificado) | Infraestrutura |

---

### Vetor 3 — Vazamento massivo de PII via API (violação LGPD)

| Aspecto | Detalhe |
|---|---|
| **ID** | VET-03 |
| **Ativo alvo** | A1 (Geolocalização), A2 (Dados financeiros), A6 (API Gateway) |
| **Categoria STRIDE** | Information Disclosure + Elevation of Privilege |
| **Descrição do ataque** | Atacante descobre um endpoint da API REST que lista propriedades cadastradas (`GET /api/propriedades`) sem autenticação adequada — seja por misconfiguration (endpoint esquecido sem `[Authorize]`), BOLA (Broken Object Level Authorization — IDOR) ou falha no filtro de campos. O resultado retorna nome completo, CPF, coordenadas GPS precisas, área cultivada e dados de produção de todos os agricultores da plataforma. |
| **Pré-condição** | Endpoint exposto à internet sem autenticação, ou token JWT com permissão excessiva, ou falha de BOLA que permite um agricultor consultar dados de outro |
| **Cenário narrativo** | Um pesquisador de segurança (ou atacante) executa fuzzing na API pública do 2F-AGRO e descobre que `GET /api/propriedades?page=1&size=1000` retorna a lista completa de agricultores com CPF e coordenadas. Publica os dados em fórum público. Resultado: violação massiva da LGPD (art. 42 — responsabilidade civil), multa da ANPD de até 2% do faturamento, e — pior — risco físico real para os agricultores cujas fazendas lucrativas agora são de conhecimento público. |
| **Impacto** | **Crítico** — violação LGPD massiva com multa administrativa; dano reputacional irreversível; risco de violência física (roubo de safra, invasão); ação civil coletiva |
| **Probabilidade** | **Alta** — BOLA é a vulnerabilidade #1 do OWASP API Security Top 10 (2023); APIs REST são superfície de ataque primária; misconfiguration é comum em sprints sob pressão |
| **Risco** | **Crítico** (impacto crítico × probabilidade alta) |

**Mitigações propostas:**

| # | Controle | Camada |
|---|---|---|
| M3.1 | **Autenticação obrigatória em todos os endpoints:** nenhum endpoint expõe dados sem JWT válido; middleware `[Authorize]` aplicado globalmente no API Gateway, com lista explícita de exceções (apenas `/health` e `/auth/login`) | Autenticação |
| M3.2 | **Autorização a nível de objeto (RBAC + owner check):** agricultor só acessa sua própria propriedade; cooperativa acessa apenas propriedades dos seus membros; EMATER acessa dados agregados (anonimizados). Implementação via `IAuthorizationHandler` customizado que valida `propriedadeId == claims.userId` | Autorização |
| M3.3 | **Filtragem de campos por role (field-level security):** o serializer JSON aplica filtro por role — `agricultor` nunca recebe CPF de outro agricultor; `cooperativa` recebe coordenadas truncadas (2 casas decimais); apenas `admin` vê dados completos, e com log de auditoria | Dados |
| M3.4 | **Rate limiting e throttling:** 100 requisições/minuto por IP; 20 req/min por token; requests de listagem paginadas com `size` máximo de 50; detecção de scraping (padrão de paginação sequencial completa) | Disponibilidade |
| M3.5 | **Logging e auditoria de acesso a PII:** toda consulta que retorna dados pessoais gera registro no log de auditoria (quem, quando, qual dado, IP de origem) — registro das operações conforme LGPD art. 37; retenção de 5 anos definida por política interna, alinhada ao prazo prescricional de reparação civil; alertas automáticos para consultas em volume anormal (>100 registros em 1h) | Monitoramento |
| M3.6 | **Testes automatizados de autorização (CI/CD):** suite de testes de integração que verifica: (a) endpoints sem `[Authorize]` falham no CI; (b) agricultor A não consegue acessar dados de agricultor B; (c) listagem completa retorna 403 para roles não-admin | Prevenção |
| M3.7 | **Política de minimização de dados (LGPD art. 6º, III):** endpoints retornam apenas os campos estritamente necessários para cada operação; CPF nunca trafega em responses de listagem; coordenadas são armazenadas criptografadas (AES-256) e só descriptografadas no backend sob demanda | Dados |

---

### Vetor 4 — DDoS contra a API Gateway em período crítico de safra

| Aspecto | Detalhe |
|---|---|
| **ID** | VET-04 |
| **Ativo alvo** | A6 (API Gateway), A7 (Links de dados espaciais) |
| **Categoria STRIDE** | Denial of Service |
| **Descrição do ataque** | Atacante executa ataque volumétrico (DDoS) ou de aplicação (HTTP flood, slowloris) contra a API Gateway do 2F-AGRO durante o período crítico de safra (plantio ou colheita), impedindo que agricultores acessem alertas agroclimáticos. A indisponibilidade da API também interrompe a ingestão de telemetria das estações IoT e a consulta às APIs espaciais (NASA POWER / CPTEC-INPE), paralisando o pipeline inteiro de alertas. |
| **Pré-condição** | Endpoint público exposto na internet (necessário para o app mobile funcionar); botnets de aluguel acessíveis na dark web (~US$ 50–200 para ataque de algumas horas) |
| **Cenário narrativo** | Durante a temporada de plantio no semiárido de Pernambuco (fevereiro–março), um concorrente ou agente mal-intencionado contrata um serviço de DDoS e lança 50 Gbps de tráfego contra api.2f-agro.com.br usando uma botnet de ~10.000 nós. A API Gateway fica indisponível por 6 horas. Nesse intervalo, o CPTEC emite um alerta de seca iminente para a região de Petrolina-PE, mas Seu João e outros 2.000 agricultores cadastrados não recebem a notificação. Sem o aviso, 40% dos agricultores não irrigam preventivamente e perdem parte significativa da safra de milho — prejuízo de R$ 500–5.000 por família. |
| **Impacto** | **Alto** — perda econômica coletiva para agricultores; interrupção do serviço em momento crítico da safra; agricultores sem alternativa digital de consulta; possível perda de confiança na plataforma |
| **Probabilidade** | **Média** — DDoS-as-a-Service é acessível e barato; API pública é alvo trivial; porém plataformas com CDN/WAF (Cloudflare) reduzem significativamente a eficácia do ataque |
| **Risco** | **Alto** (impacto alto × probabilidade média) |

**Mitigações propostas:**

| # | Controle | Camada |
|---|---|---|
| M4.1 | **CDN + WAF com mitigação DDoS integrada:** Cloudflare absorve tráfego volumétrico (L3/L4) e HTTP flood (L7) antes de chegar à origem; geo-blocking restringe tráfego ao Brasil; challenge automático (Turnstile) em endpoints públicos | Rede |
| M4.2 | **Auto-scaling horizontal:** Horizontal Pod Autoscaler (HPA) no Kubernetes escala os pods da API Gateway de 3 para até 10 conforme carga de CPU/memória, absorvendo picos legítimos que passem pela CDN | Infraestrutura |
| M4.3 | **Modo degradado read-only automático:** se a carga ultrapassa o limiar definido (CPU > 90% ou latência p99 > 2 s por 2 min), o sistema bloqueia automaticamente escritas e mantém apenas leituras — alertas existentes e dados cacheados continuam acessíveis ao agricultor | Aplicação |
| M4.4 | **Cache local no app mobile (offline-first):** o app armazena os últimos alertas e dados da propriedade em AsyncStorage; mesmo com a API indisponível, o agricultor consulta alertas recentes e histórico localmente | Mobile |
| M4.5 | **Alertas de disponibilidade no SIEM:** Grafana monitora latência p99 e taxa de erros 5xx; se latência > 2 s ou erros > 5% por 2 min, aciona alerta de possível DDoS e notifica o Coordenador de SI | Monitoramento |

---

## 5. Matriz de risco consolidada

| Vetor | Ativo(s) | STRIDE | Probabilidade | Impacto | Risco | Prioridade |
|---|---|---|---|---|---|---|
| VET-01 — MITM Telemetria | A5, A1 | I, S | Média | Alto | **Alto** | 2 |
| VET-02 — Envenenamento ML | A3, A4, A5 | T, E | Média | Crítico | **Crítico** | 1 |
| VET-03 — Vazamento PII | A1, A2, A6 | I, E | Alta | Crítico | **Crítico** | 1 |
| VET-04 — DDoS em período de safra | A6, A7 | D | Média | Alto | **Alto** | 2 |

**Escala de risco:**

```
              │ Baixo    │ Médio    │ Alto             │ Crítico
──────────────┼──────────┼──────────┼──────────────────┼──────────────
 Alta         │ Médio    │ Alto     │ Crítico          │ Crítico ←VET-03
 Média        │ Baixo    │ Médio    │ Alto ←VET-01,04  │ Crítico ←VET-02
 Baixa        │ Baixo    │ Baixo    │ Médio            │ Alto
──────────────┴──────────┴──────────┴──────────────────┴──────────────
 Probabilidade      Impacto →
```

**Nota — Repúdio (R):** a categoria não tem vetor dedicado porque é mitigada transversalmente pelos logs de auditoria imutáveis (M3.5), que registram quem acessou/alterou qual dado e quando, em todas as operações sensíveis.

---

## 6. Referências

- OWASP API Security Top 10 (2023) — https://owasp.org/API-Security/
- STRIDE Threat Model — Microsoft SDL
- LGPD — Lei nº 13.709/2018 (arts. 5º, 6º, 37, 42, 46)
- LoRaWAN Security Whitepaper — LoRa Alliance
- NIST SP 800-183 — Networks of Things (IoT Security)
- Spec técnico 2F-AGRO § 4.6 — Cybersecurity

---

*Documento gerado em junho/2026 como parte da entrega de Cybersecurity (Global Solution — FIAP 3ES).*
