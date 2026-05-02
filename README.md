# HealthGO
Teste Técnico - Arquiteto de Software Sênior | A HealthGo é uma healthtech brasileira especializada em diagnóstico respiratório — testes de hidrogênio, metano e sulfeto no ar expirado. Desenvolvemos tecnologia que permite a profissionais de saúde realizar exames mais precisos, confiáveis e acessíveis em toda a América Latina.

# ADR — Decisões Arquiteturais Críticas
### Plataforma ECG — Monitoramento Cardíaco em Escala
> Versão 1.3 | Última revisão: 01/05/2026 | Responsável: Fábio Muniz (Arquiteto de Software)

---

## ADR-001 · Pseudonimização no Edge (a mais crítica)

**Status:** Aceito ✅

**Contexto:**
Dispositivos de ECG capturam dados de saúde vinculados a `patient_id`
real. A LGPD (Art. 11) classifica dados de saúde como sensíveis —
qualquer transmissão identificável exige base legal e consentimento
explícito. O dado precisa chegar à nuvem sem ser identificável.

**Decisão:**
Aplicar HMAC-SHA256 com chave local (age-encrypted no Key Vault do
edge) antes de qualquer transmissão. A nuvem recebe apenas o token
pseudônimo. A reversão é possível somente com a chave local.
Rotação trimestral automática da chave HMAC.

**Alternativas descartadas:**

| Alternativa | Por que descartada |
|---|---|
| Pseudonimizar na nuvem | Dado identificável em trânsito → violação LGPD Art. 11 |
| Criptografia simétrica no edge | Reversível por qualquer detentor da chave → não é pseudonimização real |
| Anonimização irreversível | Impossibilita auditoria e direito de acesso do titular |

**Riscos assumidos:**
- Chave HMAC comprometida por acesso físico ao hardware → tokens
  reversíveis. **Mitigação:** rotação trimestral + alerta de acesso
  físico não autorizado.
- Perda da chave → dado inacessível (crypto-shredding não
  intencional). **Mitigação:** backup da chave no Key Vault HSM
  central com acesso restrito.

**Hipóteses que sustentam esta decisão:** H-04 (LGPD), H-05 (sistema assistivo)

---

## ADR-002 · Kafka como único barramento de eventos

**Status:** Aceito ✅

**Contexto:**
Três sistemas diferentes (Flink, ML Pipeline, Databricks) precisam
consumir o mesmo sinal de ECG de forma independente, com velocidades
diferentes e sem acoplamento entre si. Um sexto mês do projeto, o
módulo ML foi adicionado — sem nenhuma alteração no produtor.

**Decisão:**
Kafka (Confluent Cloud gerenciado) com modelo de consumer groups.
Cada consumidor tem seu offset independente. Retenção de 7 dias
permite reprocessamento e adição de novos consumidores retroativos.

**Alternativas descartadas:**

| Alternativa | Por que descartada |
|---|---|
| RabbitMQ | Mensagem deletada após consumo — impossível múltiplos consumidores independentes |
| Azure Service Bus | Sem consumer groups nativos; custo por mensagem inviável a 3M pontos/s |
| REST síncrono entre serviços | Acoplamento temporal — se ML cair, ingestor para |
| Azure Event Hubs | Tecnicamente viável, mas Kafka API-compatible sem lock-in adicional |

**Riscos assumidos:**
- Complexidade operacional alta para squad de 2. **Mitigação:**
  Confluent Cloud gerenciado (SLA 99.95%) — sem operação de broker.
- Kafka como SPOF lógico. **Mitigação:** replicação 3x,
  `min.insync.replicas=2`, circuit breaker no ingestor.

**Hipóteses que sustentam esta decisão:** H-01 (volume), H-03 (squad de 2)

---

## ADR-003 · ML em produção com validação assíncrona

**Status:** Aceito com restrições ⚠️

**Contexto:**
Detecção de arritmia (VT, AFIB) precisa ser em tempo real — latência
de validação humana é inaceitável. Mas um falso negativo em VT pode
omitir uma arritmia potencialmente fatal. Qualquer deploy sem
evidência clínica é risco regulatório e ético.

**Decisão:**
Modelo ONNX (CNN + MLP, treinado no Databricks) roda no AKS com GPU T4.
Thresholds conservadores por classe (VT: 0.65, AFIB: 0.70).
Todo alerta exibe disclaimer clínico. Nenhum deploy sem tag
`cardiology_signoff` preenchida no MLflow Model Registry
(bloqueio técnico via código — não processo manual).
Monitoramento contínuo de concept drift (Evidently AI via
Databricks Workflows), alerta automático se drift > 15% em 24h.

**Alternativas descartadas:**

| Alternativa | Por que descartada |
|---|---|
| Validação humana antes do alerta | Latência > 5min para VT — clinicamente inaceitável |
| Modelo embarcado no edge | Custo de GPU no edge por clínica inviável; atualizações complexas |
| Regras determinísticas (limiar RR) | Alta taxa de falso positivo — rejeitado pelos cardiologistas consultados |
| Somente batch (sem tempo real) | Não detecta eventos agudos — fora do requisito clínico |

**Riscos assumidos:**
- Falso negativo em VT. **Mitigação:** threshold conservador (0.65),
  retreat obrigatório semestral ou sob drift > 15%.
- Degradação silenciosa do modelo. **Mitigação:** Evidently AI +
  alerta automático em Grafana.
- Equipamentos não vistos no treino. **Mitigação:** validação de
  SNR mínimo (>20dB) antes da inferência.

**Hipóteses que sustentam esta decisão:** H-06 (qualidade ML), H-05 (sistema assistivo)

---

## ADR-004 · .NET Aspire + Dapr para serviços de aplicação

**Status:** Aceito ✅

**Contexto:**
A arquitetura tem serviços em .NET (Ingestor, Auth, Alert, Dashboard)
e não-.NET (Python edge, Python ML, Java Flink, Databricks).
Era necessário uma camada de orquestração que servisse ao
desenvolvimento local sem sacrificar portabilidade em produção
e sem ignorar os serviços não-.NET.

**Decisão:**
.NET Aspire para developer experience dos serviços .NET
(topologia em código, dashboard local, OpenTelemetry zero-config).
Dapr como sidecar em produção para abstração de infraestrutura
multi-linguagem (pub/sub, secrets, state) — Python e .NET falam
com o Dapr sidecar via HTTP, não com Kafka diretamente.

**Alternativas descartadas:**

| Alternativa | Por que descartada |
|---|---|
| Aspire sozinho em produção | Não cobre Python/Java — Edge Agent e ML Pipeline ficam sem abstração |
| Dapr sozinho sem Aspire | DX de dev local inferior — sem dashboard integrado, sem topologia em código |
| Kubernetes raw (sem Aspire/Dapr) | Boilerplate de OTel, service discovery e resiliência repetido em cada serviço |
| Service Mesh puro (Istio/Linkerd) | Complexidade operacional desproporcional para squad de 2 |

**Riscos assumidos:**
- Dois frameworks novos simultaneamente. **Mitigação:** Aspire
  entra primeiro (dev local), Dapr expande gradualmente por serviço.
- Lock-in Azure via AZD. **Mitigação:** Terraform como IaC principal;
  AZD apenas para manifests iniciais.

**Hipóteses que sustentam esta decisão:** Squad de 2, Aspire + Dapr
