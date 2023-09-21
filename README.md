# grpc-import-test

This project is used to demonstrate that there might be a problem with the built in gRPC compiler in
Quarkus and how it handles *.proto files in subdirectories.

This is a sample setup based on a real world scenario at our company since I couldn't post the real setup.

## Installing the quarkus-proto-test.jar into local maven
In order to run the test you need to install the jar in the root directory of this project into your local maven repo.

This file represents a real world project where we have our proto files in an external artifact.
Those files are stored under a subdir called `protobuf` and unfortunately we cannot change that artifact because it is built by another
group in the company.

```bash
mvn install:install-file -Dfile=quarkus-proto-test.jar -DgroupId=org.acme -DartifactId=quarkus-proto-test -Dversion=1.0 -Dpackaging=jar
```

## Compiling the code
When you attempt the compile the code you'll see errors:

```bash
mvn clean compile
```

Output
```bash
(base) ➜  grpc-import-test git:(main) mvn clean compile                                                                                                                                                                                                                                                  
[INFO] Scanning for projects...                                                                                                                                                                                                                                                                          
[INFO]                                                                                                                                                                                                                                                                                                   
[INFO] ---------------------< org.acme:grpc-import-test >----------------------                                                                                                                                                                                                                          
[INFO] Building grpc-import-test 1.0.0-SNAPSHOT                                                                                                                                                                                                                                                          
[INFO]   from pom.xml                                                                                                                                                                                                                                                                                    
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ grpc-import-test ---
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ grpc-import-test ---
[INFO] Copying 2 resources from src/main/resources to target/classes
[INFO] 
[INFO] --- quarkus:3.4.1:generate-code (default) @ grpc-import-test ---
base.proto: File not found.
extended.proto:11:1: Import "base.proto" was not found or had errors.
extended.proto:15:5: "org.acme.protos.base.User" is not defined.
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.587 s
[INFO] Finished at: 2023-09-21T16:20:23-04:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal io.quarkus.platform:quarkus-maven-plugin:3.4.1:generate-code (default) on project grpc-import-test: Quarkus code generation phase has failed: InvocationTargetException: Failed to generate Java classes from proto files: [/Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto, /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/role.proto, /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/base.proto] to /Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc with command /Users/tmulle/NetBeansProjects/grpc-import-test/target/com.google.protobuf-protoc-osx-aarch_64-exe -I=/Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-dependencies/2d160609fb74c459975eca766b93d1dc5316867f -I=/Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6 -I=/Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto --plugin=protoc-gen-grpc=/Users/tmulle/NetBeansProjects/grpc-import-test/target/io.grpc-protoc-gen-grpc-java-osx-aarch_64-exe --plugin=protoc-gen-q-grpc=/Users/tmulle/NetBeansProjects/grpc-import-test/target/quarkus-grpc3201600090399120494.sh --q-grpc_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc --grpc_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc --java_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc /Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/role.proto /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/base.proto -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
```

### What we need to happen

### Past way of doing things...
Using other grpc maven plugins like `com.github.os72` combined with the `maven-dependency` plugin we can get the protos building correctly without 
any modification to the `import` statments in our protos.

We use the `maven-dependency` plugin to download the artifact with out protos in it and expand them to a directory under the target folder.
We then configure the protoc compiler to include the target folder so we can access those imported protos.
 

```xml
<groupId>com.github.os72</groupId>
<artifactId>protoc-jar-maven-plugin</artifactId>
```

### New way using Quarkus
Since this is a new project and I wanted to use Quarkus/gRPC I decided to use the built in compiler specified in the docs.

I read https://quarkus.io/guides/grpc-getting-started

After setting things up and finally getting the external jar to download automatically it appears that there is no way to configure the protoc
compiler to go into a subfolder of the expanded artifact.

For example, using the following properties in the `application.properties` to pull down the protos from maven, I can see the files in the `target` folder
under a random generated name and under that folder a folder called `protobuf` with our protos.

ie. 
```
grpc-import-test
/target
 /hfkdhjgsfdg748678
  /protobuf
    base.proto
    role.proto
```
```
#quarkus.generate-code.grpc.scan-for-imports=org.acme:quarkus-proto-test
quarkus.generate-code.grpc.scan-for-proto=org.acme:quarkus-proto-test
quarkus.generate-code.grpc.scan-for-proto-includes."org.acme\:quarku-proto-test"=protobuf/**
```
What we need to be able to configure is go into the `protobuf` folder and all the protos under that folder.

What appears to be happening is that the config in the properties file is including the top level folder when in reality we want to ignore it and
just pull the *.protos under it so they are effectively under the same directory.

In the project proto we have:

```proto
syntax = "proto3";

option java_package = "org.acme.protos.extended";
option java_outer_classname = "ExtendedProtos";

option optimize_for = CODE_SIZE;

package org.acme.proto.extended;

// Import the base proto file
import "base.proto";

// A message representing detailed user information
message DetailedUser {
    org.acme.protos.base.User user = 1;             // Use the User message from base.proto
    org.acme.protos.base.Address address = 2;      // Use the Address message from base.proto
    string phone_number = 3;
}

// Another message using the base Address message
message Company {
    string name = 1;
    org.acme.protos.base.Address company_address = 2;
}
```

Using the old way with the other plugin, when we told it to use the `protobuf` folder as an include it worked fine.
But the compiler in quarkus is throwing an error.

If I change the import in the file to `import "protobuf/base.proto"` the compile finds the `base.proto` BUT then any imports inside
the `base.proto` fail.

```bash
[INFO] Scanning for projects...                                                                                                                                                                                                                                                                          
[INFO]                                                                                                                                                                                                                                                                                                   
[INFO] ---------------------< org.acme:grpc-import-test >----------------------                                                                                                                                                                                                                          
[INFO] Building grpc-import-test 1.0.0-SNAPSHOT                                                                                                                                                                                                                                                          
[INFO]   from pom.xml                                                                                                                                                                                                                                                                                    
[INFO] --------------------------------[ jar ]---------------------------------                                                                                                                                                                                                                          
[INFO]                                                                                                                                                                                                                                                                                                   
[INFO] --- clean:3.2.0:clean (default-clean) @ grpc-import-test ---                                                                                                                                                                                                                                      
[INFO] Deleting /Users/tmulle/NetBeansProjects/grpc-import-test/target                                                                                                                                                                                                                                   
[INFO]                                                                                                                                                                                                                                                                                                   
[INFO] --- resources:3.3.1:resources (default-resources) @ grpc-import-test ---                                                                                                                                                                                                                          
[INFO] Copying 2 resources from src/main/resources to target/classes                                                                                                                                                                                                                                     
[INFO]                                                                                                                                                                                                                                                                                                   
[INFO] --- quarkus:3.4.1:generate-code (default) @ grpc-import-test ---                                                                                                                                                                                                                                  
role.proto: File not found.                                                                                                                                                                                                                                                                              
protobuf/base.proto:9:1: Import "role.proto" was not found or had errors.                                                                                                                                                                                                                                
protobuf/base.proto:18:5: "org.acme.protos.role.Role" is not defined.                                                                                                                                                                                                                                    
extended.proto:11:1: Import "protobuf/base.proto" was not found or had errors.                                                                                                                                                                                                                           
extended.proto:15:5: "org.acme.protos.base.User" is not defined.                                                                                                                                                                                                                                         
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.                                                                                                                                                                                                                                      
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.                                                                                                                                                                                                                                      
[INFO] ------------------------------------------------------------------------                                                                                                                                                                                                                          
[INFO] BUILD FAILURE                                                                                                                                                                                                                                                                                     
[INFO] ------------------------------------------------------------------------                                                                                                                                                                                                                          
[INFO] Total time:  1.358 s                                                                                                                                                                                                                                                                              
[INFO] Finished at: 2023-09-21T16:38:22-04:00                                                                                                                                                                                                                                                            
[INFO] ------------------------------------------------------------------------                                                                                                                                                                                                                          
[ERROR] Failed to execute goal io.quarkus.platform:quarkus-maven-plugin:3.4.1:generate-code (default) on project grpc-import-test: Quarkus code generation phase has failed: InvocationTargetException: Failed to generate Java classes from proto files: [/Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto, /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/role.proto, /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/base.proto] to /Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc with command /Users/tmulle/NetBeansProjects/grpc-import-test/target/com.google.protobuf-protoc-osx-aarch_64-exe -I=/Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-dependencies/2d160609fb74c459975eca766b93d1dc5316867f -I=/Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6 -I=/Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto --plugin=protoc-gen-grpc=/Users/tmulle/NetBeansProjects/grpc-import-test/target/io.grpc-protoc-gen-grpc-java-osx-aarch_64-exe --plugin=protoc-gen-q-grpc=/Users/tmulle/NetBeansProjects/grpc-import-test/target/quarkus-grpc12223228226746760719.sh --q-grpc_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc --grpc_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc --java_out=/Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/grpc /Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/role.proto /Users/tmulle/NetBeansProjects/grpc-import-test/target/protoc-protos-from-dependencies/a037c3d013f4ac2e4ee551e85514d61e5497c4b6/protobuf/base.proto -> [Help 1]                                            
[ERROR]                                                                                                                                                                                                                                                                                                  
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.                                                                                                                                                                                                                      
[ERROR] Re-run Maven using the -X switch to enable full debug logging.                                                                                                                                                                                                                                   
[ERROR]                                                                                                                                                                                                                                                                                                  
[ERROR] For more information about the errors and possible solutions, please read the following articles:                                                                                                                                                                                                
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException     
```


