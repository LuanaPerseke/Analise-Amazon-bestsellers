[Documentação do Projeto Amazon Bestselling Books.md](https://github.com/user-attachments/files/34255764/Documentacao.do.Projeto.Amazon.Bestselling.Books.md)
# Amazon Bestselling Books (2009 - 2019)

Uma solução de Business Intelligence e Engenharia de Dados desenvolvida para analisar o desempenho de vendas, avaliações e precificação dos livros mais vendidos da Amazon. O projeto engloba desde a importação do arquivo bruto (`.csv` via Assistente de Importação de Arquivo Simples do SQL Server), limpeza, tratamento e modelagem dimensional em SQL Server (*Star Schema* com *surrogate key*), até a criação de um Dashboard Executivo e Interativo no Power BI.

## Sumário
- [Visão Geral do Projeto](#visão-geral-do-projeto)
- [Arquitetura de Dados](#arquitetura-de-dados)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Tratamento de Dados, Análise Crítica e Modelagem SQL](#tratamento-de-dados-análise-crítica-e-modelagem-sql)
  1. [Carga Inicial, Higienização com TRIM e Tradução](#1-carga-inicial-higienização-com-trim-e-tradução)
  2. [Criação da Tabela Dimensão (DIM_LIVROS & Surrogate Key)](#2-criação-da-tabela-dimensão-dim_livros--surrogate-key)
  3. [Limpeza e Padronização de Autores (Tratamento de Anomalias)](#3-limpeza-e-padronização-de-autores-tratamento-de-anomalias)
  4. [Atualização e Otimização da Tabela Fato (FATO_BESTSELLERS)](#4-atualização-e-otimização-da-tabela-fato-fato_bestsellers)
- [Dashboard Power BI](#dashboard-power-bi)
- [Como Executar o Projeto](#como-executar-o-projeto)
- [Desenvolvedora](#desenvolvedora)

---

## Visão Geral do Projeto

O objetivo deste projeto é transformar dados brutos dos livros mais vendidos da Amazon (2009 - 2019) em insights estratégicos sobre:

* **Métricas de Desempenho e Vendas:** Total de livros (550), Média de Preço ($13,10), Média da Nota (4,5) e Total de Avaliações (3,4 Mi).
* **Análise de Autores e Bestsellers:** Identificação dos autores mais avaliados (destaque para Paulo Coelho com 36 Mil avaliações) e com maior volume de títulos na lista (Jeff Kinney, Gary Chapman, Rick Riordan e Suzanne Collins).
* **Tendência de Precificação Temporal:** Evolução da média de preço dos livros ao longo dos anos (2009 a 2019).
* **Distribuição por Gênero:** Proporção entre Não Ficção (56,36%) e Ficção (43,64%).

---

## Arquitetura de Dados

```text
[bestsellers with categories.csv] (Raw Data)
       |
       ▼ (Importação via Assistente de Importação de Arquivo Simples / Flat File Import)
[TB_bestsellers]
       |
       ▼ (Higienização com TRIM, Tradução de Gêneros com CASE WHEN & Seleção de Campos)
[FATO_BESTSELLERS]
       |
       ▼ (Tratamento de Anomalias: Deduplicação e Padronização de Nomes de Autores)
[FATO_BESTSELLERS] (Atualizada)
       |
       ▼ (Estruturação do Modelo Relacional Star Schema & Criação de Surrogate Key)
-------------------------------------------------------------------------
                                   |
                                   ▼
[FATO_BESTSELLERS] ───────────► [DIM_LIVROS] (ID_LIVROS, NOME, AUTOR, GENERO)
 (Com ID_LIVROS FK)                (Com ID_LIVROS PK via IDENTITY)
-------------------------------------------------------------------------
       |
       ▼ (Importação e Construção Visual)
[Dashboard Power BI]
```

---

## Tecnologias Utilizadas

* **SQL Server:** Importação de arquivo plano via Assistente de Importação de Arquivo Simples, higienização e remoção de espaços em branco com a função `TRIM`, tradução de categorias com `CASE WHEN`, limpeza de anomalias de texto via `UPDATE`/`REPLACE`, modelagem *Star Schema*, criação de chave primária surrogada com `IDENTITY(INT, 1, 1)`, JOINs relacionais e refatoração de estrutura (`DROP COLUMN`).
* **Power BI:** Design de Dashboard Executivo (Light/Beige Theme com contraste corporativo), medidas DAX, caixa de pesquisa para consulta de autores, cartões de KPI e gráficos interativos.
* **GitHub:** Versionamento de código e documentação completa do projeto.

---

## Tratamento de Dados, Análise Crítica e Modelagem SQL

Abaixo estão os scripts SQL utilizados no projeto. O fluxo seguiu a higienização dos campos com a função `TRIM`, a tradução dos gêneros literários, a criação da chave surrogada (`ID_LIVROS`) e uma etapa de **análise crítica** realizada ao testar os filtros do Power BI, onde identificou-se uma duplicidade nos nomes de autores por falta de padronização na pontuação (ex: `'J.K. Rowling'` vs `'J. K. Rowling'` e `'George R.R. Martin'` vs `'George R. R. Martin'`).

### 1. Carga Inicial, Higienização com TRIM e Tradução

Seleção dos dados da tabela bruta `TB_bestsellers`, aplicando a remoção de espaços em branco antes e depois das strings com `TRIM` nos títulos e autores, e traduzindo a categoria do livro de inglês para português:

```sql
SELECT * FROM TB_bestsellers

SELECT 
	TRIM (name) AS NOME,
	TRIM (Author) AS AUTOR,
	User_Rating AS AVALIACAO_USUARIO,
	Reviews AS TOTAL_AVALIACOES,
	Price AS PRECO,
	Year AS ANO,
	CASE
		WHEN Genre = 'Non Fiction' THEN 'Não Ficção'
		WHEN Genre = 'Fiction' THEN	'Ficção'
	END AS GENERO
INTO FATO_BESTSELLERS
FROM TB_bestsellers
```

### 2. Criação da Tabela Dimensão (DIM_LIVROS & Surrogate Key)

Criação da tabela dimensão de livros com uma chave surrogada autoincrementável (`ID_LIVROS`) gerada a partir de uma subconsulta com registros distintos (`DISTINCT`):

```sql
SELECT
	IDENTITY(INT,1,1) AS ID_LIVROS,
	NOME,
	AUTOR,
	GENERO
INTO DIM_LIVROS
FROM (SELECT DISTINCT NOME, AUTOR, GENERO 
					  FROM FATO_BESTSELLERS) AS SUBCONSULTA
```

### 3. Limpeza e Padronização de Autores (Tratamento de Anomalias)

Durante a exploração visual no Power BI, constatou-se que autores com variações na escrita acumulavam métricas separadamente. Para garantir a integridade dos dados, aplicou-se a padronização diretamente na tabela Fato e a recriação da Dimensão unificada:

```sql
-- Padronização dos nomes de autores com inconsistência de pontuação/espaçamento
UPDATE FATO_BESTSELLERS
SET AUTOR = REPLACE(AUTOR, 'J.K. Rowling', 'J. K. Rowling');

UPDATE FATO_BESTSELLERS
SET AUTOR = REPLACE(AUTOR, 'George R.R. Martin', 'George R. R. Martin');

-- Recriação da DIM_LIVROS unificada com os dados devidamente padronizados
DROP TABLE IF EXISTS DIM_LIVROS;

SELECT 
    IDENTITY(INT, 1, 1) AS ID_LIVROS,
    NOME,
    AUTOR,
    GENERO
INTO DIM_LIVROS
FROM (
    SELECT DISTINCT NOME, AUTOR, GENERO
    FROM FATO_BESTSELLERS
) AS subconsulta;
```

### 4. Atualização e Otimização da Tabela Fato (FATO_BESTSELLERS)

Vínculo da chave estrangeira (`ID_LIVROS`) na tabela Fato e remoção de colunas textuais redundantes para estruturar o modelo estritamente relacional:

```sql
-- Adiciona a coluna de chave estrangeira
ALTER TABLE FATO_BESTSELLERS
ADD ID_LIVROS INT;

-- Atualiza a tabela Fato relacionando com a Dimensão
UPDATE F
SET F.ID_LIVROS = D.ID_LIVROS
FROM FATO_BESTSELLERS F
INNER JOIN DIM_LIVROS D
    ON F.NOME = D.NOME
   AND F.AUTOR = D.AUTOR
   AND F.GENERO = D.GENERO;

-- Remove as colunas descritivas mantendo o modelo dimensional enxuto
ALTER TABLE FATO_BESTSELLERS
DROP COLUMN NOME, AUTOR, GENERO;

-- Validação dos dados finais da Fato
SELECT * FROM FATO_BESTSELLERS;
```

---

## Dashboard Power BI

O painel foi projetado com um layout claro, elegante e focado em usabilidade executiva para facilitar a consulta rápida de títulos, autores e métricas históricas.

<img width="1323" height="738" alt="Dashboard_Bestsellers" src="https://github.com/user-attachments/assets/20119777-9995-49f1-a96d-d1b0381d3263" />


### Destaques do Painel:
* **Cards de KPIs (Topo):**
  * Total de Livros: **550**
  * Média de Preço: **$13,10**
  * Média da Nota: **4,5**
  * Total de Avaliações: **3,4 Mi**
* **Campo de Pesquisa (Canto Superior Direito):** Filtro rápido de "Consulta de Autor" para busca dinâmica na base.
* **Gráfico de Barras Horizontais (Esquerda):** Top 10 Autores Mais Avaliados (destaque para Paulo Coelho com 36 Mil e Wizards RPG Team com 17 Mil).
* **Gráfico de Linha com Área (Centro Superior):** Preço Médio ao Longo dos Anos (evolução histórica de 2009 a 2019).
* **Gráfico de Rosca (Direita Superior):** Proporção por gênero entre Não Ficção (56,36% - 310 títulos) e Ficção (43,64% - 240 títulos).
* **Tabela Detalhada (Centro Inferior):** Relação dos livros com Nome, Autor, Gênero, Preço e Média da Nota.
* **Gráfico de Colunas (Canto Inferior Direito):** Autores com Mais Bestsellers na lista (Jeff Kinney com 12, Gary Chapman com 11, Rick Riordan com 11 e Suzanne Collins com 11).

---

## Como Executar o Projeto

1. **SQL Server:**
   * Realize a importação do arquivo `.csv` utilizando o **Assistente de Importação de Arquivo Simples** (*Flat File Import Wizard*) no SQL Server.
   * Execute os scripts SQL contidos na seção *Tratamento de Dados, Análise Crítica e Modelagem SQL*.
2. **Power BI:**
   * Abra o arquivo `.pbix` do projeto no Power BI Desktop.
   * Atualize as credenciais de conexão com o banco SQL Server local.
   * Interaja com os filtros e campo de pesquisa para explorar as métricas.

---

## Desenvolvedora

**Desenvolvido por Luana Perseke**  
Conecte-se comigo no [LinkedIn](https://www.linkedin.com/in/luana-perseke-84211b179/) | [GitHub](https://github.com/LuanaPerseke)
