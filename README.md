# 🌐 Integração entre dispositivos LoRaWAN e a plataforma FIWARE

<p align="center">
  <img src="https://www.fiware.org/custom/brand-guide/img/logo/fiware/secondary/png/logo-fiware-secondary.png" width="200"/>
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/1/13/LoRaWAN_Logo.svg/2560px-LoRaWAN_Logo.svg.png" alt="LoRaWAN Logo" width="300"/>
</p>

## 📖 Sobre este Guia

Este projeto apresenta uma arquitetura para integração de dispositivos **LoRaWAN** com a plataforma aberta **FIWARE**, permitindo receber, gerenciar, persistir e visualizar dados provenientes de dispositivos IoT.

A solução utiliza o **The Things Stack (TTN)** como servidor de aplicação LoRaWAN e integra seus dados aos componentes FIWARE executados em contêineres Docker.

Como exemplo, o projeto utiliza uma aplicação de monitoramento da qualidade do ar.

A arquitetura também disponibiliza um **Módulo de Inteligência Artificial (IA) opcional**, responsável por processar determinados dados recebidos pelo FIWARE e produzir uma nova informação calculada por um modelo de Machine Learning.

> [!IMPORTANTE]
> **O Módulo de IA é opcional.**
>
> A integração básica **FIWARE + LoRaWAN + TTN + persistência + Grafana funciona independentemente do módulo de IA**.
>
> Portanto, quem deseja apenas receber, gerenciar, armazenar e visualizar os dados dos dispositivos IoT pode seguir somente as etapas principais deste Guia e ignorar a seção **🤖 Módulo Opcional de Inteligência Artificial**.

---

# 📚 Sumário

* [📌 Contextualização](#-contextualização)
* [🏗️ Arquitetura do Projeto](#️-arquitetura-do-projeto)

  * [Orion Context Broker](#orion-context-broker)
  * [IoT Agent LoRaWAN](#iot-agent-lorawan)
  * [MongoDB](#mongodb)
  * [Cygnus](#cygnus)
  * [PostgreSQL](#postgresql)
  * [Grafana](#grafana)
  * [Módulo de IA opcional](#módulo-de-ia-opcional)
* [🔧 Pré-requisitos](#-pré-requisitos)

  * [Docker e Docker Compose](#docker-e-docker-compose)
  * [TTN](#ttn)
* [🚀 Instalação do Projeto](#-instalação-do-projeto)
* [🔐 Configuração das variáveis de ambiente](#-configuração-das-variáveis-de-ambiente)
* [📡 Configuração da aplicação LoRaWAN no TTN](#-configuração-da-aplicação-lorawan-no-ttn)
* [📤 Registro do dispositivo no IoT Agent](#-registro-do-dispositivo-no-iot-agent)
* [🔄 Fluxo dos dados](#-fluxo-dos-dados)
* [📦 Persistência com Cygnus e PostgreSQL](#-persistência-com-cygnus-e-postgresql)
* [📊 Visualização com Grafana](#-visualização-com-grafana)
* [🤖 Módulo Opcional de Inteligência Artificial](#-módulo-opcional-de-inteligência-artificial)

  * [Objetivo](#objetivo)
  * [Arquitetura da IA](#arquitetura-da-ia)
  * [Estrutura do módulo](#estrutura-do-módulo)
  * [Modelo de Machine Learning](#modelo-de-machine-learning)
  * [Construção da imagem Docker](#construção-da-imagem-docker)
  * [Inicialização do serviço](#inicialização-do-serviço)
  * [Subscription do Orion](#subscription-do-orion)
  * [Fluxo de processamento](#fluxo-de-processamento)
  * [Validação](#validação)
* [🧪 Verificação e diagnóstico](#-verificação-e-diagnóstico)
* [🔒 Segurança](#-segurança)
* [🧠 Considerações finais](#-considerações-finais)

---

# 📌 Contextualização

O **FIWARE** é uma plataforma aberta voltada ao desenvolvimento de aplicações inteligentes baseadas em dados de contexto.

Neste projeto, o FIWARE funciona como camada intermediária entre os dispositivos IoT e as aplicações responsáveis por persistência, visualização e processamento.

Os dispositivos utilizam **LoRaWAN** para transmitir suas informações.

O **The Things Stack** recebe os dados da rede LoRaWAN e disponibiliza essas informações por meio de sua camada de aplicação.

O **IoT Agent LoRaWAN** recebe esses dados e os converte para o modelo utilizado pelo **Orion Context Broker**.

O Orion passa então a gerenciar as entidades e seus atributos de contexto.

A partir desse ponto, diferentes componentes podem consumir os dados, como:

* Cygnus;
* PostgreSQL;
* Grafana;
* aplicações externas;
* APIs;
* módulos de Machine Learning;
* sistemas de análise e tomada de decisão.

---

# 🏗️ Arquitetura do Projeto

A arquitetura principal pode ser representada da seguinte forma:

![Arquitetura da integração FIWARE + LoRaWAN](docs/img/Diagrama_ic.png)

**Figura 1 — Arquitetura geral da integração FIWARE + LoRaWAN.**

O módulo de IA não interfere no caminho principal dos dados.

Ele funciona como um **consumidor adicional das informações disponibilizadas pelo Orion Context Broker**.

---

## 🧠 Orion Context Broker

O **Orion Context Broker** é o componente responsável pelo gerenciamento das informações de contexto.

Neste projeto, ele recebe as atualizações provenientes do IoT Agent LoRaWAN e mantém as entidades IoT.

Exemplo conceitual:

```json
{
  "id": "SensorCvel",
  "type": "LoraDevice",
  "Best_CO": {
    "type": "Float",
    "value": 50
  },
  "Temperatura": {
    "type": "Float",
    "value": 32
  },
  "Umidade": {
    "type": "Float",
    "value": 75
  }
}
```

O Orion também permite criar **Subscriptions**, possibilitando que outros serviços sejam notificados quando determinadas informações de contexto forem alteradas.

Essa funcionalidade é utilizada tanto pelo Cygnus quanto pelo módulo opcional de IA.

---

## 🤖 IoT Agent LoRaWAN

O **IoT Agent LoRaWAN** é responsável pela integração entre o servidor LoRaWAN e o FIWARE.

Seu papel é:

1. receber dados provenientes do servidor LoRaWAN;
2. interpretar as informações do dispositivo;
3. associar os dados a uma entidade FIWARE;
4. atualizar o Orion Context Broker.

O dispositivo é registrado por meio da API do IoT Agent.

---

## 🗃️ MongoDB

O MongoDB é utilizado pelos componentes FIWARE para armazenamento interno.

Neste projeto ele é utilizado pelo Orion Context Broker e pelo IoT Agent para manter informações necessárias ao funcionamento desses serviços.

---

## 📦 Cygnus

O **Cygnus** funciona como um conector entre o Orion Context Broker e sistemas de persistência.

Neste projeto, ele recebe notificações do Orion e envia os dados para o PostgreSQL.

---

## 🐘 PostgreSQL

O PostgreSQL funciona como banco de dados histórico.

Enquanto o Orion mantém o estado atual das entidades, o PostgreSQL permite armazenar o histórico das informações recebidas.

---

## 📊 Grafana

O Grafana é utilizado para visualizar os dados persistidos no PostgreSQL.

Por exemplo, para a estação de monitoramento de qualidade do ar, dashboards contendo as informações a seguir podem ser construídos:

* temperatura;
* umidade;
* CO;
* NO₂;
* SO₂;
* OX;
* outros atributos disponibilizados pelo dispositivo.

---

## 🤖 Módulo de IA opcional

O módulo de IA é uma camada adicional da arquitetura.

Ele não é necessário para:

* receber dados LoRaWAN;
* utilizar o Orion;
* persistir dados;
* utilizar PostgreSQL;
* criar dashboards no Grafana.

Sua finalidade é permitir processamento inteligente dos dados recebidos pelo Orion.

No exemplo implementado neste projeto, o modelo de Machine Learning recebe informações relacionadas à concentração de CO, temperatura e umidade e produz o atributo:

```text
CO_Corrigido
```

---

# 🔧 Pré-requisitos

## 🐳 Docker e Docker Compose

Todos os serviços principais são executados utilizando Docker.

Verifique a instalação:

```bash
docker version
docker compose version
```

O Docker Compose deve estar disponível como:

```bash
docker compose
```

> [!NOTE]
> Este projeto utiliza **Docker Engine + Docker Compose**. Não é necessário utilizar Docker Desktop em ambientes Linux.

---

## 🌐 Conta no The Things Stack

Para utilizar o exemplo com TTN, é necessário possuir:

* uma conta no The Things Stack;
* uma Application;
* um End Device;
* uma configuração LoRaWAN funcional;
* uma API Key apropriada;
* acesso às informações MQTT da aplicação.

O The Things Stack disponibiliza os dados da camada de aplicação por MQTT e utiliza API Keys para autenticação.

---

# 🚀 Instalação do Projeto

Clone o repositório:

```bash
git clone https://github.com/NearDeathMetal/fiware-project.git
```

Entre no projeto:

```bash
cd fiware-project
```

A estrutura principal é:

```text
fiware-project/
├── docker/
├── ml/
├── scripts/
├── datasets/
├── .env.example
├── .gitignore
└── README.md
```

---

# 🔐 Configuração das variáveis de ambiente

O projeto disponibiliza o arquivo:

```text
.env.example
```

Crie sua configuração local:

```bash
cp .env.example .env
```

Edite:

```bash
nano .env
```

### Descrição das principais variáveis

| Variável              | Finalidade                                     |
| --------------------- | ---------------------------------------------- |
| `SERVICE_PATH`        | Caminho FIWARE utilizado no Orion              |
| `ENTITY_NAME`         | Nome da entidade criada no Orion               |
| `DEVICE_ID`           | Identificador do dispositivo                   |
| `APP_EUI`             | Identificador da aplicação LoRaWAN             |
| `DEV_EUI`             | Identificador do dispositivo                   |
| `APPLICATION_ID`      | Identificador da aplicação no TTN              |
| `APPLICATION_KEY`     | Chave utilizada na configuração do dispositivo |
| `APP_SERVER_HOST`     | Endereço do servidor MQTT                      |
| `APP_SERVER_USERNAME` | Usuário MQTT                                   |
| `APP_SERVER_PASSWORD` | Senha/API Key MQTT                             |
| `PROVIDER`            | Provedor LoRaWAN                               |
| `DATA_MODEL`          | Modelo de dados utilizado pelo IoT Agent       |

> [!IMPORTANT]
> **Nunca publique o arquivo `.env`.**
>
> O arquivo `.env.example` contém apenas a estrutura das variáveis e não deve conter credenciais reais.

---

# 📡 Configuração da aplicação LoRaWAN no TTN

No The Things Stack:

1. Acesse **Applications**.
2. Selecione sua aplicação.
3. Acesse **End devices**.
4. Selecione o dispositivo.
5. Identifique:

   * Device ID;
   * AppEUI;
   * DevEUI.
  
![Informações da aplicação no TTN](docs/img/ttn-data1.png)

**Figura 2 — Informações da aplicação utilizadas na configuração do `.env`.**

Essas informações devem ser utilizadas na configuração do projeto.

---

# 🔗 Configuração MQTT

Na aplicação TTN, acesse a seção de integração MQTT.

Obtenha:

* endereço do servidor MQTT;
* usuário;
* API Key.

Para o The Things Stack, o formato do usuário pode utilizar o identificador da aplicação e o tenant, por exemplo:

```text
minha-aplicacao@ttn
```

A documentação atual do The Things Stack recomenda gerar uma API Key específica para a aplicação e copiar a chave no momento de sua criação, pois ela não fica disponível posteriormente na interface.

Preencha:

```bash
APP_SERVER_HOST="..."
APP_SERVER_USERNAME="..."
APP_SERVER_PASSWORD="..."
```

![Informações do dispositivo no TTN](docs/img/ttn-data2.png)

**Figura 3 — Informações do dispositivo utilizadas na configuração do `.env`.**

---

# 📤 Registro do dispositivo no IoT Agent

Depois de configurar o `.env`, registre o dispositivo no IoT Agent.

O projeto disponibiliza o script:

```bash
scripts/registerLoraDevice.sh
```

Execute:

```bash
bash ./scripts/registerLoraDevice.sh
```

Se necessário:

```bash
chmod +x ./scripts/registerLoraDevice.sh
```

e:

```bash
./scripts/registerLoraDevice.sh
```

O registro associa:

```text
Dispositivo LoRaWAN
        │
        ▼
   IoT Agent
        │
        ▼
Entidade FIWARE
        │
        ▼
Orion Context Broker
```

Os atributos registrados devem corresponder aos dados efetivamente enviados pelo dispositivo.

> [!NOTE]
> O conjunto de atributos apresentado no script de exemplo corresponde à estação de monitoramento utilizada neste projeto. Para outro dispositivo, adapte a lista de atributos conforme o payload e o modelo utilizado.

---

# 🚀 Inicialização dos serviços

A partir do diretório do projeto:

```bash
cd docker
docker compose up -d
```

Verifique:

```bash
docker compose ps
```

Também é possível verificar todos os contêineres:

```bash
docker ps
```

Para analisar logs:

```bash
docker compose logs --tail=100
```

Para um serviço específico:

```bash
docker compose logs --tail=100 iotagent-lora
```

---

# 🔄 Fluxo dos dados

Após a configuração, o fluxo principal é:

```text
Sensor
  │
  ▼
LoRaWAN
  │
  ▼
The Things Stack
  │
  │ MQTT
  ▼
IoT Agent LoRaWAN
  │
  │ NGSI
  ▼
Orion Context Broker
  │
  ├──────────────► Cygnus
  │                   │
  │                   ▼
  │               PostgreSQL
  │                   │
  │                   ▼
  │                Grafana
  │
  └──────────────► IA opcional
                      │
                      ▼
                  CO_Corrigido
```

O Orion é, portanto, o ponto central de distribuição dos dados de contexto.

---

# 📦 Persistência com Cygnus e PostgreSQL

Para armazenar o histórico das informações, é necessário criar uma Subscription no Orion.

Exemplo:

```bash
curl -iX POST \
  'http://localhost:1026/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' \
  -d '{
    "description": "Notify Cygnus Postgres of context changes",
    "subject": {
      "entities": [
        {
          "idPattern": ".*"
        }
      ]
    },
    "notification": {
      "http": {
        "url": "http://cygnus:5055/notify"
      }
    },
    "throttling": 5
  }'
```

Para formatar uma resposta JSON:

```bash
curl ... | jq
```

O projeto também disponibiliza:

```bash
bash ./scripts/CygnusSubscription.sh
```

Para verificar as inscrições:

```bash
bash ./scripts/SubscriptionVerification.sh
```

---

# 🐘 PostgreSQL

Entre no PostgreSQL:

```bash
docker exec -it db-postgres psql -U postgres -d postgres
```

Liste os bancos:

```sql
\list
```

Liste os schemas:

```sql
\dn
```

Liste as tabelas do schema `openiot`:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema = 'openiot'
ORDER BY table_schema, table_name;
```

Exemplo de consulta:

```sql
SELECT *
FROM openiot.airquality_sensorcvel_loradevice
LIMIT 10;
```

Para consultar determinado atributo:

```sql
SELECT recvtime, attrvalue
FROM openiot.airquality_sensorcvel_loradevice
WHERE attrname = 'Best_CO'
ORDER BY recvtime DESC
LIMIT 10;
```

Para sair:

```sql
\q
```

---

# 📊 Grafana — Visualização dos dados

O Grafana é disponibilizado pelo Docker.

Acesse:

```text
http://localhost:3003
```

A configuração inicial depende das credenciais definidas no ambiente.

---

## 🔌 Configurando o PostgreSQL no Grafana

No Grafana:

**Connections → Data Sources → Add data source → PostgreSQL**

Utilize, conforme a configuração do Compose:

```text
Host: postgres-db:5432
Database: postgres
User: postgres
Password: <senha configurada>
TLS/SSL Mode: disable
```

Clique em:

```text
Save & Test
```

---

## 📈 Criando um painel

Crie um dashboard:

**Dashboards → New → Add visualization**

Selecione a fonte PostgreSQL.

Exemplo de consulta:

```sql
SELECT
    recvtime::timestamp AS "time",
    NULLIF(attrvalue, 'null')::float AS "CO"
FROM
    openiot.airquality_sensorcvel_loradevice
WHERE
    attrname = 'Best_CO'
ORDER BY
    "time" ASC;
```

O mesmo princípio pode ser aplicado aos demais atributos.

---

# 🤖 Módulo Opcional de Inteligência Artificial

> [!IMPORTANT]
> **Esta seção é opcional.**
>
> A integração FIWARE + LoRaWAN pode funcionar normalmente sem instalar ou executar o módulo de IA.
>
> O módulo deve ser implementado somente quando houver necessidade de realizar processamento adicional dos dados utilizando Machine Learning.

---

## 🎯 Objetivo

O módulo de IA foi desenvolvido para utilizar os dados disponibilizados pelo Orion Context Broker e gerar uma estimativa adicional denominada:

```text
CO_Corrigido
```

O módulo utiliza um modelo de **Random Forest Regressor** treinado previamente.

A API recebe uma notificação do Orion, extrai os atributos necessários, executa o modelo e atualiza novamente a entidade no Orion.

---

# 🏗️ Arquitetura da IA

A integração opcional adiciona o seguinte fluxo:

```text
                    Orion Context Broker
                           │
                           │ Subscription
                           ▼
                    ┌───────────────┐
                    │    ml-api     │
                    │    FastAPI    │
                    └───────┬───────┘
                            │
                            ▼
                    RandomForestRegressor
                            │
                            ▼
                       CO_Corrigido
                            │
                            │ PATCH
                            ▼
                    Orion Context Broker
```

O Orion continua sendo o responsável pelo gerenciamento da entidade.

A API de IA apenas processa a informação e devolve o resultado ao Orion.

---

# 📁 Estrutura do módulo

O módulo encontra-se no diretório:

```text
ml/
├── Dockerfile
├── docker.sh
├── requirements.txt
├── RF_Regressor.joblib
└── app/
    ├── main.py
    ├── model-ml.py
    ├── Air_Quality_Analysis_IAG_and_Envcity_Data.py
    ├── NotebookMatheus2023.ipynb
    ├── RF_Regressor.joblib
    └── envcity_df_sp_dataset_2023.csv
```

### Componentes principais

| Arquivo                          | Função                                   |
| -------------------------------- | ---------------------------------------- |
| `Dockerfile`                     | Define a imagem Docker da API            |
| `requirements.txt`               | Dependências Python                      |
| `main.py`                        | API FastAPI e integração com Orion       |
| `RF_Regressor.joblib`            | Modelo treinado                          |
| `model-ml.py`                    | Processo relacionado ao treinamento      |
| `NotebookMatheus2023.ipynb`      | Análise e desenvolvimento do modelo      |
| `envcity_df_sp_dataset_2023.csv` | Dataset utilizado no treinamento/análise |

---

# 🧠 Modelo de Machine Learning

O modelo utilizado atualmente é:

```text
RandomForestRegressor
```

As variáveis utilizadas na entrada do modelo são:

```text
e2sp_co
e2sp_co_we
e2sp_co_ae
e2sp_temp
pin_umid
```

Correspondência conceitual:

| Entrada      | Origem        |
| ------------ | ------------- |
| `e2sp_co`    | `Best_CO`     |
| `e2sp_co_we` | `CO_WE`       |
| `e2sp_co_ae` | `CO_AE`       |
| `e2sp_temp`  | `Temperatura` |
| `pin_umid`   | `Umidade`     |

O modelo retorna:

```text
CO_Corrigido
```

---

# 🐳 Construção da imagem Docker

O módulo possui seu próprio `Dockerfile`.

Entre no diretório:

```bash
cd ml
```

Construa a imagem:

```bash
docker build -t ml-api .
```

Para uma reconstrução completa:

```bash
docker build --no-cache -t ml-api .
```

O `Dockerfile` instala as dependências e copia o modelo e o código da aplicação.

A aplicação é executada com:

```text
uvicorn app.main:app
```

na porta:

```text
8000
```

---

# 🚀 Inicialização do serviço de IA

O `docker-compose.yml` possui um serviço específico para a API de Machine Learning.

A imagem utilizada atualmente é:

```yaml
image: ml-api
```

Depois de construir a imagem:

```bash
cd docker
docker compose up -d ml-api
```

Verifique:

```bash
docker compose ps
```

Para consultar os logs:

```bash
docker logs ml-api
```

ou, dependendo do nome atribuído pelo Compose:

```bash
docker compose logs --tail=100 ml-api
```

---

# 🔎 Verificação da API

A API FastAPI disponibiliza documentação automática.

Acesse:

```text
http://localhost:8000/docs
```

Também é possível verificar os endpoints publicados:

```bash
curl -s http://localhost:8000/openapi.json | jq
```

Entre os endpoints disponíveis estão:

```text
/
 /notifyCO
 /orion/entities
 /orion/status
 /orion/subscribe
 /prediction
```

---

# 🔔 Subscription do Orion para a IA

A API de IA não precisa consultar o Orion continuamente.

Em vez disso, utiliza uma **Subscription**.

O Orion envia uma notificação para:

```text
http://ml-api:8000/notifyCO
```

A Subscription deve monitorar os atributos necessários ao modelo:

```text
Best_CO
CO_WE
CO_AE
Temperatura
Umidade
```

Um exemplo conceitual:

```json
{
  "description": "Subscribe to LoraDevice updates",
  "subject": {
    "entities": [
      {
        "idPattern": ".*",
        "type": "LoraDevice"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://ml-api:8000/notifyCO",
      "attrs": [
        "Best_CO",
        "CO_WE",
        "CO_AE",
        "Temperatura",
        "Umidade"
      ]
    }
  },
  "throttling": 0
}
```

> [!NOTE]
> A configuração exata da entidade e do `fiware-servicepath` deve corresponder à configuração utilizada pelo seu dispositivo.

---

# 🔄 Fluxo de processamento da IA

Quando o Orion recebe uma alteração em um dos atributos monitorados:

```text
Best_CO
CO_WE
CO_AE
Temperatura
Umidade
```

a Subscription envia uma notificação para:

```text
POST /notifyCO
```

A API:

1. recebe a notificação;
2. identifica a entidade;
3. extrai os cinco atributos;
4. monta o vetor de entrada;
5. executa o Random Forest;
6. obtém `CO_Corrigido`;
7. atualiza a entidade no Orion.

Conceitualmente:

```text
Notificação Orion
       │
       ▼
   /notifyCO
       │
       ▼
Extração dos atributos
       │
       ▼
DataFrame
       │
       ▼
RandomForestRegressor
       │
       ▼
CO_Corrigido
       │
       ▼
PATCH /v2/entities/<id>/attrs
       │
       ▼
Orion
```

---

# 🧪 Testando a IA

Depois que o serviço estiver funcionando, pode-se alterar um atributo monitorado da entidade.

Por exemplo:

```bash
curl -iX PATCH \
  'http://localhost:1026/v2/entities/sensores-novos/attrs' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' \
  -d '{
    "Best_CO": {
      "type": "Number",
      "value": 60
    }
  }'
```

Depois consulte a entidade:

```bash
curl -s \
  'http://localhost:1026/v2/entities/sensores-novos' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' | jq
```

O resultado esperado é a presença do atributo:

```json
"CO_Corrigido": {
  "type": "Number",
  "value": 2.515522
}
```

O valor exato depende dos dados utilizados pelo modelo.

---

# 🔁 Evitando loop de notificações

O atributo:

```text
CO_Corrigido
```

não deve fazer parte da condição da Subscription utilizada para disparar a IA.

O fluxo correto é:

```text
Best_CO
CO_WE
CO_AE
Temperatura
Umidade
       │
       ▼
      IA
       │
       ▼
CO_Corrigido
```

e não:

```text
CO_Corrigido
       │
       ▼
      IA
       │
       ▼
CO_Corrigido
       │
       ▼
      ...
```

Essa separação evita que a própria saída do modelo gere novas execuções desnecessárias.

---

# 🧩 IA como componente opcional

A arquitetura permite utilizar o projeto em diferentes níveis.

### Nível 1 — Integração básica

```text
LoRaWAN
   ↓
TTN
   ↓
IoT Agent
   ↓
Orion
```

### Nível 2 — Persistência

```text
LoRaWAN
   ↓
TTN
   ↓
IoT Agent
   ↓
Orion
   ↓
Cygnus
   ↓
PostgreSQL
```

### Nível 3 — Visualização

```text
PostgreSQL
     ↓
  Grafana
```

### Nível 4 — Inteligência Artificial

```text
Orion
  ↓
ML API
  ↓
Random Forest
  ↓
CO_Corrigido
  ↓
Orion
```

Dessa forma, o usuário pode implementar somente os componentes necessários ao seu projeto.

---

# 🧪 Verificação e diagnóstico

## Verificar os contêineres

```bash
docker ps
```

ou:

```bash
docker compose ps
```

## Ver logs

```bash
docker compose logs --tail=100
```

## Logs do IoT Agent

```bash
docker compose logs --tail=100 iotagent-lora
```

## Logs do Orion

```bash
docker compose logs --tail=100 orion
```

## Logs do Cygnus

```bash
docker compose logs --tail=100 cygnus
```

## Logs da IA

```bash
docker compose logs --tail=100 ml-api
```

---

# 🔎 Consultando uma entidade no Orion

Exemplo:

```bash
curl -s \
  'http://localhost:1026/v2/entities/sensores-novos' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' | jq
```

Para consultar todas:

```bash
curl -s \
  'http://localhost:1026/v2/entities' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' | jq
```

---

# 🔔 Consultando as Subscriptions

```bash
curl -s \
  'http://localhost:1026/v2/subscriptions' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /airQuality' | jq
```

Essa consulta permite verificar se as inscrições do Cygnus e da IA estão ativas.

---

# 🔒 Segurança

As credenciais do TTN não devem ser armazenadas diretamente no código.

Utilize:

```text
.env
```

para informações sensíveis.

O arquivo:

```text
.env.example
```

deve conter somente os nomes das variáveis e valores de exemplo vazios.

Nunca publique:

* API Keys;
* senhas MQTT;
* tokens;
* credenciais do banco;
* credenciais do Grafana;
* chaves de dispositivos.

> [!WARNING]
> Uma API Key do The Things Stack deve ser tratada como uma credencial. Crie chaves com somente as permissões necessárias ao serviço que irá utilizá-las.

---

# 📌 Observações sobre o módulo de IA

O módulo de Machine Learning deste projeto possui três responsabilidades principais:

```text
1. Receber dados do Orion
2. Executar o modelo
3. Atualizar o Orion
```

Ele **não substitui**:

* o IoT Agent;
* o Orion;
* o Cygnus;
* o PostgreSQL;
* o Grafana.

Também não é necessário instalar o módulo de IA para utilizar o restante da arquitetura.

Sua adoção depende do objetivo da aplicação.

---

# 🧠 Considerações finais

Este projeto demonstra uma arquitetura modular para aplicações IoT utilizando:

* **LoRaWAN** para comunicação com os dispositivos;
* **The Things Stack** para gerenciamento da aplicação LoRaWAN;
* **IoT Agent LoRaWAN** para integração com FIWARE;
* **Orion Context Broker** para gerenciamento de contexto;
* **MongoDB** para armazenamento interno dos componentes FIWARE;
* **Cygnus** para persistência;
* **PostgreSQL** para armazenamento histórico;
* **Grafana** para visualização;
* **FastAPI + Random Forest** para processamento inteligente opcional.

A principal característica da arquitetura é sua modularidade.

O fluxo fundamental permanece:

```text
Dispositivo
    ↓
LoRaWAN
    ↓
TTN
    ↓
IoT Agent
    ↓
Orion
```

A partir do Orion, outros serviços podem ser adicionados conforme a necessidade:

```text
                  ┌──► Cygnus ──► PostgreSQL ──► Grafana
                  │
Dispositivo ──► Orion
                  │
                  └──► IA ──► CO_Corrigido
```

Assim, a Inteligência Artificial é tratada como uma **extensão da arquitetura**, e não como requisito para a integração FIWARE + LoRaWAN.

---

# 📚 Referências

* [FIWARE](https://www.fiware.org/)
* [FIWARE Orion Context Broker](https://fiware-orion.readthedocs.io/)
* [FIWARE IoT Agent LoRaWAN](https://fiware-lorawan.readthedocs.io/)
* [The Things Stack](https://www.thethingsindustries.com/)
* [Docker](https://www.docker.com/)
* [PostgreSQL](https://www.postgresql.org/)
* [Grafana](https://grafana.com/)
* [FastAPI](https://fastapi.tiangolo.com/)
* [Scikit-learn](https://scikit-learn.org/)

---

## 🤝 Comunidade e colaboração

Caso encontre problemas ou tenha sugestões:

* abra uma Issue no repositório;
* proponha melhorias por Pull Request;
* consulte a documentação oficial dos componentes utilizados;
* contribua com exemplos e melhorias para a documentação.

Este projeto foi desenvolvido como parte de atividades de pesquisa e desenvolvimento relacionadas à integração de tecnologias IoT, FIWARE, LoRaWAN e Inteligência Artificial.
