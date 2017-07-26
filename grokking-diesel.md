Question:
- statement about custom derive correct?
- primary key requirement statement below seems flimsy.
- rust capitilisation? (pick one)
- am I mixing statements and expressions?
- am I mising structs and objects?

Outline

## Introduction/getting started

Diesel (http://diesel.rs) is an ORM (Object-relational mapping) and Query Builder written in Rust. It supports Postgresql, Mysql and SQLite. It makes use of Rust's custom derive functionality to generate all the code you need in order to get all the power of Rust's type system. This means that you get compile-time validation of your code, thus eliminating possible runtime errors and giving you lightning fast code.

Diesel is very powerful, but just following the examples might still leave you scratching your head, asking questions like "where do these types come from?", "what should I add here", "how does this work?". This article is meant to shed a little bit of light on the topic from the perspective of a intermediate-level rust developer.

## First steps

You should first run through the Getting Started guide. http://diesel.rs/guides/getting-started/. By far the easiest way to get a Postgresql server running for your tests is by using Docker (www.docker.com), the following one-liner should do the trick:

    docker run -d --rm --name postgres -e POSTGRES_USER=username -e POSTGRES_PASSWORD=password -p 127.0.0.1:5432:5432 postgres
    
> This is _not_ for production use and the whole container will be removed when the container is stopped, use `docker stop postgres`, but it is super handy for the example as you can use everything on the site through copy and paste.

### A word about the format of the rest of this guide

I'm going to assume that you have worked through the Getting Started guide, so if you get lost, or find that details are missing in this article, it was probably intentionally omitted to keep the article concise, I'm not going to deal with configuring the project and `Cargo.toml`, etc.
    
## Core components and community

At its core diesel consists of 4 main components:

The diesel crate, diesel_codegen (the code generator) and the diesel cli, which by now you should have been introduced to.

The last major component, and 'secret weapon' (according to me) is the the test suite in the official repo (https://github.com/diesel-rs/diesel/tree/master/diesel_tests), it served as my guide whenever I got stuck, although it will be replaced by documentation through the massive effort the project and its contributors are currently undertaking to improve the documentation.

### Environment

When I started off with Diesel I was baffled by the `schema.rs` macro `infer_schema!` in the example (more on this below), which used `dotenv:DATABASE_URL"`, the `dotenv` dependency is just a convenience library to allows you to put your `DATABASE_URL` in a hidden environment file, ala `.env` (dot env, get it).

This implies that you can also just specify `DATABASE_URL` in your environment, specially handy when you run your Diesel binary in a docker container ala 12-Factor app [1] 
        
The `diesel` cli tool also read the `.env` file, but will also accept the `--database-url` parameter.

> Definitely check out the `examples/sqlite/getting_started_step_3/README.md` file in the repo to learn how to configure the `DATABASE_URL` for SQLite, it doesn't use a URI format (`DATABASE_URL=file:test.db"`) 

## Basic flow

The core workflow for creating an app built with Diesel can be broken down as follows:

### Design a Schema

This is very obvious to seasoned SQL veterans, luckily for the rest of us the `diesel` cli's migrations subcommand allow us to easily iterate on our design and even evolve it over time. That said, it is super useful to have a clear idea of what you want your database to look like up front, and it should be noted that diesel only works with tables that have a primary key. 

### Create Migrations

Follow the pattern in the Getting Started guide, and note that you can add more migrations at any time using the `diesel migration generate` subcommand. Migrations will be run in order using `diesel migration run`. You can rerun the last migration by issuing `diesel migration redo` and if you truly get stuck, you can run the following command, but please never do so on a production database, `diesel database reset`.

When designing your tables you should plurals of your table names, diesel will take the model `User` and search for the table `users`. You can define custom table names, but knowing the diesel designers' assumptions will probably spare you some confusion.

Diesel will take `PascalCase` rust structs (which probably will describe a single object, or row in your database table) and translate them into `snake_case` table names with a `s` tucked at the end to "pluralise" it. For example `AFancyNamedObject` will be assumed to map to a table named `a_fancy_named_objects`.

3. Infer the Schema for use in Rust

Diesel has the ability to inspect your actual database and infer a schema for use in rust, this is in turn used to create the necessary DSL (domain specific language) that allows you to interact with your database in safe, lightning fast, strongly typed fashion.

The Getting Started guide and examples use the `infer_schema!` macro, the major disadvantage of using this macro is, firstly, that you have _no_ idea how diesel actually interprets your database's types (this matters for your models, which will be revealed later) and secondly it requires a bootstrapped database instance during compile time, bit of a pain when compiling as part of a pipeline that might not have your database and compiler toolchain adjacent to each other.

I recommend the use of the `diesel print-schema` subcommand, simple copy and paste the inferred schema into a file in your project, in the example this is called `schema.rs` but it just as easily be pasted into `lib.rs` or `main.rs`. 

What you will see is a `table!` macro similar to

    table! {
	    users {
        id -> Integer,
        name -> VarChar,
       }
    }
    
You want to grab the datatypes (`Integer` and `VarChar` in this case) and immediately run over to `docs.rs/diesel` and plug the into the search bar. This will type you straight to `diesel::types::Foo` (just click on `diesel::types`) and allow you to inspect the implemented `ToSql` and `FromSql` traits for each type. For example `Integer` maps to `i32` in rust. This is super useful when implementing your models, or wrestling with compile-time errors.
  
### Create Models

As with the schema you don't _have_ to put your models in the `models.rs` file, I recommend splitting your models out using the modules systems, something like the following might assist, especially when dealing with lots of models. I also point this out as I initially struggled to split the "magic" used to drive diesel from idiomatic rust. It turns out diesel is just good old familiar rust once you know how it is structured and now how to deal with the generated code.

    models/
      users/
        mod.rs
      posts/
        mod.rs
        
Models are normal rust structs that map to your tables, they normally represent a single object in a single row and your table represents the collection of these objects. Diesel assumes that the `User` object will be stored in the `users` table, although you can arrive it with some directives as discussed next.

An example model expressed as a rust struct is as follows, plucked straight from the Getting Started guide:

    #[derive(Queryable)]
      pub struct Post {
      pub id: i32,
      pub title: String,
      pub body: String,
      pub published: bool,
    }

Note that diesel's SQL types don't support unsigned integres on postgresql, and that it supports some other custom datatypes too, such as MACADDR (implemented as a `[u8; 6]`), so it might be worthwhile checking `diesel::pg::types` in the docs.
 
### Derives, aka "The Codegen Magic"

Diesel's code generator is primarily used to embue your rust structs with SQL magic without you having to handcode a ton of functionality, it also turns the rust type system into the magic ingredients that can be used to construct fast and reliable SQL that gets transmitted to the database.

In order to imbue your structs with the SQL goodness you use rust's custom derive functionality, for example:

    #[derive(Queryable)]
      pub struct Post {
      pub id: i32,
      ...
            
I loved stumbling across the following in the tests, it truly made me smile at the genius of the authors, but it also made me scratch my head, where on earth are these things used and how do they work?

    #[derive(PartialEq, Eq, Debug, Clone, Queryable, Identifiable, Insertable, AsChangeset, Associations)]
    #[table_name = "users"]
    pub struct User {
        pub id: i32,
        pub name: String,
        pub hair_color: Option<String>,
    }
        
I'm so glad you asked...

First, a 'pro tip' from a not-a-pro, you should install the cargo subcommand `expand`

    `cargo install cargo-expand`
    
This allows you to run, the following command and actually _see_ what is generated (warning, this guide assumes that you know rust, but the following might make even a seasoned rust developer's eyes water, feel free to skip it or use as bedtime reading).
      
    `cargo exand`
    
The full glory of the generated DSL is revealed... of particular interest is the columns for each model, to parse the output is left as an exercise for the bored reader, or the readers who like a challenge.

Lets look at the derives in more detail.

#### PartialEq, Eq, Debug, Clone

These are your standard rust derives that most developers rely on, you probably want at least `Debug` if you like `println!` (or the shiny new `eprintln!` debugging).

#### Queryable

The first thing that most developers usually want to do with their databases is query the data within it, in order to do this you need to decorate your `struct` as follows:

    #[derive(Queryable)]
    struct User {
        id: i32,
        firstname: String,
        lastname: String,
        age: i32,
    }
    
This will cause `diesel_codegen` to generate the query DSL needed for `diesel` to be able to execute queries based on your objects. The code above represents one "row" in a database table called `users`, with columns `firstname`, `lastname` and `age`.

A primary key column is mandatory to work with `diesel`, but is probably good practice anyway.

How to use the DSL to construct queries is addressed later in the article.

#### Insertable

Unless you are coming from an existing system, you probably want to insert data into your database too. A typical "insertable" object would look like this:

    #[derive(Insertable)]
    #[table_name="users"]
    struct NewUser {
        firstname: String,
        lastname: String,
        age: i32,
    }
    
Note that we have dropped the `id` field as the SQL server will handle this for us, by default (this might change for advanced use-cases).

Also note that we now explicitly name the table, `users`, as there is no direct correlation between the struct name and the table name.

#### Identifiable

Inevitably you are going to use SQL joins to construct results from more than one table at a time. In order for the join to successfully resolve the exact object in your target table this table needs to be annotated as follows:

    #[derive(Identifiable)]
    struct User {
        id: i32,
        firstname: String,
        lastname: String,
        age: i32,
    }

By default `diesel` will assume your primary key is called `id`, if it is not you can override it as follows:

    #[derive(Identifiable)]
    #[primary_key(guid)]
    struct User {
        guid: i32,
        firstname: String,
        lastname: String,
        age: i32,
    }
    
#### Associations

The tables that you want to enrich with your "Identifiable" data needs to be annotated as follows:

    #[derive(Associations)]
    #[belongs_to(User)]
    struct ActiveUsers {
        id: i32,
        user_id: i32,
        last_active: NaiveDateTime
    }
    
This will allow you to join the `User` data to this table by looking up the user, defined by the field `user_id`, corresponding `id` field in the `User` table. In the event where your foreign key field is not specified in the pattern `type_id`, you will need to manually map it, as in:

    #[derive(Associations)]
    #[belongs_to(User, foreign_key="user_lookup_key")]
    struct ActiveUsers {
        id: i32,
        user_lookup_key: i32,
        last_active: NaiveDateTime
    }
    
#### AsChangeset

Full disclosure, I have no idea what this is for :-) Maybe it will be covered in a follow-up guide. To attempt to grok this yourself have a look at `diesel::query_builder::AsChangeset` in the docs.
        
## Using Diesel

At this point, you are ready to actually _do_ something with `diesel`, probably insert some data and make some queries. This is covered by multiple examples in the Getting Started guide. There are, however, some items that need to be unpacked to, hopefully, make the lightbulbs go on.

First and foremost, at this point, it is super important to underscore that you are now entering the normal `_rust_` world. What I mean by this is that the `codegen` and "magic" bits of `diesel` now take a backseat.

This implies that when you look at example code, there is no more slight of hand, trust the complier and take time to unpack the statements and expressions like you would normally do.

You'll find that you are calling normal methods, passing normal data-structures (or references to them) and that you can `println!`, debug, step, re-order and organise like you would any other `rust` app. It took me an unfortunately long time to `grok` this, but it is an important insight and will allow you to code without fear.

### Connecting to the database

The `postgresql` example uses the code the function `pub fn establish_connection() -> PgConnection` to wrap the connection into a convenient function call for re-use in the rest of the example code.

`PgConnection` encapsulates the handle to the `posgresql` for you, keep this object around or recreate it on demand. When needed look at the `r2d2` crate to create a pool of database connections.

Everything you execute using `diesel` will depend on the connection object to be available.

### Inserting data

The guide specifies the following function in `lib.rs`, lets step through it:

    pub fn create_post(conn: &PgConnection, title: &str, body: &str) -> Post {
        use schema::posts;

        let new_post = NewPost {
            title: title,
            body: body,
        };

        diesel::insert(&new_post).into(posts::table)
            .get_result(conn)
            .expect("Error saving new post")
    }

First note the reference to the connection `&PgConnection`, a new `PgConnection` can aslo be instantiated by calling something like:

    let conn = establish_connection();

This would not be a great idea, as you will be creating a new connection everytime the function is called, but it serves to illustrate that there is nothing magical about a connection, unless you start to DOS your database server with too many connections, then you will of course also discover that there is nothing magical about a server either, but in a different context.

The next line, `use schema::posts;`, stumped me for a long time, as this is using generated code. In the `diesel::insert` statement, we see the use of `posts::table`. At this point is might be a good idea to take the output of `cargo expand` and have a look at it in an IDE of sorts. Try the following:

    # Check out a copy of diesel
    git clone https://github.com/diesel-rs/diesel.git
    cd diesel/examples/postgres/getting_started_step_3/
    
    # Build the example (assuming your postgres instance is ready 
    # and running, see the docker hint above)
    echo DATABASE_URL=postgres://username:password@localhost/diesel_demo > .env
    diesel setup
    cargo build
    
    # Expand the code code (assuming use installed cargo-expand)
    cargo expand --lib > expanded.rs
    
If you search for `mod schema` under it you will see `mod posts`, and there you'll find a  `table` struct (empty).

Next we have a new object (which is `Insertable`) called `new_post`. 

Lastly we have the actual `diesel` statement `insert`. If you consult the documentation you will see that the `diesel` module has 5 functions, `insert`, `delete`, `insert_default_values`, `select` and `update`.

If you look at the `insert` function you'll see it accepts `records` (in this case our new user, but it can also be a `Vec<T>`) and returns an `IncompleteInsertStatement` object, which has one method called `into()` which accepts your `table` struct. The patterns to remember, in other words, you could have called `into(schema::posts::table)` and avoided the use statement.

In this case `schema` is the name of _your_ module, due to the schema being generated in the `schema.rs` file.

### Querying data
 
 
 > To be continued...
 
 
 
 
 
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

### Diesel CLI

The Diesel CLI deserves special mention, you will use it to create your migrations and execute them. Other subcommands that warrant special mention are:

    diesel database reset
    
This will drop your database 

[1] https://12factor.net
