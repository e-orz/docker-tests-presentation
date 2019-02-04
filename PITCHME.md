# Dockerized 3rd Party Components in ScalaTest

---

## Agenda

* Starting and stopping a Docker container before and after a test Suite.
* Accessing the container from the test
* Container logs
* Starting and stopping multiple containers (sequentially or in parallel)
* Linux and Mac support (Windows on best effort basis)
* Running the tests alongside the real running product
* Fast start and stop
* Monitoring crashed test JVM (e.g. pressing stop in Intellij)
* Keeps the host machine clean after the tests (live demo)
* Container vs. embedded (jar hell😟, large community)
* Debugging in Intellij!
* Running a single test and not the whole test suite (even from Intellij)
* Special case study: Kafka ("compose" in code, private network, exposed port discovery)
* Jenkins support (mapping the docker socket)

---

## The TestContainers Library


*lkj
lkjflkdsj
* kfjdslj
* pure Java library - docker Cli is not needed
* self contained - all depenencies are sharded (won't confilict with apache when trying to use the Unix socket
)
*  simple (almost declerative) API
* 8

+++
@snap[text-15 text-gold]
Code presenting rrrrrepository source file template.
@snapend
```scala
class TestESSuiteDemo extends FlatSpec with ForAllTestContainer {
  val elasticsearchVersion: String = "6.5.4"

  override val container = {
    val scalaContainer = GenericContainer(s"docker.elastic.co/elasticsearch/elasticsearch-oss:$elasticsearchVersion",
      exposedPorts = Seq(9200),
      waitStrategy = Wait.forHttp("/").forPort(9200).forStatusCode(200),
      env = Map("discovery.type" -> "single-node", "ES_JAVA_OPTS" -> "-Xms2000m -Xmx2000m")
    )
    scalaContainer.configure { container =>
      val logger = new Slf4jLogConsumer(LoggerFactory.getLogger(s"elasticsearch-oss:$elasticsearchVersion"))
      container.withLogConsumer(logger)
    }
    scalaContainer
  }

  "TestEsSuiteDemo" should "work" in {
    val output = scala.io.Source.fromURL(s"http://${container.containerIpAddress}:${container.mappedPort(9200)}").mkString
    print(s"The output:\n$output\n")
    assert(true)
  }

}
```
@[1](Start the container before the suite and close it after)
@[4](the container)
@[5](container declaration)
@[6](the container)
@[5-16](getting access to the internal Java container)
