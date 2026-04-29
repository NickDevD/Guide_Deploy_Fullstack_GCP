# Guide Deploy Fullstack GCP
**Este guia fornece o passo a passo completo para realizar o deploy de uma arquitetura com custo zero (ou quase nulo), ideal para projetos de portfólio e testes.**

## **🏗️ Arquitetura**

- **Banco de Dados: Supabase (PostgreSQL)**
- **Backend: Java Spring Boot no Google Cloud Run (Serverless)**
- **Frontend: React/Vite no Firebase Hosting**
- **Segurança: GCP Secret Manager**

## **☁️ Preparação no Cloud Shell**

Acesse o [Google Cloud Console](https://console.cloud.google.com/) e abra o **Cloud Shell**.

## ♻️ 1. Clonar e Configurar Variáveis

Substitua os valores abaixo pelos do seu projeto:

```bash
# 1. Clone o seu projeto
git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
cd SEU_REPOSITORIO

# 2. Defina as variáveis de ambiente para automatizar os comandos
export PROJECT_ID=$(gcloud config get-value project)
export REGION="adicione a região"
export REPO_NAME="adicione um nome ao repositório que vai armazenar sua imagem"
export IMAGE_NAME="adicione um nome a imagem - escolha o nome que desejar"
export SERVICE_NAME="adicione um nome ao serviço que vai rodar"
```




## 🛠️ 2. Preparando o Terreno (GCP)

Ative as APIs necessárias para execução do projeto. Por padrão, todas veem desativadas para não consumir recursos.

- API Secret Manager: Para Gerenciar as secrets adicionadas
- API Artifact Registry: Permite criar repositórios para armazenamneto de imagens docker dentro do GCP

```bash
# Ativar APIs essenciais
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com
```

- Repositório Criado via comando Cloud Shell

```bash
# Criar Repositório no Artifact Registry
gcloud artifacts repositories create $REPO_NAME \
    --repository-format=docker \
    --location=$REGION \
    --description="Repositorio Docker para o Projeto (Adicione o nome do seu projeto)"
```




## 🔐 3. Cofre de Segredos (Secret Manager)

Não trafegue senhas em texto puro. Vamos usar o Secret Manager.

Cada Projeto possui uma configuração de keys de acesso. Para o exemplo abaixo, foram adicionadas ao Secret as chaves

- JWT - Autenticação
- Login User - Nome de Usuário com acesso de admininstrador
- Admin Senha - Senha do Usuário administrador
- Senha do Banco de Dados Externo - Supabase

```bash
# Criar os segredos (Substitua os valores entre aspas)
echo -n "sua_chave_jwt_secreta" | gcloud secrets create sai_jwt_secret --data-file=-
echo -n "admin_login" | gcloud secrets create sai_admin_user --data-file=-
echo -n "admin_senha_pura" | gcloud secrets create sai_admin_password --data-file=-
echo -n "senha_do_supabase" | gcloud secrets create sai_db_password --data-file=-
```

**Após adicionar as secrets que são essenciais para o projeto é necessário liberar o acesso para que o cloud run possa ler estes segredos e preencher os espaços automaticamente.**

```bash
# Liberar acesso para o Cloud Run ler os segredos
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```




## ☕ 4. Deploy do Backend (Java)

Certifique-se de que seu `application.properties` ou arquivo de configuração de variáveis de ambiente dentro d projeto está configurado para ler as variáveis corretamente, pois durente o processo de deploy este arquivo será consultado para coleta destes valores.

Importante que os comandos a seguir sejam executados dentro da pasta onde o Dockerfile, com a config do backend, está localizado.

```bash
# 1. Build e Push da Imagem
cd backend
gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/$IMAGE_NAME .
```

**Com a imagem montada e armazenada, basta rodar o comando de Deploy.** 

```bash
# 2. Deploy para o Cloud Run
gcloud run deploy $SERVICE_NAME \
  --image $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/$IMAGE_NAME \
  --region $REGION \
  --allow-unauthenticated \
  --set-env-vars "SPRING_DATASOURCE_URL=jdbc:postgresql://[SEU_HOST_AQUI]:5432/postgres,SPRING_DATASOURCE_USERNAME=[SEU_USUARIO]" \
--set-secrets "ADMIN_LOGIN=[NOME_SEGREDO_LOGIN]:latest,ADMIN_PASSWORD_LINE=[NOME_SEGREDO_SENHA]:latest,JWT_SECRET=[NOME_SEGREDO_JWT]:latest,SPRING_DATASOURCE_PASSWORD=[NOME_SEGREDO_DB]:latest"
```

🛑 Ao fim do deploy, um link com a URL do backend será gerado, guarde-o, pois vamos utilizar durante o deploy do Frontend.

**🛑 ATENÇÃO**

#### Variáveis de Ambiente (`-set-env-vars`)

Estas são informações de configuração que **não** são sensíveis.

- **`SPRING_DATASOURCE_URL`**: Substitua `[HOST_DO_BANCO]` pelo endereço do seu banco de dados (ex: Supabase ou Cloud SQL).
- **`SPRING_DATASOURCE_USERNAME`**: O usuário do banco de dados (geralmente `postgres`).

#### Segredos do Secret Manager (`-set-secrets`)

Estas variáveis buscam os valores que você salvou no "Cofre" do GCP no Passo 2. O formato é `VARIAVEL_NO_PROJETO=NOME_DO_SEGREDO_NO_GCP:latest`.




## ⚛️ 5. Frontend (Firebase Hosting)

O segredo aqui é injetar a URL do backend que acabamos de criar.

Importante que os comandos a seguir sejam executados dentro da pasta onde o Dockerfile, com a config do frontend, está localizado.

O comando abaixo cria uma variável de ambiente que armazenará o link do backend de forma automática.

```bash
cd ../front

# Criar o arquivo .env dinamicamente
echo "VITE_API_URL=$(gcloud run services describe $SERVICE_NAME --region $REGION --format='value(status.url)')" > .env
```

## Build do Projeto Frontend




### 🔥 Console  Firebase

Vincule o projeto no Console do Firebase - Importante para o GCP Identificar o Projeto Correto no Terminal

Siga estes passos rápidos:

1. Acesse o [Console do Firebase](https://console.firebase.google.com/).
2. Clique em **"Adicionar projeto"** (ou "Add project").
3. **Não digite um nome novo.** Clique no campo de nome e você verá uma lista dos seus projetos existentes do Google Cloud.
4. Selecione o [PROJETO].
5. Clique em continuar e aceite os termos. Isso vai "ativar" as funções do Firebase dentro do seu projeto do Google Cloud.
</aside>

Após selecionar e configurar qual projeto será a base do build, vamos instalar as ferramentas necessárias.

```bash
# Instala o CLI do Firebase (se ainda não instalou)
npm install -g firebase-tools

# Instala as dependências do seu projeto React
npm install

# Gera a pasta 'dist' com o código otimizado para produção
npm run build
```

### 🔥 Inicializar o Firebase Hosting




- Iniciar Login no Firebase

```bash
firebase login --no-localhost
```

- Iniciar o processo de hosting do projeto no Firebase

```bash
firebase init hosting
```

**Configurações do `init`:**

- **Project:** Select an existing project (NOME_PROJETO).
- **Public directory:** Selecione de acordo com o seu projeto
- **Single-page app:** Selecione de acordo com o seu projeto
- **GitHub Action:** Selecione de acordo com o seu projeto


- Ao final deste processo, com sucesso do Deploy o link para o frontend será gerado e já poderá ser acessado.

- Projeto no ar e rodando com o menor custo possível🎉🚀





## ⚠️Guia de Solução de Problemas

### 🛑Port Error no Cloud Run

Certifique-se de substituir `SEU_HOST_BANCO_DADOS` pela URL real do seu banco (ex: `db.xrezbaqbekify.supabase.co`). Se essa variável estiver errada ou incompleta, o container não iniciará e o Cloud Run reportará um erro de porta/timeout.

### 🧰Extrair Logs do Serviço

Comando essencial para mostrar os logs do serviço no terminal, para identificar erros facilmente:

```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=NOME_DO_SERVICO" --limit=20 --format="value(textPayload)"
```
