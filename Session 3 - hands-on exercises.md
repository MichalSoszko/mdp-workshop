# Session 3

## GetInData Modern Data Platform: Test and document models

Welcome to the **GetInData Modern Data Platform** workshop hands-on `session 3`. 

By the end of this tutorial, you will learn how to:
- improve your dbt pipeline by using modern project structuring conventions
- apply jinja macro in your SQL code
- publish your work to DEV using Git and CICD
- connect your data with Looker Studio to create simple reports

Target environment will be Google Cloud Platform's: `BigQuery & Data Studio`, `Vertex AI Managed Notebook`, `VSCode` as IDE. 

This tutorial uses our DataOps JupyterLab image gcp-1.5.0.
For more versions and images check out [our public repo](https://github.com/getindata/jupyter-images/tree/master/jupyterlab-dataops).


# Exercises

## Introduction

A proper structuring of a project in dbt is crucial because it allows for efficient and organized development of data models. With a well-defined structure, it becomes easier to manage dependencies between models, track changes, and maintain a consistent approach to development. This helps ensure the accuracy and reliability of the data produced by the models, and facilitates collaboration among team members working on the project. 

In section 3 we will refactor the pipeline we already created and apply modern project structuring approach. Then we will commit our work to remote repository. The repository is stored in GitLab, a web-based repository manager that is part of our modern data stack. GitLab also provides continuous integration and delivery, allowing our code (after some treatment) to be distributed across various components of our modern data stack, such as the scheduler, data catalog, ingestion tool, and BI tool.

In this chapter we will reforge our set of dbt models into a well organized and structured project, dividing it into three logic layers: `staging`, `intermediate` and `mart`. 

- `Marts layer` is a place 

## Refactoring the dbt pipeline

### Staging layer

`Staging layer` is a place where data is cleaned (by formating, spliting / concating, json extracting etc.), column naming convention is applied, as well as some basic calculation and conversions are included. In this layer we avoid joining models.

1. In `models` folder inside of your project create subfolder called `00_raw_data`.

2. Move (ie. by using drag and drop) both `source_order_items.yml` and `source_users.yml` into newly created `models/00_raw_data folder`.

    The result should look as follows:

    <img width="400" alt="image" src="Images/dbt_refactor_01.png" >

3. Perform a code check by running `dbt compile` command in the command line.

4. Create second subfolder in `models` directory, call it `01_staging`.

5. Inside of the `01_staging` folder create new model file called `stg_order_items.sql`.

6. Add the following SQL statement:

    ```
    with source as (
        select * from {{ source('raw_data', 'order_items') }}
    ),
    cleaned as (
        select
            id                      as order_item_id,
            order_id                as order_id,
            user_id                 as user_id,
            product_id              as product_id,
            inventory_item_id       as inventory_item_id,
            status                  as order_status,
            created_at              as order_created_at,
            shipped_at              as order_shipped_at,
            delivered_at            as order_delivered_at,
            returned_at             as order_returned_at,
            sale_price              as order_item_sale_price
        from source
    )
    select * from cleaned
    ```

    Note, this is the staging model, where we are referencing the source (`{{ source (....) }}`) placed in `00_raw_data` folder. For dbt it does not matter where the model is stored - and that is because all model names need to be unique. Inside of SQL we changed column names applying our custom naming convention.

7. Perform a code check by running `dbt compile` command in the command line.

8. Exercise: Repeat steps `5` - `7` for new model file called `stg_users`. 
    
    > Hint: You can choose whatever column naming convention you prefer.

    > Hint2: Use Bigquery to check schema for `raw_data.users` table.

    <details>
    <summary>Preview example of the resulting SQL statement</summary>
    <br>
    <pre>
    with source as (
      select * from {{ source('raw_data', 'users') }}
    ),
    cleaned as (
        select
            id                  user_id,
            first_name          user_first_name,
            last_name           user_last_name,
            email               user_email,
            age                 user_age,
            gender              user_gender,
            state               user_address_state,
            street_address      user_street_address,
            postal_code         user_postal_code,
            city                user_city,
            country             user_country,
            latitude            user_geo_latitude,
            longitude           user_geo_longitude,
            traffic_source      user_acc_traffic_source,
            created_at          user_acc_created_at
        from source
    )
    select * from cleaned
    </pre>
    </details>

9. Create the third staging model, this time for our `seed_tax_rates` CSV file. Call it `stg_tax_rates.sql`. Put the following code inside and perform `dbt compile` check:
    
    ```
    with tax_rates as (
        select * from {{ ref( 'seed_tax_rates' ) }}
    ),
    cleaned as (
        select 
            Country         as country,
            Tax_Rate        as tax_rate,

            trim(Country)   as country_cleaned
        from tax_rates
    )
    select * from cleaned
    ```

    Note that in this model, apart lowercasing column names, we added new column called `country_cleaned`. you might have noticed that in `model_order_items_with_tax.sql` the join between `order_items_with_country` column and `tax_rates` is performed on `trim(tax_rates.Country)`. The purpose of staging layer is to avoid such complications for downstream models if possible. This is a simple example but you could imagine a complicated join on a case-when statement in several subqueries making the code a little bit harder to understand.

After performing steps 1-9 your `models/` you should obtain a similar image:

<img width="300" alt="image" src="Images/dbt_refactor_02.png" >


### Intermediate layer

`Intermediate layer` is a place where we apply more advanced conversions, business logic, calculations, joins etc. Tables (views) we create here are close in shape to what we'd like to use for reporting. However, for clarity, intermediate models are generaly hidden from BI tools.
