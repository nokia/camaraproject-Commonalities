# API Testing Guidelines

## Table of Contents
- [Introduction](#introduction)
- [Test plan coverage](#test-plan-coverage)
  - [Independent operations](#independent-operations)
  - [Operations modifying resources](#operations-modifying-resources)
  - [Notifications](#notifications)
  - [Error scenarios](#error-scenarios)
- [Test plan design guidelines](#test-plan-design-guidelines)
  - [Location of feature file](#location-of-feature-files)
  - [Feature structure](#feature-structure)
  - [Environment variables](#environment-variables)
  - [Background section](#background-section)
  - [Scenario structure](#scenario-structure)
  - [Request setup](#request-setup)
  - [Request sending](#request-sending)
  - [Response validation](#response-validation)
- [References](#references)

## Introduction
This document captures guidelines for the API testing in CAMARA project. These guidelines are applicable to every API to be worked out under the CAMARA initiative.

## Test plan coverage

Based on the decision taken in the [Release Management](https://github.com/camaraproject/ReleaseManagement) and Commonalities working groups, and as documented in [API Release Process](https://wiki.camaraproject.org/display/CAM/API+Release+Process)(https://wiki.camaraproject.org/display/CAM/API+Release+Process#APIReleaseProcess-APIreadinesschecklist), at least one Gherkin feature file per API will be added to the API repo folder for test plans, and this test plan must fulfill the minimum criteria of readiness with respect to API test cases and documentation:

* Release candidates must have at least a basic API test plan, covering sunny day scenarios. Scenarios are expected to validate that the implemented interface is compliant with the specification, in terms of syntax and support for the mandatory aspects. 
* Any public release must include a full test plan, covering both sunny and rainy day scenarios, to be considered stable. This may include unusual or not explicitly specified error scenarios, semantic validations, and corner cases.

If subprojects also intend to add test implementations, an aligned single implementation agreed among all provider implementors could be added to the main subproject repo. If no alignment is possible, each provider implementer will add the test implementation to their own repos.

* Test plans are not meant to certify the accuracy or veracity of the implementation's responses but to ensure that the interface complies with the API specification and that the API is fully implemented.

* When the API allows options or alternative approaches, the test plan should cover those situations and define the expected behaviour for inputs or options not supported.
  - Implementations will need to inform in advance to the API tester about which subset of options is supported. 

### Independent operations

Operations that execute an action or retrieve information from the operator without modifying a resource will usually be tested in its own dedicated feature file. Splitting a test plan into several features is a test plan design decision, taking into account the complexity of the test plan.

Test plan must validate that providing a valid input, a success response is received and that the response body complies with the JSON schema in the API spec.

For operations receiving an input request body with optional properties, several scenarios should be included, testing different sets of valid inputs.


### Operations modifying resources

For path resources with CRUD (Create/Read/Update/Delete) operations, some scenario should validate that the new state of the resource is coherent with the result of the operation:

* For Create operations:
  - If operation success returns the created object, validate that the properties of the response object are coherent with the request object
  - Validate that the GET operation can read the created object for the item id.

* For Update operations:
  - If operation success returns the updated object, validate that the properties of the response object are coherent with the request object
  - Validate that the GET operation can read the updated object for the item id, with the modified properties.

* For Delete operations:
  - Validate that trying to read the successfully deleted object, with the GET operation for the deleted item id, returns the error specified in the API spec, typically 404.

### Notifications

For operation with implicit subscriptions:

* Check that when a webhook (i.e. callbackURL) is provided, the expected events are received in the `sink`, with the right `sinkCredential`, for those situations specified in the API.
* If the API allows updating a previously provided webhook:
  - If the `sink` is modified, validate that events are received in the modified value.
  - If the `sink` can be nullified, validate that events are not longer received.
  - If the `sinkCredential` is modified, validate that the new token is used for the events.

For the explicit subscriptions model:

* Validate that the subscribed events are received in the `sink`, with the right `sinkCredential`, for those situations specified in the API.
* For subscriptions that provide `subscriptionExpireTime`, validate that the subscribed events are not longer received after the expiration time.
* For subscriptions that provide `subscriptionMaxEvents`, validate that the subscribed events are not longer received after the maximum events limit is reached.
* Validate that after a subscription is deleted, the subscribed events are not longer received.

### Error scenarios

* Test plan must be aligned with API specification regarding the coverage of HTTP statuses in the responses.
  - All errors explicitly documented in the API spec must be covered by one or more dedicated error scenarios.
  - On the contrary, HTTP Statuses which are not explicitly documented in the API specification should not be covered in the test plan. If any of the non-mandatory "status" in API Design Guidelines is relevant in the context of a given API and deserves to have a specific test, it needs to be documented in the API specification as well.
  - In particular operations that expect a `device` object allowing one or more identifiers to be provided, must include scenarios to test the error cases defined, aligned with the device error management section of the API design guidelines. 

* It must be validated that the HTTP status code and response property `code` are exactly those specified in the API spec.

* It must be validated that the property `message` is present, but its value is free for the implementation to decide. Test plan may validate that message is human friendly and coherent with the scenario.

For operations with a request body, several scenarios are expected to test different sets of input data which do not comply with the request body schema, testing that an error, typically `400` `INVALID_ARGUMENT` is received. Typical tests:
- Values with invalid format
- Mandatory properties are missing

## Test plan design guidelines

* Granularity of the feature file must be decided at the project level, but it is recommended to:
    -	group in one file all scenarios testing one closely related API capability (that can cover one or several endpoints).
    -	provide several files when one CAMARA API (yaml) covers several independent functions that can be provided independently

* Third-person pronoun usage in feature file is advisable as using the third person, conveys information in a more official manner.


### Location of feature files

The feature files will reside under 

`{Subproject_Repository}/code/Test_definitions/file-name.feature`

E.g.: 

`https://github.com/camaraproject/DeviceLocation/blob/main/code/Test_definitions/location-verification.feature`


Filename:

- APIs with a single feature file will use the api-name as file name: E.g. `location-verification.feature`
- For APIs with several operations that split the test plan in one feature file per operation, recommendation is to add the `operationId` to the api-name as file name: `quality-on-demand-createSession.feature`.
- APIs with several feature files, not split by operation, should use the api-name along some other description, e.g.: `quality-on-demand-notifications.feature`

### Feature structure

A feature will typically test an independent API operation or several closely related API operations. It will consist of several scenarios testing the server responses to the API operations under different conditions and input content in the requests, covering both success and error scenarios. Each scenario will validate that the response complies with the expected HTTP status, that the response body syntactically complies with the specified JSON schema, and, when applicable, that some properties have the expected semantic values.

For the Feature description, API name and version must be included. When the feature is focused on an operation, it also should be included, e.g.:

```
Feature: CAMARA Device location verification API, v0.2.0 - Operation verifyLocation
```

#### Feature Context

Every feature may include a context after the `Feature` tag to provide relevant information about the implementation and execution of the tests.

This feature context is RECOMMENDED unless it does not provide additional value (e.g. all the info could have been given in the Background section for a certain API)

For the feature context, the following template should be used:

```
  # Input to be provided by the implementation to the tester
  #
  # Implementation indications:
  # * 
  #
  # Testing assets:
  # * 
  #
  # References to OAS spec schemas refer to schemas specified in {apiname}.yaml
```

An example for this feature context is depicted below:

```
    # Input to be provided by the implementation to the tester
    #
    # Implementation indications:
    # * apiRoot: API root of the server URL
    # * List of device identifier types which are not supported, among: phoneNumber, ipv4Address, ipv6Address.
    #   For this version, CAMARA does not allow the use of networkAccessIdentifier, so it is considered by default as not supported.
    # * List of application server IP formats which are not supported, among ipv4 and ipv6.
    #
    # Testing assets:
    # * A device object applicable for Quality On Demand service.
    # * A device object identifying a device commercialized by the implementation for which the service is not applicable, if any.
    #
    # References to OAS spec schemas refer to schemas specified in quality-on-demand.yaml
```

### Environment variables

Commonly, some values to fill the request bodies will not be known in advance and cannot be specified as part of the feature file, as they will be specific for the test environment, and they will have to be provided by the implementation to the tester as a separate set of environment or configuration variables.

How those variables are set and feed into the testing execution will depend on the testing tool (Postman environment, context in Behave, etc.). 

A first step is to identify potential environment variables writing the test plan, e.g.: 


| variable       | description                                                                                                                       |
|----------------|-----------------------------------------------------------------------------------------------------------------------------------|
| apiRoot        | API root of the server URL                                                                                                        |
| locatedDevice  | A device object which location is known by the network when connected. To test all scenarios, at least 2 valid devices are needed |
| knownLatitude  | The latitude where locatedDevice is known to be located                                                                           |
| knownLongitude | The longitude where locatedDevice is known to be located                                                                          |

This list of input data to be requested may be kept in a separated file. The Structure of this file is yet TBD.

### Background section

At a beginning of the feature file, a `Background` section is expected, including common configuration steps and setting up variables and values which are applicable to every scenario. This way, scenarios do not need to replicate those steps.

Background section example:

```
Background:
    Given an environment at "apiRoot" 
    And the resource "{path_resource}"                                                              |
    And the header "Content-Type" is set to "application/json"
    And the header "Authorization" is set to a valid access token
    And the header "x-correlator" complies with the schema at "#/components/schemas/XCorrelator"
```

### Scenario structure

* The structure of the steps for API testing will typically be:
  - A `Given` block setting up the request
  - A single `When` step sending the request
  - A `Then` block validating the response

* It is recommended to only have one When/Then block per scenario. However, in case of complex scenarios, several When/Then blocks can be concatenated in the same scenario.

* Each scenario will have a tag with the scenario unique identifier. The recommended format for scenario identifier is:

```
@<(mandatory)feature_identifier>_<(mandatory)number>_<(optional)short detail in lower case and using underscore “_” as the separator>
```
E.g.: `@location_verification_10_device_empty`

* `Scenario Outline` may be used at convenience with an `Examples` section. 


### Request setup

`Given` steps generally put the system in a well-defined state. For API testing, this typically implies setting up the request according to the scenario preconditions, filling the necessary path, query, header and/or body parameters.   

For coherence and to ease reusability of testing implementations, a common syntax should be followed by all testing plans:

```
* the path parameter "{name}" is set as "{value}"
* the path parameters:
    | param_name | param_value |
    | ---------- | ----------- |
    | xxx        | yyy         |
* the header "{name}" is set as "{value}"
* the headers:
    | param_name | param_value |
    | ---------- | ----------- |
    | xxx        | yyy         |
* the query parameter "{name}" is set as "{value}"
* the query parameters:
    | param_name | param_value |
    | ---------- | ----------- |
    | xxx        | yyy         |
* the request body property "{json_path}" is set as "{value}"
* the request body:
    ```
    {
      "xxx": "yyy"
    }
    ```
```

To refer to properties in request bodies and response, [JSON path](https://www.ietf.org/archive/id/draft-goessner-dispatch-jsonpath-00.html) notation is recommended.

Alternatively, instead of specifying exact values for specific parameters or properties, testing plans can describe the value with a human-friendly sentence, to let the API tester fill the request with some value that complies with the requirement, e.g.:

```
* the path parameter "id" is that of an existing item
* the query parameter "startDate" is in the past
* the request body property "$.device" is a valid device which location is known
```

Testing plans should identify the list of input values that have to be provided by the implementation, and maintain this list as an asset of the test plan. 


### Request sending

For scenarios testing individual operations, the request will be sent by means of a single `When` step, indicating the `operationId` specified in the spec, as the `operationId` uniquely identifies the HTTP method and resource path. E.g.:

```
When the request "{operationId}" is sent
```

For scenarios involving several requests, further requests may be included as additional `When` steps after the response to the first one is validated. E.g.:

```
Scenario: Delete an item and check that is deleted
  Given an existing item 
  When the request "deleteItem" is sent
  Then the response status is 204
  When the request "getItem" is sent
  Then the response status is 404
```


But the general recommendation is to avoid creating complex scenarios and split logic in simpler scenarios as much as possible.

### Response validation

`Then` steps will validate that response is as expected by the scenario. Typically, these steps will be written in the form of assertions, checking that some condition is complied.

Typical response validations:

- that HTTP status code is the integer value specified, e.g.:

```
- the response status code is 200
```

- response header validations, e.g.:

```
- the response header "x-correlator" has same value as the request header "x-correlator" 
- the response header "Content-Type" is "application/json"
```

- response schema validations. Reference to the schema in the OAS spec may be explicit or implicit, e.g.:

```
- the response body complies with the OAS schema for this operation and status code
- the response body complies with the OAS schema at "{oas_schema_json_path}"
```

- additional constraints for optional or dependent properties in the response body, e.g.:

```
- the response property "{json_path}" exists
- the response property "{json_path}" does not exist
- the response property "{json_path}" only exists if property "{json_path}" exists
- the response property "{json_path}" only exists if property "{json_path}" is "{value}"
```

- for error scenarios, the response property `code` must be checked, e.g.:

```
- the response property "$.code" is "INVALID_ARGUMENT"
```

## References

* [Feature files]( https://copyprogramming.com/howto/multiple-feature-inside-single-feature-file#multiple-feature-inside-single-feature-file)
* [Scenario Identifier](https://support.smartbear.com/cucumberstudio/docs/tests/best-practices.html#scenario-content-set-up-writing-standards)
* [One when/then](https://cucumber.io/docs/gherkin/reference/)
