
# 🍺 Projeto de Ingestão e Modelagem de Dados - Open Brewery DB

Este projeto realiza a ingestão de dados da API pública [Open Brewery DB](https://www.openbrewerydb.org/), salvando os dados brutos em PostgreSQL e posteriormente no catálogo do Databricks, seguindo a arquitetura Medallion (camadas bronze, silver e gold). O pipeline está versionado no GitHub e executado por Jobs no Databricks.

---

## ⚙️ Etapa 1 - Infraestrutura local com Docker Compose

Utilizamos um `docker-compose.yml` para levantar dois serviços:

- **PostgreSQL**: para persistência inicial dos dados brutos
- **Jupyter Notebook**: ambiente de desenvolvimento em Python

### 🔧 Como subir:

```bash
docker compose up -d
```

---

## 🔄 Etapa 2 – Ingestão com Airbyte (via Docker e `abctl`)

Para facilitar testes locais e reproduzir o pipeline de ingestão de forma prática, o **Airbyte** foi configurado de duas formas distintas:

### 🔹 1. Acesso à documentação oficial

O Airbyte Open Source pode ser explorado em [https://airbyte.com/product/airbyte-open-source](https://airbyte.com/product/airbyte-open-source).

---

### 🔹 2. Execução via Docker (manual)

O Airbyte também foi executado manualmente via Docker (sem uso de `docker-compose`).  
Este modo permite rodar os containers do Airbyte diretamente com o Docker Desktop, sem acoplamento à infraestrutura principal do projeto.

### 🔹 3. Execução via `abctl` (linha de comando)

Seguindo a [documentação oficial](https://docs.airbyte.com/deploying-airbyte/on-your-computer/abctl), o Airbyte também foi instalado via CLI (`abctl`), permitindo uma forma prática e controlada de gerenciar instâncias locais:

#### 📦 Passos principais:

1. **Verificar arquitetura do sistema**  
   Acesse: `Configurações > Sistema > Sobre` e identifique se é `AMD` ou `ARM`.

2. **Download e instalação do `abctl`**  
   Faça o download da versão correta para seu sistema operacional, extraia os arquivos e adicione a pasta ao `PATH`.

3. **Instalação do Airbyte local**  
   Com Docker Desktop aberto, rode:

   ```bash
   abctl local install
   ```

4. **Recuperar credenciais padrão** (usuário/senha para acessar o painel do Airbyte):

   ```bash
   abctl local credentials
   ```

---

### 🔧 Configuração do Airbyte:

- **Fonte (source):** PostgreSQL (Docker local)
- **Destino (destination):** Databricks Delta Lake (Unity Catalog)
- **Tabela destino:** `projto_ambev.public.raw_breweries`

Dessa forma, a ingestão é automatizada e mantida atualizada de forma confiável, servindo como base para o restante do pipeline (bronze → silver → gold).

---

## 🧱 Etapa 3 - Arquitetura Medallion (Bronze, Silver, Gold)

### 🔹 Bronze

- Persistência do JSON bruto vindo do PostgreSQL
- Tabela: `projto_ambev.bronze.breweries_json`

### 🔹 Silver

- Conversão do campo `raw_data` (string JSON) para `STRUCT` usando `from_json`
- Extração dos principais campos: `id`, `name`, `brewery_type`, `city`, `state`, `latitude`, etc.
- Tabela: `projto_ambev.silver.breweries`

### 🔹 Gold

Agregações analíticas:

- Total de cervejarias por estado → `gold.total_por_estado`
- Total por tipo de cervejaria → `gold.total_por_tipo`
- Top 10 cidades com mais cervejarias → `gold.top10_cidades`

---

## 🔄 Etapa 4 - Orquestração com Databricks Jobs

Criamos um Job com múltiplas tasks no Databricks que executa:

1. Notebook Bronze → Silver
2. Notebook Silver → Gold

Também é possível configurar agendamento e alertas via interface gráfica.

---

## 🗂️ Estrutura final no Databricks

```
projto_ambev/
├── public/
│   └── raw_breweries
├── bronze/
│   └── breweries_json
├── silver/
│   └── breweries
└── gold/
    ├── total_por_estado
    ├── total_por_tipo
    └── top10_cidades
```

---

## 💻 Tecnologias utilizadas

- Python 3
- Docker + Docker Compose
- PostgreSQL
- Jupyter Notebook
- Spark (Databricks)
- Delta Lake
- Unity Catalog
- GitHub
- Airbyte
- Databricks Jobs

---

## 📒 Notebooks do Projeto

Este repositório também inclui notebooks utilizados nas etapas do pipeline de dados:

### 🔹 `Extraindo_dados_brutos.ipynb`
📥 Executado localmente no Jupyter Notebook.  
Responsável por extrair dados da API [Open Brewery DB](https://www.openbrewerydb.org/) e inserir os registros no banco **PostgreSQL** que está rodando em um container Docker.

### 🔹 `1_bronze_copia_dados.ipynb`
📦 Notebook executado no **Databricks**.  
Lê os dados do PostgreSQL e os salva no formato Delta Lake, compondo a camada **Bronze**, onde os dados são mantidos em sua forma bruta.

### 🔹 `2_silver_normalizar_dados.ipynb`
🧹 Também executado no **Databricks**.  
Transforma os dados brutos da Bronze, usando funções como `from_json` e `selectExpr`, para extrair e normalizar os principais campos, criando a camada **Silver**.

### 🔹 `3_gold_dados_mensurados.ipynb`
📊 Notebook final executado no **Databricks**.  
Agrega os dados normalizados da camada Silver em métricas e indicadores analíticos, compondo a camada **Gold**, que serve de base para dashboards e consumo analítico.

---

## 🧠 Observações Técnicas - Boas Práticas com Delta Lake no Databricks

Pensando em cenários de Big Data com grandes volumes de dados e múltiplas transformações, algumas boas práticas foram consideradas (ou podem ser implementadas futuramente) no uso do **Delta Lake** no Databricks:

- ✅ **Particionamento de dados**: melhora a performance de leitura e escrita, especialmente em consultas filtradas por colunas temporais como `date`, `ano_mes`, `event_date`, etc.
- 🔁 **Time Travel (`VERSION AS OF`)**: permite acessar versões anteriores da tabela, útil para auditoria, debug e rollback de transformações.
- 🧹 **Vacuum**: remove arquivos antigos e não referenciados para economizar armazenamento e manter a performance.

  ```sql
  VACUUM nome_da_tabela RETAIN 168 HOURS;
  ```

- 📊 **Z-Ordering** (quando aplicável): otimiza a ordenação dos dados internamente, melhorando ainda mais o desempenho de queries.
- 💾 **OPTIMIZE**: compacta pequenos arquivos e melhora o desempenho geral de leitura.

Essas práticas são fundamentais para manter um **data lakehouse saudável, performático e escalável**.

---

## 👨‍💻 Autor

Rafael Carlos dos Santos  
Engenheiro de Dados | [LinkedIn](https://www.linkedin.com/in/rafaelcarlossantos/)
