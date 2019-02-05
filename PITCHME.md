### Dockerized 3rd Party Components in ScalaTest

---

## Why?

* Mocks are faster but are not the _real_ thing
* Running embedded can cause issues
	- jar hell
	- not supported by the developers
	- adapt the code for the tests
* Docker community is large
	- Most of the apps have ready made images

---

## The TestContainers Library

* pure Java library - docker CLI is not needed
* self contained - all dependencies are shaded (won't conflict with Apache when trying to use the Unix socket)
*  simple (almost declarative) API

---

### Simple Start

* Starting and stopping a Docker container before and after a test Suite.
* Accessing the container from the test
* Container logs
* Linux and Mac support (Windows on best effort basis)

+++

#### Starting ES container

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
    val addr = container.containerIpAddress
    val port = container.mappedPort(9200)
    val output = scala.io.Source.fromURL(s"http://$addr:$port").mkString
    print(s"The output:\n$output\n")
    assert(true)
  }

}
```
@[1](Start the container before the suite and close it after)
@[4](override the container)
@[5](container declaration)
@[6](exposing ports)
@[7](Wait until the wait condition met)
@[8](environments variable inject to the container)
@[10-13](getting access to the internal Java container)
@[17-23](demo test)
@[18](getting the container ip address (mac ready!))
@[19](getting the mapped port accessible from the host machine)

+++

### Live Demo

* Fast start and stop
* Monitoring crashed test JVM (e.g. pressing stop in Intellij)
* Keeps the host machine clean after the tests
* Debugging in Intellij!
* Running a single test and not the whole test suite (even from Intellij)

---

### Multiple Containers

```scala
  def elasticsearch(elasticsearchVersion: String): GenericContainer = {
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

  def cassandra(cassandraVersion: String): GenericContainer = {
    val scalaContainer = GenericContainer(s"cassandra:$cassandraVersion",
      waitStrategy = Wait.forLogMessage(".*Starting listening for CQL clients.*\n", 1),
      env = Map("JVM_OPTS" -> "-Xms1G -Xmx1G")
    )
    scalaContainer.configure { container =>
      val logger = new Slf4jLogConsumer(LoggerFactory.getLogger(container.getDockerImageName))
      container.withLogConsumer(logger)
    }
    scalaContainer
  }
```

@[16](Another wait strategy)

+++

#### The Trait (reusable for several test suites)

```scala
trait EsCasDockerSuite extends ForAllTestContainer { this:Suite =>
  def cassandraVersion: String
  def elasticsearchVersion: String
  val elasticsearchContainer = ContainerHelpers.elasticsearch(elasticsearchVersion)
  val cassandraContainer = ContainerHelpers.cassandra(cassandraVersion)
  override val container = MultipleContainers(cassandraContainer, elasticsearchContainer)
}
```
@[6](Note: the containers won't start in parallel)

+++

#### The Actual Test Suite

```scala
class TestESCasSuite extends FlatSpec with EsCasDockerSuite {
  override def elasticsearchVersion: String = "6.5.4"
  override def cassandraVersion: String = "3.11.3"

  "EsCasDockerSuite" should "work" in {
    assert(true)
  }
}
```

---

### Starting the Containers in Parallel

```scala
class MultipleContainersParallelExecution private(containers: Seq[LazyContainer[_]]) extends Container {
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.concurrent.duration._

  override def finished()(implicit description: Description): Unit = {
    val f = Future.traverse(containers)(lazyContainer => Future(lazyContainer.finished()(description)))
    Await.ready(f, 5.minutes)
  }

  override def succeeded()(implicit description: Description): Unit = {
    val f = Future.traverse(containers)(lazyContainer => Future(lazyContainer.succeeded()(description)))
    Await.ready(f, 5.minutes)
  }

  override def starting()(implicit description: Description): Unit = {
    val f = Future.traverse(containers)(lazyContainer => Future(lazyContainer.starting()(description)))
    Await.ready(f, 5.minutes)
  }

  override def failed(e: Throwable)(implicit description: Description): Unit = {
    val f = Future.traverse(containers)(lazyContainer => Future(lazyContainer.failed(e)(description)))
    Await.ready(f, 5.minutes)
  }
}

object MultipleContainersParallelExecution {

  def apply(containers: LazyContainer[_]*): MultipleContainersParallelExecution = new MultipleContainersParallelExecution(containers)
}
```

---

## Special Case Study
#### Kafka

+++

### The Challenge

![Kafka Communication](assets/kafka-communication.png)

+++

### Solution?

![Kafka Multi Broker](assets/kafka-multi-broker.png)

+++

#### The Solution <br /> Kafka and Zookeeper Containers

```scala
def kafkaAndZookeeper(kafkaVersion: String, zookeeperVersion: String): KafkaZookeeperContainers = {
  val internalNetwork = Network.newNetwork()
  lazy val zookeeperContainer = {
    val scalaContainer = GenericContainer(s"zookeeper:$zookeeperVersion",
      exposedPorts = Seq(2181),
      waitStrategy = Wait.forLogMessage(".*binding to port 0.0.0.0/0.0.0.0:2181.*\n", 1)
    )
    scalaContainer.configure { container =>
      container.withNetwork(internalNetwork)
      container.withNetworkAliases("zookeeper")
      val logger = new Slf4jLogConsumer(LoggerFactory.getLogger(container.getDockerImageName))
      container.withLogConsumer(logger)
    }
    scalaContainer
  }
  lazy val kafkaContainer = {
    val externalPort = 9092
    val internalPort = 10000 + externalPort
    val brokerId = 1
    val scalaContainer = GenericContainer(s"wurstmeister/kafka:$kafkaVersion",
      exposedPorts = Seq(externalPort),
      env = Map(
        "KAFKA_ZOOKEEPER_CONNECT" -> s"zookeeper:2181",
        "KAFKA_LISTENERS" -> s"INTERNAL://0.0.0.0:$internalPort,EXTERNAL://0.0.0.0:$externalPort",
        "KAFKA_LISTENER_SECURITY_PROTOCOL_MAP" -> s"INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT",
        "KAFKA_INTER_BROKER_LISTENER_NAME" -> s"INTERNAL",
        "KAFKA_CREATE_TOPICS" -> s"persist_topic:1:1,persist_topic.priority:1:1,index_topic:1:1,index_topic.priority:1:1",
        "KAFKA_BROKER_ID" -> s"$brokerId"
      ),
      waitStrategy = Wait.forLogMessage(".*KafkaServer.*started.*\n", 1)
    )
    scalaContainer.configure { container =>
      container.withNetwork(internalNetwork)
      val networkAlias = s"kafkaBroker-$brokerId"
      container.withNetworkAliases(networkAlias)
      //The network alias can be used the advertising listeners later (for multi brokers configuration)
      container.addEnv("KAFKA_ADVERTISED_LISTENERS", s"INTERNAL://$networkAlias:$internalPort")
      val logger = new Slf4jLogConsumer(LoggerFactory.getLogger(container.getDockerImageName))
      container.withLogConsumer(logger)
    }
    scalaContainer
  }
  val combined = MultipleContainers(zookeeperContainer, kafkaContainer)
  KafkaZookeeperContainers(kafkaContainer, zookeeperContainer, combined)
}
```
@[2](Declare a new Docker network (won't be created until the actual run))
@[9](Add this container the the network)
@[10](Give this container a DNS name inside the network)
@[17](The INTERNAL port of Kafka to be used by EXTERNAL clients)
@[18](The INTERNAL port to be used by Kafka brokers/INTERNAL clients)
@[23](Zookeeper hostname is known in advance)
@[24](Configure Kafka to listen to TWO ports)
@[26](Inter broker communication will be done using the internal port)
@[28](Set the Kafka broker ID manually)
@[37](Advertise only the INTERNAL network configuration (for now))

+++

#### The Actual Suite

```scala
trait KafkaZookeeperDockerSuite extends ForAllTestContainer { this: Suite =>
  def zookeeperVersion: String
  def kafkaVersion: String
  val KafkaZookeeperContainers(kafkaContainer, zookeeperContainer, combined) = ContainerHelpers.kafkaAndZookeeper(kafkaVersion, zookeeperVersion)
  override val container = combined

  override def afterStart(): Unit = {
    super.afterStart()
    // scalastyle:off
    kafkaContainer.configure{ container =>
      val result = container.execInContainer("bash", "-c", "${KAFKA_HOME}/bin/kafka-configs.sh " +
        "--bootstrap-server localhost:19092 --entity-type brokers --entity-name 1 --alter --add-config " +
        s"advertised.listeners=[EXTERNAL://${kafkaContainer.containerIpAddress}:${kafkaContainer.mappedPort(9092)},INTERNAL://kafkaBroker-1:19092]")
      val stdOut = result.getStdout.trim
      if (stdOut != "Completed updating config for broker: 1.") {
        val stdErr = result.getStderr.trim
        throw new Exception(s"Couldn't change Kafka's advertised listeners config for broker 1. stdout: [$stdOut]. stderr: [$stdErr]")
      }
    }
    // scalastyle:on
  }
}
```
@[12](Advertising the EXTERNAL network configuration)
@[13-18](Making sure the configuration succeeded)


+++
#### Using the Suite

```scala
class TestKZSuite extends FlatSpec with KafkaZookeeperDockerSuite {
  override def zookeeperVersion: String = "3.4.13"
  override def kafkaVersion: String = "2.12-2.1.0"

  "KafkaZookeeperDockerSuite" should "work" in {
    //dummy line for breakpoint
    val a = 10
    assert(true)
  }
}
```

---

### Final Notes

* The library also supports:
	- Dockerfile
	- compose (even without compose installed)
* The container can reach the host (Linux and mac!)
	- Expose the host port `kjhkjh`
	- Reach from the container using `lkjlkj`
* Works perfectly in Jenkins (thanks Amos) but the docker socket should be available
	
