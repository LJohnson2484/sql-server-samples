# SQL Assessment API

SQL Assessment API provides a mechanism to evaluate the configuration of your SQL Server for best practices. The API is delivered with a ruleset containing best practice rules suggested by SQL Server Team. This ruleset is enhancing with the release of new versions but at the same time, the API is built with the intent to give a highly customizable and extensible solution. So, users can tune the default rules and create their own ones. The API can be used to assess SQL Server versions 2012 and higher and Azure SQL Database Managed Instance (more to come).

Learn more about the API on the [SQL Assessment API docs page](https://docs.microsoft.com/en-us/sql/sql-assessment-api/sql-assessment-api-overview).

## QuickStart.md

Learn how to assess your SQL Server configuration for best practices in 2 simple steps.

## config.json

This is the default set of rules shipped with SQL Assessment API. Feel free to open issues to have us fix or add rules. Also, we're happy to see your pull requests to this file.

## DisablingBuiltInChecks_sample.json

Contains three parts. First shows how you can disable a specified rule by its ID. The second disables all the rules with the "TraceFlag" tag. The last disables to run all rules of the default ruleset (using the DefaultRuleset tag) against databases named "DBName1" and "DBName2".

## MakingCustomChecks_sample.json

Demonstrates how to make a custom ruleset containing two checks. The sample contains two sections: `rules` and `probes`. `Rules` is for rule (sometimes referred to as check) definitions. Usually, rules are best practices or a company's internal policies that should be applied to SQL Server configuration. Here's one of the rules from this sample with explanations for each line:

```
{
    "target": {                                                                  //Target describes a SQL Server object the check is supposed to run against
        "type": "Database",                                                          //This check targets Database object
        "version": "[13.0,)",                                                        //Applies to SQL Server 2016 and higher
                                                                                     //Another example: "[12.0,13.0)" reads as "any SQL Server version >= 12.0 and < 13.0"
        "platform": "/^(Windows|Linux)$/",                                           //Applies to SQL Server on Windows and Linux
        "engineEdition": "OnPremises, ManagedInstance",                              //Applies to SQL on Premises and Azure SQL Managed Instance. Here you can also filter specific editions of SQL Server
        "name": { "not": "/^(master|tempdb|model)$/" }                               //Applies to any database excluding master, tempdb, and msdb
    },
    "id": "QueryStoreOn",                                                        //Rule ID
    "itemType": "definition",                                                    //Can be "definition" or "override". First is to declare a rule, the latter is to override/customize an existing rule. See also DisablingBuiltInChecks_sample.json
    "tags": [ "CustomRuleset", "Performance", "QueryStore", "Statistics" ],      //Tags combine rules in different subsets.
    "displayName": "Query Store should be active",                               //Short name for the rule
    "description": "The Query Store feature provides you with insight on query plan choice and performance. It simplifies performance troubleshooting by helping you quickly find performance differences caused by query plan changes. Query Store automatically captures a history of queries, plans, and runtime statistics, and retains these for your review. It separates data by time windows so you can see database usage patterns and understand when query plan changes happened on the server. While Query Store collects queries, execution plans and statistics, its size in the database grows until this limit is reached. When that happens, Query Store automatically changes the operation mode to read-only and stops collecting new data, which means that your performance analysis is no longer accurate.",
                                                                                //A more detailed explanation of a best practice or policy that the rule check
    "message": "Make sure Query Store actual operation mode is 'Read Write' to keep your performance analysis accurate",
                                                                                //Usually, it's for recommendation what user should do if the rule raises up an alert
    "helpLink": "https://docs.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store",
                                                                                //Reference material
    "probes": [ "Custom_DatabaseConfiguration" ],                               //List of probes that are used to get the required data for this check. See below to know more about probes.
    "condition": {                                                              //Condition object is to define "good" and "bad" state, the latter is when the rule should raise an alert. When the condition is true, it means that the checked object complies with the best practice or policy. Otherwise, the rule raises an alert (it actually adds its message to the resulting set of recommendations)
        "equal": [ "@query_store_state", 2 ]                                        //It means that the variable came from the probe should be equal to 2
    }
}

```

`Probes` describes how and where to get the required data to check compliance with a rule. You can use T-SQL queries as well as methods from some assemblies. The probe below uses a T-SQL query.

```
"Custom_DatabaseConfiguration": [                                               //Probe name is used to reference the probe from a rule
                                                                                //Probe can have a few implementations that will be used for different targets
                                                                                //This probe has two implementations for different version of SQL Server
    {
        "type": "SQL",                                                          //Probe uses a T-SQL query to get the required data. Use 'CLR' for assemblies.
        "target": {                                                             //Probes have their own target, usually to separate implementation for different versions, editions, or platforms. Probe targets work the same way as rule targets do.
            "type": "Database",
            "version": "(,12.0)",                                               //This target is for SQL Server of versions prior to 2014
            "platform": "/^(Windows|Linux)$/",
            "engineEdition": "OnPremises, ManagedInstance"
        },
        "implementation": {                                                     //Implementation object with a T-SQL query. This probe is used in many rules, that's why the query return so many fields
            "query": "SELECT db.is_auto_create_stats_on, db.is_auto_update_stats_on, 0 AS query_store_state, db.collation_name, (SELECT collation_name FROM master.sys.databases (NOLOCK) WHERE database_id = 1) AS master_collation, db.is_auto_close_on, db.is_auto_shrink_on, db.page_verify_option, db.is_db_chaining_on, NULL AS is_auto_create_stats_incremental_on, db.is_trustworthy_on, db.is_parameterization_forced FROM [sys].[databases] (NOLOCK) AS db WHERE db.[name]=@TargetName"
        }
    },
    {                                                                           //This implementation object is to get the required data from SQL Server 2014 (look at target.version)
        "type": "SQL",
        "target": {
            "type": "Database",
            "version": "[12.0, 13.0)",
            "platform": "/^(Windows|Linux)$/",
            "engineEdition": "OnPremises, ManagedInstance"
        },
        "implementation": {
        "query": "SELECT db.is_auto_create_stats_on, db.is_auto_update_stats_on, 0 AS query_store_state, db.collation_name, (SELECT collation_name FROM master.sys.databases (NOLOCK) WHERE database_id = 1) AS master_collation, db.is_auto_close_on, db.is_auto_shrink_on, db.page_verify_option, db.is_db_chaining_on, db.is_auto_create_stats_incremental_on, db.is_trustworthy_on, db.is_parameterization_forced FROM [sys].[databases] (NOLOCK) AS db WHERE db.[name]=@TargetName"
        }
    },
    {
        "type": "SQL",                                                          //This implementation object is to get the required data from SQL Server 2016 and up (look at target.version)
        "target": {
        "type": "Database",
        "version": "[13.0,)",
        "platform": "/^(Windows|Linux)$/",
        "engineEdition": "OnPremises, ManagedInstance"
        },
        "implementation": {
            "useDatabase": true,                                                //Use this key if your query requires to run on a database that is being assessed (it's a replacement for 'USE <DATABASENAME>;')
            "query": "SELECT db.is_auto_create_stats_on, db.is_auto_update_stats_on, (SELECT CAST(actual_state AS DECIMAL) FROM [sys].[database_query_store_options]) AS query_store_state, db.collation_name, (SELECT collation_name FROM master.sys.databases (NOLOCK) WHERE database_id = 1) AS master_collation, db.is_auto_close_on, db.is_auto_shrink_on, db.page_verify_option, db.is_db_chaining_on, db.is_auto_create_stats_incremental_on, db.is_trustworthy_on, db.is_parameterization_forced FROM [sys].[databases] (NOLOCK) AS db WHERE db.[name]=@TargetName"
        }
    }
]
```