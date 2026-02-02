---
marp: true
theme: gaia
paginate: false
# class: invert
style: |
  pre {
    font-size: 0.8em;
  }
  section {
    print-color-adjust: exact;
    -webkit-print-color-adjust: exact;
  }
  h1, h2 {
    text-align: center;
  }
  section::before {
    content: '';
    position: absolute;
    top: 20px;
    right: 20px;
    width: 80px;
    height: 80px;
    background-image: url('./axual-logo.svg');
    background-size: contain;
    background-repeat: no-repeat;
  }
---
<!-- _class: lead  -->
# JUnit5 extensions
### for fun and profit

---
## About this talk

* About how (and why) to write JUnit5 extensions
* Uses @KSMLTestExtension as an example
* We'll skip over KSML internals where we can

---
## The challenge
```yaml
streams:
  test_input:
    topic: input_topic
  test_output:
    topic: output_topic

pipelines:
  main:
    from: test_input
    via:
      - type: mapKey
        mapper:
          code: |
            return key[:4].upper()
          resultType: string
    to: test_output

```
<!-- 
* We would like automated tests for this
* There are a lot of KSML operations
* some have multiple variants: code, expression
-->
---
## Solution: Kafka TopologyTestDriver
```java
        final var uri = ClassLoader.getSystemResource("pipelines/test-copying.yaml").toURI();
        final var path = Paths.get(uri);
        final var definition = YAMLObjectMapper.INSTANCE.readValue(Files.readString(path), JsonNode.class);
        final var definitions = ImmutableMap.of("definition",
                new TopologyDefinitionParser("test").parse(ParseNode.fromRoot(definition, "test")));
        var topologyGenerator = new TopologyGenerator("some.app.id");
        final var topology = topologyGenerator.create(streamsBuilder, definitions);

        try (TopologyTestDriver driver = new TopologyTestDriver(topology)) {
            var inputTopic = driver.createInputTopic("test_input", new StringSerializer(), new StringSerializer());
            var outputTopic = driver.createOutputTopic("test_output", new StringDeserializer(), new StringDeserializer());
            inputTopic.pipeInput("key1", "value1");
            var keyValue = outputTopic.readKeyValue();
            
            // do asserts on the topic contents
        }
```
* boilerplate obscuring the test intent (more if using AVRO)
* boilerplate not easily generalized
<!--
All KSML does is create a Kafka Streams topology, in test setup we can call these classes.
Once we have the topology, we could use TopologyTestDriver to put some data through and verify thet 
the pipeline does what we want. 
However there is quite a bit of boilerplate code that is not easy to generalize. Considering creating a 
superclass but that idea won't fly.
-->
---
## JUnit5 extensions

Extension points related to certain events in test execution
Five main types:
* <span style="color: red">conditional test execution</span>
* <span style="color: red">lifecycle callbacks</span>
* test instance post processing
* parameter resolution
* exception handling

[docs.junit.org/5.0.1/api/org/junit/jupiter/api/extension](https://docs.junit.org/5.0.1/api/org/junit/jupiter/api/extension/package-summary.html)
<!-- 
There are 5 main types of extension points. For our extension, we are interested in
* conditional test execution: the tests only run on GraalVM.
* lifecycle callbacks so we can set the test driver up before the test, and clean up afterwards.
-->

---
## Conditional test execution
Verify that the test is running on GraalVM.
```java
    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(final ExtensionContext extensionContext) {
        if (extensionContext.getTestMethod().isEmpty()) {
            // at class level verification
            log.debug("Check for GraalVM");
            if (Version.getCurrent().isRelease()) {       
                return ConditionEvaluationResult.enabled("running on GraalVM");
            }
            log.warn("KSML tests need GraalVM to run, test disabled");
            extensionContext.publishReportEntry("KSML tests need GraalVM, test disabled");
            return ConditionEvaluationResult.disabled("KSML tests need GraalVM to run");
        }
        return ConditionEvaluationResult.enabled("on method");
    }

```
<!-- 
This extension point gets called once for the test class, and once for each test method. We're checking on the
global level, and disable the test as a whole if not on Graal.
-->