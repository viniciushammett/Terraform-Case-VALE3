# Plano de Observabilidade

> Visibilidade L3 (rede) e L7 (aplicação) com foco em detecção rápida e baixo custo operacional.

---

## Coleta de Logs

### NSG Flow Logs (Camada L3/L4)

**O que captura:** Todo tráfego que passa pelos NSGs - IPs de origem/destino, portas, protocolo, bytes transferidos, decisão (Allow/Deny).

**Configuração:**
```
Destino:      Azure Storage Account (retenção 30 dias) → Log Analytics Workspace
Versão:       NSG Flow Logs v2 (inclui bytes/pacotes por fluxo)
Intervalo:    60 segundos
Traffic Analytics: habilitado (agrega fluxos para visualização no mapa de topologia)
```

**Valor:** Permite identificar tentativas de acesso direto à `snet-app` bypassando o Gateway, picos de tráfego suspeitos, e IPs bloqueados frequentes.

---

### Application Insights (Camada L7)

**O que captura:** Requests HTTP (status, duração, dependências), exceptions, traces, custom events.

**Integração:**
- App Service conectado ao Application Insights via `APPLICATIONINSIGHTS_CONNECTION_STRING` (variável de ambiente - sem SDK obrigatório)
- Para visibilidade de dependências (Storage, banco), instrumentar com SDK: `applicationinsights` package

**Dashboards prontos:** Failure rate, response time percentiles, live metrics, dependency map.

---

## 2 Métricas-Chave

### Métrica 1 - Taxa de Erros HTTP 5xx (Application Insights)

```
Alerta: requests/failed > 5% das requests nos últimos 5 minutos
Severidade: P1
Ação: Notificação via Action Group → email + webhook Slack
```

**Justificativa:** Erro 5xx indica falha na API (não no cliente). Acima de 5% em janela de 5 minutos é anomalia que exige investigação imediata. Abaixo desse threshold evita alertas falsos por spikes pontuais.

---

### Métrica 2 - Fluxos Negados pelo NSG (NSG Flow Logs → Log Analytics)

```kql
AzureNetworkAnalytics_CL
| where FlowStatus_s == "D"                    // D = Denied
| where SubNet_s == "snet-app"
| summarize NegadosPorMinuto = count() by bin(TimeGenerated, 1m), SrcIP_s
| where NegadosPorMinuto > 10

Alerta: > 10 fluxos negados/minuto vindos do mesmo IP para snet-app
Severidade: P2
Ação: Log no SIEM + notificação para time de segurança
```

**Justificativa:** Tentativas repetidas de acesso direto à subnet de app (bypassando o Gateway) indicam reconhecimento ou ataque direcionado. O threshold de 10/minuto diferencia scanning ativo de erros de configuração pontuais.

---

## Retenção e Custo

| Dado | Retenção | Custo estimado |
|---|---|---|
| NSG Flow Logs (Storage) | 30 dias | ~USD 2–5/mês |
| Log Analytics Workspace | 90 dias (padrão) | Pay-per-GB (~USD 2,76/GB) |
| Application Insights | 90 dias | Primeiros 5 GB/mês gratuitos |

> Para ambientes de produção, considerar Azure Monitor Workbooks para dashboards consolidados e Azure Sentinel se houver requisito de SIEM.

---