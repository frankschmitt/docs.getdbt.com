---
title: "Unit tests"
sidebar_label: "Unit tests"
description: "Learn how to use unit tests on your SQL models."
search_weight: "heavy"
id: "unit-tests"
keywords:
  - unit test, unit tests, unit testing, dag
---
:::note closed beta

Support for unit testing dbt models is available to dbt Cloud customers who have chosen to [Keep on latest version](/docs/dbt-versions/upgrade-dbt-version-in-cloud#keep-on-latest-version). This is currently in closed beta, starting with select customers and rolling out to wider availability through February and March.

For dbt Core, unit tests will be available in v1.8, planned for late April 2024. The v1.8.0-b1 release of `dbt-core` & dbt Labs-maintained adapters is planned for February 28.

:::

Historically, dbt's test coverage was confined to [“data” tests](/docs/build/data-tests), assessing the quality of input data or resulting datasets' structure. However, these tests could only be executed _after_ building a model. 

Now, we are introducing a new type of test to dbt - unit tests. In software programming, unit tests validate small portions of your functional code, and they work much the same way here. Unit tests allow you to validate your SQL modeling logic on a small set of static inputs _before_ you materialize your full model in production. Unit tests enable test-driven development, benefiting developer efficiency and code reliability. 

## Before you begin

- We currently only support unit testing SQL models.
- We currently only support adding unit tests to models in your _current_ project.
- If your model has multiple versions, by default the unit test will run on *all* versions of your model. Read [unit testing versioned models](#unit-testing-versioned-models) for more information.
- Unit tests must be defined in a YML file in your `models/` directory.

Read the [reference doc](/reference/resource-properties/unit-tests) for more details about formatting your unit tests.

### When to add a unit test to your model

You should unit test a model:
- When your SQL contains complex logic:
    - Regex
    - Date math
    - Window functions
    - `case when` statements when there are many `when`s
    - Truncation
    - Recursion
- When you're writing custom logic to process input data, similar to creating a function.
- We don't recommend conducting unit testing for functions like `min()` since these functions are tested extensively by the warehouse. If an unexpected issue arises, it's more likely a result of issues in the underlying data rather than the function itself. Therefore, fixture data in the unit test won't provide valuable information.
- Logic for which you had bugs reported before.
- Edge cases not yet seen in your actual data that you want to handle.
- Prior to refactoring the transformation logic (especially if the refactor is significant).
- Models with high "criticality" (public, contracted models or models directly upstream of an exposure).

## Unit testing a model

This example creates a new `dim_customers` model with a field `is_valid_email_address` that calculates whether or not the customer’s email is valid: 

<file name='dim_customers.sql'>

```sql
with customers as (

    select * from {{ ref('stg_customers') }}

),

accepted_email_domains as (

    select * from {{ ref('top_level_email_domains') }}

),
	
check_valid_emails as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customers.email,
	      coalesce (regexp_like(
            customers.email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$'
        )
        = true
        and accepted_email_domains.tld is not null,
        false) as is_valid_email_address
    from customers
		left join accepted_email_domains
        on customers.email_top_level_domain = lower(accepted_email_domains.tld)

)

select * from check_valid_emails
```
</file>

The logic posed in this example can be challenging to validate. You can add a unit test to this model to ensure the `is_valid_email_address` logic captures all known edge cases: emails without `.`, emails without `@`, and emails from invalid domains.

<file name='dbt_project.yml'> 

```yaml
unit_tests:
  - name: test_is_valid_email_address
    description: "Check my is_valid_email_address logic captures all known edge cases - emails without ., emails without @, and emails from invalid domains."
    model: dim_customers
    given:
      - input: ref('stg_customers')
        rows:
          - {email: cool@example.com,    email_top_level_domain: example.com}
          - {email: cool@unknown.com,    email_top_level_domain: unknown.com}
          - {email: badgmail.com,        email_top_level_domain: gmail.com}
          - {email: missingdot@gmailcom, email_top_level_domain: gmail.com}
      - input: ref('top_level_email_domains')
        rows:
          - {tld: example.com}
          - {tld: gmail.com}
    expect:
      rows:
        - {email: cool@example.com,    is_valid_email_address: true}
        - {email: cool@unknown.com,    is_valid_email_address: false}
        - {email: badgmail.com,        is_valid_email_address: false}
        - {email: missingdot@gmailcom, is_valid_email_address: false}

```
</file>

The previous example defines the mock data using the inline `dict` format, but you can also use `csv` either inline or in a separate fixture file. 

You only have to define the mock data for the columns you care about. This enables you to write succinct and _specific_ unit tests.

:::note

The direct parents of the model that you’re unit testing (in this example, `stg_customers` and `top_level_email_domains`) need to exist in the warehouse before you can execute the unit test.

Use the [`--empty`](/reference/commands/build#the---empty-flag) flag to build an empty version of the models to save warehouse spend. 

```bash

dbt run --select "stg_customers top_level_email_domains" --empty

```

Alternatively, use `dbt build` to, in lineage order:

- Run the unit tests on your model.
- Materialize your model in the warehouse.
- Run the data tests on your model.

:::

Now you’re ready to run this unit test. You have a couple of options for commands depending on how specific you want to be: 

- `dbt test --select dim_customers` runs _all_ of the tests on `dim_customers`.
- `dbt test --select "dim_customers,test_type:unit"` runs all of the _unit_ tests on `dim_customers`.
- `dbt test --select test_is_valid_email_address` runs the test named `test_is_valid_email_address`.

```shell

dbt test --select test_is_valid_email_address
16:03:49  Running with dbt=1.8.0-a1
16:03:49  Registered adapter: postgres=1.8.0-a1
16:03:50  Found 6 models, 5 seeds, 4 data tests, 0 sources, 0 exposures, 0 metrics, 410 macros, 0 groups, 0 semantic models, 1 unit test
16:03:50  
16:03:50  Concurrency: 5 threads (target='postgres')
16:03:50  
16:03:50  1 of 1 START unit_test dim_customers::test_is_valid_email_address ................... [RUN]
16:03:51  1 of 1 FAIL 1 dim_customers::test_is_valid_email_address ............................ [FAIL 1 in 0.26s]
16:03:51  
16:03:51  Finished running 1 unit_test in 0 hours 0 minutes and 0.67 seconds (0.67s).
16:03:51  
16:03:51  Completed with 1 error and 0 warnings:
16:03:51  
16:03:51  Failure in unit_test test_is_valid_email_address (models/marts/unit_tests.yml)
16:03:51    

actual differs from expected:

@@ ,email           ,is_valid_email_address
→  ,cool@example.com,True→False
   ,cool@unknown.com,False
...,...             ,...


16:03:51  
16:03:51    compiled Code at models/marts/unit_tests.yml
16:03:51  
16:03:51  Done. PASS=0 WARN=0 ERROR=1 SKIP=0 TOTAL=1

```

The clever regex statement wasn’t as clever as initially thought, as the model incorrectly flagged `cool@example.com` as an invalid email address.

Updating the regex logic to `'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'` (those pesky escape characters) and rerunning the unit test solves the problem:

```shell

dbt test --select test_is_valid_email_address
16:09:11  Running with dbt=1.8.0-a1
16:09:12  Registered adapter: postgres=1.8.0-a1
16:09:12  Found 6 models, 5 seeds, 4 data tests, 0 sources, 0 exposures, 0 metrics, 410 macros, 0 groups, 0 semantic models, 1 unit test
16:09:12  
16:09:13  Concurrency: 5 threads (target='postgres')
16:09:13  
16:09:13  1 of 1 START unit_test dim_customers::test_is_valid_email_address ................... [RUN]
16:09:13  1 of 1 PASS dim_customers::test_is_valid_email_address .............................. [PASS in 0.26s]
16:09:13  
16:09:13  Finished running 1 unit_test in 0 hours 0 minutes and 0.75 seconds (0.75s).
16:09:13  
16:09:13  Completed successfully
16:09:13  
16:09:13  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1

```

Your model is now ready for production! Adding this unit test helped catch an issue with the SQL logic _before_ you materialized `dim_customers` in your warehouse and will better ensure the reliability of this model in the future. 

## Unit testing versioned models

When a unit test is added to a model, it will run on all versions of the model by default.
Using the example in this article, if you have versions 1, 2, and 3 of `dim_customers`, the `test_is_valid_email_address` unit test will run on all 3 versions.

To only unit test a specific version (or versions) of a model, include the desired version(s) in the model config:

```yml

unit_tests::
  - name: test_is_valid_email_address
    model: dim_customers
      versions:
        include: 
          - 2
    ...

```

In this scenario, if you have version 1, 2, and 3 of `dim_customers `, my `test_is_valid_email_address` unit test will run on _only_ version 2.

To unit test all versions except a specific version (or versions) of a model, you can exclude the relevant version(s) in the model config:

```yml

unit_tests:
  - name: test_is_valid_email_address
    model: dim_customers
      versions:
        exclude: 
          - 1
    ...

```
So, if you have versions 1, 2, and 3 of `dim_customers`, your `test_is_valid_email_address` unit test will run on _only_ versions 2 and 3.

If you want to unit test a model that references the pinned version of the model, you should specify that in the `ref` of your input:

```yml

unit_tests:
  - name: test_is_valid_email_address
    model: dim_customers
    given: 
      - input: ref('stg_customers', v=1)
      ...

```

## Unit testing incremental models

When configuring your unit test, you can override the output of macros, vars, or environment variables. This enables you to unit test your incremental models in "full refresh" and "incremental" modes. 

When testing an incremental model, the expected output is the __result of the materialization__ (what will be merged/inserted), not the resulting model itself (what the final table will look like after the merge/insert).

For example, say you have an incremental model in your project:

<File name='my_incremental_model.sql'>

```sql

{{
    config(
        materialized='incremental'
    )
}}

select * from {{ ref('events') }}
{% if is_incremental() %}
where event_time > (select max(event_time) from {{ this }})
{% endif %}

```

</File>

You can define unit tests on `my_incremental_model` to ensure your incremental logic is working as expected:

```yml

unit_tests:
  - name: my_incremental_model_full_refresh_mode
    model: my_incremental_model
    overrides:
      macros:
        # unit test this model in "full refresh" mode
        is_incremental: false 
    given:
      - input: ref('events')
        rows:
          - {event_id: 1, event_time: 2020-01-01}
    expect:
      rows:
        - {event_id: 1, event_time: 2020-01-01}

  - name: my_incremental_model_incremental_mode
    model: my_incremental_model
    overrides:
      macros:
        # unit test this model in "incremental" mode
        is_incremental: true 
    given:
      - input: ref('events')
        rows:
          - {event_id: 1, event_time: 2020-01-01}
          - {event_id: 2, event_time: 2020-01-02}
          - {event_id: 3, event_time: 2020-01-03}
      - input: this 
        # contents of current my_incremental_model
        rows:
          - {event_id: 1, event_time: 2020-01-01}
    expect:
      # what will be inserted/merged into my_incremental_model
      rows:
        - {event_id: 2, event_time: 2020-01-02}
        - {event_id: 3, event_time: 2020-01-03}

```

There is currently no way to unit test whether the dbt framework inserted/merged the records into your existing model correctly, but [we're investigating support for this in the future](https://github.com/dbt-labs/dbt-core/issues/8664).

## Additional resources

- [Unit testing reference page](/reference/resource-properties/unit-tests)
- [Supported data formats for mock data](/reference/resource-properties/data-formats)
- [Unit testing versioned models](/reference/resource-properties/unit-testing-versions)
- [Unit test inputs](/reference/resource-properties/unit-test-input)
- [Unit test overrides](/reference/resource-properties/unit-test-overrides)