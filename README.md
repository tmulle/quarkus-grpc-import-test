# grpc-import-test:protobuf-maven-plugin

This project is used to demonstrate that there might be a problem with the `protobuf-maven-plugin` in
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
(base) ➜  grpc-import-test git:(protobuf-maven-plugin-test) mvn clean compile                                                                                                                                                                                                                         
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Detecting the operating system and CPU architecture
[INFO] ------------------------------------------------------------------------
[INFO] os.detected.name: osx
[INFO] os.detected.arch: aarch_64
[INFO] os.detected.bitness: 64
[INFO] os.detected.version: 13.5
[INFO] os.detected.version.major: 13
[INFO] os.detected.version.minor: 5
[INFO] os.detected.classifier: osx-aarch_64
[INFO] 
[INFO] ---------------------< org.acme:grpc-import-test >----------------------
[INFO] Building grpc-import-test 1.0.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ grpc-import-test ---
[INFO] Deleting /Users/tmulle/NetBeansProjects/grpc-import-test/target
[INFO] 
[INFO] --- protobuf:0.6.1:compile (compile) @ grpc-import-test ---
[INFO] Building protoc plugin: quarkus-grpc-protoc-plugin
[INFO] Compiling 1 proto file(s) to /Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/protobuf/java
[ERROR] PROTOC FAILED: base.proto: File not found.
extended.proto:11:1: Import "base.proto" was not found or had errors.
extended.proto:15:5: "org.acme.protos.base.User" is not defined.
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.

[ERROR] /Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto [0:0]: base.proto: File not found.
extended.proto:11:1: Import "base.proto" was not found or had errors.
extended.proto:15:5: "org.acme.protos.base.User" is not defined.
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.

[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.886 s
[INFO] Finished at: 2023-09-21T17:23:18-04:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.xolstice.maven.plugins:protobuf-maven-plugin:0.6.1:compile (compile) on project grpc-import-test: protoc did not exit cleanly. Review output for more information. -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException

```

### New way using Quarkus
Since this is a new project and I wanted to use Quarkus/gRPC I decided to use the built in compiler specified in the docs.

I read https://quarkus.io/guides/grpc-getting-started#protobuf-maven-plugin

After setting things up and finally getting the external jar to download automatically it appears that there is no way to configure the protoc
compiler to go into a subfolder of the expanded artifact.

*protobuf-maven config*
```xml
 <plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>      
    <version>0.6.1</version>
    <configuration>
        <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}</protocArtifact> 
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
        <protocPlugins>
            <protocPlugin>
                <id>quarkus-grpc-protoc-plugin</id>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-grpc-protoc-plugin</artifactId>
                <version>3.4.1</version>
                <mainClass>io.quarkus.grpc.protoc.plugin.MutinyGrpcGenerator</mainClass>
            </protocPlugin>
        </protocPlugins>
<!--                    <includes>
            <include>${project.build.outputDirectory}/**/*.proto</include>
        </includes>
        <protoSourceRoot>src/main/proto</protoSourceRoot>-->
    </configuration>
    <executions>
        <execution>
            <id>compile</id>
            <goals>
                <goal>compile</goal>
                <goal>compile-custom</goal>
            </goals>
        </execution>
        <execution>
            <id>test-compile</id>
            <goals>
                <goal>test-compile</goal>
                <goal>test-compile-custom</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Using the configuration above causes the protos to be found in the main project but not in the dependency included in the pom.

The files appear under the `protoc-dependencies` folder under a random name and `protobuf` folder.

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

If I change the import in the file to `import "protobuf/base.proto"` the compile finds the `base.proto` BUT then any imports inside
the `base.proto` fail.

```bash
(base) ➜  grpc-import-test git:(protobuf-maven-plugin-test) mvn clean compile                                                                                                                                                                                                                            
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Detecting the operating system and CPU architecture
[INFO] ------------------------------------------------------------------------
[INFO] os.detected.name: osx
[INFO] os.detected.arch: aarch_64
[INFO] os.detected.bitness: 64
[INFO] os.detected.version: 13.5
[INFO] os.detected.version.major: 13
[INFO] os.detected.version.minor: 5
[INFO] os.detected.classifier: osx-aarch_64
[INFO] 
[INFO] ---------------------< org.acme:grpc-import-test >----------------------
[INFO] Building grpc-import-test 1.0.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ grpc-import-test ---
[INFO] Deleting /Users/tmulle/NetBeansProjects/grpc-import-test/target
[INFO] 
[INFO] --- protobuf:0.6.1:compile (compile) @ grpc-import-test ---
[INFO] Building protoc plugin: quarkus-grpc-protoc-plugin
[INFO] Compiling 1 proto file(s) to /Users/tmulle/NetBeansProjects/grpc-import-test/target/generated-sources/protobuf/java
[ERROR] PROTOC FAILED: role.proto: File not found.
protobuf/base.proto:9:1: Import "role.proto" was not found or had errors.
protobuf/base.proto:18:5: "org.acme.protos.role.Role" is not defined.
extended.proto:11:1: Import "protobuf/base.proto" was not found or had errors.
extended.proto:15:5: "org.acme.protos.base.User" is not defined.
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.

[ERROR] /Users/tmulle/NetBeansProjects/grpc-import-test/src/main/proto/extended.proto [0:0]: role.proto: File not found.
protobuf/base.proto:9:1: Import "role.proto" was not found or had errors.
protobuf/base.proto:18:5: "org.acme.protos.role.Role" is not defined.
extended.proto:11:1: Import "protobuf/base.proto" was not found or had errors.
extended.proto:15:5: "org.acme.protos.base.User" is not defined.
extended.proto:16:5: "org.acme.protos.base.Address" is not defined.
extended.proto:23:5: "org.acme.protos.base.Address" is not defined.

[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.887 s
[INFO] Finished at: 2023-09-21T17:29:14-04:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.xolstice.maven.plugins:protobuf-maven-plugin:0.6.1:compile (compile) on project grpc-import-test: protoc did not exit cleanly. Review output for more information. -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
  
```

If I uncomment the `includes` section of the plugin so that I can try to include the correct folders, then the main protos are no longer found
with the message `No proto files to compile` even though they are still there in the project.

I even tried to set the `protoSourceRoot` to the same as the default, but still same issue.

```xml
<includes>
    <include>${project.build.outputDirectory}/**/*.proto</include>
</includes>
<protoSourceRoot>src/main/proto</protoSourceRoot>
```


