# lev

Documentação Técnica: Arquitetura e Modelagem (AWS & Athena)

Esta documentação detalha a estrutura de dados implementada no AWS Athena para o projeto de análise de vendas, utilizando o modelo Star Schema (Modelo Estrela) e o formato Parquet para otimização de custos e performance.

1. Arquitetura de Dados

A solução segue os princípios de um Modern Data Lakehouse:

Bronze (Raw): Dados brutos em CSV carregados no S3.

Gold (Curated): Tabelas dimensionais e de factos em formato Parquet, tratadas para integridade referencial.

2. Diagrama Entidade-Relacionamento (ERD)

O diagrama abaixo representa a modelagem final consumida pelo Power BI.

erDiagram
    FACT_SALES ||--o{ DIM_CUSTOMER : "links via customer_id"
    FACT_SALES ||--o{ DIM_PRODUCT : "links via product_id"
    FACT_SALES ||--o{ DIM_LOCATION : "links via postal_code"

    FACT_SALES {
        int row_id PK
        string order_id
        date order_date
        string customer_id FK
        string product_id FK
        string postal_code FK
        double sales
        int quantity
        double discount
        double profit
    }

    DIM_CUSTOMER {
        string customer_id PK
        string customer_name
        string segment
    }

    DIM_PRODUCT {
        string product_id PK
        string product_name
        string category
        string sub_category
    }

    DIM_LOCATION {
        string postal_code PK
        string city
        string state
        string region
        string country
    }


3. Scripts DDL (AWS Athena)

3.1. Criação do Banco de Dados

CREATE DATABASE IF NOT EXISTS levdb;


3.2. Tabela Raw (Bronze)

Mapeamento direto do CSV original no S3.

CREATE EXTERNAL TABLE IF NOT EXISTS levdb.raw_sales (
  row_id INT,
  order_id STRING,
  order_date STRING,
  ship_date STRING,
  ship_mode STRING,
  customer_id STRING,
  customer_name STRING,
  segment STRING,
  country STRING,
  city STRING,
  state STRING,
  postal_code STRING,
  region STRING,
  product_id STRING,
  category STRING,
  sub_category STRING,
  product_name STRING,
  sales DOUBLE,
  quantity INT,
  discount DOUBLE,
  profit DOUBLE
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 's3://teste-lev-2026/raw/sales/'
TBLPROPERTIES ('skip.header.line.count'='1');


3.3. Tabela Dimensional: Clientes

CREATE TABLE levdb.dim_customer
WITH (format = 'PARQUET', external_location = 's3://teste-lev-2026/gold/dim_customer/') AS
SELECT DISTINCT
    customer_id,
    customer_name,
    segment
FROM levdb.raw_sales;


3.4. Tabela Dimensional: Produtos (Deduplicada)

Lógica aplicada para garantir um único nome por ID de produto.

CREATE TABLE levdb.dim_product
WITH (format = 'PARQUET', external_location = 's3://teste-lev-2026/gold/dim_product/') AS
SELECT product_id, product_name, category, sub_category
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY product_name) as rn
    FROM (SELECT DISTINCT product_id, product_name, category, sub_category FROM levdb.raw_sales)
) WHERE rn = 1;


3.5. Tabela Dimensional: Localidade (Deduplicada)

Tratamento para inconsistências de CEP/Cidade.

CREATE TABLE levdb.dim_location
WITH (format = 'PARQUET', external_location = 's3://teste-lev-2026/gold/dim_location/') AS
SELECT postal_code, city, state, region, country
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY postal_code ORDER BY city) as rn
    FROM (SELECT DISTINCT postal_code, city, state, region, country FROM levdb.raw_sales)
) WHERE rn = 1;


3.6. Tabela de Factos: Vendas

CREATE TABLE levdb.fact_sales
WITH (format = 'PARQUET', external_location = 's3://teste-lev-2026/gold/fact_sales/') AS
SELECT
    row_id,
    order_id,
    CAST(parse_datetime(order_date, 'MM/d/yyyy') AS DATE) as order_date,
    customer_id,
    product_id,
    postal_code,
    sales,
    quantity,
    discount,
    profit
FROM levdb.raw_sales;

4. Dicionário de Medidas (DAX)

Query no DAX STUDIO:

SELECT
  [Name]        AS [Medida],
  [Expression]  AS [DAX],
  [Description] AS [Descricao]
FROM
  SYSTEM.TMSCHEMA_MEASURES
ORDER BY
  [Name]
  
Abaixo estão listadas as principais métricas implementadas no Power BI para suporte à análise de negócio.

Nome da Medida
Fórmula DAX

Lucro Total
SUM ( fact_sales[profit])

Total Vendas
SUM ( fact_sales[Sales] )

Margem de Lucro
DIVIDE ( [Lucro Total], [Total Vendas] )

Número de Pedidos
DISTINCTCOUNT ( fact_sales[order_id] )

Quantidade Vendida
SUM ( fact_sales[quantity])

Percentual de Desconto
AVERAGE ( fact_sales[discount] )

Lucro com Prejuízo
CALCULATE ( [Lucro Total], FILTER ( fact_sales, fact_sales[profit] < 0 ) )

Vendas com Prejuízo
CALCULATE ( [Total Vendas], FILTER ( fact_sales, fact_sales[profit] < 0 ) )

Percentual Vendas com Prejuízo
DIVIDE ( [Vendas com Prejuízo], [Total Vendas] )

Margem Apenas Prejuízo
DIVIDE ( [Lucro com Prejuízo], [Vendas com Prejuízo] )

Rank Cliente por Lucro
RANKX ( ALL ( dim_customer[customer_id] ), [Lucro Total], , DESC, DENSE )

Rank Cliente por Vendas
RANKX ( ALL ( dim_customer[customer_id] ), [Total Vendas], , DESC, DENSE )

Lucro Total Top 10 Clientes
IF ( [Rank Cliente por Lucro] <= 10, [Lucro Total], BLANK() )

Total Vendas Top 10 Clientes
IF ( [Rank Cliente por Vendas] <= 10, [Total Vendas] )

5. Notas de Qualidade de Dados

Conversão de Tipos: A coluna order_date foi convertida de String para Date para permitir inteligência de tempo no Power BI.

Integridade Referencial: Foram removidas duplicatas das dimensões que causavam fan-out (multiplicação de linhas) nas métricas financeiras.

Performance: O uso de Parquet resultou numa redução de aproximadamente 80% no volume de dados lidos pelo Athena em comparação com o CSV original.
