# Superbadge: Extended User Access and Restriction

<div align="center">

<img src="doc/imagens/logo.png" alt="Salesforce Logo" width="180"/>

<br/>

![Superbadge Badge](doc/imagens/badge.png)

<br/>

![Challenge Complete](doc/imagens/congratulations.png)

<br/>

**Salesforce Trailhead Superbadge**  
Build effective sharing solutions to provide the right access to the right records.

[![Salesforce](https://img.shields.io/badge/Salesforce-00A1E0?style=for-the-badge&logo=salesforce&logoColor=white)](https://trailhead.salesforce.com)
[![SFDX](https://img.shields.io/badge/SFDX-CLI-blue?style=for-the-badge)](https://developer.salesforce.com/tools/salesforcecli)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

</div>

---

## Autor

**Leandro da Silva Stampini**  
Salesforce Developer | Trailblazer  
GitHub: [@stampini81](https://github.com/stampini81)

---

## Visão Geral

Este repositório documenta a implementação do **Superbadge Extended User Access and Restriction** do Salesforce Trailhead. O desafio aborda configurações avançadas de segurança e compartilhamento de dados no Salesforce, incluindo:

- Configuração de **Organization-Wide Defaults (OWD)**
- Criação de **Roles** e hierarquias de compartilhamento
- Regras de compartilhamento baseadas em **proprietário** e **critérios**
- **Restriction Rules** para filtrar acesso a registros de tarefas

### Caso de Uso

A empresa **Thunderground**, uma startup de e-commerce, está expandindo para o mercado EMEA (Europa, Oriente Médio e África) e precisa garantir que a equipe de vendas, o auditor de GDPR e as partes interessadas tenham o acesso correto aos registros corretos, em conformidade com o GDPR.

---

## Pré-requisitos

- [Salesforce CLI (SF CLI)](https://developer.salesforce.com/tools/salesforcecli) instalado
- Git instalado
- Org Developer Edition especial para este Superbadge ([signup aqui](https://trailhead.salesforce.com/promo/orgs/usersuperbadge))
- Conta no [Trailhead](https://trailhead.salesforce.com)

---

## Estrutura do Projeto

```
superbadge-extended-user-access-and-restriction/
├── doc/
│   └── imagens/
│       └── badge.png
├── force-app/
│   └── main/
│       └── default/
│           ├── roles/
│           │   ├── GDPR_Auditor.role-meta.xml
│           │   ├── Technical_Sales_Manager.role-meta.xml
│           │   └── Technical_Sales_Representative.role-meta.xml
│           ├── sharingRules/
│           │   ├── Account.sharingRules-meta.xml
│           │   ├── Contract.sharingRules-meta.xml
│           │   ├── Opportunity.sharingRules-meta.xml
│           │   └── Task.sharingRules-meta.xml
│           └── restrictionRules/
│               ├── Sales_Manager_Task_Restriction.restrictionRule-meta.xml
│               └── Sales_Rep_Task_Restriction.restrictionRule-meta.xml
├── manifest/
│   └── package.xml
├── .forceignore
├── .gitignore
├── sfdx-project.json
└── README.md
```

---

## Passo a Passo de Implementação

### Etapa 0 — Configuração do Ambiente

#### 0.1. Clone o repositório

```bash
git clone https://github.com/stampini81/superbadge-extended-user-access-and-restriction.git
cd superbadge-extended-user-access-and-restriction
```

#### 0.2. Autentique a org Salesforce via browser

```bash
sf org login web --alias superbadge-user-access --set-default --instance-url https://login.salesforce.com
```

> Uma janela do browser será aberta. Faça login com suas credenciais da Developer Edition especial e clique em **Allow**.

---

### Desafio 1 — Configurações Gerais de Segurança de Registros

#### 1.1. Configurar Organization-Wide Defaults (OWD) — Manual

> **Nota:** Esta configuração deve ser feita manualmente via UI pois os OWDs não são deployáveis via Metadata API.

1. Acesse **Setup** → pesquise por **Sharing Settings** → clique em **Edit**
2. Configure os seguintes objetos:

| Objeto | Configuração Padrão |
|--------|-------------------|
| **Account** | `Private` |
| **Contract** | `Private` |
| **Opportunity** | `Private` |

3. Clique em **Save**

> ⚠️ **Importante:** Esta é a etapa mais crítica. Sem o OWD configurado como Private, o deploy dos roles com nível de acesso `None` para Oportunidades falhará.

#### 1.2. Criar os Roles via deploy de metadados

Após configurar o OWD, execute o deploy dos 3 novos roles:

```bash
sf project deploy start \
  --metadata "Role:GDPR_Auditor" \
  --metadata "Role:Technical_Sales_Manager" \
  --metadata "Role:Technical_Sales_Representative" \
  --target-org superbadge-user-access \
  --wait 30
```

Os roles criados seguem a hierarquia abaixo:

```
CEO
├── General Counsel
│   └── GDPR Auditor          ← Novo (sem acesso a opps de terceiros)
└── SVP, Sales & Marketing
    └── VP, International Sales
        └── EMEA Sales
            └── Technical Sales Manager    ← Novo (pode editar todas as opps)
                └── Technical Sales Representative  ← Novo (sem acesso a opps de terceiros)
```

| Role | Reporta para | Acesso a Oportunidades |
|------|-------------|----------------------|
| `GDPR_Auditor` | General Counsel | Sem acesso a opps que não são suas em contas que possui |
| `Technical_Sales_Manager` | EMEA Sales | Pode editar todas as opps em contas que possui |
| `Technical_Sales_Representative` | Technical Sales Manager | Sem acesso a opps que não são suas em contas que possui |

---

### Desafio 2 — Acesso Cross-Funcional a Registros

#### 2.1. Criar o Grupo Público "Operations" — Via DML

```bash
# Criar o grupo público
sf data create record \
  --sobject Group \
  --values "Name='Operations' DeveloperName='Operations' Type='Regular' Description='Operations team for provisioning closed won opportunities.'" \
  --target-org superbadge-user-access
```

> Anote o ID retornado. Substitua `<GROUP_ID>` nos comandos abaixo.

#### 2.2. Adicionar roles de Customer Support ao grupo

```bash
# Consultar IDs internos dos grupos de role
sf data query \
  --query "SELECT Id, Name, Type, RelatedId FROM Group WHERE Type = 'Role' AND RelatedId IN ('CUSTOMER_SUPPORT_NA_ROLE_ID', 'CUSTOMER_SUPPORT_INTL_ROLE_ID')" \
  --target-org superbadge-user-access
```

> Os IDs dos roles Customer Support estão disponíveis via:
> ```bash
> sf data query --query "SELECT Id, Name FROM UserRole WHERE DeveloperName IN ('CustomerSupportNorthAmerica','CustomerSupportInternational')" --target-org superbadge-user-access
> ```

```bash
# Adicionar Customer Support, North America ao grupo
sf data create record \
  --sobject GroupMember \
  --values "GroupId='<GROUP_ID>' UserOrGroupId='<CS_NA_ROLE_GROUP_ID>'" \
  --target-org superbadge-user-access

# Adicionar Customer Support, International ao grupo
sf data create record \
  --sobject GroupMember \
  --values "GroupId='<GROUP_ID>' UserOrGroupId='<CS_INTL_ROLE_GROUP_ID>'" \
  --target-org superbadge-user-access
```

#### 2.3. Deploy das Sharing Rules

Após criar os roles e o grupo Operations, execute o deploy das regras de compartilhamento:

```bash
sf project deploy start \
  --metadata "SharingRules:Account" \
  --metadata "SharingRules:Opportunity" \
  --target-org superbadge-user-access \
  --wait 30
```

**Regras criadas:**

| Nome | Objeto | Tipo | Compartilha com | Critério |
|------|--------|------|-----------------|---------|
| `Operations_Visibility` | Opportunity | Critério | Grupo Operations | StageName = Closed Won AND Provisioned = False |
| `GDPR_Auditor_Opportunity_Visibility` | Opportunity | Proprietário | Role GDPR_Auditor | Opps de EMEA Sales e subordinados |
| `GDPR_Auditor_Account_Visibility` | Account | Critério | Role GDPR_Auditor | European Union = True |

---

### Desafio 3 - Controle de Acesso a Tarefas

Este desafio deve ser configurado no objeto **Task**, em **Setup > Object Manager > Task > Restriction Rules**. O Trailhead valida duas restriction rules ativas, com descricao preenchida, criterios de usuario por role e criterios de registro exatamente alinhados ao cenario.

#### 3.1. Obter os IDs reais dos roles

Os IDs dos roles sao unicos por org e sao usados no **User Criteria** das restriction rules:

```bash
sf data query \
  --query "SELECT Id, Name, DeveloperName FROM UserRole WHERE DeveloperName IN ('Technical_Sales_Manager','Technical_Sales_Representative')" \
  --target-org superbadge-user-access
```

> Os IDs comecam com `00E`. Nesta org, os valores usados foram `00Ebm00000Hgk4fEAB` para `Technical_Sales_Manager` e `00Ebm00000HgorSEAR` para `Technical_Sales_Representative`.

#### 3.2. Criar `Sales_Manager_Task_Restriction`

Na UI, crie a rule em **Task > Restriction Rules > New Rule**:

| Campo | Valor |
|-------|-------|
| Rule Name | `Sales_Manager_Task_Restriction` |
| Full Name | `Sales_Manager_Task_Restriction` |
| Description | `Allows Technical Sales Managers to see only tasks from their department` |
| Active | Marcado |
| User Criteria | `$User.UserRoleId` equals `00Ebm00000Hgk4fEAB` |
| Record Criteria | `[Task].Owner.User.Department` equals current user `$User.Department` |

Metadata equivalente:

**`force-app/main/default/restrictionRules/Sales_Manager_Task_Restriction.restrictionRule-meta.xml`**
```xml
<recordFilter>Owner:User.Department=$User.Department</recordFilter>
<targetEntity>Task</targetEntity>
<userCriteria>$User.UserRoleId='00Ebm00000Hgk4fEAB'</userCriteria>
```

> Importante: o criterio do gerente nao deve ser por `Owner.UserRoleId`. O criterio que passou foi por departamento: o departamento do dono da task igual ao departamento do usuario atual.

#### 3.3. Criar `Sales_Rep_Task_Restriction`

Na UI, crie a segunda rule em **Task > Restriction Rules > New Rule**:

| Campo | Valor |
|-------|-------|
| Rule Name | `Sales_Rep_Task_Restriction` |
| Full Name | `Sales_Rep_Task_Restriction` |
| Description | `Allows Technical Sales Representatives to see only the tasks they own.` |
| Active | Marcado |
| User Criteria | `$User.UserRoleId` equals `00Ebm00000HgorSEAR` |
| Record Criteria | `[Task].Owner.User.Id` equals current user `$User.Id` |

Metadata equivalente:

**`force-app/main/default/restrictionRules/Sales_Rep_Task_Restriction.restrictionRule-meta.xml`**
```xml
<recordFilter>Owner:User.Id=$User.Id</recordFilter>
<targetEntity>Task</targetEntity>
<userCriteria>$User.UserRoleId='00Ebm00000HgorSEAR'</userCriteria>
```

#### 3.4. Deploy das Restriction Rules

Neste projeto, as restriction rules devem ser implantadas pelo formato MDAPI em `deploy-mdapi/restrictionRules`:

```bash
sf project deploy start \
  --metadata-dir deploy-mdapi \
  --target-org superbadge-user-access \
  --wait 30
```

**Regras de Restrição criadas:**

| Nome | Aplica-se a | Filtro de Usuário | Filtro de Registro |
|------|------------|-------------------|-------------------|
| `Sales_Manager_Task_Restriction` | Technical Sales Managers | `$User.UserRoleId='00Ebm00000Hgk4fEAB'` | `Owner:User.Department=$User.Department` |
| `Sales_Rep_Task_Restriction` | Technical Sales Representatives | `$User.UserRoleId='00Ebm00000HgorSEAR'` | `Owner:User.Id=$User.Id` |

#### 3.5. Resultado

Com esses criterios, o Superbadge foi validado com sucesso:

![Superbadge concluido](doc/imagens/congratulations.png)

---

## Deploy Completo (Após OWD Configurado)

Para fazer o deploy de todos os metadados em um único comando:

```bash
sf project deploy start \
  --manifest manifest/package.xml \
  --target-org superbadge-user-access \
  --wait 60
```

---

## Retrieve — Recuperar Metadados da Org

Para baixar os metadados configurados na org:

```bash
sf project retrieve start \
  --manifest manifest/package.xml \
  --target-org superbadge-user-access
```

---

## Verificação dos Desafios

### Verificar roles criados

```bash
sf data query \
  --query "SELECT Id, Name, DeveloperName, ParentRoleId FROM UserRole ORDER BY Name" \
  --target-org superbadge-user-access
```

### Verificar grupo Operations e membros

```bash
# Verificar grupo
sf data query \
  --query "SELECT Id, Name, DeveloperName, Type FROM Group WHERE DeveloperName = 'Operations'" \
  --target-org superbadge-user-access

# Verificar membros
sf data query \
  --query "SELECT Id, GroupId, UserOrGroupId FROM GroupMember WHERE GroupId IN (SELECT Id FROM Group WHERE DeveloperName = 'Operations')" \
  --target-org superbadge-user-access
```

### Verificar Restriction Rules ativas

```bash
sf data query \
  --use-tooling-api \
  --query "SELECT Id, DeveloperName, TargetEntity, UserCriteria, RecordFilter, IsActive FROM RestrictionRule WHERE DeveloperName IN ('Sales_Manager_Task_Restriction','Sales_Rep_Task_Restriction')" \
  --target-org superbadge-user-access
```

---

## Conceitos Abordados

| Conceito | Descrição |
|----------|-----------|
| **OWD (Org-Wide Defaults)** | Define o nível de acesso padrão para todos os registros de um objeto |
| **Role Hierarchy** | Estrutura hierárquica que permite que gerentes vejam registros de seus subordinados |
| **Owner-Based Sharing Rules** | Compartilha registros com base em quem é o proprietário do registro |
| **Criteria-Based Sharing Rules** | Compartilha registros com base em critérios específicos dos campos do registro |
| **Public Groups** | Agrupamento de usuários, roles ou outros grupos para simplificar o compartilhamento |
| **Restriction Rules** | Restringe o acesso a registros além do que o modelo de compartilhamento permite |

---

## Recursos

- [Salesforce Sharing Settings Documentation](https://help.salesforce.com/s/articleView?id=sf.security_sharing_owd_about.htm)
- [Restriction Rules](https://help.salesforce.com/s/articleView?id=sf.security_restriction_rule.htm)
- [Salesforce CLI Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/)
- [Trailhead: Data Security](https://trailhead.salesforce.com/content/learn/modules/data_security)
- [User Access Superbadge: Trailhead Challenge Help](https://help.salesforce.com/s/articleView?id=sf.trailhead_superbadge_extended_user_access.htm)

---

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

---

<div align="center">

Desenvolvido com ❤️ por **Leandro da Silva Stampini**

<img src="doc/imagens/mascote.png" alt="Codey - Salesforce Mascot" width="150"/>

[![Trailhead Profile](https://img.shields.io/badge/Trailhead-Profile-00A1E0?style=flat&logo=salesforce)](https://trailhead.salesforce.com)

</div>
