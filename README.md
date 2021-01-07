# Postman Contract Test Generator
When building APIs, a common need is to validate the shape of the requests and responses. You want to verify the implementation of the API matches the definition document. This is typically done through *contract tests*, which exercise required fields, optional fields, and things like query and path parameters.

[Postman](https://www.postman.com/) has the ability to group requests together and associate them to your API as a contract test. The intention of this feature is to build a [collection](https://www.postman.com/collection/) manually to add coverage to your API.

# Objective
To build an automated way to provide an exhaustive set of tests providing close to 100% coverage for contract tests. The **contract test generator** should not need to be maintained, it should be able to dynamically create all tests with every run based on the Open API Specification (OAS) document defining the API.

The generator tests the **implementation** of a given API -- not the definition. It should use the definition document as a ruleset and compare the responses of the generated requests to it.

# What It Does

## Execution
Every run, the generator will perform the following actions:

* Load the OAS for the provided API/workspace
* Validate the definition is in the proper format
    * All parameters, schemas, and responses have `description` and `example` provided for every field
    * `Servers` are defined
    * User-defined governance settings configured in the *enviroment*
* Build schema test array
* Loop through every test in the array and submit the request to the API
* Validate status code and response body schema against OAS

## Schema Test Definition
For every path defined in the Open API Spec, a `json` object will be created in the following format to define the schema test:

```
{
  "path": "", // Combines url from server and path
  "parameters": [ ], // All parameters defined at the path
  "method": "", // Path method
  "allowedRole": "", // If using role based apps, grabs the first allowable role
  "responses": [ // All expected responses for the path
    {
      "statusCode": 200, // Status code defined in responses array
      "$ref": "#/components/responses/Ok" // Grabs either referenced response or inline defined schema
    }
  ],
  "success": true, // Boolean for if this test is expected to be a success
  "description": "Has all required fields", // Generated description for what is being tested
  "body": { }, // Generated body in JSON
  "name": "" // Generated request name with method, path, description, and success
}
```

## Test Generation
The generator will build request bodies for each endpoint using the **example** provided for every property. If there is no example provided for an individual property, it will not be included in the request. Examples will be used for all parameters and schema properties.

For accurate tests, it is advised *to use real values from your test environment in your examples* in order to get valid responses back during test execution. This means using known, hardcoded or seeded values in the system to define your API in the OAS.

The request body will be generated with the required fields only. The generator will take the request body with all required fields and create an array of mutations against it. For each required field in the request body, two mutations will be created:

* Omit the required field
* Leave the required field blank

Once all the mutations have been created, the generator proceeds to execute the tests.

### Example
Let's take an example API that maintains cars. The car schema as defined in the OAS would be:

```yaml
components:
  schemas:
    Car:
      type: object
      required:
        - make
        - model
        - year
      properties:
        make:
          type: string
          example: Nissan
        model:
          type: string
          example: Pathfinder
        year:
          type: number
          example: 2015
        color:
          type: string
          example: red
```

The generator will create the following request body for the schema:

```json
{
  "make": "Nissan",
  "model": "Pathfinder",
  "year": 2015
}
```

It will also make mutations for every property:
```json
{
  "model": "Pathfinder",
  "year": 2015
}
```
and

```json
{
  "make": "",
  "model": "Pathfinder",
  "year": 2015
}
```

The generator is building these tests to verify the implementation of the API matches what is defined in the OAS. It will check to see if a valid http status code (400) is given if a missing required field is submitted.


# What It Does NOT Do
The generator will **not** build a collection to be run at a later date. It builds and executes the tests at runtime, allowing for the test to dynamically update as changes are made to the Open API Spec.

# Requirements
The Open API Spec tested by the contract test generator is required to be in a specific format. Below are the minimum requirements for the tests to execute:

* The `Servers` object is defined and has at least one server with a description
* Every schema property must have an example defined
* The following fields in the provided environment are configured
    * `env-apiKey` - Integration API Key for Postman
    * `env-workspaceId` - Identifier for the workspace that contains the API to be tested
    * `env-requireParamExample` - Must be set to true (the collection will still run, but this will show you where any problems lie)
    * `env-server` - Matches the description of the server to be tested in the `Servers` object
* Request bodies are defined in the `#/components/schemas` section of the OAS and are referred to by using `$ref`

# Authentication
Authentication is the only piece of the generator that needs to be handled by the consumer. It is set up assuming all requests that execute against your API use the same authentication method.

## Configuration
If your API uses authentication, you will be required to configure it on the collection itself.

1. Right click the collection in your Postman workspace and select **Edit**.
2. Click on the **Authorization** tab and configure the type of Authentication your API requires.
3. If using OAuth2.0, you may [reference my blog post](https://www.readysetcloud.io/blog/allen.helton/how-to-automate-oauth2-token-renewal-in-postman-864420d381a0/) on how to automate the token renewal.
4. Click **Update** to save your changes

**NOTE-** If your API uses a standard api key header like `x-api-key` this step is unnecessary. You may just add it in the `#/components/parameters` section and it will be included automatically in each request.
```yaml
components:
  parameters:
    ApiKey:
      name: x-api-key
      in: header
      example: 982345jsdw0971ls09812354
      schema:
        type: string
```

## Setup
In this repo there are two files you need to import into your [Postman](https://postman.com/) workspace:
* Contract Test Generator *collection*
* Contract Test Generator *Environment*

If you are unsure how to import these into Postman, please [refer to this guide](https://kb.datamotion.com/?ht_kb=postman-instructions-for-exporting-and-importing).

# Extensions
If you wish to use extensions with your Open API Spec to assist with the test generation, see details below for supported extensions:

## x-amazon-apigateway-integration
If you use AWS API Gateway and practice [API-first development](https://www.readysetcloud.io/blog/allen.helton/api-first-development-with-postman/), chances are you use the [x-amazon-apigateway-integration extension](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration.html). This extension allows you to identify how each endpoint proxies to an AWS service. For API-first development, you are able to to mock out the proxy and response for endpoints that are not implemented yet. 

When using this extension, you can add the following snippet to tell API Gateway to mock out the response.
```yaml
x-amazon-apigateway-integration:
  responses:
    200:
      statusCode: 200
  passthroughBehavior: when_no_match
  requestTemplates:
    application/json: |
      {
        'statusCode': 200
      }
  type: mock
```

The generator will skip over any endpoint methods that have `type: mock` defined in this extension. No tests will be run for mocked endpoints.

## x-postman-variables
Instead of relying completely on seeded, or known, data in the system for the generated tests, this extension allows you to use objects created in the generated tests in subsequent tests. For example, if you have an endpoint that creates a `book` object by doing a **POST** to `/books`, this extension will allow you to save the returned id as a collection variable and use it in other API calls, like doing a **GET** on `/books/{bookId}`.

**Saving a variable**

In the `responses` section of an endpoint method you can add the following snippet to save a collection variable.
```yaml
responses:
  201:
    $ref: `#/components/responses/Created`
    x-postman-variables:
      - type: save
        name: bookId
        path: .id
```

`type` - Must be *save* in order to save the value to a collection variable
`name` - What to name the collection variable
`path` - Json-path to the property you want. Currently does not support arrays. It must start with a '.'

The extension is an array, so you can add as many saves as you'd like. 

**Consuming a variable**

The extension must be added to a parameter for consumption. This could be a header, query param, or path param. It works with both inline and ref parameters.
```yaml
parameters:
  bookId:
      name: bookId
      in: path
      description: Unique identifier for the book
      required: true
      schema:
        type: string
        example: 23SovYJfRZ5Wt7jpZEPHVo
      x-postman-variables:
        - type: load
          name: bookId
```

Whenever this parameter is used, the test generator will load the value from the collection variable. If the collection variable does not exist or has no value, it will fall back to the provided example.

# Environment
Below is a list of environment variables currently consumed by the generator:

* `env-apiKey` - Your Postman API key used to access the Postman API
* `env-minApiCount` - The minimum amount of APIs required in a provided workspace
* `env-maxApiCount` - The maximum amount of APIs required in a provided workspace
* `env-workspaceId` - The identifier of the workspace you wish to generate tests for
* `env-requireParamDescription` - Flag denoting if assertions should be run that require a parameter description - **BOOLEAN**
* `env-requireParamExample` - Flag denoting if assertions should be run that require a parameter example - **BOOLEAN**
* `env-paramDescriptionMinLength` - The minimum length a description should be. This is only used if `env-requireParamDescription` is true
* `env-paramDesciptionMaxLength` - The maximum length a description should be. This is only used if `env-requireParamDescription` is true
* `env-securityExtensionName` - If using a role-based API, the name of the extension you use to denote allowed roles on an endpoint
* `env-roleHeaderName` - If using a role-based API, the name of the header where you supply the user's assumed role
* `env-server` - The description of which `server` element to use. This is for the base url of your API. [See OAS Documentation](https://swagger.io/docs/specification/api-host-and-base-path/)
* `env-runComponentTests` - Flag denoting if assertions should be run validating schema adherence - **BOOLEAN**
* `env-runContractTests` - Flag denoting if contract tests should be generated and run - **BOOLEAN**
* `env-schemaPropertyExceptions` - The names of any properties that should not be validated in the component tests - **ARRAY OF STRINGS**
* `env-jsonToYaml` - NPM Package that converts yaml to json. DO NOT EDIT!

# Contact
You may contact me by any of the social media channels below:

[![Twitter][1.1]][1] [![GitHub][2.1]][2] [![LinkedIn][3.1]][3] [![Ready, Set, Cloud!][4.1]][4]

[1.1]: http://i.imgur.com/tXSoThF.png
[2.1]: http://i.imgur.com/0o48UoR.png
[3.1]: http://i.imgur.com/lGwB1Hk.png
[4.1]: https://readysetcloud.s3.amazonaws.com/logo.png

[1]: http://www.twitter.com/allenheltondev
[2]: http://www.github.com/allenheltondev
[3]: https://www.linkedin.com/in/allen-helton-85aa9650/
[4]: https://readysetcloud.io
