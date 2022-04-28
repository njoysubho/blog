---
title: "Spring Rest Openapi"
date: 2022-04-24T08:23:28+02:00
draft: false
---

Till now we have seen how to generate a new spring boot application and then how to containerize it. However our application till does not have any functionality. Today we will see how to create a REST API with Spring boot. However we will take a schema first approach and we will generate the REST API stub using the schema. This article will show how a OpenAPI specification looks like and how can we generate our REST API stub using the schema.

## OpenAPI Specification

APIs are contract between the application and consumers of the application. These consumers can be machines or humans. OpenAPI is a specification to write your API contract in human and machine readable format. It standardizes the way we can describe our API. The entire specififaction can be found here `https://spec.openapis.org/oas/v3.1.0` . 

## Our First API Spec

Let's create a new service, I call it `inventory-service`. We now know how to generate a new spring boot application. We add a yml openAPI spec file in `src/resources/spec/inventory-api.yml` . A minimal API can look as follows -  

```yaml
openapi: "3.0.3"
info:
  title: inventory-api
  version: 1.0.0
paths:
  /products:
    get:
      description: Get All Products
      operationId: getAllProducts
      responses:
        '200':
          description: All products are returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ListOfProducts'
        '404':
          description: No Product returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
    post:
      description: Add A Product to inventory
      operationId: addProduct
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Product'
      responses:
        '201':
          description: Product added successfully
          headers:
            location:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /products/{id}:
    get:
      description: Get A Product By ID
      operationId: getProductById
      parameters:
        - in: path
          name: id
          schema:
            type: string
            format: uuid
          required: true
      responses:
        '200':
          description: Get a product by id
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
        '404':
          description: No Product returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
    put:
      description: Update A Product
      operationId: updateProduct
      parameters:
        - in: path
          name: id
          schema:
            type: string
            format: uuid
          required: true
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Product'
      responses:
        '200':
          description: Created product is  returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '404':
          description: Product Does not Exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
components:
  schemas:
    ListOfProducts:
      type: array
      items:
        $ref: '#/components/schemas/Product'

    Product:
      type: object
      properties:
        id:
          type: string
          format: UUID
    Error:
      type: object
      properties:
        message:
          type: string
```
It is a very minimal API . We can see in `paths` section we have description of our API. Each API endpoint has it's optional request body and response body. We can also define if some custom headers are required, path parameters , query parameters and so on. 
In the `components` section we define our models and these are referenced in our API. 

I will not go deeper into the OpenAPI spec but because it is very vast but we can always consult the spec for our specific use case . 

## Generating REST API for Spring

Now that we have our OpenAPI spec, there are plugins and tool available to generate code from our spec. We can use the `openapi-generator` https://openapi-generator.tech/docs/installation to generate our REST API. We can use cli to generate our REST API. 
 However there is also a maven plugin https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin which we will use to generate our source .

The maven plugin uses the openapi-generator to generate the source code.

 To use the maven-plugin we will add it to the build section like following -

 ```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>5.4.0</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/spec/inventory-api.yml</inputSpec>
                <generatorName>spring</generatorName>
                <generateSupportingFiles>false</generateSupportingFiles>
                <configOptions>
                    <basePackage>com.sab.inventory</basePackage>
                    <sourceFolder>src/java/main</sourceFolder>
                    <interfaceOnly>true</interfaceOnly>
                    <modelPackage>com.sab.inventory.dto</modelPackage>
                    <apiPackage>com.sab.inventory.api</apiPackage>
                    <skipDefaultInterface>true</skipDefaultInterface>
                    <openApiNullable>false</openApiNullable>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
 
 ``` 


 Both the plugin and the actual openapi-generator has lot of config options we can check it from https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin and https://openapi-generator.tech/docs/generators/spring. 

 In the above example I have used the bare minimum config . I will explain them below . 

    * `inputSpec` - This is the path to the OpenAPI spec file.
    * `generatorName` - ooenapi-generator can produce source code multiple language and framework. Because we want to generate for Spring I chose spring as the generator name.
    * `generateSupportingFiles` - When I generated first I saw some unecessary supporting files were generated as I do not need those I turned it to false.
    * `configOptions` - Properties that maps directly to the generator options.
       ** `basePackage` - package name for your generated sources
       ** `sourceFolder` - folder where your generated sources will be placed
       **  `interfaceOnly` - We can either generate only REST API interface or we can generate some default code. I make it true as I want to generate only REST API interface.
       ** `modelPackage` - package name for your generated model/dto classes.
       ** `apiPackage` - package name for your generated API classes.
       **   `skipDefaultInterface` - We can skip the genration of default methods in the interface.
       ** `openApiNullable` - With the value true it will generate an import of `org.openapitools.jackson.nullable.JsonNullable` however I didn't need it so I make it false.

So what is the output of the above code ? Below is how our REST API stub looks like 

```java
/**
 * NOTE: This class is auto generated by OpenAPI Generator (https://openapi-generator.tech) (5.4.0).
 * https://openapi-generator.tech
 * Do not edit the class manually.
 */
package com.sab.inventory.api;

import com.sab.inventory.dto.Error;
import com.sab.inventory.dto.Product;
import java.util.UUID;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.Parameters;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.multipart.MultipartFile;

import javax.validation.Valid;
import javax.validation.constraints.*;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import javax.annotation.Generated;

@Generated(value = "org.openapitools.codegen.languages.SpringCodegen", date = "2022-04-23T22:00:49.464591+02:00[Europe/Berlin]")
@Validated
@Tag(name = "products", description = "the products API")
public interface ProductsApi {

    /**
     * POST /products
     * Add A Product to inventory
     *
     * @param product  (optional)
     * @return All products are returned (status code 201)
     *         or No Product returned (status code 400)
     */
    @Operation(
        operationId = "addProduct",
        responses = {
            @ApiResponse(responseCode = "201", description = "All products are returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Product.class))),
            @ApiResponse(responseCode = "400", description = "No Product returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Error.class)))
        }
    )
    @RequestMapping(
        method = RequestMethod.POST,
        value = "/products",
        produces = { "application/json" },
        consumes = { "application/json" }
    )
    ResponseEntity<Product> addProduct(
        @Parameter(name = "Product", description = "", schema = @Schema(description = "")) @Valid @RequestBody(required = false) Product product
    );


    /**
     * GET /products
     * Get All Products
     *
     * @return All products are returned (status code 200)
     *         or No Product returned (status code 404)
     */
    @Operation(
        operationId = "getAllProducts",
        responses = {
            @ApiResponse(responseCode = "200", description = "All products are returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Product.class))),
            @ApiResponse(responseCode = "404", description = "No Product returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Error.class)))
        }
    )
    @RequestMapping(
        method = RequestMethod.GET,
        value = "/products",
        produces = { "application/json" }
    )
    ResponseEntity<List<Product>> getAllProducts(
        
    );


    /**
     * GET /products/{id}
     * Get A Product By ID
     *
     * @param id  (required)
     * @return All products are returned (status code 200)
     *         or No Product returned (status code 404)
     */
    @Operation(
        operationId = "getProductById",
        responses = {
            @ApiResponse(responseCode = "200", description = "All products are returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Product.class))),
            @ApiResponse(responseCode = "404", description = "No Product returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Error.class)))
        }
    )
    @RequestMapping(
        method = RequestMethod.GET,
        value = "/products/{id}",
        produces = { "application/json" }
    )
    ResponseEntity<Product> getProductById(
        @Parameter(name = "id", description = "", required = true, schema = @Schema(description = "")) @PathVariable("id") UUID id
    );


    /**
     * PUT /products/{id}
     * Update A Product
     *
     * @param id  (required)
     * @param product  (optional)
     * @return Created product is  returned (status code 200)
     *         or Error (status code 400)
     *         or Product Does not Exist (status code 404)
     */
    @Operation(
        operationId = "updateProduct",
        responses = {
            @ApiResponse(responseCode = "200", description = "Created product is  returned", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Product.class))),
            @ApiResponse(responseCode = "400", description = "Error", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Error.class))),
            @ApiResponse(responseCode = "404", description = "Product Does not Exist", content = @Content(mediaType = "application/json", schema = @Schema(implementation =  Error.class)))
        }
    )
    @RequestMapping(
        method = RequestMethod.PUT,
        value = "/products/{id}",
        produces = { "application/json" },
        consumes = { "application/json" }
    )
    ResponseEntity<Product> updateProduct(
        @Parameter(name = "id", description = "", required = true, schema = @Schema(description = "")) @PathVariable("id") UUID id,
        @Parameter(name = "Product", description = "", schema = @Schema(description = "")) @Valid @RequestBody(required = false) Product product
    );

}
```

Now we have our API interface , we can now create our controller and implement the methods. 

So that's it for this post, I will actually implement the API using tests so watch out for the next post. 