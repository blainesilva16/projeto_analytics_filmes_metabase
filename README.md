# Projeto de Engenharia de Dados utilizando dados sobre filmes no Metabase

Projeto para o desafio 1 do DPT, que consiste em uma análise exploratória de um dataset de filmes

## 🏗️ Arquitetura do Pipeline de Dados

Para este desafio, foi implementada uma arquitetura de Modern Data Stack focada em escalabilidade e custo-benefício.

- Ingestão de Dados: Carga de arquivos brutos (CSV) para o Google BigQuery.
- Data Warehouse (Storage): Camada de dados brutos e persistência no BigQuery.
- Transformação (Modeling): Criação de Views Analíticas em SQL para limpeza, desnormalização e cálculo de métricas.
- Infraestrutura Cloud: Deploy de uma instância AWS EC2 (t3.xlarge).
- Containers: Orquestração do Metabase via Docker para garantir portabilidade e isolamento.
- Visualização (BI): Conexão segura via Service Account entre Metabase e BigQuery para dashboards em tempo real.

## 🛠️ Passos de Execução

### 1. Configuração do Data Warehouse

- Criação do Dataset no BigQuery.
- Carga dos arquivos belief_data.csv, movie_elicitation_set.csv, movies.csv, ratings_for_additional_users.csv, user_rating_history.csv e user_recommendation_history.csv, todos baixados do Grouplens.
- Criação de uma Service Account no GCP com permissões de BigQuery Data Viewer, BigQuery Job User e BigQuery Metadata Viewer.

### 2. Infraestrutura AWS

- Provisionamento de instância EC2 rodando Amazon Linux/Ubuntu.
- Configuração de Security Groups permitindo tráfego na porta 3000 (Metabase).
- Instalação do Docker e execução da imagem oficial:

```bash
docker run -d -p 3000:3000 --name metabase metabase/metabase
```

### 3. Criação de Tabelas Externas

- Utilização de um código Python para extrair as informações dos arquivos csv do GCS para criar tabelas externas onde todos os dados eram do tipo STRING

### 4. Criação de Tabelas Analíticas e de Views

Execução de scripts SQL para criar a camada de consumo (analytics_dataset), como:

- fact_ratings: Unificação e limpeza de tipos de dados.
- vw_user_activity: Métricas de retenção e churn.
- vw_ratings_heatmap: Agregação temporal para análise de comportamento.

### 5. Visualização de Dados com Metabase

A etapa final do projeto consistiu em transformar as views analíticas em um Dashboard interativo, contendo as seguintes características:

1. Construção das "Questions" (Perguntas)
Cada visualização no dashboard nasceu de uma "Question", utilizando uma mistura de interface No-Code e Consultas Nativas (SQL):

- Análise de Coorte e Churn: Utilização do Editor SQL para calcular a taxa de retenção ano a ano, unindo as métricas de primeira e última avaliação dos usuários através de CTEs (Common Table Expressions).
- Heatmap de Engajamento: Configuração de uma Pivot Table (Tabela Dinâmica) cruzando dias da semana e faixas de horário, com aplicação de formatação condicional (escala de cores) para identificar picos de tráfego.
- Distribuição de Qualidade (Scatter Plot): Mapeamento de milhares de filmes em um gráfico de dispersão para identificar a correlação entre o volume de votos e a nota média, permitindo separar "Blockbusters" de "Filmes Cult".

2. Layout do Dashboard
   
O dashboard foi projetado seguindo princípios de Data Storytelling:

- Camada de Resumo (Topo): KPIs de alto nível (Rating Médio Global, Total de Filmes e Volume de Avaliações) para uma visão imediata da saúde da plataforma.
- Camada de Comportamento (Meio): O Heatmap e o gráfico de Entrada de Usuários, respondendo "Quem são nossos usuários e quando eles estão ativos?".
- Camada de Performance de Conteúdo (Base): Top Gêneros e o Scatter Plot de popularidade, respondendo "Quais conteúdos performam melhor?".

3. Diferenciais Técnicos Aplicados

- Sincronização de Metadados: Configuração de varredura (scan) para garantir que as alterações de tipos de dados no BigQuery fossem refletidas instantaneamente na interface.
- Otimização de Performance: Uso de filtros de agregação diretamente na interface do Metabase para reduzir o processamento de dados desnecessários na AWS EC2.

## 📊 Queries Principais
Aqui vale destacar a query de Métricas de Engajamento e Churn que utilizei para um gráfico no dashboard:

```bash
WITH base AS (
  SELECT 
    EXTRACT(YEAR FROM primeira_avaliacao) AS ano_entrada,
    EXTRACT(YEAR FROM ultima_avaliacao) AS ano_saida
  FROM `projeto-id.analytics_dataset.vw_user_activity`
),
entradas AS (
  SELECT ano_entrada AS ano, COUNT(*) AS qtd_primeiras
  FROM base GROUP BY 1
),
saidas AS (
  SELECT ano_saida AS ano, COUNT(*) AS qtd_ultimas
  FROM base GROUP BY 1
),
metricas_unificadas AS (
  SELECT 
    COALESCE(e.ano, s.ano) AS ano_referencia,
    IFNULL(e.qtd_primeiras, 0) AS entradas,
    IFNULL(s.qtd_ultimas, 0) AS saidas
  FROM entradas e
  FULL OUTER JOIN saidas s ON e.ano = s.ano
)
SELECT 
  ano_referencia,
  entradas,
  saidas,
  -- Cálculo de Churn Simples: (Saídas / Total de Entradas acumuladas até o ano)
  ROUND(SAFE_DIVIDE(saidas, SUM(entradas) OVER (ORDER BY ano_referencia)), 4) AS taxa_churn
FROM metricas_unificadas
ORDER BY ano_referencia ASC
```

## 📈 Dashboard de Insights

Alguns insights do dashboard:

### 📊 Principais Visualizações e Insights

| Visualização | Descrição | Arquivo |
| :--- | :--- | :--- |
| **Heatmap de Engajamento** | Picos de acesso por dia e hora (Pico: Sáb/Dom 15h-18h). | `img/heatmap.png` |
| **Scatter Plot** | Correlação Popularidade vs Qualidade. | `img/scatter.png` |
| **Entrada vs Churn** | Crescimento de base e retenção anual. | `img/user_activity.png` |

## 💡 Principais Descobertas
- Janela de Ouro: O maior volume de engajamento ocorre aos sábados e domingos entre 15h e 18h.
- Convergência de Notas: Filmes com mais de 2.000 avaliações tendem a estabilizar a nota média entre 3.5 e 4.2, enquanto filmes de baixo volume apresentam alta volatilidade (outliers).
- Boom de Usuários: Identificado um crescimento exponencial de novos usuários a partir de 2021, com uma taxa de retenção entre 36% e 64%.
