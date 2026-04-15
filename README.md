# Exposição Segura de API na Azure - Arquitetura da solução

> Exposição segura de uma API interna para um app web na Azure, com IaC validável, regras de segurança documentadas e plano de observabilidade.

---

## Estrutura do Repositório

```
terraform-case-vale3/
├── README.md                        ← Este arquivo
├── docs/
│   ├── arquitetura.md               ← Escolhas de arquitetura (rede, segurança, HA)
│   └── plano-observabilidade.md     ← Plano de observabilidade resumido
├── diagrams/
│   └── architecture.svg             ← Diagrama da arquitetura simples
├── iac/
│   ├── main.tf                      ← Terraform: VNet + Subnets + NSG
│   ├── variables.tf                 ← Variáveis configuráveis
│   ├── outputs.tf                   ← Outputs do deploy
│   └── README.md                    ← Instruções init/plan/deploy
└── security/
    └── nsg-rules.json               ← Regras NSG com justificativas
```

---

##  Iniciar Projeto

### Pré-requisitos

- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.5.0
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) >= 2.50.0
- Conta Azure com permissões de Contributor na subscription

### Deploy em 3 passos

```bash
# 1. Clone o repositório
git clone https://github.com/<seu-usuario>/terraform-case-vale3.git
cd terraform-case-vale3/iac

# 2. Autentique na Azure
az login
az account set --subscription "<SUBSCRIPTION_ID>"

# 3. Init, Plan e Apply
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Validação sem deploy (modo simulado)

```bash
cd iac/
terraform init
terraform validate   # Valida sintaxe sem provisionar nada
terraform plan       # Exibe o que seria criado
```

---

## Resumo das Decisões de Arquitetura

| Componente | Decisão | Justificativa |
|---|---|---|
| Rede | VNet + 2 subnets (Web/App) | Isolamento de camadas por NSG |
| Firewall | NSG + WAF (sem Azure Firewall) | Custo-benefício adequado ao escopo |
| Acesso ao Storage | Managed Identity | Zero secrets, sem rotação manual |
| Exposição da API | Private Endpoint + Application Gateway WAF | API nunca exposta publicamente |
| Alta Disponibilidade | Availability Zones + Health Probes + Autoscale | SLA 99,9% conceitual |
| Observabilidade | NSG Flow Logs + Application Insights | Visibilidade L3/L7 |

---

## Documentação Completa

- **Arquitetura detalhada** → [`docs/arquitetura.md`](docs/arquitetura.md)
- **Plano de observabilidade** → [`docs/plano-observabilidade.md`](docs/plano-observabilidade.md)
- **IaC + deploy** → [`iac/README.md`](iac/README.md)
- **Regras de segurança** → [`security/nsg-rules.json`](security/nsg-rules.json)

---

## Observações

- O IaC pode ser validado localmente com `terraform validate` sem acesso à Azure.

---
