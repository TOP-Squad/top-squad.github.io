---
layout: post
title:  "Multithreaded Rust: kies je channel!"
author: sander.hautvast
categories: [ Rust, multithreading ]
image: assets/images/stephane-gagnon-NLgqFA9Lg_E-unsplash.jpg
beforetoc: "De standaard channels zijn niet altijd de beste"
featured: true
hidden: false
lang: nl
---
Stel je voor dat je de woorden in een bestand moet tellen (aantal per uniek woord). Het eindresultaat in het geheugen is dus een _Map_ met als _key_ het woord en als _value_ het aantal.

Misschien bevat de input veel ruis, zoals leestekens, getallen of zelfs onmogelijke UTF karakters. Je wil de ruis wegfilteren. Dat zorgt ervoor dat de verwerkingstijd per woord waarschijnlijk langer is dan de tijd voor I/O. In dat geval is het misschien zinvol om één (main) thread de file te laten lezen en een x-aantal andere de verwerking te laten doen. Elke job krijgt een eigen `Map<String, usize>` voor de telling. De afzonderlijke tellingen worden tot slot samengevoegd (bijvoorbeeld in de main thread). Een soort fork-join dus. 

Daarnaast willen we dat het proces niet zoveel geheugen gaat bevatten als het bestand groot is, want dan kan vele gigabytes beslaan. Regel voor regel inlezen en gelijk verwerken dus.

**Channels**

Er vindt op 2 manieren inter-thread communicatie plaats: 
1. verdelen van ingelezen woorden over tellers (jobs in threadpool)
2. samenvoegen van deelresultaten van de tellers

Dé manier om dit (safe) te doen is met zogenaamde [Channels](https://doc.rust-lang.org/rust-by-example/std_misc/channels.html). Rust channels zijn conceptueel gelijk aan die van _Go_ en _Kotlin_. In _Java_ zou je waarschijnlijk een `Deque` implementatie gebruiken die `java.util.concurrent` levert, of werken met een threadpool.

Je kunt (misschien) ook zonder Channels, maar is _niet-idiomatisch_ én zorgt dat je waarschijnlijk vastloopt in compiler klachten over _ownership_ (zoals ik...)

Laten we een `threadpool` maken voor een x-aantal workers, die allemaal van een channel lezen. Frameworks zoals [rayon](https://crates.io/crates/rayon) doen dit al voor je. Het doel is hier iets te vertellen over channels. Ik gebruik [crossbeam_channel](https://crates.io/crates/crossbeam-channel) als implementatie, want ze claimen dat die beter is dan `std::sync::mpsc::channel`, maar voor wat we hier doen, maakt het eigenlijk niet zoveel uit (ik zag geen verschil).

```rust
pub struct ThreadPool {
  workers: Vec<Worker>,
  sender: crossbeam_channel::Sender<Message>,
}

enum Message {
  NewJob(Job),
  Terminate
}

type Job = Box<dyn FnOnce() + Send + 'static>;
```

Deze code is gebaseerd (met wijzigingen) op een voorbeeld in het [rust book](https://doc.rust-lang.org/book/ch20-02-multithreaded.html) (Listing 20-15 en verder). Er zijn zoveel workers als er threads zijn (parameter). Een worker kijkt constant of er een `Message::NewJob` binnenkomt en verwerkt deze dan, totdat er een einde-signaal (`Message::Terminate`) arriveert.

```rust
impl ThreadPool {
  pub fn new(nworkers: usize) -> Self{
    assert!(nworkers > 0);

    let (sender,receiver) = crossbeam_channel::unbounded();
    let receiver = Arc::new(Mutex::new(receiver)); //wrap de receiver in mutex voor gebruik in threads

    let mut workers = Vec::with_capacity(nworkers);

    for _ in 0..nworkers {
      workers.push(Worker::new(Arc::clone(&receiver)));
    }

    Self{
      workers,
      sender,
    }
  }
}
```

En zo ziet een worker eruit:

```rust
struct Worker {
  thread: Option<thread::JoinHandle<()>>,
}

impl Worker{
  fn new(receiver: Arc<Mutex<crossbeam_channel::Receiver<Message>>>) -> Self {
    loop{
      let thread = thread::spawn(move||{
        let maybe_message = receiver.lock().unwrap().recv_timeout(Duration::from_secs(1));

        if let Ok(message) = maybe_message{
          match message {
            Message::NewJob(job) => {
              job(); // voer de aan de pool aangeboden functie uit
            },
            Message::Terminate => {
              break; // regulier einde
            }
          }
        } else {
          break; // deze situatie treedt op als de receiver nooit iets heeft ontvangen
        }
      });
    }
  }
}
```

(Wat ik niet heb laten zien is dat de Message niet 1 woord bevat, maar een buffer van 10.000 regels. Buffering is slim om context-switching te voorkomen en memory [efficient](https://mechanical-sympathy.blogspot.com/2012/08/memory-access-patterns-are-important.html) te lezen.)

_Op deze manier zou je denken dat je zonder eerst het hele bestand in te lezen de telling kunt doen._ 

**Toch ?**

Mijn bestand was 35 Gb groot. Het proces ook ongeveer. Niet zo handig! Wat is er aan de hand?

De [docs](https://docs.rs/crossbeam-channel/latest/crossbeam_channel/):<br/>
`bounded` creates a channel of bounded capacity, i.e. there is a limit to how many messages it can hold at a time.<br/>
`unbounded` creates a channel of unbounded capacity, i.e. it can hold any number of messages at a time.<br/>

Ik had in eerste instantie zonder diep na te denken unbounded genomen. Dat betekent dat het proces het bestand in gaat lezen, zo snel als de I/O op de machine dat toelaat. De verwerking was puur in het geheugen, maar omdat die toch trager ging dan het inlezen, stapelden de jobs (inclusief de ingelezen data) zich op in het channel en daarmee in het geheugen. 
De oplossing was vrij simpel, namelijk door in de ThreadPool constructor unbounded te vervangen door bounded (rg. 49):

```rust
let (sender,receiver) = crossbeam_channel::bounded(nworkers * 2);
```

Het aantal nworkers*2 komt niet heel nauw, maar op deze manier zorg je dat er voor een draaiende thread altijd een volgende klaar staat om op te pakken.

Daarna zakte het geheugengebruik in naar ca 3 Gb.

**Conclusie**<br/>
Channels zijn generieke constructen. Het komt aan op de manier waarop ze gebruikt worden. Het kan zijn dat een _bounded_ channel onwenselijk is, bijvoorbeeld als het maximum op geen enkele manier te bepalen is, of wanneer de _lees_ snelheid gegarandeerd hoog genoeg is. Het geheugengebruik per job is natuurlijk ook een factor van belang. Als dat allemaal niet aan de hand is, is het veiliger om _bounded_ te nemen. Kleine wijziging, groot verschil!

<div style="text-align: right">∞</div>