# Web Fuzzing

This is a document on how to use the Web fuzzer without using the CI Fuzz local installation, using only the CI Fuzz server

## Prerequisites

This repository contains a template and documentation to set up fuzzing for your own Web App/Web API. You need to have your own CI Fuzz server installed. In the near future, an account in CI Fuzz SaaS platform will also be supported.

This template will work with few modifications for Java HTTP APIs (this use case is also described in this readme).

With some modifications, it can also be used to fuzz Java GRPC APIs and Go APIs (both HTTP and GRPC).

## Repository setup

Copy the .code-intelligence directory to the root of the Git repository where the source code of your application/API resides. Modify the following files:

### .code-intelligence/web_services.yaml

If your application consists of several microservices, add one web service entry for each microservice and name them appropriately. Otherwise, the default name can be used.

If your application does not have web controllers written in the Springboot framework, or if you already configured fuzzing but you have problems with the endpoint analysis step when starting fuzzing, add a file with OpenAPI specifications. The path is relative to the root directory of this git repository.

### .code-intelligence/fuzz_targets/fuzzallendpoints.yaml

Modify base_url to the URL where your application under test will be accessible, if it is predictable and will be always the same. Otherwise, leave the default.

Make sure that network traffic from the CI Fuzz server to the application is allowed on the port where the application is running.

If you changed or added web service name(s) in the previous file, change/add them here to match.

### .code-intelligence/fuzz-targets

Use one or more of the files here to add the steps needed to authenticate to your application or set it's initial state. More info [here](https://help.code-intelligence.com/configure-http-headers)

### .code-intelligence/fuzzallendpoints_seed_corpus

For the initial fuzzing setup, you can keep this empty, as it is.

This is further explained later on in this Readme.

## Configure project in CI Fuzz Web Interface

Push the changes to your online repository.

Go to the CI Fuzz web interface and create a project from this repository. Instructions can be found [here](https://help.code-intelligence.com/using-the-web-app-new)

## Write a CI/CD pipeline (or a regular shell script, to try it out)

CICD script is documented [here](https://help.code-intelligence.com/continuous-fuzzing-setup-new)


Make sure that the name of the web service in every app/service started with a java agent match the entries in web_services.yaml and fuzzallendpoints.yaml. If you did not change those, use the default name: mywebservice. If you give the java agent aa different service_name, the fuzz test will not work correctly!

Setup is now complete and you can run your pipeline.

When running, your application's output should say that it is instrumenting Java classes and that it is connecting to the CI Fuzz server:

INFO: Got status 'OK' from fuzzing server

In your CI Fuzz web interface, in the Web Services view, your web service should be green when it is running, but fuzzing has not started yet. It is OK if it shows red during fuzzing.

## Working with seeds

By default, CI Fuzz will generate HTTP requests based on the results of automatic endpoint analysis of your application. After you ran fuzzing, if you have low code coverage, you can improve it by putting valid raw HTTP requests in this directory, in text files ending with .http extension. 

You can also add multiple requests in one file, if you want to fuzz more complex scenarios in your application (for example create something and then delete it). In order to do this, separate the requests with a newline and set correct Content-Length header for all requests that have a body, except the last request, where it is not needed.


For the out of process fuzzer to reach any meaningful code coverage good seeds should be provided.
When running a fuzz target seeds are automatically generated using the OpenAPI spec or Spring endpoint
analysis. In some cases these automatically generated seeds will require some user modification.

When you use this template, then the automatic seeds can only be found on the CI Fuzz server's file system, in the CI Fuzz data directory. To find the CI Fuzz data directory:
```
grep DATA_DIR /etc/cifuzz/*env /opt/ci-fuzz-*/cifuzz.env
```
Inside this directory, look for your project, an in there you should see a `corpus` directory with some automatic seeds inside. 

The format is plain `.http` files. However, users should add valid http requests as new files in the relevant directory in the fuzzing project's git repository, not to this corpus directory.

Note: The http request parsing is fairly strict, for example the `Content-Length` header has to be set correctly for all requests except the last and an
empty line has to be entered before the request body.

A default SSL configuration is used for the communication which first looks for an OpenSSL provider and if not found
falls back on the JDK SSL provider (see [GrpcSslContexts.java#L229](https://github.com/grpc/grpc-java/blob/814655cdde5797854289ce5c2ec3e7b80ce0cf44/netty/src/main/java/io/grpc/netty/GrpcSslContexts.java#L229)).

## Troubleshooting

- Check that the fuzzing server can be reached from the web application. This could require editing the firewall settings,
docker networks, cluster settings ...

## Tips and Tricks

- The "REST Client" VS Code extension is great for sending .http files as requests from the IDE.
https://marketplace.visualstudio.com/items?itemName=humao.rest-client

