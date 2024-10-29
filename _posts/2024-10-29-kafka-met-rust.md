---
layout: post
title: "Een kafka producer in Rust"
author: sander.hautvast
categories: [rust, kafka]
image:  
alt:
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Dit is iets wat je gewoon kunt doen, terwijl je op je pipeline zit te wachten:
1. kafka starten in een docker container
2. een kafka consumer op een topic starten in dezelfde container
3. een rust projectje maken dat een bericht stuurt naar de topic.

>Er zijn trouwens meer blogs die hetzelfde doen op andere manieren. Hieronder volgt de eenvoudigste:

#### kafka in een docker container

Dit kun je vinden in de [quickstart](https://kafka.apache.org/quickstart) van kafka: 

```bash
docker run -p 9092:9092 apache/kafka:3.8.0
```

#### een kafka consumer op een topic in dezelfde container
```bash
docker ps
```
Geeft:
![docker ps](/assets/images/kafka-rust/dockerps.png)
```bash
docker exec -it $CONTAINER_ID /bin/bash
```
en binnen deze shell:
```bash
/opt/kafka/bin/kafka-console-consumer.sh --topic rustonomicon  --bootstrap-server localhost:9092
```

Dit is de snelste manier om een consumer te starten, die eenvoudigweg op de commandline de binnenkomende berichten logt. 

#### Een kafka producer in Rust

Natuurlijk is er ook een rust client voor Kafka. Het is niet een officieel ondersteunde en je hebt daarom in principe geen garantie dat hij alles goed doet, of blijft doen na een update. Maar ik denk dat het in de praktijk niet zo'n vaart zal lopen. Het project heeft een permissieve MIT licentie, dus daar heb je ook geen hinder van. 
Het bevat ook een redelijk aantal _examples_, voorbeeldcode die je met cargo --example kunt draaien. Zie de [repo](https://github.com/kafka-rust/kafka-rust/tree/master/examples).

Ik heb de _console-producer.rs_ gepakt en uitgekleed tot de _bare minimum_ om duidelijk te laten zien hoe eenvoudig het kan zijn:

```rust
use anyhow::Result;
use kafka::client::KafkaClient;
use kafka::producer::{Producer, Record};

fn main() -> Result<()> {
    let mut client = KafkaClient::new(vec!["localhost:9092".into()]);
    client.set_client_id("kafka-rust-console-producer".into());
    client.load_metadata_all()?;

    let mut producer = Producer::from_client(client).create()?;

    let record = Record::from_value("rustonomicon", "Hello from rust");
    producer.send(&record)?;
    println!("Sent: {}", record.value);

    Ok(())
}
```
Hoe krijg je dit aan de praat?
1. installeer [rust](https://www.rust-lang.org/tools/install)
2. ```
mkdir hello-kafka && cd hello-kafka
cargo init
```
4. zit dit in Cargo.toml (die nu aangemaakt is):
```toml
[dependencies]
kafka = "0.10"
anyhow = "1.0"
```
5. zet de bovenstaande code in src/main.rs (bestaande code verwijderen).
6. cargo run
![cargo run](/assets/images/kafka-rust/cargorun.png)
7. resultaat in de docker terminal
![kafka consumer](/assets/images/kafka-rust/consumer.png)

Het vervolg zou zijn om niet alleen tekst uit te wisselen via kafka, maar objecten, met behulp van [apache avro](https://avro.apache.org/), het serialisatie format dat kafka native ondersteunt. Daar ga ik snel eens naar kijken.
<div style="text-align: right">∞</div>