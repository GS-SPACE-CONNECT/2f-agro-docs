# Product Backlog — 2F-AGRO

> **Global Solution · FIAP · 1º semestre 2026 · 3ES — Engenharia de Software**
> Matéria: **Testing, Compliance & Quality Assurance** · Entrega: Product Backlog (40 pts)

---

## Visão do Produto

**2F-AGRO** é uma plataforma que combina dados de satélite + sensores IoT de campo + IA para que **pequenos agricultores familiares** tomem decisões agronômicas com a mesma qualidade que grandes fazendas — de forma acessível, offline-first e sem precisar entender de tecnologia.

---

## Épicos

| # | Épico | Descrição |
|---|-------|-----------|
| **EP1** | Modelagem de Risco Agronômico (ML) | Pipeline de Machine Learning para prever risco de perda de safra e perfil agronômico |
| **EP2** | Coleta de Dados Multi-camada | Ingestão de dados de satélite (NDVI), sensores IoT da estação edge e câmera |
| **EP3** | App Mobile Acessível (Field App) | Aplicativo Android offline-first com interface acessível para o agricultor |
| **EP4** | Dashboard Cooperativa (Web) | Painel web com mapa de risco regional para gestores de cooperativas |
| **EP5** | Alertas e Recomendações Inteligentes | Motor de regras e notificações push/áudio com recomendações agronômicas diretas |
| **EP6** | Identificação de Pragas via IA (CV) | Módulo de visão computacional para detectar pragas e doenças por foto da folha |
| **EP7** | Segurança, LGPD e Compliance | Autenticação OAuth2/JWT, auditoria de acesso e consentimento LGPD |
| **EP8** | Visão Coletiva Regional | Compartilhamento opt-in de dados entre produtores da mesma cooperativa |

---

## Backlog Priorizado

> **Escala de Prioridade:** CRÍTICA > ALTA > MÉDIA > BAIXA
> **Story Points (Fibonacci):** 1, 2, 3, 5, 8, 13, 21

---

### SPRINT 1 — Base da Plataforma (Crítico / MVP)

---

#### US-01 — Cadastro de Propriedade Rural

**Épico:** EP3 — App Mobile Acessível

> **Como** agricultor familiar, **quero** cadastrar minha propriedade no app informando nome, localização GPS e culturas plantadas, **para** que o sistema possa monitorar minha lavoura especificamente.

**Critérios de Aceite:**
- [ ] Formulário com nome da propriedade, cultura(s) e localização (GPS automático)
- [ ] Localização capturada automaticamente via GPS do celular com confirmação visual no mapa
- [ ] Validação offline: dados salvos localmente e sincronizados quando houver sinal
- [ ] Limite de 5 culturas por propriedade na versão inicial
- [ ] Confirmação visual após cadastro ("Propriedade salva!")
- [ ] Interface com ícones grandes e texto em fonte ≥ 18sp

**Prioridade:** CRÍTICA | **Story Points:** 5

---

#### US-02 — Visualização de Risco Agronômico no App

**Épico:** EP1 — Modelagem de Risco Agronômico (ML)

> **Como** agricultor familiar, **quero** ver o nível de risco da minha lavoura com uma cor semafórica (🟢🟡🔴), **para** entender rapidamente se devo me preocupar sem precisar interpretar dados técnicos.

**Critérios de Aceite:**
- [ ] Tela principal exibe risco atual com cor: Verde (baixo), Amarelo (médio), Vermelho (alto)
- [ ] Exibição do motivo em texto simples (ex.: "Vegetação fraca + seca chegando")
- [ ] Atualização a cada 6h automaticamente quando online
- [ ] Última atualização exibida em formato legível ("Atualizado há 2h")
- [ ] Funciona offline mostrando última leitura com aviso de dados desatualizados

**Prioridade:** CRÍTICA | **Story Points:** 8

---

#### US-03 — Alerta de Risco em Áudio

**Épico:** EP5 — Alertas e Recomendações Inteligentes

> **Como** agricultor familiar com baixa alfabetização, **quero** receber alertas em formato de áudio em português brasileiro, **para** entender o risco mesmo sem saber ler.

**Critérios de Aceite:**
- [ ] Botão "Ouvir Alerta" visível em toda tela de risco
- [ ] Áudio gerado por TTS (expo-speech) em pt-BR com voz clara
- [ ] Texto do áudio em linguagem simples (ex.: "Atenção Seu João, sua roça corre risco. Irrigue hoje.")
- [ ] Funciona completamente offline (síntese local, sem internet)
- [ ] Áudio interrompível ao pressionar novamente o botão

**Prioridade:** CRÍTICA | **Story Points:** 5

---

#### US-04 — Ingestão de Dados de Satélite (NDVI)

**Épico:** EP2 — Coleta de Dados Multi-camada

> **Como** sistema 2F-AGRO, **quero** ingerir automaticamente os arquivos CSV com dados de satélite (NDVI, cobertura de nuvens, umidade do solo), **para** alimentar os modelos de ML com dados atualizados.

**Critérios de Aceite:**
- [ ] Job agendado (cron) processa CSV a cada 6h
- [ ] Validação de schema: rejeita CSV com colunas faltantes e registra erro
- [ ] Dados persistidos no PostgreSQL com timestamp de ingestão
- [ ] Log de cada ingestão com quantidade de registros processados/rejeitados
- [ ] Alerta interno em caso de falha consecutiva por mais de 12h

**Prioridade:** CRÍTICA | **Story Points:** 5

---

#### US-05 — Inferência de Risco por ML

**Épico:** EP1 — Modelagem de Risco Agronômico (ML)

> **Como** serviço de ML do 2F-AGRO, **quero** executar os modelos de Regressão Logística e Árvore de Decisão sobre os dados mais recentes de cada propriedade, **para** gerar o índice de risco atual de perda de safra.

**Critérios de Aceite:**
- [ ] API REST `POST /api/ml/inferir` recebe `propriedade_id` e retorna `{ risco: "alto|medio|baixo", probabilidade: 0.87, motivo: "..." }`
- [ ] Latência de resposta < 500ms para propriedade individual
- [ ] Modelo pré-carregado em memória (não recarrega por requisição)
- [ ] Log de inferência com inputs, output e timestamp para auditoria
- [ ] Fallback para regra simples (NDVI < 0.3 = alto) se modelo indisponível

**Prioridade:** CRÍTICA | **Story Points:** 8

---

#### US-06 — Autenticação Segura (OAuth2 / JWT)

**Épico:** EP7 — Segurança, LGPD e Compliance

> **Como** agricultor, **quero** fazer login no app com CPF e senha, **para** acessar apenas os dados da minha propriedade com segurança.

**Critérios de Aceite:**
- [ ] Login via CPF + senha; senha hasheada com bcrypt (salt ≥ 12)
- [ ] JWT emitido com expiração de 8h; refresh token válido por 30 dias
- [ ] Bloqueio automático após 5 tentativas incorretas por 15 min
- [ ] HTTPS obrigatório em todas as rotas de autenticação (TLS 1.3)
- [ ] Logout invalida token no servidor (blacklist)

**Prioridade:** CRÍTICA | **Story Points:** 8

---

### SPRINT 2 — Visão Computacional e Notificações

---

#### US-07 — Fotografia de Folha para Detecção de Praga

**Épico:** EP6 — Identificação de Pragas via IA (CV)

> **Como** produtor de tomate, **quero** fotografar uma folha suspeita com meu celular pra saber se é praga, **para** agir antes que a infestação se espalhe e comprometa toda a safra.

**Critérios de Aceite:**
- [ ] Botão "Fotografar Folha" abre câmera nativa do celular
- [ ] Imagem processada em < 3 segundos no dispositivo (modelo YOLO lite)
- [ ] Resultado mostra: nome da praga/doença, confiança (%) e ação recomendada
- [ ] Histórico de diagnósticos acessível offline
- [ ] Funciona com iluminação baixa e fundo não uniforme (campo real)

**Prioridade:** ALTA | **Story Points:** 13

---

#### US-08 — Notificação Push de Alerta de Risco

**Épico:** EP5 — Alertas e Recomendações Inteligentes

> **Como** agricultor, **quero** receber uma notificação push no meu celular quando o risco da minha lavoura subir para ALTO, **para** ser avisado mesmo sem abrir o app.

**Critérios de Aceite:**
- [ ] Push enviado em até 5 min após detecção de risco ALTO pelo ML
- [ ] Notificação contém: nível de risco, cultura afetada e ação principal recomendada
- [ ] Respeita horário configurável pelo agricultor (padrão: 6h-20h)
- [ ] Clique na notificação abre diretamente a tela de risco da propriedade
- [ ] Funciona em Android 7+ (Expo Notifications)

**Prioridade:** ALTA | **Story Points:** 5

---

#### US-09 — Recomendação de Ação Agronômica

**Épico:** EP5 — Alertas e Recomendações Inteligentes

> **Como** agricultor familiar sem formação técnica, **quero** receber uma recomendação clara e direta do que fazer quando o risco subir, **para** tomar a ação correta sem precisar chamar um agrônomo.

**Critérios de Aceite:**
- [ ] Recomendação exibida em até 2 frases curtas (ex.: "Irrigue a lavoura 3 hoje" / "Adie o plantio 4 dias")
- [ ] Recomendação gerada a partir das regras da Árvore de Decisão + contexto da cultura
- [ ] Botão "Fazer isso agora" registra ação tomada para acompanhamento
- [ ] Ação recomendada diferente por cultura (tomate ≠ feijão ≠ milho)
- [ ] Disponível em áudio (mesmo mecanismo de US-03)

**Prioridade:** ALTA | **Story Points:** 8

---

#### US-10 — Coleta de Dados da Estação Edge (Raspberry Pi)

**Épico:** EP2 — Coleta de Dados Multi-camada

> **Como** sistema, **quero** receber automaticamente os dados dos sensores da estação meteorológica instalada na fazenda (temperatura, umidade, chuva), **para** complementar os dados de satélite com leituras locais mais precisas.

**Critérios de Aceite:**
- [ ] Estação envia dados a cada 15 min via MQTT
- [ ] Serviço de ingestão persiste cada leitura com timestamp e `propriedade_id`
- [ ] Reconexão automática em caso de queda de sinal (backoff exponencial)
- [ ] Dados da estação mesclados com dados de satélite no pipeline ML
- [ ] Alerta interno se estação ficar > 2h sem enviar (possível falha de energia)

**Prioridade:** ALTA | **Story Points:** 8

---

#### US-11 — Histórico de Risco da Propriedade

**Épico:** EP3 — App Mobile Acessível

> **Como** agricultor, **quero** ver o histórico de risco da minha lavoura nos últimos 30 dias, **para** entender a evolução e identificar padrões (ex.: risco sobe toda vez que não chove por 5 dias).

**Critérios de Aceite:**
- [ ] Gráfico de linha simples mostrando nível de risco por dia (últimos 30 dias)
- [ ] Cores semafóricas nos pontos do gráfico
- [ ] Toque em um ponto exibe detalhes daquele dia (motivo + ação tomada se houver)
- [ ] Carrega offline com dados armazenados localmente
- [ ] Exportação do histórico como imagem (share para WhatsApp)

**Prioridade:** ALTA | **Story Points:** 5

---

### SPRINT 3 — Dashboard Cooperativa e Visão Regional

---

#### US-12 — Mapa de Risco Regional da Cooperativa

**Épico:** EP4 — Dashboard Cooperativa (Web)

> **Como** gestor de cooperativa, **quero** ver no painel web um mapa com todas as propriedades dos meus associados coloridas pelo nível de risco, **para** coordenar uma ação preventiva antes que a perda se espalhe pela região.

**Critérios de Aceite:**
- [ ] Mapa interativo (Leaflet) com um pino por propriedade
- [ ] Cor do pino = nível de risco (verde/amarelo/vermelho)
- [ ] Filtro por cultura (ex.: mostrar só propriedades de tomate)
- [ ] Clique no pino exibe detalhes da propriedade (nome, risco, última atualização)
- [ ] Dados atualizados em tempo real (WebSocket ou polling a cada 5 min)

**Prioridade:** MÉDIA | **Story Points:** 8

---

#### US-13 — Relatório Regional para Exportação

**Épico:** EP4 — Dashboard Cooperativa (Web)

> **Como** gestor de cooperativa, **quero** exportar um relatório CSV das propriedades com maior risco, **para** apresentar ao EMATER e solicitar suporte técnico coordenado.

**Critérios de Aceite:**
- [ ] Botão "Exportar CSV" disponível no dashboard
- [ ] CSV contém: nome da propriedade, produtor, cultura, nível de risco, probabilidade, data
- [ ] Filtro por nível de risco antes de exportar (ex.: só risco ALTO)
- [ ] Exportação em < 3s para cooperativas com até 200 propriedades
- [ ] Nome do arquivo inclui data (`relatorio-risco-2026-06-01.csv`)

**Prioridade:** MÉDIA | **Story Points:** 3

---

#### US-14 — Compartilhamento Opt-in de Dados entre Vizinhos

**Épico:** EP8 — Visão Coletiva Regional

> **Como** agricultor, **quero** optar por compartilhar os dados da minha propriedade com os vizinhos da cooperativa, **para** que todos se beneficiem de alertas regionais mais precisos.

**Critérios de Aceite:**
- [ ] Configuração de privacidade clara: "Compartilhar dados com cooperativa: SIM/NÃO"
- [ ] Padrão: NÃO compartilhar (opt-in explícito conforme LGPD)
- [ ] Dados compartilhados anonimizados no mapa (sem nome do produtor)
- [ ] Revogar consentimento a qualquer momento apaga dados do agregado
- [ ] Registro de consentimento com timestamp para auditoria LGPD

**Prioridade:** MÉDIA | **Story Points:** 5

---

#### US-15 — Alerta Coletivo para a Cooperativa

**Épico:** EP8 — Visão Coletiva Regional

> **Como** gestor de cooperativa, **quero** disparar um alerta coletivo para todos os produtores de uma região quando mais de 30% das propriedades estiverem em risco ALTO, **para** coordenar ação preventiva em escala.

**Critérios de Aceite:**
- [ ] Trigger automático quando ≥ 30% das propriedades da cooperativa atingem risco ALTO
- [ ] Alerta enviado por push + SMS (fallback) para todos os produtores afetados
- [ ] Mensagem inclui: região afetada, percentual em risco e recomendação coletiva
- [ ] Histórico de alertas coletivos disponível no dashboard
- [ ] Gestor pode suprimir alerta falso positivo com justificativa

**Prioridade:** MÉDIA | **Story Points:** 8

---

### SPRINT 4 — Segurança, Compliance e Polimento

---

#### US-16 — Consentimento e Transparência LGPD

**Épico:** EP7 — Segurança, LGPD e Compliance

> **Como** agricultor, **quero** ver quais dados meus estão sendo coletados e poder solicitar a exclusão dos meus dados, **para** ter controle sobre minha privacidade conforme a Lei Geral de Proteção de Dados.

**Critérios de Aceite:**
- [ ] Tela "Meus Dados" lista todos os dados coletados (localização, sensores, fotos)
- [ ] Botão "Excluir minha conta e dados" com confirmação em dois passos
- [ ] Exclusão efetiva em 72h; confirmação por e-mail ou SMS
- [ ] Termo de consentimento exibido no cadastro em linguagem simples (não jurídica)
- [ ] Log de auditoria de acesso aos dados do usuário disponível para o próprio usuário

**Prioridade:** ALTA | **Story Points:** 8

---

#### US-17 — Auditoria de Acesso a Dados

**Épico:** EP7 — Segurança, LGPD e Compliance

> **Como** administrador do sistema, **quero** ter um log completo de quem acessou quais dados de qual propriedade, **para** demonstrar conformidade LGPD em auditorias e investigar acessos suspeitos.

**Critérios de Aceite:**
- [ ] Todo acesso a `GET /api/propriedades/{id}` gera registro: `user_id`, `propriedade_id`, `timestamp`, `ip`
- [ ] Logs imutáveis (append-only) armazenados separadamente do banco principal
- [ ] Retenção de 5 anos conforme LGPD
- [ ] Interface de busca de logs por usuário, propriedade ou período
- [ ] Alerta automático para acesso fora do horário normal (22h-6h)

**Prioridade:** ALTA | **Story Points:** 5

---

#### US-18 — Modo Offline Completo do App

**Épico:** EP3 — App Mobile Acessível

> **Como** agricultor em região rural com cobertura de sinal instável, **quero** usar todas as funções básicas do app sem internet, **para** não ficar sem informação nos dias de sinal ruim.

**Critérios de Aceite:**
- [ ] Visualização de risco, histórico e recomendações disponíveis offline
- [ ] Dados sincronizados automaticamente quando sinal retorna (background sync)
- [ ] Banner informativo quando offline ("Sem sinal — dados de X horas atrás")
- [ ] Consumo de dados < 5 MB/mês em uso normal (imagens comprimidas, cache local)
- [ ] Sem degradação de performance offline vs. online

**Prioridade:** ALTA | **Story Points:** 8

---

#### US-19 — Interface Adaptada para Baixa Alfabetização

**Épico:** EP3 — App Mobile Acessível

> **Como** produtor com dificuldade de leitura, **quero** navegar pelo app usando principalmente ícones e cores, **para** entender as informações sem precisar ler textos longos.

**Critérios de Aceite:**
- [ ] Todo elemento interativo tem ícone + label de no máximo 2 palavras
- [ ] Fonte mínima de 18sp em todas as telas
- [ ] Paleta restrita a cores semafóricas (verde/amarelo/vermelho) para status de risco
- [ ] Nenhuma tela requer rolagem em celulares com tela ≥ 5" para as informações críticas
- [ ] Testado com 3 usuários rurais de baixa alfabetização (usability test)

**Prioridade:** ALTA | **Story Points:** 5

---

#### US-20 — Integração com EMATER via API Externa

**Épico:** EP8 — Visão Coletiva Regional

> **Como** extensionista da EMATER, **quero** receber automaticamente os relatórios de risco das regiões que acompanho, **para** planejar minhas visitas técnicas com base em dados reais.

**Critérios de Aceite:**
- [ ] Webhook configurável por região para envio de relatório diário às 7h
- [ ] Payload em formato JSON padronizado (compatível com sistemas EMATER/MAPA)
- [ ] Autenticação via API Key no header (`X-2FAGRO-KEY`)
- [ ] Retry automático em caso de falha (3 tentativas com backoff)
- [ ] Dashboard EMATER exibe status de entrega dos webhooks

**Prioridade:** MÉDIA | **Story Points:** 8

---

#### US-21 — Clustering de Propriedades por Perfil Agronômico

**Épico:** EP1 — Modelagem de Risco Agronômico (ML)

> **Como** gestor de cooperativa, **quero** ver meus produtores agrupados por perfil agronômico (ex.: zona seca, área chuvosa, encosta vulnerável), **para** direcionar orientações específicas para cada grupo.

**Critérios de Aceite:**
- [ ] K-Means (k=4) segmenta propriedades em 4 perfis com labels interpretáveis
- [ ] Cada propriedade exibe seu perfil no dashboard ("Zona Seca — Alto risco de seca/incêndio")
- [ ] Mapa filtrável por cluster
- [ ] Reclustering automático mensal para refletir mudança de estação
- [ ] Exportação de propriedades por cluster em CSV

**Prioridade:** MÉDIA | **Story Points:** 5

---

#### US-22 — Diagnóstico de Praga com Histórico e Tendência

**Épico:** EP6 — Identificação de Pragas via IA (CV)

> **Como** produtor, **quero** ver o histórico dos diagnósticos de praga da minha lavoura e se alguma praga está aumentando de frequência, **para** identificar infestações crônicas antes que se tornem críticas.

**Critérios de Aceite:**
- [ ] Tela de histórico lista: data, praga detectada, confiança e ação tomada
- [ ] Gráfico de frequência por praga no último mês
- [ ] Alerta automático se a mesma praga aparecer 3x em 7 dias
- [ ] Exportação do histórico como PDF (para agrônomo)
- [ ] Integração com US-09 (recomendação de ação específica para a praga)

**Prioridade:** BAIXA | **Story Points:** 5

---

#### US-23 — Painel de Administração do Sistema

**Épico:** EP7 — Segurança, LGPD e Compliance

> **Como** administrador técnico do 2F-AGRO, **quero** um painel para monitorar saúde dos serviços, volume de inferências ML e erros recentes, **para** garantir a disponibilidade da plataforma.

**Critérios de Aceite:**
- [ ] Painel exibe: uptime dos microsserviços, inferências/hora, erros nas últimas 24h
- [ ] Alertas de saúde quando inferências caem > 50% vs. média dos últimos 7 dias
- [ ] Acesso restrito a role `admin` (separado do acesso de cooperativa)
- [ ] Métricas retidas por 90 dias
- [ ] Exportação de métricas via API para Grafana/Prometheus

**Prioridade:** BAIXA | **Story Points:** 8

---

#### US-24 — Suporte Multilíngue (Português + Libras QR)

**Épico:** EP3 — App Mobile Acessível

> **Como** agricultor surdo, **quero** acessar vídeos de Libras via QR code nas telas de recomendação, **para** entender as orientações agronômicas de forma completa e inclusiva.

**Critérios de Aceite:**
- [ ] QR code em cada tela de recomendação aponta para vídeo em Libras hospedado localmente
- [ ] Vídeos disponíveis offline (cache em primeiro acesso com Wi-Fi)
- [ ] Tamanho total dos vídeos < 50 MB (vídeos curtos, 30s cada)
- [ ] Opção de desativar QR codes (usuário não surdo)

**Prioridade:** BAIXA | **Story Points:** 3

---

#### US-25 — Relatório de Impacto para ONGs e Financiadores

**Épico:** EP8 — Visão Coletiva Regional

> **Como** parceiro ONG (CONTAG / MST), **quero** receber um relatório mensal com métricas de impacto (propriedades salvas, alertas disparados, área monitorada), **para** demonstrar resultados aos financiadores e ao Pronaf.

**Critérios de Aceite:**
- [ ] Relatório gerado automaticamente no 1º dia de cada mês
- [ ] Contém: total de alertas, propriedades em risco evitado, área total monitorada (ha)
- [ ] Formato PDF com gráficos e dados agregados (sem dados pessoais — anonimizado)
- [ ] Enviado automaticamente por e-mail aos parceiros cadastrados
- [ ] Link de acesso permanente ao relatório (URL pública, não autenticada)

**Prioridade:** BAIXA | **Story Points:** 5

---

## Resumo do Backlog

| Sprint | Histórias | Story Points | Foco |
|--------|-----------|--------------|------|
| Sprint 1 | US-01 a US-06 | 39 pts | MVP: Cadastro, Risco ML, Auth, Ingestão |
| Sprint 2 | US-07 a US-11 | 39 pts | Visão Computacional, Notificações, Histórico |
| Sprint 3 | US-12 a US-15 | 24 pts | Dashboard Cooperativa, Visão Regional |
| Sprint 4 | US-16 a US-25 | 52 pts | Segurança, LGPD, Polimento, Integrações |
| **Total** | **25 histórias** | **154 pts** | |

---

## Critérios de Priorização (MoSCoW)

| Must Have (MVP) | Should Have | Could Have |
|---|---|---|
| US-01, US-02, US-03, US-04, US-05, US-06 | US-07, US-08, US-09, US-10, US-11, US-16, US-17, US-18, US-19 | US-12, US-13, US-14, US-15, US-20, US-21, US-22, US-23, US-24, US-25 |

---

*Backlog elaborado pelo time 2F-AGRO · QA Owner: @roji-menez · FIAP 3ES · GS 2026.1*
