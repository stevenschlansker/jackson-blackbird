# Jackson Blackbird
_Upgrade your Afterburner for your Java 11+ Environment 🚀_

The [Afterburner](https://github.com/FasterXML/jackson-modules-base/tree/master/afterburner)
has long been your engine of choice for maximum Jackson performance.
But in the brave new Java 11 world, the trusty Afterburner is showing its age.
It uses horrifying bytecode manipulation and cracks `Unsafe.defineClass` which will
[stop working soon](https://github.com/FasterXML/jackson-modules-base/issues/37).

Blackbird is Afterburner Mk II - powered by 100% renewable Lambda based fuel.

The [LambdaMetafactory](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/invoke/LambdaMetafactory.html)
introduces a standard Java API for dynamically instantiating function objects.
The current OpenJDK implementation generates anonymous classes in a somewhat similar fashion
to the classic Afterburner.  While the metafactory cannot generate comparably specialized
implementations, we can write needed adapters as simple Java code and use the metafactory
to create distinct call sites for every needed access path.  This should allow each accessor to
have a monomorphic call profile and easily inline for maximum performance.

If the module proves useful, eventually I hope to integrate it into `jackson-modules-base` or even
as a default strategy for Jackson 3.

## Status

Blackbird is experimental and needs brave pilots to take her for test flights!
The code is written to fail-fast and explode on the tarmac rather than later at runtime.

Blackbird passes all the original Afterburner tests (except a couple that didn't make sense anymore).

## Usage

### Maven dependency

To use module on Maven-based projects, use following dependency:

```xml
<dependency>
  <groupId>com.fasterxml.jackson.module</groupId>
  <artifactId>jackson-module-blackbird</artifactId>
  <version>2.10.0-SNAPSHOT</version>
</dependency>
```

Since there's no releases yet, you'll have to build it locally.

### Non-Maven

Upgrade to Maven.

### Registering module

To use the the Module in Jackson, simply register it with the ObjectMapper instance:

```java
ObjectMapper mapper = new ObjectMapper()
mapper.registerModule(new BlackbirdModule());
```

after which you just do data-binding as usual:

```java
Value val = mapper.readValue(jsonSource, Value.class);
mapper.writeValue(new File("result.json"), val);
```

If you're really brave and running with a modulepath, you may need to grant Blackbird access to your classes.
This is done by calling the caller sensitive `MethodHandles.lookup()`.  The module constructor allows you to customize
which lookup instances are used for what classes.

### What is optimized?

Following things are optimized:

* For serialization (POJOs to JSON):
    * Getter methods are inlined using generated lambdas instead of reflection
    * Serializers for small number of 'primitive' types (`int`, `long`, `boolean`, `String`) are replaced with lambda-specialized generators, instead of getting delegated to `JsonSerializer`s
* For deserialization (JSON to POJOs):
    * Calls to default (no-argument) constructors are lambda-fied instead of using reflection
    * Calls to @JsonCreate factory methods and constructors with arguments get a slightly less efficient lambda based implementation
    * Setter methods are called using lambdas instead of reflection
    * Deserializers for small number of 'primitive' types (`int`, `long`, `boolean`, `String`) are replaced with lambda-specialized parsers, instead of getting delegated to `JsonDeserializer`s
 
### ... and what is not?

* Any sort of direct field access.  This may be possible to support someday but requires additional JDK support.  Use methods instead.
* Streaming parser/generator access
* Tree Model: there isn't much that can be done at this level

## Performance

Blackbird has been lightly performance tested using `jmh` on openjdk 11 and 12.
For reading and writing a moderately complex bean, Blackbird performance seems
to be almost exactly on par wtihAfterburner - up to 20% better than vanilla Jackson in some cases.
At that point the method invocation overhead disappears behind the heat generated
from the long parser and UTF8 decoder.

I used [jitwatch](https://github.com/AdoptOpenJDK/jitwatch) with `hsdis` to verify the generated code at a
superficial level.  The generated call sites do seem to be good targets for inlining, although unfortunately
[jitwatch currently can't analyze lambdas very well](https://github.com/AdoptOpenJDK/jitwatch/issues/282) so
more advanced analysis may have to come later.