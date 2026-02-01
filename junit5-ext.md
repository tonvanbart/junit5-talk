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
---
<!-- _class: lead  -->
# JUnit5 extensions
### for fun and profit

---
# About this talk

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
        try (TopologyTestDriver driver = new TopologyTestDriver(topology)) {
            TestInputTopic<String, String> inputTopic = driver.createInputTopic("test_input", new StringSerializer(), new StringSerializer());
            var outputTopic = driver.createOutputTopic("test_output", new StringDeserializer(), new StringDeserializer());
            inputTopic.pipeInput("key1", "value1");
            var keyValue = outputTopic.readKeyValue();
            
            // do asserts on the topic contents
        }

```
<!--
All KSML does is create a Kafka Streams topology.
Once we have that, we could use TopologyTestDriver to put some data through and verify thet the pipeline does what we want. 
-->
---

