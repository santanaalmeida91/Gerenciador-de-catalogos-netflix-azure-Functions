# Gerenciador de Catálogos da "Netflix" — Projeto DIO

> Projeto: **Gerenciador de Catálogos da Netflix com Azure Functions e Banco de Dados**

## 1. Objetivo

Criar uma aplicação serverless para gerenciar catálogos de filmes e séries (CRUD), demonstrando integração entre **Azure Functions**, **Banco de Dados (Azure SQL / Cosmos DB)**, automação de infraestrutura e **CI/CD** via GitHub Actions. O projeto serve como entrega prática para a DIO, evidenciando arquitetura, testes e deploy automatizado.

---

## 2. Visão geral / Arquitetura

* **Frontend (opcional)**: SPA simples (React) para consumir APIs.
* **API**: Azure Functions (HTTP Trigger) expondo endpoints REST: `GET /catalogs`, `GET /catalogs/{id}`, `POST /catalogs`, `PUT /catalogs/{id}`, `DELETE /catalogs/{id}`.
* **Banco de Dados**: duas opções recomendadas:

  * **Azure SQL Database** (relacional) — bom para consultas SQL e relacionamentos.
  * **Azure Cosmos DB (Core/SQL API)** — documento JSON, escala global, baixa latência.
* **Infraestrutura**: provisionamento via **Bicep** ou **Terraform** (ex.: Function App, Storage Account, SQL/Cosmos, Application Insights).
* **CI/CD**: GitHub Actions para build, testes e deploy automático para Azure.

Representação simplificada:

```
[User] --> [Azure Frontend (React)] --> [Azure Functions (HTTP)] --> [Azure SQL / Cosmos DB]
                                  \--> [Application Insights]
```

---

## 3. Stack tecnológica sugerida

* Backend: **Azure Functions** com **TypeScript** (Node.js 18+) ou **.NET 8 (C#)** — escolha conforme sua afinidade.
* Infra: **Bicep** (recomendado) ou **Terraform**.
* Banco de dados: **Azure SQL** ou **Cosmos DB**.
* CI/CD: **GitHub Actions**.
* Observabilidade: **Application Insights** + logs no Storage / AppInsights.

---

## 4. Estrutura do repositório (sugestão)

```
netflix-catalog-manager/
├─ .github/workflows/ci-cd.yml
├─ infra/                # Bicep ou Terraform
│  ├─ main.bicep
│  └─ params.dev.json
├─ backend/
│  ├─ functions/
│  │  ├─ getCatalogs/index.ts
│  │  ├─ getCatalogById/index.ts
│  │  ├─ createCatalog/index.ts
│  │  ├─ updateCatalog/index.ts
│  │  └─ deleteCatalog/index.ts
│  ├─ host.json
│  ├─ local.settings.json.sample
│  └─ package.json
├─ frontend/ (opcional)
│  └─ react-app/
├─ docs/
│  ├─ architecture.md
│  └─ runbook.md
├─ README.md
└─ LICENSE
```

---

## 5. README mínimo de apresentação (exigido para a DIO)

No documento `README.md` do repositório inclua:

* Nome do projeto
* Objetivo e escopo
* Tecnologias utilizadas
* Diagrama da arquitetura
* Passo-a-passo para rodar localmente
* Como executar os testes
* Instruções de deploy (GitHub Actions)
* Critérios de avaliação (features implementadas)

---

## 6. Exemplo rápido: Azure Function — `createCatalog` (TypeScript)

```ts
// backend/functions/createCatalog/index.ts
import { AzureFunction, Context, HttpRequest } from "@azure/functions";
import { v4 as uuidv4 } from "uuid";
import { insertCatalog } from '../../lib/db';

const httpTrigger: AzureFunction = async function (context: Context, req: HttpRequest): Promise<void> {
  try {
    const payload = req.body;
    if (!payload || !payload.title) {
      context.res = { status: 400, body: { error: 'title is required' } };
      return;
    }

    const newCatalog = {
      id: uuidv4(),
      title: payload.title,
      description: payload.description || null,
      type: payload.type || 'movie',
      year: payload.year || null,
      createdAt: new Date().toISOString()
    };

    await insertCatalog(newCatalog);

    context.res = {
      status: 201,
      body: newCatalog
    };
  } catch (err) {
    context.log.error(err);
    context.res = { status: 500, body: { error: 'internal server error' } };
  }
};

export default httpTrigger;
```

> `insertCatalog` seria uma função no módulo `backend/lib/db.ts` que encapsula a lógica de persistência (SQL/Cosmos).

---

## 7. Exemplo de esquema para Azure SQL

```sql
CREATE TABLE Catalog (
  Id UNIQUEIDENTIFIER PRIMARY KEY,
  Title NVARCHAR(250) NOT NULL,
  Description NVARCHAR(MAX) NULL,
  Type NVARCHAR(50) NULL,
  Year INT NULL,
  CreatedAt DATETIME2 NOT NULL
);
```

---

## 8. Provisionamento (Bicep) — esqueleto

```bicep
param location string = resourceGroup().location
param functionName string = 'netflix-catalog-func'

resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: '${toLower(functionName)}sa'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${functionName}-plan'
  location: location
  sku: { name: 'Y1', tier: 'Dynamic' }
}

resource functionApp 'Microsoft.Web/sites@2022-03-01' = {
  name: functionName
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        { name: 'AzureWebJobsStorage'; value: storage.properties.primaryEndpoints.blob }
        { name: 'FUNCTIONS_WORKER_RUNTIME'; value: 'node' }
      ]
    }
  }
}
```

---

## 9. GitHub Actions (fluxo básico)

Arquivo: `.github/workflows/ci-cd.yml` (esqueleto):

```yaml
name: CI/CD
on:
  push:
    branches: [ main, master ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install Dependencies
        run: |
          cd backend
          npm install
      - name: Run Tests
        run: |
          cd backend
          npm test
      - name: Build
        run: |
          cd backend
          npm run build
      - name: Deploy to Azure Functions
        uses: azure/functions-action@v1
        with:
          app-name: ${{ secrets.AZURE_FUNCTIONAPP_NAME }}
          package: ./backend
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
```

> Configure `AZURE_PUBLISH_PROFILE` como segredo no GitHub (obtenha do painel do Azure — *Get publish profile*).

---

## 10. Testes e qualidade

* Escreva testes unitários para a lógica de persistência (Jest ou xUnit).
* Testes de integração podem usar uma instância local do banco (localdb, or Cosmos Emulator).
* Inclua linting (ESLint / dotnet format) e checks no pipeline.

---

## 11. Entregáveis mínimos (para nota)

1. Repositório no GitHub com README claro.
2. Ao menos 3 Azure Functions implementadas (List, Create, GetById) e funcionando localmente.
3. Script de provisionamento (Bicep ou Terraform) mínimo para Function App e Banco.
4. Workflow do GitHub Actions que realiza build e deploy (mesmo que deploy para um slot de teste).
5. SQL schema / modelo de documentos.
6. Documentação curta de como executar localmente e como implantar.

---

## 12. Roadmap de implementação (tarefas sugeridas)

1. Inicializar repositório + README.
2. Criar projeto Azure Functions (TypeScript).
3. Implementar camada de persistência (DB adapter) com interface única (Repository Pattern).
4. Implementar funções: List, GetById, Create, Update, Delete.
5. Escrever testes unitários.
6. Criar Bicep/Terraform para infra.
7. Configurar GitHub Actions e secrets.
8. Testar deploy e corrigir observability.
9. Criar frontend simples (opcional).

---

## 13. Critérios extras para nota alta (diferença de excelência)

* Autenticação mínima (Azure AD / JWT) para proteger endpoints.
* Paginação, filtros e pesquisa por título/ano/genre.
* Monitoramento (Application Insights) com dashboards.
* Tratamento robusto de erros e logs estruturados.
* Testes de integração e cobertura mínima (ex.: 60%).

---

## 14. Como eu posso ajudar agora

Escolha uma das opções abaixo e eu executo/gero:

* Gerar o `README.md` completo e pronto para o GitHub (com badges).
* Gerar o scaffold de arquivos do backend (Azure Functions) com código inicial e testes.
* Gerar o arquivo Bicep/Terraform completo para provisionar infra mínima.
* Gerar o workflow do GitHub Actions pronto.

Diga qual opção prefere que eu gere agora que eu criarei os arquivos correspondentes no repositório de projeto (ou disponibilizo aqui para download).

---

*Documento gerado automaticamente para suportar entrega do projeto DIO.*
