# Work in progress


# Out of process fuzzing

This is a preliminary document on how to use the out of process fuzzer without using the local installation with VS Code extension

## Project setup

An already configured project can be used to set up out of process fuzz targets. For new projects a quick setup is
possible by executing `cictl create project <project_dir> --web-app-template`. This will create a project without
a docker image and without a build script, neither of which are needed for out of process fuzzing. The `cictl` output
should give some hints on how to configure the java agent.

Note: any configured build script and docker image will not be used for out of process fuzzing.

## Configure fuzzing agent

The `fuzzing_agent.jar` needs to be loaded with every java application. For simple applications that
can be run with `java -jar <application.jar>` it is possible to add the java agent directly in the java command:

    java -javaagent=/path/to/fuzzing_agent.jar=service_name=<project_name>/web_services/<service_name> -jar <application.jar>
    
Additional arguments for configuring the fuzzing agent:

- `instrumentation_includes` - A ":" separated list of package glob patterns to include in the instrumentation
- `instrumentation_excludes` - A ":" separarted list of package glob patterns to exclude from instrumentation
- `fuzzing_server_host` - The IP or hostname of the fuzzing server (default 127.0.0.1)
- `fuzzing_server_port` - The port on which the fuzzing gRPC server is listening (default 6773)
- ...

Example:

    -javaagent=/path/to/fuzzing_agent.jar=service_name=projects/xyz/web_services/abc,instrumentation_includes=com.my_comp.**:org.other_org.**,fuzzing_server_host=123.43.1.4

## List running agents

After starting the application with the fuzzing agent attached it should print something like "INFO: Got status 'OK' 
from fuzzing server" on stdout. The agent will continuously ping the fuzzing server and wait for a fuzzer to start.
All agents that have registered with the fuzzing server can be listed by running

    cictl list webservices
    
Example output:

    NAME                                                       JAVA COMMAND                                                           LAST CONTACT
    projects/21-points-e9664ce4/web_services/21-points         build/libs/21-points-5.0.3.jar                                         -
    projects/AltoroJ-c59c7caa/web_services/AltoroJ             org.apache.catalina.startup.Bootstrap start                            -
    projects/AltoroJ_project-d1ef63f9/web_services/AltoroJ     org.apache.catalina.startup.Bootstrap start                            4s ago
    projects/securityrat-oop-f6109f24/web_services/securityrat securityrat.jar fuzzing_agent_deploy.jar --spring.profiles.active=prod -

The "Last Contact" column can be used to verify that an agent is currently running.

## Create a fuzz target

Creating a fuzz target is a matter of telling the fuzzing server which web services should be fuzzed. 

    cictl create webapptarget --web_services=<web_service_name> --web_services=<another_web_service_name> ...
    
Glob patterns work instead of full web service names. Create a fuzz target with all web services for a project:   

    cictl create webapptarget <name> --web_services=<project_name>/web_services/*
    
Additional options:
 - `--openapi` - path to an OpenAPI spec (yaml or json) which will be used to generate seeds and guide the mutations
 - `--base_url` - URL at which the application under test is reachable (defaults to `127.0.0.1:8080`)

Not providing an OpenAPI spec triggers the default Spring endpoint analysis on the running web application. However, it
will only correctly work if the required Spring dependencies are part of the web application dependencies.

## Working with seeds

For the out of process fuzzer to reach any meaningful code coverage good seeds should be provided.
When creating a fuzz target seeds are automatically generated using the OpenAPI spec or Spring endpoint
analysis. In some cases these automatically generated seeds will require some user modification.

The seeds can be found in the directory  `<project_dir>/.code-intelligence/<fuzz_target_name>_seed_corpus` and the format
is plain `.http` files. Users can add valid http requests as new files in this directory.

Note: The http request parsing is fairly strict, for example the `Content-Length` header has to be set correctly and an
empty line has to be entered before the request body.

## Configuring Login Pages

There are three methods to configure authorization for the out of process fuzzer:

1. Adding login requests to `<project_dir>/.code-intelligence/fuzz_targets/<target_name>_initial_requests.http`. The initial
requests will be sent before the fuzzer starts and any cookies set during the initial requests will be used for fuzzing.
Example:

```
POST /WebGoat/login HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded
Content-Length: 29
Origin: http://localhost:8080
Connection: keep-alive
Referer: http://localhost:8080/WebGoat/login

username=admin&password=admin
```
    
2. Setting constant headers in `<project_dir>/.code-intelligence/fuzz_targets/<target_name>_headers.http`. The headers 
defined in this file will be set for every request the fuzzer sends. Example:

    Authorization: Basic YWRtaW46YWRtaW4=
    
3. A shell script at `<project_dir>/.code-intelligence/fuzz_targets/<target_name>_headers.sh` that generates constant
headers. Example:

```
token=$(curl -s -X POST "http://localhost:8080/api/login" -H  "Content-Type: application/json" -d "{  \"username\": \"admin\",  \"password\": \"admin\"}" | jq .Authorization | tr -d '"')
echo "Authorization: $token"
```


## Starting the fuzzer

Same as any other fuzz target.

## Encrypted communication

Add the option `tls=true` to the fuzzing agent when the ci-daemon has TLS enabled. This option only applies to the
communication between the fuzzing agent and ci-daemon. The gRPC communication between fuzzing agent and fuzzer is
unencrypted!

A default SSL configuration is used for the communication which first looks for an OpenSSL provider and if not found
falls back on the JDK SSL provider (see [GrpcSslContexts.java#L229](https://github.com/grpc/grpc-java/blob/814655cdde5797854289ce5c2ec3e7b80ce0cf44/netty/src/main/java/io/grpc/netty/GrpcSslContexts.java#L229)).

## API Token

Running the fuzzing agent inside a CI/CD environment requires providing the fuzzing agent with an API token to
authenticate at the ci-daemon. A valid personal or organization token is supported and can be passed with the option
`api_token=<token>`.

## Troubleshooting

- If the web application is not running on the same host as the the fuzzing server has to be started with e.g.
 `--listen_address=0.0.0.0:6773` since it by default only listens on localhost.
- Check that the fuzzing server can be reached from the web application. This could require editing the firewall settings,
docker networks, cluster settings ...

## Tips and Tricks

- The "REST Client" VS Code extension is great for sending .http files as requests from the IDE.
https://marketplace.visualstudio.com/items?itemName=humao.rest-client
- The `instrumentation_includes` option to the fuzzing agent is an important setting that can be tweaked depending on
the project. If this option is not set the agent will default to instrumenting all packages. This might cause errors
for certain large projects that include many dependencies. If that is the case a simple heuristic is to instrument the
user code + some dependencies that are especially interesting (the user / customer should know).

## Future Work

- Concurrent requests
- Full exception policy support
- More bug detectors
- Better mutations
