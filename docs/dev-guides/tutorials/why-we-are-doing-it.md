# Why We Are Doing It

## Why bother to do it programmatically? Why not UI?
In some cases, you might want to construct Metadata events directly and use programmatic ways to emit that metadata to DataHub.

## What are the main use-cases?
You can do a lot things with APIs, below are some of the use cases. 

### Basic Usage
* [Adding Tags](http://localhost:3000/docs/dev-guides/tutorials/adding-tags)
* [Adding Terms]()
* [Adding Ownership]()

### Advanced Usage 
* Adding Tags on Entities Based on Entity Type
* Ingesting Entity from CSV Files
* Adding Column-level Lineages

## Our APIs (TBD)
Datahub supports 3 APIs : GraphQL, SDKs and OpenAPI. Each method has different usage and format. 
Here's a good grasp of what each API can do. 

|                          | GraphQL | SDK | OpenAPI |
|--------------------------|--------|---|---|
| Add Tags/Terms/Ownership | ✅      |||
| Create Dataset           ||| ✅        |
| Delete Dataset           ||| ✅  |
| Search Dataset           ||| ✅  |


