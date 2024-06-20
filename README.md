# Análise DB Northwind

## Questões a serem respondidas:
   * Qual foi o total de receitas no ano de 1997?
   * Análise de crescimento mensal e o cálculo de YTD
   * Qual é o valor total que cada cliente já pagou até agora?
   * Separe os clientes em 5 grupos de acordo com o valor pago por cliente
   * Agora somente os clientes que estão nos grupos 3, 4 e 5 para que seja feita uma análise de Marketing especial com eles
   * Identificar os 10 produtos mais vendidos.
   * Quais clientes do Reino Unido pagaram mais de 1000 dólares?

## Configuração Inicial

### Manualmente

Utilize o arquivo SQL fornecido, `nortwhind.sql`, para popular o banco de dados.

### Com Docker e Docker Compose

**Pré-requisito**: Instale o Docker e Docker Compose

* [Começar com Docker](https://www.docker.com/get-started)
* [Instalar Docker Compose](https://docs.docker.com/compose/install/)

### Passos para configuração com Docker:

1. **Iniciar o Docker Compose** 
    Execute o comando abaixo para subir os serviços:
    
    ```
    docker-compose up
    ```
    
    Aguarde as mensagens de configuração, como:
    
    ```csharp
    Creating network "northwind_psql_db" with driver "bridge"
    Creating volume "northwind_psql_db" with default driver
    Creating volume "northwind_psql_pgadmin" with default driver
    Creating pgadmin ... done
    Creating db      ... done
    ```
       
2. **Conectar o PgAdmin** 
    Acesse o PgAdmin pelo URL: [http://localhost:5050](http://localhost:5050), com a senha `postgres`. 

    Configure um novo servidor no PgAdmin:
        
        * **Aba General**:
            * Nome: db
        * **Aba Connection**:
            * Nome do host: db
            * Nome de usuário: postgres
            * Senha: postgres Em seguida, selecione o banco de dados "northwind".

## Criação das views (Relatórios)

1. **Relatórios de Receita**
    
    * Qual foi o total de receitas no ano de 1997?

    ```sql
    CREATE VIEW total_revenues_1997_view AS
    SELECT SUM((order_details.unit_price) * order_details.quantity * (1.0 - order_details.discount)) AS total_revenues_1997
    FROM order_details
    INNER JOIN (
        SELECT order_id 
        FROM orders 
        WHERE EXTRACT(YEAR FROM order_date) = '1997'
    ) AS ord 
    ON ord.order_id = order_details.order_id;
    ```

2.  **Análise de crescimento mensal e o cálculo de YTD**

    ```sql
    CREATE VIEW view_receitas_acumuladas AS
    WITH ReceitasMensais AS (
        SELECT
            EXTRACT(YEAR FROM orders.order_date) AS Ano,
            EXTRACT(MONTH FROM orders.order_date) AS Mes,
            SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS Receita_Mensal
        FROM
            orders
        INNER JOIN
            order_details ON orders.order_id = order_details.order_id
        GROUP BY
            EXTRACT(YEAR FROM orders.order_date),
            EXTRACT(MONTH FROM orders.order_date)
    ),
    ReceitasAcumuladas AS (
        SELECT
            Ano,
            Mes,
            Receita_Mensal,
            SUM(Receita_Mensal) OVER (PARTITION BY Ano ORDER BY Mes) AS Receita_YTD
        FROM
            ReceitasMensais
    )
    SELECT
        Ano,
        Mes,
        Receita_Mensal,
        Receita_Mensal - LAG(Receita_Mensal) OVER (PARTITION BY Ano ORDER BY Mes) AS Diferenca_Mensal,
        Receita_YTD,
        (Receita_Mensal - LAG(Receita_Mensal) OVER (PARTITION BY Ano ORDER BY Mes)) / LAG(Receita_Mensal) OVER (PARTITION BY Ano ORDER BY Mes) * 100 AS Percentual_Mudanca_Mensal
    FROM
        ReceitasAcumuladas
    ORDER BY
        Ano, Mes;
    ```

3. **Qual é o valor total que cada cliente já pagou até agora?**

    ```sql
    CREATE VIEW view_total_revenues_per_customer AS
    SELECT 
        customers.company_name, 
        SUM(order_details.unit_price * order_details.quantity * (1.0 - order_details.discount)) AS total
    FROM 
        customers
    INNER JOIN 
        orders ON customers.customer_id = orders.customer_id
    INNER JOIN 
        order_details ON order_details.order_id = orders.order_id
    GROUP BY 
        customers.company_name
    ORDER BY 
        total DESC;

4. **Separe os clientes em 5 grupos de acordo com o valor pago por cliente**

    ```sql
    CREATE VIEW view_total_revenues_per_customer_group AS
    select
        c.company_name as nome_cliente,
        sum(od.unit_price * od.quantity * (1 - od.discount)) as valor_pago,
        NTILE(5) OVER (ORDER BY sum(od.unit_price * od.quantity * (1 - od.discount)) desc) AS grupo_cliente
    from orders as o
    inner join customers as c on o.customer_id = c.customer_id
    inner join order_details as od on o.order_id = od.order_id
    group by c.company_name

5. **Agora somente os clientes que estão nos grupos 3, 4 e 5 para que seja feita uma análise de Marketing especial com eles**

    ```sql
    CREATE VIEW clients_to_marketing AS
    with agrupamento_clientes as (
        select
        c.company_name as nome_cliente,
        sum(od.unit_price * od.quantity * (1 - od.discount)) as valor_pago,
        NTILE(5) OVER (ORDER BY sum(od.unit_price * od.quantity * (1 - od.discount)) desc) AS grupo_cliente
        from orders as o
        inner join customers as c on o.customer_id = c.customer_id
        inner join order_details as od on o.order_id = od.order_id
        group by c.company_name
    )
    select
    *
    from agrupamento_clientes
    where grupo_cliente in ('3','4','5')

6. **Identificar os 10 produtos mais vendidos.**

    ```sql
    create view top_10_products as
        select 
        pro.product_name,
        sum(od.unit_price * od.quantity * (1 - od.discount)) as somatória
        from order_details as od
        inner join products as pro on od.product_id = pro.product_id
        group by pro.product_name
        order by somatória desc

7. **Quais clientes do Reino Unido pagaram mais de 1000 dólares**

    ```sql
    create view uk_clients_who_pay_more_then_1000 as
        with pay_per_client as 
        (
        select
            c.contact_name as contact_name,
            o.ship_country as country,
            sum(od.unit_price * od.quantity * (1 - od.discount)) as valor_pago
        from orders as o
        inner join customers as c on c.customer_id = o.customer_id
        inner join order_details as od on od.order_id = o.order_id
        group by c.contact_name, o.ship_country
        )
    select
        contact_name,
        valor_pago
    from pay_per_client	
    where country = 'UK' and valor_pago > 1000
    order by contact_name asc

## Resultados / Respostas

Após a criação das views, basta seguir as querys abaixo para responder cada pergunta:

   1. Qual foi o total de receitas no ano de 1997?
        ```sql
            select * from total_revenues_1997_view

   2. Análise de crescimento mensal e o cálculo de YTD
        ```sql
            select * from view_receitas_acumuladas


   3. Qual é o valor total que cada cliente já pagou até agora?
        ```sql
            elect * from view_total_revenues_per_customer

   4. Separe os clientes em 5 grupos de acordo com o valor pago por cliente
        ```sql
            select * from view_total_revenues_per_customer_group

   5. Agora somente os clientes que estão nos grupos 3, 4 e 5 para que seja feita uma análise de Marketing especial com eles
        ```sql
            select * from clients_to_marketing

   6. Identificar os 10 produtos mais vendidos.
        ```sql
            select * from view top_10_products

   7. Quais clientes do Reino Unido pagaram mais de 1000 dólares?
        ```sql
            select * from view uk_clients_who_pay_more_then_1000



!!. **Parar o Docker Compose** Pare o servidor iniciado pelo comando `docker-compose up` usando Ctrl-C e remova os contêineres com:
    
    ```
    docker-compose down
    ```
    
!!. **Arquivos e Persistência** Suas modificações nos bancos de dados Postgres serão persistidas no volume Docker `postgresql_data` e podem ser recuperadas reiniciando o Docker Compose com `docker-compose up`. Para deletar os dados do banco, execute:
    
    ```
    docker-compose down -v
    ```