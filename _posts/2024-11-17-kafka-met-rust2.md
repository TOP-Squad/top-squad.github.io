---
layout: post
title: "Verder met kafka en avro"
author: sander.hautvast
categories: [rust, kafka, java, avro]
image:  assets/images/kafka-rust/kafkaplus.png
alt:
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
In de vorige [episode](/kafka-met-rust) kon je zien hoe je met een eenvoudige setup kafka kunt benaderen voor het versturen van eenvoudige berichten. Dat ging heel snel. Nu wordt het wat meer werk...

Berichten in kafka zijn niets anders dan byte streams. Dus elk protocol voor berichten is denkbaar zolang producer en consumer elkaar begrijpen. Vaak wordt er _avro_ gebruikt. Het is gewoon een naam, geen afkorting.

In deze blog doe ik het volgende:
1. kafka starten in docker compose, met een [KRaft](https://kafka.apache.org/documentation/#kraft) configuratie.
2. een [schema registry](https://docs.confluent.io/platform/current/schema-registry/index.html) opzetten
3. een java producer die berichten met objecten in [avro](https://avro.apache.org/) formaat verstuurt
4. een rust consumer die deze berichten kan ontvangen

#### 1. Een KRaft kafka cluster in docker compose

>Waarom? 

Dit is een iets meer real-world setup. Dat wil zeggen: we zorgen voor failover door drie instanties van kafka te starten. Voorheen was het hiervoor nodig om ook [zookeeper](https://zookeeper.apache.org/) te starten, maar vanaf dit jaar is het sein op [groen](https://cwiki.apache.org/confluence/display/KAFKA/KIP-833%3A+Mark+KRaft+as+Production+Ready) gegaan voor Kafka KRaft in productie. 

KRaft is een implementatie van het [raft](https://raft.github.io/) protocol. Dit is een manier waarop servers data onderling kunnen repliceren. [Hier](http://thesecretlivesofdata.com/raft/) wordt dat heel duidelijk uitgelegd.

**docker compose kafka service**

```yaml
services:
  ...
  kafka1:
    image: confluentinc/cp-kafka:latest
    hostname: kafka1
    container_name: kafka1
    ports:
      - "39092:39092"
    environment:
      KAFKA_LISTENERS: BROKER://kafka1:19092,EXTERNAL://kafka1:39092,CONTROLLER://kafka1:9093
      KAFKA_ADVERTISED_LISTENERS: BROKER://kafka1:19092,EXTERNAL://localhost:39092
      KAFKA_PROCESS_ROLES: 'controller,broker'
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka1:9093,2@kafka2:9093,3@kafka3:9093'
      CLUSTER_ID: 'mycluster'
    volumes:
      - kafka1-data:/var/lib/kafka/data
```
_Ik heb een paar details hier niet bij gezet (zie de [repo](https://github.com/shautvast/kafka-blog/blob/main/dockercompose-cluster.yaml) voor de gehele setup)_

En dat dan dus drie keer.

#### 2. een Schema registry

>Waarom?

Het is een apart product, dat niet van apache kafka komt, maar van confluent. Het wordt er vaak naast gedeployed voor het beheren van schema versies.

Het houdt bij welke schema bij welk topic hoort. Als je avro gebruikt, zal het de eerste keer dat je een bericht in een (nieuw) formaat opstuurt een nieuwe schema versie aanmaken. Deze heet doorgaans ${KAFKA_TOPIC}-value, 

Met de API kun je die dan bijvoorbeeld opvragen. Bijvoorbeeld:

```bash
curl http://localhost:8081/subjects
```

geeft je een lijst van alle bestaande schema's. 

**docker compose schema-registry**
```yaml
services:
...
schema-registry:
    image: confluentinc/cp-schema-registry:latest
    hostname: schema-registry
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_LISTENERS: http://schema-registry:8081
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka1:19092,PLAINTEXT://kafka2:19093,PLAINTEXT://kafka3:19094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,BROKER:PLAINTEXT,EXTERNAL:PLAINTEXT
      SCHEMA_REGISTRY_DEBUG: "true"
```

#### 3. een java/avro producer

Maak een avsc bestand. Dit is de schema definitie. Een java class of iets anders kan eruit gegenereerd worden.

```json
{
  "namespace": "example.avro",
  "type" : "record",
  "name" : "Person",
  "doc" : "Ape descendent creature dwelling on planet Earth.",
  "fields" : [
    {
      "name" : "name",
      "type" : "string"
    },
    {
      "name" : "favorite_number",
      "type" : "int"
    },
    {
      "name" : "height",
      "type" : "double"
    }
  ]
}
```
Ziet er eenvoudig uit, toch? Kijk [hier](https://avro.apache.org/docs/1.11.4/getting-started-java/) om verder te lezen.

Met een maven plugin maak je de source voor java. Er is vast wel iets soortgelijks voor _gradle_.
```xml
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>${avro.version}</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>schema</goal>
            </goals>
            <configuration>
                <stringType>String</stringType>
                <sourceDirectory>${project.basedir}/src/main/resources/avro/</sourceDirectory>
                <outputDirectory>${project.build.directory}/generated/java/</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```
Vergeet niet om `<stringType>String</stringType>` op te nemen in de _configuration_. Zonder deze instelling wordt een avro string in java een CharSequence. Er is vast een goede reden voor, maar ik vind het _jammer_.

En vergeet niet om de outputDirectory toe te voegen als source directory (build-helper-maven-plugin).

**dependencies:**
* org.apache.avro:avro:1.11.4
* org.apache.kafka:kafka-clients:3.8.0
* io.confluent:kafka-avro-serializer:7.7.1

De java code voor de producer ziet er zo uit:

```java
public static void main(String[] args) {
    Properties properties = new Properties();
    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:39092,localhost:39093,localhost:39094");
    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
    properties.put("schema.registry.url", "http://localhost:8081");

    Person person = new Person();
    person.setName("Arthur Dent");
    person.setFavoriteNumber(42);
    person.setHeight(1.90);

    ProducerRecord<String, Person> record = new ProducerRecord<>("rustonomicon", "arthur", person);

    try (KafkaProducer<String, Person> producer = new KafkaProducer<>(properties)) {
        Future<RecordMetadata> future = producer.send(record);
        // onderstaande is alleen voor de demo, niet in productie doen.
        RecordMetadata metadata = future.get();
        System.out.println("Message sent to partition " + metadata.partition() + ", offset " + metadata.offset());

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

* kafka producers en consumers praten met het hele cluster, in plaats van dat alles door dezelfde _leader/master_ gaat. Dit is de manier waarop kafka de load verdeelt.
* de KafkaProducer communiceert met de schema registry
* producer.send is async

#### 4. een rust consumer die deze berichten kan ontvangen

Het rust ecosysteem is een stuk onrustiger dan dat van java. Dat maakte dat ik hier veel moeite had om een set libraries te vinden die goed met elkaar samenwerken. Het gaat dan om kafka, avro en de schema-registry. [3] gaf uiteindelijk de oplossing.

```toml
apache-avro = "0.17.0"
anyhow = "1.0"
serde = "1.0"
serde_derive = "1.0"
rdkafka = "0.36"
tokio = { version = "1", features = ["full"] }
futures = "0.3.28"
schema_registry_converter = { version = "4.2", features = ["avro"] }
tokio-macros = "2.4.0"
```

De code is weer zo kort mogelijk en ontdaan van allerlei zaken die in het echt niet mogen ontbreken, zoals nettere error-handling, logging en wellicht een Sender, voor het draaien van de listener in een andere thread dan de uiteindelijke message-handler.

Omdat deze consumer niet _ahead-of-time_ op de hoogte is van het Person type, dat we in java hadden, is de code niet symmetrisch. In plaats van een rust variant van de Person (is ook mogelijk), hebben we hier een generiek `Record` object (enum variant). De inhoud van een Record is een lijst tuples (key,value). Een HashMap was te luxueus, zeg maar.

`group.id` is een manier om de load over verschillende consumers te verdelen, mocht dat nodig zijn. Elke consumer met een gelijke group.id zal maar een subset van de berichten krijgen om te verwerken.

```rust
use apache_avro::types::Value;
use rdkafka::{
    consumer::{CommitMode, Consumer, StreamConsumer},
    ClientConfig, Message,
};
use schema_registry_converter::async_impl::{avro::AvroDecoder, schema_registry::SrSettings};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("group.id", "mygroup")
        .set(
            "bootstrap.servers",
            "localhost:39092,localhost:39093,localhost:39094",
        )
        .create()?;
    let avro_decoder = AvroDecoder::new(SrSettings::new("http://localhost:8081".into()));

    consumer.subscribe(&["rustonomicon"])?;

    while let Ok(message) = consumer.recv().await {
        let value_result = avro_decoder.decode(message.payload()).await?.value;
        if let Value::Record(value_result) = value_result {
            println!("{:?}", value_result.get(0));
        }

        consumer.commit_message(&message, CommitMode::Async)?;
    }

    Ok(())
}

```

met dank aan
1. [https://medium.com/@katyagorshkova/docker-compose-for-running-kafka-in-kraft-mode-20c535c48b1a](https://medium.com/@katyagorshkova/docker-compose-for-running-kafka-in-kraft-mode-20c535c48b1a)
2. [https://medium.com/slalom-technology/introduction-to-schema-registry-in-kafka-915ccf06b902](https://medium.com/slalom-technology/introduction-to-schema-registry-in-kafka-915ccf06b902)
3. [https://medium.com/@omprakashsridharan/rust-multi-module-microservices-part-4-kafka-with-avro-f11204919da5](https://medium.com/@omprakashsridharan/rust-multi-module-microservices-part-4-kafka-with-avro-f11204919da5)
4. [https://romanglushach.medium.com/the-evolution-of-kafka-architecture-from-zookeeper-to-kraft-f42d511ba242](https://romanglushach.medium.com/the-evolution-of-kafka-architecture-from-zookeeper-to-kraft-f42d511ba242)
5. Jetbrains AI assistant

<div style="text-align: right">∞</div>