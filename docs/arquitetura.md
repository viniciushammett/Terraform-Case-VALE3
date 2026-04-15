# Escolhas de Arquitetura

---

## 1. Rede

A topologia usa uma **única VNet (10.0.0.0/16)** dividida em duas subnets com responsabilidades distintas:

| Subnet | CIDR | Função |
|---|---|---|
| `snet-web` | 10.0.1.0/24 | Application Gateway (WAF) — borda pública |
| `snet-app` | 10.0.2.0/24 | App Service (API) — camada privada |

**Por que não Azure Firewall?**
O Azure Firewall custa ~USD 1.000/mês antes de qualquer tráfego. Para este escopo, uma API exposta via WAF sem roteamento hub-and-spoke, o custo não se justifica. NSG (L3/L4) + WAF (L7) entregam a proteção necessária com zero custo adicional de infraestrutura. Se o escopo crescer para múltiplas VNets ou FQDN filtering centralizado, Azure Firewall seria reavaliado.

---

## 2. Identidade

O App Service recebe uma **System-Assigned Managed Identity** e o role `Storage Blob Data Reader` (RBAC mínimo) no Storage Account. Nenhuma secret, nenhuma connection string em código, o SDK Azure autentica via token efêmero do Azure AD. Isso elimina toda a classe de riscos de credential leak e rotação manual.

---

## 3. Segurança de Borda - Private Endpoint + WAF v2

A API **nunca recebe tráfego direto da internet**. O fluxo obrigatório é:

```
Internet → Application Gateway WAF v2 (snet-web)
               ↓  Private Endpoint
           App Service API (snet-app)
               ↓  Managed Identity
           Storage Account
```

**Por que Private Endpoint e não IP público + WAF?**
Com IP público no App Service, um atacante que descobre o FQDN bypassa o WAF completamente. O Private Endpoint garante que a API só seja alcançável de dentro da VNet, mesmo que o Gateway seja comprometido, a superfície de ataque fica contida.

### Regras essenciais NSG + WAF

| Camada | Regra | Motivo |
|---|---|---|
| L3 — NSG | Allow 443 inbound: `snet-web` → `snet-app` | Única origem autorizada para a API |
| L3 — NSG | Allow 443 outbound: `snet-app` → `Storage` (Service Tag) | Acesso ao blob via Managed Identity |
| L3 — NSG | Allow 443 outbound: `snet-app` → `AzureActiveDirectory` | Aquisição de token MSAL |
| L3 — NSG | Deny all inbound `0.0.0.0/0` — prioridade 4096 | Catch-all: nega tudo não explicitamente permitido |
| L7 — WAF | OWASP CRS 3.2 + rate limiting 1.000 req/min por IP | Proteção contra SQLi, XSS, DDoS de aplicação |

---

## 4. Alta Disponibilidade — SLA 99,9%

Três pilares conceituais sem overengineering:

**Availability Zones** — Application Gateway v2 é zone-redundant nativamente. App Service em `PremiumV3` distribui instâncias entre AZ 1, 2 e 3 automaticamente. Uma AZ inteira pode cair sem downtime.

**Health Probes** — Application Gateway verifica `/health` a cada 30s. Instâncias não-saudáveis são removidas do pool de forma automática e silenciosa.

**Autoscaling** — App Service escala entre mínimo 2 e máximo 10 instâncias. Trigger: CPU > 70% por 5 minutos. O mínimo de 2 instâncias garante redundância mesmo em baixo tráfego.

Nenhum SPOF identificado na topologia.

---

## 5. Decisões Resumidas

| Componente | Escolha | Justificativa |
|---|---|---|
| Firewall | NSG + WAF (sem Azure Firewall) | Custo vs. benefício inadequado para o escopo |
| Exposição da API | Private Endpoint | API nunca acessível diretamente da internet |
| Identidade | Managed Identity | Zero secrets, sem rotação, RBAC mínimo |
| Alta disponibilidade | AZ + Health Probes + Autoscale | SLA 99,9% sem SPOF |
| IaC | Terraform | Validável localmente, portável, open source |
| Observabilidade | NSG Flow Logs + App Insights | Visibilidade L3 e L7 com custo mínimo |

---
