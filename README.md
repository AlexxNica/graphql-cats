## GraphQL Compatibility Acceptance Tests

[![Join the chat at https://gitter.im/graphql-cats/graphql-cats](https://badges.gitter.im/graphql-cats/graphql-cats.svg)](https://gitter.im/graphql-cats/graphql-cats?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

The aim of this project is to provide a set of compatibility acceptance tests for libraries 
that implement the [GraphQL specification](https://github.com/facebook/graphql). Since GraphQL 
server implementations are written in different programming languages, this project defines test cases in a 
language independent format ([YAML](http://yaml.org)).

Using this test suite has following advantages:

* The users and the author of a GraphQL library can be more confident that a library is compliant to the GraphQL 
  specification and if not, which parts of the library are not compliant
* It reduces amount of work a library implementor needs to do
* Makes it much easier for existing implementations to keep up with the specification changes

### Contribution & Open Questions

At the moment this project finds itself in an infant phase. Any contributions are very welcome! Please join us in 
a [gitter chat](https://gitter.im/graphql-cats/graphql-cats). If you would like to 
join this effort and be part of the project, please let us know.

There are still a number of open questions that we need to figure out together:

* [File format and structure](https://github.com/graphql-cats/graphql-cats/issues/3)
* [Error message assertions](https://github.com/graphql-cats/graphql-cats/issues/5)
* [Versioning & Distribution](https://github.com/graphql-cats/graphql-cats/issues/4)

### Repository format

`scenarios` folder contains a set of `*.yaml` and `*.graphql` files. Every subfolder represents a particular part of GraphQL specification. 
`*.yaml` files contain scenarios with a list of test cases. `*.graphql` files contain IDL schema definitions that are used in some of the 
scenarios (they are explicitly referenced). 

### Scenario File Format

Every scenario is a [YAML](http://yaml.org) file with following structure: 

* `scenario` - _String_ - the name of this scenario
* `background` - _Object_ (optional) - common definitions used by all of the tests
  * `schema` - _String_ (optional) - inline GraphQL IDL schema definition
  * `schema-file` - _String_ (optional) - IDL schema definition file path relative to the scenario file 
  * `test-data` - _Object_ (optional) - test data used for query execution and directives 
  * `test-data-file` - _String_ (optional) - test data file path relative to the scenario file. File can be in either JSON or YAML format.   
* `tests` - _Array of Objects_ - list of tests
  * `name` - _String_ - a name of the test
  * `given` - _Object_ - input information for the test
    * `query` - _String_ - the GraphQL query to execute an action against
    * `schema` - _String_ (optional) - inline GraphQL IDL schema definition
    * `schema-file` - _String_ (optional) - IDL schema definition file path relative to the scenario file
    * `test-data` - _Object_ (optional) - test data used for query execution and directives
    * `test-data-file` - _String_ (optional) - test data file path relative to the scenario file. File can be in either JSON or YAML format.
  * `when` - _Object_ - action that should be performed in the test. See the **Actions** section for a list of available actions.
  * `then` - _Object_ | _Arrays of Objects_ - assertions that verify result of an action. See the **Assertions** section for a list of available actions.

Definitions in the `given` part of a test may override definitions defined in the `background` section.
    
#### Actions

* **Query parsing**
  * `parse` - _Any_ - just parses the query 
* **Query validation**
  * `validate` - _Array of Strings_ - the list of validation rule names to validate a query against. This action will only validate query without executing it. 
* **Query execution**
  * `execute` - _Object_ - executes a query
    * `operation-name` - _String_ (optional) - the name of an operation to execute (in case query contains more than one)
    * `variables` - _Object_ (optional) - variables for query execution
    * `validate-query` - _Boolean_ (optional) - `true` if query should be validated during the execution, `false` otherwise (`true` by default) 
    * `test-value` - _String_ (optional) - the name of a field defined in the `test-data`. This value should be passed as a root value to an executor.  
    
#### Assertions

* **Validation/Parsing is successful**
  * `passes` - _Any_ - verifies that validation was successful. Only applicable in conjunction with query validation/parsing action  
* **Parsing syntax error**
  * `syntax-error` - _Any_ -  query contains a syntax error. Only applicable in conjunction with query parsing action (No text matching takes place, it just verifies the fact that there is a syntax error)  
* **Data match**
  * `data` - _Object_ - compares the `data` object with the result of a query execution. Only applicable in conjunction with query execution action   
* **Error count**
  * `error-count` - _Number_ - number of the errors in execution/validation results  
* **Error contains match**
  * `error` - _String_ - execution/validation results contain provided error message (provided error message may contain only part of the actual message)  
  * `loc` - _Array of Objects_ | _Array of Arrays of Numbers_ (optional) - a list of error locations
    * `line` - _Number_ 
    * `column` - _Number_ 
* **Error regex match**
  * `error-regex` - _String_ - execution/validation results contain provided error message (uses provided regular expressions to match an error message)  
  * `loc` - _Array of Objects_ | _Array of Arrays of Numbers_ (optional) - a list of error locations
    * `line` - _Number_ 
    * `column` - _Number_ 
* **Execution exception contains match**
  * `exception` - _String_ - execution may throw an exception during the execution (for instance, if operation name is not provided, but query contains more than one named operation). 
    This assertion verifies the message of this exception (provided error message may contain only part of the actual message). 
    Only applicable in conjunction with query execution action    
* **Execution exception regex match**
  * `error-regex` - _String_ - execution may throw an exception during the execution (for instance, if operation name is not provided, but query contains more than one named operation). 
    This assertion verifies the message of this exception (uses provided regular expressions to match an error message). 
    Only applicable in conjunction with query execution action  

### Execution Semantics

All test cases and scenarios are self-contained. This means that they contain the schema definition, query and test data for an execution.
Schema definition expected to be materialized in executable form. By default, fields' `resolve` function must return the values provided 
with the `test-data`/`test-data-file` and `test-value` properties which are defined in a scenario file. Some fields may have special behaviour
which is defined via schema directives which you can find in the next section.
 
### Schema Directives

#### @resolveString

```graphql
directive @resolveString(value: String!) on FIELD_DEFINITION
```

Resolves `String` field with provided `value`. Value may contain a placeholders which refer to the field arguments and must be replaced during the field resolution.
  
**Example:**

for given schema definition:

```graphql
type Query {
  article(title: String): String @resolveString(value: "Test article with title '$title'")
}

schema {
  query: Query
}
```

and query:

```graphql
{
  article(title: Foo)    
}
```

result of an execution should produce following JSON:

```json
{
  "data": {
    "article": "Test article with title 'Foo'"
  }
}
```

#### @argumentsJson

```graphql
directive @argumentsJson on FIELD_DEFINITION
```

Resolves `String` field with compact JSON representation of all field arguments.

#### @resolvePromiseString

```graphql
directive @resolvePromiseString(value: String!) on FIELD_DEFINITION
```

Resolves `String` field with provided `value` which may contain argument placeholders. Internally field should be resolved with successful 
promise or an equivalent of promise. Ideally short delay should be applied before actual resolution. If implementation does not support
promises, then it may return an eager value.

#### @resolveEmptyObject

```graphql
directive @resolveEmptyObject on FIELD_DEFINITION
```

Resolves `ObjectType` field with an empty object.

#### @resolveTestData

```graphql
directive @resolveTestData(name: String!) on FIELD_DEFINITION
```

Resolves field with a value provided via `test-data` property.

#### @resolvePromiseTestData

```graphql
directive @resolvePromiseTestData(name: String!) on FIELD_DEFINITION
```

Resolves field with a value provided via `test-data` property. Internally field should be resolved with successful 
promise or an equivalent of promise. Ideally short delay should be applied before actual resolution. If implementation does not support
promises, then it may return an eager value.

#### @resolvePromise

```graphql
directive @resolvePromise on FIELD_DEFINITION
```

Resolves field with context value (as in the default case). Internally field should be resolved with successful 
promise or an equivalent of promise. Ideally short delay should be applied before actual resolution. If implementation does not support
promises, then it may return an eager value.

#### @resolveError

```graphql
directive @resolveError(message: String!) on FIELD_DEFINITION
```

Field resolution fails with provided error `message`.

#### @resolveErrorList

```graphql
directive @resolveErrorList(values: [String!]!, message: [String!]!) on FIELD_DEFINITION
```

Field of type `[String]` fails with provided errors `messages`. Even though field resolution fails, it still must produce a value which contains 
provided `values`.

#### @resolvePromiseReject

```graphql
directive @resolvePromiseReject(message: String!) on FIELD_DEFINITION
```

Field resolution fails with provided error `message`. Internally field should be resolved with rejected 
promise or an equivalent of promise. Ideally short delay should be applied before actual resolution. If implementation does not support
promises, then it may return an eager error.

#### @resolvePromiseReject

```graphql
directive @resolvePromiseRejectList(values: [String!]!, message: [String!]!) on FIELD_DEFINITION
```

Field of type `[String]` fails with provided errors `messages`. Even though field resolution fails, it still must produce a value which contains 
provided `values`. Internally field should be resolved with rejected 
promise or an equivalent of promise. Ideally short delay should be applied before actual resolution. If implementation does not support
promises, then it may return an eager error.

### Test Data

Test data is a JSON/YAML object where every top-level field may be referenced by name within a scenario (with `test-value` property), 
schema directive (`@resolveTestData`, `@resolvePromiseTestData`) or test data itself (with `{"$ref": "name"}` value).