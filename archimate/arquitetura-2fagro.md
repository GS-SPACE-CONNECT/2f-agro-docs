# Arquitetura 2F-AGRO — ArchiMate

> **Global Solution · FIAP · 1º semestre 2026 · 3ES — Engenharia de Software**
> Matéria: **Testing, Compliance & Quality Assurance** · Entregável: ArchiMate (40 pts)
> Ferramenta: Archi 5.9 · Notação: ArchiMate 3.2

---

## Visão Geral da Solução

**2F-AGRO** é uma plataforma de monitoramento agronômico que combina:
- **Dados de satélite** (NDVI, umidade do solo, cobertura de nuvens)
- **Sensores IoT** na fazenda (estação meteorológica Raspberry Pi solar)
- **Inteligência Artificial** (ML + Visão Computacional)

…para que **pequenos agricultores familiares** recebam alertas de risco de safra e recomendações agronômicas no celular, sem precisar entender de tecnologia.

---

## Camada 1 — Visão de Motivação

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  VISÃO DE MOTIVAÇÃO                                                         │
├──────────────────────┬──────────────────────┬───────────────────────────────┤
│  STAKEHOLDERS        │  DRIVERS             │  OBJETIVOS / METAS            │
│                      │                      │                               │
│  🧑‍🌾 Agricultor      │  ⚠️ Deserto           │  🎯 Reduzir perda de         │
│     Familiar         │    tecnológico na     │     safra em ≥ 30%            │
│                      │    agricultura        │                               │
│  👥 Cooperativa /    │    familiar           │  🎯 Democratizar              │
│     Assentamento     │                      │     tecnologia espacial        │
│                      │  ⚠️ Mudanças          │     para pequenos             │
│  🏛️ EMATER / MAPA   │    climáticas e       │     produtores                │
│                      │    risco crescente    │                               │
│  🏦 Banco / Pronaf   │    de safra           │  🎯 Aumentar resiliência      │
│                      │                      │     climática regional         │
│  🌱 ONG (CONTAG,    │  ⚠️ Acesso limitado   │                               │
│     MST)             │    a crédito rural    │  🎯 Escalar apoio             │
│                      │    por risco          │     extensionista             │
│  👩‍💻 Time 2F-AGRO    │    desconhecido       │     (EMATER)                 │
└──────────────────────┴──────────────────────┴───────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  REQUISITOS                          │  RESTRIÇÕES                         │
│                                      │                                     │
│  ✅ Funcionar offline (< 5MB/mês)    │  ⛔ Dataset simulado (CSV FIAP)    │
│  ✅ Interface acessível (áudio +     │  ⛔ Equipe de 5 pessoas             │
│     ícones, fonte ≥ 18sp)            │  ⛔ Prazo: 09/06/2026              │
│  ✅ Android 7+ (celular básico)      │  ⛔ Sem infraestrutura de           │
│  ✅ Conformidade LGPD                │     campo real (simulação)          │
│  ✅ Resposta de risco em < 500ms     │                                     │
│  ✅ Disponibilidade ≥ 99% (cloud)   │                                     │
└──────────────────────────────────────┴─────────────────────────────────────┘

ODS ATENDIDOS: ODS 1 · ODS 2 (primário) · ODS 8 · ODS 13 · ODS 15
```

### Relações de Motivação

| De | Relação | Para |
|---|---|---|
| Agricultor Familiar | influencia | Meta: Reduzir perda de safra |
| Driver: Deserto tecnológico | realiza | Meta: Democratizar tecnologia |
| Driver: Mudanças climáticas | realiza | Meta: Aumentar resiliência |
| Meta: Reduzir perda | refina | Req: Funcionar offline |
| Meta: Democratizar | refina | Req: Interface acessível |
| Req: LGPD | limita | Dados compartilhados |

---

## Camada 2 — Camada de Negócio

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ATORES & PAPÉIS                                                            │
├───────────────────┬───────────────────┬────────────────────────────────────┤
│  ATOR             │  PAPEL            │  RESPONSABILIDADE                  │
│                   │                   │                                    │
│  Agricultor       │  Produtor         │  Usa app, tira fotos, executa      │
│  Familiar         │  Rural            │  recomendações, compartilha dados  │
│                   │                   │                                    │
│  Gestor de        │  Gestor           │  Monitora cooperativa no           │
│  Cooperativa      │  Regional         │  dashboard, dispara alertas        │
│                   │                   │  coletivos, exporta relatórios     │
│                   │                   │                                    │
│  Extensionista    │  Agrônomo         │  Recebe relatórios via webhook,    │
│  EMATER           │                   │  planeja visitas técnicas          │
│                   │                   │                                    │
│  Admin            │  Administrador    │  Monitora saúde do sistema,        │
│  2F-AGRO          │  de Sistema       │  gerencia usuários e compliance    │
└───────────────────┴───────────────────┴────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  FLUXO DE PROCESSOS DE NEGÓCIO                                              │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │  Monitorar  │───▶│  Detectar   │───▶│ Recomendar  │───▶│ Acompanhar  │ │
│  │ Propriedade │    │Risco Agron. │    │   Ação      │    │ Resultado   │ │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐    ┌─────────────┐                                        │
│  │  Identificar│───▶│ Recomendar  │                                        │
│  │ Praga (foto)│    │  Tratamento │                                        │
│  └─────────────┘    └─────────────┘                                        │
│                                                                             │
│  ┌─────────────────────────────────────────┐                               │
│  │  Coordenar Ação Coletiva (cooperativa)  │                               │
│  └─────────────────────────────────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  SERVIÇOS DE NEGÓCIO            │  OBJETOS DE NEGÓCIO                     │
│                                 │                                          │
│  📋 Diagnóstico Agronômico      │  🌾 Lavoura / Propriedade Rural         │
│  🔔 Alerta Preventivo de Risco  │  ⚠️ Alerta de Risco de Safra           │
│  💡 Recomendação Inteligente    │  📄 Recomendação Agronômica             │
│  🗺️ Visão Coletiva Regional     │  📊 Relatório de Safra / Impacto       │
│  🔍 Identificação de Pragas     │  📷 Diagnóstico de Praga (foto)         │
└─────────────────────────────────┴──────────────────────────────────────────┘
```

---

## Camada 3 — Camada de Aplicação

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  COMPONENTES DE APLICAÇÃO                                                   │
│                                                                             │
│  ┌──────────────────┐    ┌──────────────────────────────────────────────┐  │
│  │  📱 App Mobile   │    │           🌐 Web Dashboard                   │  │
│  │  React Native    │    │           React + Leaflet                    │  │
│  │  + Expo          │    │                                              │  │
│  │                  │    │  • Mapa de risco regional                    │  │
│  │  • Offline-first │    │  • Gestão da cooperativa                     │  │
│  │  • Android 7+    │    │  • Exportação CSV/PDF                        │  │
│  │  • Áudio TTS     │    │  • Webhooks EMATER                           │  │
│  │  • Câmera YOLO   │    │                                              │  │
│  └────────┬─────────┘    └──────────────────┬───────────────────────────┘  │
│           │                                  │                              │
│           └──────────────┬───────────────────┘                              │
│                          ▼                                                   │
│           ┌──────────────────────────────┐                                  │
│           │   🔐 API Gateway             │                                  │
│           │   C# .NET 8 Web API          │                                  │
│           │   • Roteamento               │                                  │
│           │   • Rate Limiting            │                                  │
│           │   • OAuth2 / JWT             │                                  │
│           └────┬──────┬──────┬──────┬───┘                                  │
│                │      │      │      │                                        │
│       ┌────────▼─┐ ┌──▼───┐ ┌▼────┐ ┌▼──────────┐                        │
│       │ Serviço  │ │Serv. │ │Serv.│ │  Serviço  │                        │
│       │ ML Agro  │ │Visão │ │Alert│ │  Auth +   │                        │
│       │(Python + │ │Pragas│ │ as  │ │  LGPD     │                        │
│       │scikit-l) │ │(YOLO)│ │(C#) │ │  (C#)     │                        │
│       └────────┬─┘ └──────┘ └─────┘ └───────────┘                        │
│                │                                                             │
│       ┌────────▼──────────────────────────┐                                │
│       │   Serviço Ingestão                │                                │
│       │   (Python + pandas)               │                                │
│       │   • CSV Satélite (NDVI)           │                                │
│       │   • MQTT Edge (sensores)          │                                │
│       └───────────────────────────────────┘                                │
│                                                                             │
│  DADOS                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Dados NDVI   │  │ Dados Sensor │  │ Imagem Folha │  │ Perfil Agro- │  │
│  │ (Satélite)   │  │ (Edge IoT)   │  │ (Praga CV)   │  │ nômico (ML)  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Serviços de Aplicação

| Serviço | Endpoint | Tecnologia | Função |
|---------|----------|-----------|--------|
| REST API Propriedades | `GET/POST /api/propriedades` | C# .NET 8 | CRUD de propriedades rurais |
| Inferência ML | `POST /api/ml/inferir` | Python FastAPI | Retorna risco em < 500ms |
| Detecção de Pragas CV | `POST /api/visao/praga` | Python + YOLOv8 | Classifica praga em foto |
| Push Notifications | Expo Push | Expo + FCM | Alerta push no celular |
| OAuth2 / JWT Auth | `POST /api/auth/token` | C# Identity | Autenticação segura |
| Webhook EMATER | `POST {url_emater}` | C# + HttpClient | Relatório diário automático |

---

## Camada 4 — Camada de Tecnologia

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  INFRAESTRUTURA CLOUD (Azure / AWS)                                         │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  NUVEM                                                             │    │
│  │                                                                    │    │
│  │  ┌─────────────────┐   ┌──────────────────┐   ┌────────────────┐  │    │
│  │  │  Kubernetes /   │   │  PostgreSQL       │   │  RabbitMQ      │  │    │
│  │  │  Docker         │   │  + PostGIS        │   │  (mensageria)  │  │    │
│  │  │  (microsserviços│   │  (geodados +      │   │               │  │    │
│  │  │  containerizados│   │   séries temporais│   │  → notif push  │  │    │
│  │  └─────────────────┘   └──────────────────┘   └────────────────┘  │    │
│  │                                                                    │    │
│  │  ┌──────────────────────────────┐   ┌────────────────────────┐    │    │
│  │  │  S3 / Blob Storage           │   │  HTTPS / TLS 1.3       │    │    │
│  │  │  (imagens folhas + modelos   │   │  (toda comunicação      │    │    │
│  │  │   ML pkl + pesos YOLO .pt)   │   │   criptografada)        │    │    │
│  │  └──────────────────────────────┘   └────────────────────────┘    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  DISPOSITIVOS DE CAMPO                                                      │
│                                                                             │
│  ┌──────────────────────────┐    ┌─────────────────────────────────────┐   │
│  │  🛰️ SATÉLITE SIMULADO    │    │  📡 ESTAÇÃO EDGE (Fazenda)          │   │
│  │  (CSV FIAP)              │    │  Raspberry Pi Zero 2W               │   │
│  │                          │    │  + Painel Solar 5W                  │   │
│  │  • NDVI                  │    │  + Raspbian Custom OS               │   │
│  │  • Umidade do solo       │    │                                     │   │
│  │  • Cobertura de nuvens   │    │  Sensores:                          │   │
│  │  • Temperatura           │    │  • DHT22 (temp + umidade)           │   │
│  │  • Chuva prevista        │    │  • Pluviômetro                      │   │
│  │                          │    │  • Anemômetro                       │   │
│  │  → via CSV ingestão 6h   │    │                                     │   │
│  └──────────────────────────┘    │  → MQTT → Nuvem (15 min)            │   │
│                                  └─────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────┐                              │
│  │  📱 CELULAR DO AGRICULTOR (Android 7+)   │                              │
│  │  • App React Native (Expo)               │                              │
│  │  • Câmera (YOLO lite on-device)          │                              │
│  │  • Armazenamento local (SQLite)          │                              │
│  │  • Conexão: 4G / 3G / WiFi              │                              │
│  └──────────────────────────────────────────┘                              │
│                                                                             │
│  REDES DE COMUNICAÇÃO                                                       │
│  ┌─────────────────────────┐   ┌─────────────────────────────────────────┐ │
│  │  Internet (4G/5G/WiFi)  │   │  LoRa / WiFi Local (Edge → Roteador)   │ │
│  │  App ↔ Cloud            │   │  Estação ↔ Internet                    │ │
│  └─────────────────────────┘   └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Stack Tecnológica Completa

| Camada | Tecnologia | Versão | Uso |
|--------|-----------|--------|-----|
| **Mobile** | React Native + Expo | RN 0.73 | App do agricultor |
| **Web** | React + Leaflet | React 18 | Dashboard cooperativa |
| **API Gateway** | C# .NET 8 Web API | .NET 8 | Roteamento + Auth |
| **Alertas** | C# .NET 8 + EF Core | .NET 8 | Regras agronômicas |
| **ML** | Python + FastAPI + scikit-learn | Python 3.10 | Inferência de risco |
| **CV (Pragas)** | Python + YOLOv8 + OpenCV | YOLO v8 | Detecção em folhas |
| **Ingestão** | Python + pandas | Python 3.10 | CSV + MQTT → BD |
| **Auth** | C# Identity + JWT | .NET 8 | OAuth2 + LGPD |
| **Banco** | PostgreSQL + PostGIS | PG 15 | Dados geoespaciais |
| **Fila** | RabbitMQ | 3.12 | Push notifications |
| **Storage** | S3 / Azure Blob | — | Imagens + modelos |
| **Edge OS** | Raspbian Custom | Debian 12 base | Estação IoT |
| **Container** | Docker + Kubernetes | K8s 1.29 | Deploy cloud |
| **Rede** | MQTT + HTTPS/TLS 1.3 | — | Comunicação segura |

---

## Diagrama de Fluxo Integrado (Macro)

```
🧑‍🌾 Agricultor                                           👥 Cooperativa
     │                                                         │
     ▼                                                         ▼
 📱 App Mobile ←────────────────── 🌐 Web Dashboard
     │                                     │
     └──────────────────┬──────────────────┘
                        ▼
              🔐 API Gateway (C# .NET 8)
                        │
          ┌─────────────┼─────────────────┐
          ▼             ▼                 ▼
    🤖 ML Agro    👁️ Visão Pragas    🔔 Alertas
    (Python)      (Python+YOLO)     (C# .NET)
          │                              │
          ▼                              ▼
    📥 Ingestão ←────── RabbitMQ ←─── Notif
          │
    ┌─────┴───────┐
    ▼             ▼
 🛰️ Satélite   📡 Estação Edge
  (CSV FIAP)   (Raspberry Pi)
                   │
             DHT22 + Chuva +
             Anemômetro + Solar
```

---

## Relações de Arquitetura (Resumo)

### Motivation → Business
| Stakeholder | Serve | Processo |
|---|---|---|
| Agricultor Familiar | usa | Monitorar Propriedade → Detectar Risco |
| Gestor de Cooperativa | usa | Coordenar Ação Coletiva |
| EMATER | recebe | Relatório Regional (webhook) |

### Business → Application
| Processo de Negócio | Realizado por | Componente |
|---|---|---|
| Detectar Risco Agronômico | Serviço ML Agro | FastAPI + scikit-learn |
| Identificar Praga por Foto | Serviço Visão Pragas | YOLOv8 + OpenCV |
| Recomendar Ação | Serviço Alertas | C# .NET 8 |
| Monitorar Propriedade | App Mobile | React Native |

### Application → Technology
| Componente | Depende de | Infraestrutura |
|---|---|---|
| Serviço ML Agro | PostgreSQL + S3 | Cloud (modelos .pkl) |
| Serviço Visão Pragas | S3 + GPU | Pesos YOLO (.pt) |
| Serviço Notificação | RabbitMQ | Fila de mensagens |
| App Mobile | SQLite local | Armazenamento offline |
| Estação Edge | Raspbian + MQTT | IoT na fazenda |

---

*Arquitetura elaborada no Archi 5.9 · Notação ArchiMate 3.2 · QA Owner: @roji-menez · FIAP 3ES · GS 2026.1*
