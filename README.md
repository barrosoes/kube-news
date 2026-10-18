# Kube-News

Uma aplicação de notícias desenvolvida em NodeJS para demonstrar o uso de containers e Kubernetes.

## 📋 Sobre o Projeto

O projeto Kube-News é uma aplicação web simples desenvolvida em Node.js, projetada como exemplo para demonstrar o uso de contêineres. É um portal de notícias que permite criar, visualizar e gerenciar artigos através de uma interface web.

### 🚀 Funcionalidades Principais

- Listagem de notícias na página inicial
- Criação de novas notícias através de formulário
- Visualização detalhada de cada notícia
- API REST para inserção em massa de notícias
- Endpoints de health check para monitoramento
- Coleta de métricas para Prometheus

## 🛠️ Tecnologias Utilizadas

- **Backend**: Node.js com Express.js
- **Frontend**: EJS (Embedded JavaScript) como motor de templates
- **Banco de Dados**: PostgreSQL com Sequelize ORM
- **Monitoramento**: Prometheus (via express-prom-bundle)

## 📦 Estrutura do Projeto

```
/
├── src/                      # Código-fonte principal
│   ├── models/               # Modelos de dados
│   │   └── post.js           # Definição do modelo Post
│   ├── views/                # Templates EJS
│   │   ├── partial/          # Componentes parciais (header, footer)
│   │   ├── edit-news.ejs     # Formulário de edição
│   │   ├── index.ejs         # Página principal
│   │   └── view-news.ejs     # Visualização de notícia
│   ├── static/               # Arquivos estáticos (CSS, imagens)
│   ├── middleware.js         # Middlewares personalizados
│   ├── server.js             # Ponto de entrada da aplicação
│   ├── system-life.js        # Endpoints de health check
│   └── package.json          # Dependências
├── popula-dados.http         # Arquivo para popular o banco com dados de exemplo
└── README.md                 # Documentação
```

## 🔧 Configuração

### Pré-requisitos

- Node.js
- PostgreSQL
- Docker (opcional, para containerização)
- Kubernetes (opcional, para orquestração)

### Variáveis de Ambiente

Para configurar a aplicação, defina as seguintes variáveis de ambiente:

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| DB_DATABASE | Nome do banco de dados | kubedevnews |
| DB_USERNAME | Usuário do banco de dados | kubedevnews |
| DB_PASSWORD | Senha do usuário | Pg#123 |
| DB_HOST | Endereço do banco de dados | localhost |
| DB_PORT | Porta do banco de dados | 5432 |
| DB_SSL_REQUIRE | Habilitar SSL para conexão | false |

## 🚀 Instalação e Execução

### Execução Local

1. Clone o repositório
2. Instale as dependências:
   ```bash
   cd src
   npm install
   ```
3. Configure as variáveis de ambiente necessárias
4. Inicie a aplicação:
   ```bash
   npm start
   ```
5. Acesse a aplicação em [http://localhost:8080](http://localhost:8080)

### População de Dados de Exemplo

Utilize o arquivo `popula-dados.http` para inserir notícias de exemplo:

```bash
# Com uma ferramenta como o REST Client no VS Code ou curl
POST http://localhost:8080/api/post
Content-Type: application/json
# Conteúdo do arquivo popula-dados.http
```

## ☸️ Kubernetes e GitOps

O estado do cluster vem exclusivamente de `k8s/` na branch `main`; o ArgoCD (`argocd/kube-news-application.yaml`) aplica com `prune` e `selfHeal`. Alterações manuais via `kubectl` em recursos de `k8s/` são revertidas.

### Fluxo de deploy

```
push em src/** (main) → GitHub Actions (.github/workflows/deploy.yaml)
  build → Trivy (informativo) → push barrosoes/kube-news:sha-<7>
  → commit "chore(deploy): kube-news sha-<7> [skip ci]" em k8s/app.yaml
  → ArgoCD sincroniza → Deployment atualizado
```

- **Gatilho:** push em `main` que altere `src/**` (ou `workflow_dispatch`). Commits só em `k8s/` ou `README.md` não disparam o pipeline.
- **Tag:** sempre `sha-<7 primeiros caracteres do commit>`, imutável; não existe `latest`.
- **Secrets do GitHub:** `DOCKERHUB_USERNAME` e `DOCKERHUB_TOKEN`.
- **Scan:** o resultado do Trivy aparece em Security > Code scanning e no resumo da execução, mas não bloqueia o deploy (modo informativo).
- **Rollback:** `git revert <commit chore(deploy)>` e `git push`; o ArgoCD volta a imagem anterior.

### Segredos (Sealed Secrets)

O controller roda em `kube-system` (`argocd/sealed-secrets-application.yaml`) e o Secret do banco está versionado cifrado em `k8s/sealed-secret-kube-news-db.yaml`.

Criar ou rotacionar um segredo (sem gravar o valor em claro no repositório):

```bash
kubectl create secret generic kube-news-db -n kube-news \
  --from-literal=DB_DATABASE=<db> --from-literal=DB_USERNAME=<user> --from-literal=DB_PASSWORD=<senha> \
  --dry-run=client -o yaml | kubeseal -o yaml > k8s/sealed-secret-kube-news-db.yaml
```

Trocar `DB_PASSWORD` no Secret não altera a senha do usuário no Postgres já inicializado; altere-a também no banco (`ALTER USER`).

**Backup da chave:** a chave privada do controller foi salva fora do repositório, em `~/backups/sealed-secrets/kube-news-oke-key.yaml`. Sem ela, os SealedSecrets não são decifráveis se o cluster for recriado. Para gerar de novo:

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > <arquivo-seguro>
```

**Rollback dos Sealed Secrets:** antes de remover o SealedSecret do git, tire do Secret a anotação `sealedsecrets.bitnami.com/managed` e o `ownerReference`; caso contrário a remoção apaga o Secret e derruba a aplicação.

```bash
kubectl annotate secret kube-news-db -n kube-news sealedsecrets.bitnami.com/managed-
kubectl patch secret kube-news-db -n kube-news --type=json -p '[{"op":"remove","path":"/metadata/ownerReferences"}]'
```

## 📊 Monitoramento e Health Checks

A aplicação disponibiliza endpoints para monitoramento e também recursos para simular cenários de falha, muito úteis para testar a resiliência em ambientes Kubernetes:

### Endpoints de Monitoramento
- `/health` - Verifica o estado atual da aplicação (retorna status da aplicação e hostname da máquina)
- `/ready` - Verifica se a aplicação está pronta para receber tráfego
- `/metrics` - Métricas do Prometheus (geradas pelo express-prom-bundle)

### Simulação de Falhas (Chaos Engineering)
- `/unhealth` - (PUT) Altera o estado da aplicação para não saudável. Todas as requisições subsequentes receberão status code 500.
- `/unreadyfor/:seconds` - (PUT) Simula indisponibilidade temporária por um número específico de segundos. Durante este período, o endpoint `/ready` retornará status code 500.

Estes recursos de simulação de falhas são extremamente úteis para testar:
- Comportamento de probes de liveness e readiness no Kubernetes
- Políticas de retry e circuit breaker
- Mecanismos de failover
- Resiliência geral da sua infraestrutura

## 🔒 Modelo de Dados

O projeto utiliza um único modelo `Post` com os seguintes campos:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| title | String | Título da notícia (limite: 30 caracteres) |
| summary | String | Resumo da notícia (limite: 50 caracteres) |
| content | String | Conteúdo completo (limite: 2000 caracteres) |
| publishDate | Date | Data de publicação |


