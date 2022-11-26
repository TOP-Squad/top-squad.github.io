---
layout: post
title:  "Tekst detectie met OpenCV in rust"
author: sander.hautvast
categories: [Rust, OpenCV, Deep Learning]
image: assets/images/typos.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
[OpenCV](https://opencv.org/) is een schatkist vol tools voor alles wat visueel is. Niet alleen beeldbewerking zoals in tekenprogramma's, maar ook geavanceerde algoritmes, waar iemand ooit op afgestudeerd is. De laatste jaren vind je er ook steeds meer toepassingen van _deep learning_. 

OpenCV is oorspronkelijk een C library, later werd dat C++, maar je kunt er ook mee werken vanuit java, python, én rust...

De installatie op linux en macos is eenvoudig. Installeer eerst opencv, bijvoorbeeld met `brew install opencv`. En start een nieuw rust project en voeg aan de Cargo.toml toe: `opencv = "0.7.4"`. 

Tekst detectie is het vinden van tekst in een plaatje. 
![plaatje](/assets/images/traffic-sign.png)

Klinkt simpel, maar er komt heel wat bij kijken. Het algoritme in OpenCV gebruikt een DCNN (Deep Convolutional Neural Network). Het bereikt goede resultaten onder diverse belichtingsomstandigheden en is niet afhankelijk van de specifieke taal. 

Begin met deze imports en een main functie om een window te tonen en het plaatje te laden.
```rust
use opencv::{prelude::*, highgui};
use opencv::core::{Point, Scalar, Size, Vector};
use opencv::dnn::TextDetectionModel_DB;
use opencv::highgui::imshow;
use opencv::imgcodecs::{imread, IMREAD_COLOR, imwrite};
use opencv::imgproc::{INTER_CUBIC, polylines, resize};


fn main() -> anyhow::Result<()> {
    highgui::named_window("window", highgui::WINDOW_NORMAL)?;
    let mut img = imread("<PATH_TO_IMAGE>", IMREAD_COLOR)?;
...
```

Het eerste wat we vervolgens moeten doen is _resizen_ naar 736x736, omdat het voor dit tekst detectie model de beste resultaten geeft.

```rust
    let mut resized = Mat::default();
    let input_size = Size::new(736, 736);
    resize(&img, &mut resized, input_size, 0_f64, 0_f64, INTER_CUBIC)?;
```

`Mat` staat voor matrix. Het is het standaard formaat voor afbeeldingen in OpenCV.

Nu zijn we klaar voor het deep learning deel. We doen dat door een _pre-trained model_ te [downloaden](https://drive.google.com/uc?export=dowload&id=19YWhArrNccaoSza0CfkXlA8im4-lAGsR) en het onnx bestand in de project root te plaatsen.

Lees het bestand in met de onderstaande regel:
```rust
    let net = opencv::dnn::read_net_from_onnx("DB_TD500_resnet50.onnx")?;
``` 

```rust
    let mut model = TextDetectionModel_DB::new(&net)?;
    model.set_binary_threshold(0.3)?
        .set_polygon_threshold(0.5)?
        .set_max_candidates(200)?
        .set_unclip_ratio(2.0)?;

    let scale = 1.0 / 255.0;
    let mean = Scalar::from((122.67891434, 116.66876762, 104.00698793));

    model.set_input_params(scale, input_size, mean, false, false)?;
```
Deze hyperparameters heb ik overgenomen uit deze [tutorial](https://github.com/opencv/opencv/blob/master/doc/tutorials/dnn/dnn_text_spotting/dnn_text_spotting.markdown). Ze worden daar _helaas_ niet verder uitgelegd.


Alles staat nu klaar om de detectie te starten. Nog wel even een `Vector` instantieren voor de resultaten (een lijst polygonen, geimplementeerd als een `Vector` van `Point`s)
```rust
    let mut det_results: Vector<Vector<Point>> = Vector::new();
    model.detect(&resized, &mut det_results)?;
```
En hiermee teken je de polygonen in de afbeelding

```rust
    polylines(&mut resized, &det_results, true, Scalar::from((0.0, 255.0, 0.0)), 2, 1, 0)?;
```

En natuurlijk willen we ook wat zien. Daarvoor herstellen we de oorspronkelijke aspect ratio, door wéér een `resize` uit te voeren. Daarvoor bepalen we eerst de `orig_size` en voeren we de functie uit, waarbij we de geheugenruimte van de input image hergebruiken. 
```rust
    let orig_size = img.size()?;
    resize(&mut resized, &mut img, orig_size, 0_f64, 0_f64, INTER_CUBIC)?;

    imshow("window", &img)?;

    let mut params = Vector::new();
    params.push(0); //default strategy
    imwrite("result.png", &img, &params)?;
```

Regel 4 toont de afbeelding op het scherm en de rest van de code schrijft het naar een bestand. Let op dat het programma vaak in de achtergrond start, waardoor je eerst _alt/cmd tab_ moet doen om het ook echt te zien.

Tot slot een stukje code om het programma te laten wachten op `esc` om netjes af te sluiten.
```rust
    loop {
        let key = highgui::wait_key(1)?;
        if key == 27 {
            break;
        }
    }

    Ok(())
}
```

En dit is het resultaat:
![plaatje](/assets/images/traffic-sign-detected.png)

_PS. de regelnummers beginnen elk blok weer met 1, maar je mag ze in je hoofd doornummeren. Ik kreeg dit niet voor elkaar met kramdown..._

maar je kunt ook in de gitlab repo kijken: [https://gitlab.com/sander-hautvast/opencv-textdetection-rust](https://gitlab.com/sander-hautvast/opencv-textdetection-rust)


