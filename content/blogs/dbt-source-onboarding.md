
---
title: "DBT Source automation"
date: 2023-04-07
draft: false
github_link: "https://medium.com/dev-genius/streamline-your-data-onboarding-with-dbt-codegen-package-8ecb5caab03c"
author: "Lior Kaufman"
tags:
  - DBT
  - Data Modeling
  - ETL/ELT
# image: /images/post.jpg
description: "Post about using the DBT codegen package and CLI tools to automate manual DBT tasks"
toc: 
---

Streamline Your Data Onboarding with DBT Codegen package
========================================================

**Introduction**

In today’s data-driven world, [dbt](https://docs.getdbt.com/) (data build tool) has become the go-to open-source transformation tool for turning raw data into actionable insights. In this post, we’ll explore dbt’s organized layer approach (staging, intermediate, and marts/final tables) and how automation can make onboarding new sources more efficient, using a real-life example from my own experience.

**DBT: Data Transformation Done Right**

DBT allows data engineers and analysts to perform complex data manipulations using a simple, maintainable, and version-controlled codebase. By treating data transformations as code, dbt enables teams to build modular, reusable, and testable data pipelines.

**The Layered Approach: Staging, Intermediate, and Marts/Final Tables**

Organizing data transformations into layers creates a clear and maintainable pipeline:

1.  **Staging Layer:** The first layer ingests raw data from various sources and lightly transforms it, creating a consistent schema for further transformations.
2.  **Intermediate Layer:** Here, data from the staging layer is further transformed into a structured format, involving complex calculations, aggregations, and business logic.
3.  **Marts/Final Tables Layer:** The final layer consists of well-defined, curated tables optimized for end-user consumption, answering specific business questions for reporting and analytics.

**Automating Staging with DBT-Codegen: A Real-life Example**

When onboarding DBT in my company, I initially onboarded five tables manually. Realizing I still had 30 more tables to onboard for the specific project I was working on, I knew there had to be a more efficient way. That’s when I discovered automation using dbt-codegen ([https://github.com/dbt-labs/dbt-codegen](https://github.com/dbt-labs/dbt-codegen)) and found it significantly streamlined the process.

dbt-codegen provides macros that automate the staging layer creation, such as the `generate_source` macro, which generates YAML code for a source file.

Here is the official documentation from [gitlab repo](https://github.com/dbt-labs/dbt-codegen#generate_source-source)

Another useful macro provided by dbt-codegen is `generate_base_model`. This macro generates the SQL code for a base model (named `stg_tablename`) based on a source table, again here is the official documentation from the [gitlab repo.](https://github.com/dbt-labs/dbt-codegen#generate_base_model-source)

**Using the Generate_Base_Model and Generate_Source Macros to Onboard Multiple Sources Efficiently**

After onboarding 30 sources using the `generate_base_model` and `generate_source` macros, I decided to onboard even more sources. For my next project, I needed to onboard about 60 sources.

Although copy-pasting 60times is doable, I knew there had to be a more efficient approach. By utilizing two readily available CLI tools, `sed` and the `>` operator, I was able to save time and reduce manual work.

`sed` is a powerful stream editor for filtering and transforming text, while the `>` operator is used for output redirection, allowing you to save the output of a command to a file. By combining these two tools, I was able to run the `generate_source` command as follows:

```
dbt run-operation generate_source \  
  --args '{"schema_name": "schema_name", "database_name": "db_name"}' \  
  | sed '1d;2s/^.\{14\}//' \  
  > models/myfolder/sources.yml
```

The sed command here uses `1d` to delete the first line of output, and `2s/^.\{14\}//` to remove the first 14 characters from the beginning of the second line. This effectively eliminates the first line and a specific number of characters from the second line. **Keep in mind that you may need to adjust the number of spaces or lines to remove based on your environment.**

The output redirection operator (`>`) lets you create a file and save to it the output directly from the terminal. Using this 2 tools I did not even need to copy paste a the source file anymore.  
Here is the generate_base_model commands I used.

```
dbt run-operation generate_base_model \  
  --args '{"source_name": "your_source_name", "table_name": "table_name"}' \  
  | sed '1,2d' \  
  > models/myfolder/stg_tablename.sql
```

I only had to remove the first 2 lines of the base models for each table, and while I could have manually used the same technique 60 times. I figured I could build a quick looping script to save me the manual work of changing the table name and the file name each time. Here is a template of the script I ended up writing:

```
for x in $(sed -n '5,$s/- name://p' models/myfolder/sources.yml) do   
dbt run-operation generate_base_model --args '{"source_name": "source_name", "table_name": '$x'}' \  
| sed 1,2d > models/myfolder/stg_$x.sql
```

Here's a breakdown of what each part of the script does:

1.  `for x in $(sed -n '5,$s/- name://p' models/myfolder/sources.yml); do \`: This line iterates over the output of the `sed` command. The `sed` command extracts table names from the `sources.yml` file. It starts from the fifth line (`5,`) and goes until the end of the file (`$`). It looks for lines starting with `- name:` and removes the `- name:` prefix, printing only the table names. The `for` loop assigns each table name to the variable `x` and repeats the loop for each table name.
2.  `dbt run-operation generate_base_model \`: This line runs the `generate_base_model` macro, which generates the base model SQL code for the specified source and table name.
3.  `--args '{"source_name": "your_source_name", "table_name": "'$x'"}' \`: This line passes the source name and table name as arguments to the `generate_base_model` macro. The table name is set to the variable `x` from the loop.
4.  `| sed 1,2d > models/myfolder/stg_$x.sql; \`: This line pipes the output of the `generate_base_model` macro to another `sed` command, which removes the first two lines of the output (`1,2d`). The remaining output is then redirected to a new file named `stg_$x.sql`, where `$x` is the table name. This creates a new staging model file for each table in the `sources.yml`.

I hope this post was helpful for you in learning how to streamline the onboarding process of new data sources using dbt and some CLI tools.
