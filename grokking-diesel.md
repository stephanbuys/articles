

Outline

## Introduction/getting started

Say something about diesel cli 

## Basic flow

- design Schema
- create migrations
- create models (models.rs)
  - derives (see below)
- rust schema (schema.rs - and macros)

Under the hood:

- DATABASE_URL
- diesel print schema

Recommended:
- cargo-expand

## Migrations

### Schema

infer_schema!
	diesel print-schema

## Structs

- PascalCase to pascal_case(s), or AUserName to a_user_name(s)
- Types:
   - diesel::types
   - timestamps: http://docs.diesel.rs/diesel/pg/types/sql_types/index.html
   - number: http://docs.diesel.rs/diesel/pg/types/sql_types/index.html
     There are no unsigned ints
     
### Derives

- QueryAble
- Identifiable
- Insertable
  - annotate column names
  - specify table name
- AsChangeset
- Associations     
  - belongs_to
    - User -> user_id: i32,
    - foreign key 
  - belonging_to
    -	some examples
        
### Using

 - schema::table
	  - dsl
		  - columns
      -::table
      
How it works

table is an empty struct
select and filter build up "pseudo queries"
the you call something from the LoadDSL on the query

### Building queries

http://docs.diesel.rs/diesel/prelude/index.html
	DSLs
Select statements can have:
	SelectStatement {
	    select: 
	    from: 
	    distinct: NoDistinctClause,
	    where_clause: NoWhereClause,
	    order: NoOrderClause,
	    limit: NoLimitClause,
	    offset: NoOffsetClause,
	    group_by: NoGroupByClause
	}
Expressions
	http://docs.diesel.rs/diesel/expression_methods/global_expression_methods/trait.ExpressionMethods.html#method.desc      

### Getting results

Getting results
	get_result
	get_results
	load
	first
		limit(1).get_result
		
Mostly returned as the struct in your models.rs, or Vec of struct, or tuples		

## The unofficial cheatsheets

Check `diesel_tests/tests` in the repo for copious examples.




