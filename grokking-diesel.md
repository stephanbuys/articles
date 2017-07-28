Outstanding Questions:
- statement about custom derive correct?
- primary key requirement statement below seems flimsy.
- Rust capitilisation? (pick one)
- am I mixing statements and expressions?
- am I mising structs and objects
- can we safely skip joins, etc (complex queries)? this article is already _dense_

## Introduction

 Diesel (http://diesel.rs) is an ORM (Object-relational mapping) and Query Builder written in Rust. It supports Postgresql, Mysql and SQLite. It makes use of Rust's custom derive functionality to generate all the code you need in order to get all the power of Rust's type system when interacting with a database. This means that you get compile-time validation of your code, thus eliminating possible runtime errors.
 Diesel is very powerful, but by just following the examples might still leave you scratching your head, asking questions like "where do these types come from?", "what should I add here", "how does this work?". This article is meant to shed a little bit of light on the topic from the perspective of a intermediate-level Rust developer.

There are many topics that this article will not cover, but the hope is that you will be able to "grok" enough of Diesel so that the rest becomes obvious and that you understand how it works, to the extent that new territory that you explore in this crate also become simpler to understand.

## First Steps

You should first run through the Getting Started guide an the official Diesel website http://diesel.rs/guides/getting-started/, this article focusses on Postgresql. By far the easiest way to get a Postgresql server running for your tests is by using Docker (www.docker.com), the following one-liner should do the trick:

    docker run -d --rm --name postgres -e POSTGRES_USER=username -e POSTGRES_PASSWORD=password -p 127.0.0.1:5432:5432 postgres
    
> This is _not_ for production use, the whole container will be removed when the container is stopped, use `docker stop postgres`, but it is super handy for the getting started guide.

### A word about the format of the rest of this article

This articles assumes that you have worked through the Getting Started guide, it deliberately glosses over a whole set of features, this is intentional and done in order to keep the article as concise as possible.
    
## Core Components And Community

At its core Diesel consists of 4 main components:

The `diesel` crate, `diesel_codegen` crate (the code generator) and the `diesel` cli, which by now you should have been introduced to.

The last major component, and 'secret weapon' or 'unofficial guide' (according to me) is the the test suite in the official repo (https://github.com/diesel-rs/diesel/tree/master/diesel_tests), it served as my guide whenever I got stuck, although it will surely be superseded by documentation as the project continues to evolve.

### Environment

When I started off with Diesel I was baffled by the `infer_schema!` macro in `schema.rs` (more on this below), which in turn used `dotenv:DATABASE_URL"`. The `dotenv` crate is just a convenience library that allows you to put your `DATABASE_URL` in a hidden environment file, ala `.env` (dot env, get it?).

It follows that you can also just specify `DATABASE_URL` in your environment, this is particularly handy when you run your Diesel-using Rust binary in a docker container ala 12-Factor apps [1].
        
The Diesel cli tool also reads the `.env` file, if it is not available you will need to define `DATABASE_URL` in the environment or pass it the `--database-url` parameter.

> Definitely check out the `examples/sqlite/getting_started_step_3/README.md` file in order to learn how to configure the `DATABASE_URL` for SQLite, it doesn't use a URI format.

## Basic Flow

The core workflow for creating an app built with Diesel can be broken down as follows:

### Design A Schema

This is very obvious to seasoned SQL veterans, luckily for the rest of us the `diesel` cli's `migration` subcommand allows us to easily iterate on our design and even evolve it over time. It is, however, very useful to have a clear idea of what you want your database to look like up front, and it should be noted that Diesel only works with tables that have a primary key. 

### Create Migrations

Follow the pattern in the Getting Started guide, and note that you can add more migrations at any time using the `diesel migration generate` subcommand. Migrations will be run in order whenever you run `diesel migration run`. You can rerun the last migration by issuing `diesel migration redo` and if you truly get stuck, you can run the following command, but please never do so on a production database, `diesel database reset`.

When designing your tables you should put plurals of your table names, for example, Diesel will take the model `User` and search for the table `users`. You can define custom table names, but knowing the Diesel developers' assumptions will probably spare you some confusion.

Diesel will take `PascalCase` Rust structs (which will probably describe single objects, or rows in your database table) and translate them into `snake_case` table names with a `s` tucked at the end to "pluralise" it. For example `AFancyNamedObject` will be assumed to map to a table named `a_fancy_named_objects`.

### Infer The Schema From The Database

Diesel has the ability to inspect your actual database and infer a schema for use in Rust, this will be used to create the necessary DSL (domain specific language) that allows you to interact with your database in safe, lightning fast and strongly typed fashion.

The Getting Started guide and examples use the `infer_schema!` macro. The major disadvantage of using this macro is, firstly, that you have _no_ idea how Diesel actually interprets your database's types (this matters for your models, which will be revealed later) and secondly it requires a bootstrapped database instance during compile time, which can be a bit of a pain when compiling as part of a pipeline that might not have your database and compiler toolchain available to each other.

It is recommended that you use of the `diesel print-schema` subcommand, simply copy and paste the inferred schema into a file in your project (in the Getting Started guide this is `schema.rs` but it could just as easily be pasted into `lib.rs` or `main.rs`). 

What you will see is a `table!` macro similar to:

    table! {
	    users {
        id -> Integer,
        name -> VarChar,
       }
    }
    
You want to grab the datatypes (`Integer` and `VarChar` in this case) and immediately run over to `docs.rs/diesel` and plug it into the doc search bar. This will take you straight to `diesel::types::Foo` (also explore `diesel::types`) and allow you to inspect the implemented `ToSql` and `FromSql` traits for each type. For example `Integer` maps to `i32` in Rust. This is super useful when implementing your models, or wrestling with compile-time errors.
  
### Create Models

As with the schema you don't _have_ to put your models in the `models.rs` file, it is recommend that you split your models out using the Rust modules system, something like the following might assist, especially when dealing with lots of models. 

> I initially struggled to split the "magic" used to drive Diesel from idiomatic Rust. It turns out Diesel is just good old familiar Rust once you know how it is structured and now how to deal with the generated code.

    models/
      users/
        mod.rs
      posts/
        mod.rs
        
Models are normal Rust structs that map to your tables, they normally represent a single object in a single row and your table represents the collection of these objects. Diesel assumes that the `User` object will be stored in the `users` table, although you can override it with some directives as discussed next.

An example model expressed as a Rust struct is as follows, plucked straight from the Getting Started guide:

```
#[derive(Queryable)]
  pub struct Post {
  pub id: i32,
  pub title: String,
  pub body: String,
  pub published: bool,
}
```

Note that diesel's SQL types don't support unsigned integers in Postgresql, and that it supports some other custom datatypes too, such as MACADDR (implemented as a `[u8; 6]`), so it might be worthwhile checking `diesel::pg::types` in the docs.
 
### Derives, aka "The Codegen Magic"

Diesel's code generator is primarily used to embue your Rust structs with SQL magic without you having to handcode a ton of functionality, it also turns the Rust type system into the magic ingredient that can be used to construct the fast and reliable SQL that gets transmitted to the database.

In order to imbue your structs with the SQL goodness you use Rust's custom derive functionality, for example:

```
#[derive(Queryable)]
pub struct Post {
    pub id: i32,
    ...
```
  
A more complex example:
            
> I loved stumbling across the following in the tests, it truly made me smile at the genius of the authors, but it also made me scratch my head, where on earth are these things used and how do they work?

```
#[derive(PartialEq, Eq, Debug, Clone, Queryable, Identifiable, Insertable, AsChangeset, Associations)]
#[table_name = "users"]
pub struct User {
    pub id: i32,
    pub name: String,
    pub hair_color: Option<String>,
}
```
        
First, a 'pro tip' from a novice, you should install the cargo subcommand `expand`:

`cargo install cargo-expand`
    
This allows you to run, the following command and actually _see_ what is generated (warning, this guide assumes that you know Rust, but the following might make even a seasoned Rust developer's eyes water, feel free to skip it or use as bedtime reading).
      
`cargo exand`
    
The full glory of the generated DSL is revealed... of particular interest is the columns for each model. Parsing the output is left as an exercise for the bored reader, or the readers who like a challenge.

Lets look at the derives in more detail.

#### PartialEq, Eq, Debug, Clone

These are your standard Rust derives that most developers rely on when appropriate, you probably want at least `Debug` if you like `println!` (or the shiny new `eprintln!`) for  debugging.

#### Queryable

The first thing that most developers usually want to do with their databases is query the data within it, in order to do this you need to decorate your `struct` as follows:

```
#[derive(Queryable)]
struct User {
    id: i32,
    firstname: String,
    lastname: String,
    age: i32,
}
```
    
This will cause `diesel_codegen` to generate the query DSL needed by Diesel. The code above represents one row in a database table called `users`, with columns `firstname`, `lastname` and `age`.

A primary key column is mandatory to work with Diesel, but is probably good practice anyway.

How to use the DSL to construct queries is addressed later in the article.

#### Insertable

Unless you are coming from an existing system, you probably want to insert data into your database too. A typical "insertable" object would look like this:

```
#[derive(Insertable)]
#[table_name="users"]
struct NewUser {
    firstname: String,
    lastname: String,
    age: i32,
}
```
    
Note that we have dropped the `id` field as the SQL server will handle this for us (this might change for advanced use-cases).

Also note that we now explicitly name the table, `users`, as there is no direct correlation between the struct name and the table name.

#### Identifiable

Inevitably you are going to use SQL joins to construct results from more than one table at a time. In order for the join to successfully resolve the exact object in your target table this `struct` needs to be annotated as follows:

```
#[derive(Identifiable)]
struct User {
    id: i32,
    firstname: String,
    lastname: String,
    age: i32,
}
```

By default Diesel will assume your primary key is called `id`, if it is not you can override it as follows:

```
#[derive(Identifiable)]
#[primary_key(guid)]
struct User {
    guid: i32,
    firstname: String,
    lastname: String,
    age: i32,
}
```
    
#### Associations

The tables that you want to enrich with your `Identifiable` data needs to be annotated as follows:

```
#[derive(Associations)]
#[belongs_to(User)]
struct ActiveUsers {
    id: i32,
    user_id: i32,
    last_active: NaiveDateTime
}
```
    
This will allow you to join the `User` data to this table by looking up the user, defined by the field `user_id`, corresponding to the `id` field in the `User` table. In the event where your foreign key field is not specified in the pattern `type_id`, you will need to manually map it:

```
#[derive(Associations)]
#[belongs_to(User, foreign_key="user_lookup_key")]
struct ActiveUsers {
    id: i32,
    user_lookup_key: i32,
    last_active: NaiveDateTime
}
```
        
## Using Diesel 

At this point, you are ready to actually _do_ something with Diesel, probably insert some data and make some queries. This is covered by multiple examples in the Getting Started guide. There are, however, some items that need to be unpacked to, hopefully, make the lightbulbs go on.

First and foremost, at this point, it is super important to underscore that you are now entering the _normal_ Rust world. What I mean by this is that the codegen and magic bits of Diesel now take a backseat.

This implies that when you look at example code, there is no more slight of hand, trust the complier and take time to unpack the statements and expressions like you would normally do. You'll find that you are calling normal methods, passing normal data-structures (or references to them) and that you can `println!`, debug, step, re-order and organise like you would any other Rust app or crate. It took me an unfortunately long time to comprehend this, but it is an important insight and will allow you to code without fear.

### Connecting to the database

The `postgresql` example uses the function `pub fn establish_connection() -> PgConnection` to wrap the connection into a convenient function call for re-use in the rest of the example code.

`PgConnection` encapsulates the handle to `posgresql` for you, keep this object around or recreate it on demand. When needed look at the `r2d2` crate to create a pool of database connections.

Everything you execute using Diesel will depend on the connection object being available.

### Inserting data

The guide specifies the following function in `src/lib.rs`, lets step through it:

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

First note the reference to the connection `&PgConnection`. A new `PgConnection` could alo be instantiated by calling something like (as is done of some of the other files in `src/bin/`:

    let conn = establish_connection();

The next line, `use schema::posts;`, stumped me for a long time, as this is using generated code. In the `diesel::insert` statement, we see the use of `posts::table`. 

At this point is might be a good idea to take the output of `cargo expand` and have a look at it in an IDE of sorts. Try the following:

    # Check out a copy of Diesel     git clone https://github.com/diesel-rs/diesel.git
    cd diesel/examples/postgres/getting_started_step_3/
    
    # Build the example (assuming your postgres instance is ready 
    # and running, see the docker hint above)
    echo DATABASE_URL=postgres://username:password@localhost/diesel_demo > .env
    Diesel setup
    cargo build
    
    # Expand the code code (assuming use installed cargo-expand)
    cargo expand --lib > expanded.rs
    
If you search for `mod schema` under the output you will also see `mod posts`, and there you'll find a `table` struct (empty, but with a bunch of Traits impl'ed).

Next we have a new object (which is `Insertable`) called `new_post`. 

Lastly we have the actual Diesel statement `insert`. If you consult the documentation you will see that the Diesel module has 5 functions, `insert`, `delete`, `insert_default_values`, `select` and `update` at its core.

If you look at the `insert` function you'll see it accepts `records: T` (in this case our new user, but it can also be a `Vec<T>`) and returns an `IncompleteInsertStatement` object, which has one method called `into()` which accepts your `table` struct. You could also have called `into(schema::posts::table)` and avoided the use statement.

In this case `schema` is the name of _your_ module, due to the schema being generated in the `schema.rs` file.

### Querying data
 
 Again, we'll refer to a function in the guide, in this case the `src/bin/show_posts.rs` file:
 
```
extern crate diesel_demo;
extern crate diesel;

use self::diesel_demo::*;
use self::diesel_demo::models::*;
use self::diesel::prelude::*;

fn main() {
    use diesel_demo::schema::posts::dsl::*;

    let connection = establish_connection();
    let results = posts.filter(published.eq(true))
        .limit(5)
        .load::<Post>(&connection)
        .expect("Error loading posts");

    println!("Displaying {} posts", results.len());
    for post in results {
        println!("{}", post.title);
        println!("----------\n");
        println!("{}", post.body);
    }
}
```

Stepping through it from the top, first we import our own crate (the example `Cargo.toml` specifies the `crate` name as `diesel_demo`). The we import the Diesel crate.

`use self::diesel_demo::*;` gives us access to the `establish_connection()` function, `use self::diesel_demo::models::*;` gives us access to the actual structs that we defined in our `models.rs` files, in this case the `Post` struct.

`use self::diesel::prelude::*;` is a necessary import that bring a whole bunch of Diesel Traits and types into scope. It is needed for Diesel to work and beyond the scope of this article to dive into.

In the `main()` function is where we encounter the use of some magic again, specifically the `use diesel_demo::schema::posts::dsl::*;` line. When I first started using Diesel I was stumped between the difference of `schema::tablename::*` imports as used above when inserting code and `schema::posts::dsl::*` dsl imports used here. 

A look at the output of `cargo expand` allows for some clarification, but in short, `schema::posts::dsl::*` brings the `columns` of your table into scope. Each column type that is generated for you has a collection of `expression_methods` implemented on it. To rephrase this, the `dsl` (domain specific language) allows us to use the `columns` in our table's names (as defined by the `schema.rs` generated code) and apply logic to it in order to construct our SQL queries. (see http://docs.diesel.rs/diesel/expression_methods/global_expression_methods/trait.ExpressionMethods.html#method.desc)
    
Extra credit if you spotted the convenience import of `table` into the dsl module (I think this is for convenience).

#### Building queries

In SQL we construct a select statement to return values from our database, in Diesel we use Rust's type system to construct type-checked, safe versions of those. This is great for anyone who has ever struggled with the fragility of SQL queries, and its implied security risks, only valid SQL should compile successfully.

The next statement `posts.filter(published.eq(true))` reflects that we want to run the `filter` method on the `posts` table (conveniently also imported into our context by the `use schema::posts::dsl::*` statement). `filter` takes a constructed filter as its input. A filter is constructed by combining columns and their expression methods.

To inspect the results of this you can rewrite the relevant code as:

```
use diesel::debug_sql;
let posts_with_sql = posts.filter(published.eq(true))
    .limit(5);

println!("SelectStatement: {:#?}", posts_with_sql);

let results = posts_with_sql
    .load::<Post>(&connection)
    .expect("Error loading posts");
```

If you look at the output to the terminal you will see a `SelectStatement` object is returned, you can also use `println!("SQL: {}", debug_sql!(posts_with_sql));` to look at the SQL that would be generated (add `#![feature(use_extern_macros)]` to the top of the file to use this macro).

The main takeaway from this section is that you first build up the relevant SQL statement by using your `table` and `columns` imported from the `dsl` module, and that this is introspect-able by `println!` and `debug_sql`.

You should familiarize yourself as much as possible with the different `expression_methods` you can call on columns - it will get you well on your way to building the SQL queries you want to use in your applications. 
 
### Getting results

The last item we'll handle in this article is actually reading your results. In the `show_posts.rs` binary this is achieved by the lines:

```
let results = ...omitted...
    .load::<Post>(&connection)
    .expect("Error loading posts");
```

As can be seen we are calling the `.load()` method of the `SelectStatement` struct that we inspected a bit closer above. The `.load()` method is generic, so we need to give the compiler some hints as to what type we want to return. The parameter for the load function is a reference to (or borrow of) the `connection` object returned by `establish_connection`.

`load` returns a result object, which in the case of this executable we deal with by just unwrapping the result with the `expect()` method. We then pass the result, if we were successful, to the `results` binding. 

See `diesel::prelude::LoadDsl` or `diesel::prelude::FirstDsl` in the docs for some alternatives to `load`. As will be seen the methods return a `QueryResult` which is just a `Result<T, Error>;` type alias.

In all the methods just mentioned, either return a single object (`QueryResult<T>`) or a vector of objects (`QueryResult<Vec<T>>`), in other words, one row or multiple rows of the selected columns in the database table.

#### Wresting with results

If we wanted to get all the results back, we could have used the code:

```
let results :Vec<Post> = posts
    .load(&connection)
    .expect("Error loading posts");
```

`load` and `get_results` are equivalent, in other words, they both return a `Vec<T>` result (`QueryResult`). Note that the code was reformatted to illustrate the return-type hint given to the compiler in the binding, `let results :Vec<Post> = ...`. 

Lastly, sometimes you will use a `select` method or `join` method (not illustrated in this article, but do check out the 'unofficial guide' aka `diesel_tests` mentioned above) which will return rows that do not map directly to the fields in your models, you should use a `tuple` to collect your results from your `load` method in that case. This may look like:

```
let results :Vec<(i32, String, String, bool)>= posts
    .load(&connection)
    .expect("Error loading posts");
```

Above we expressed the `Post` model or `struct` as a `tuple` of its constituent parts.

## What we didn't cover and what's next

There is a lot that wasn't covered in this article, even though it is quite a large volume of information to consume in its own right. As mentioned initially, the goal was to explain enough of Diesel and its structure so that it would become possible for interested developers to "grok" or understand it, and then help themselves.

I hope you made it this far and that you enjoyed the journey, please send feedback and enjoy Rust and Diesel.

[1] https://12factor.net
