---
layout: post
title: "Huffman coding in Kotlin"
author: sander.hautvast
categories: [java, jdk-22, rust, apm]
image:  assets/images/huffman/ivory.jpeg
alt: an image of a computer scientist on top of a white concrete ivory tower that is also a graph of nodes and vertices
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Ik kwam [dit artikel](https://lazamar.github.io/haskell-data-compression-with-huffman-codes) tegen dat precies uitlegt hoe Huffman codes werken, en daarbij een algoritme geeft in Haskell.

Wist je dat Huffman codes aan de basis liggen van JPEG image compression?

Ik woon niet in een ivoren toren en spreek geen Haskell. Ik kende het algoritme ook niet precies.
Ik kreeg wel zin om te kijken of de code net zo kort en fraai kan in kotlin. 

>spoiler: het komt in de buurt. Misschien is het zelfs beter!

### huffman coding
Huffman coding is een lossless compressie techniek. Stel dat je een tekst wil comprimeren, dan zijn er de volgende stappen:
1. maak een frequentie tabel voor elk karakter
2. maak een binary tree van de karakters en hun gewicht (de frequentie)
3. ken vanuit de tree een binaire code toe aan elk karakter, zodanig dat:
4. de meest voorkomende karakters de kortste code krijgen
5. de codes (achter elkaar geplakt, zoals de oorspronkelijke tekst) leesbaar blijven (zonder separators)

Voor details, lees het artikel. Ik ga het niet herhalen, maar puur de vergelijking doen van haskell en kotlin (wat ik ervan gemaakt heb althans...).

### 1. frequenties tellen

>Maak een Map van elk karakter als sleutel naar het aantal keer dat dat karakter voorkomt. 

_haskell_
```haskell
type FreqMap = Map Char Int

countFrequency :: String -> FreqMap
countFrequency = Map.fromListWith (+) . fmap (,1)
```

_kotlin_
```kotlin
typealias FreqMap = Map<Char, Int>

fun countFrequency(data: String): FreqMap =
    data.toList().groupingBy { it }.eachCount()
```

Ik had even nodig om de haskell te begrijpen. De declaratie en de implementatie staan op 2 opvolgende regels en herhalen de functie naam. `fmap (,1)` ziet eruit als een hack!

De kotlin code is net zo kort én beter leesbaar. Het _mimict_ een SQL statement: `select ..., count(1) from ... group by ...`

De type alias van kotlin verhoogt net als bij Haskell de leesbaarheid en de typesafety. En is wel zo handig dat elke `Map<Char, Int>` automatisch een FreqMap is. Het feit dat kotlin de conversie van String naar List<Char> niet automatisch doet (maar met toList) vind ik geen nadeel.

### 2. de tree opbouwen

De bedoeling is:
1. maak van de frequentie map een (op gewicht) gesorteerde lijst Leaf nodes 
2. combineer steeds twee opeenvolgende nodes in een Fork node (de merge functie)
3. zorg dat de lijst nodes (Leaf of Fork) steeds gesorteerd blijft.

_haskell_
```haskell
data HTree
  = Leaf Weight Char
  | Fork Weight HTree HTree
  deriving Eq

instance Ord HTree where
  compare x y = compare (weight x) (weight y)

buildTree :: FreqMap -> HTree
buildTree = build . sort . fmap (\(c,w) -> Leaf w c ) . Map.toList
  where
  build trees = case trees of 
    [] -> error "empty trees"
    [x] -> x
    [x:y:rest] -> build $ insert (merge x y) rest

  merge x y = Fork (weight x + weight y) x y  
```

* `weight` is een functie die de weight van de node geeft.
* `merge` is een functie die van 2 nodes een (bovenliggende) Fork node maakt met de twee input nodes als children en het gecombineerde gewicht van beide

Haskell is inderdaad briljant hier met pattern matching en ADT's (Algebraic Data Type, de HTree), maar weer moeilijker leesbaar (IMHO). Merk op dat de nested `build` functie recursief wordt aangeroepen en dat de 'aanroep' (rg 10) vóór de declaratie staat (rg 12 ev)

* Lees `$...` als: 'resultaat van ...'
* `insert` is een functie die de sorteervolgorde handhaaft als het lijst element `Ord` is.

_kotlin_
```kotlin
sealed class HTree(open val weight: Int) {
    data class Leaf(override val weight: Int, val char: Char) : HTree(weight)
    data class Fork(override val weight: Int, val left: HTree, val right: HTree) : HTree(weight)
}

fun buildTree(freqMap: FreqMap): HTree {
    val sortedFreqMap = freqMap.toList().sortedBy { it.second }
    val trees: MutableList<HTree> = sortedFreqMap.map { Leaf(it.second, it.first) }.toMutableList()

    while (trees.size > 1) {
        val first = trees.removeFirst()
        val second = trees.removeFirst()

        trees.add(merge(first, second))
        trees.sortBy { it.weight }
    }

    return trees[0]
}
```
Algoritme: Maak een boom structuur die elementen met een hoge weight (frequentie) bovenin plaatst. Op deze manier krijgen veel voorkomende karakters een korter pad en daarmee een kortere binaire code.

Kotlin komt krachtig uit de hoek met de HTree definitie, die veel compacter is dan java, maar zeker niet zo fraai als haskell's `union` (vgl rust enums!). In plaats van recursie, gebruiken we een MutableList die we _in-place_ muteren. Het leidt tot dezelfde uitkomst als de haskell code. 

Heel jammer vind ik dat we de lijst handmatig moeten sorteren, en geen gebruik kunnen maken van een SortedSet. Hoewel alle HTree instanties als data classes een correcte _equals_ methode hebben, ziet de Set ze als _duplicates_, als de comparator op gewicht leidt tot gelijke gewichten. Deze manier performt ook slechter, omdat het op volgorde toevoegen aan een lijst goedkoper is dan steeds de hele lijst sorteren. Misschien is hier nog verbetering (van deze code) mogelijk?

### 3. binaire codes toekennen

We moeten voor elke leaf node het unieke pad aflopen. We doen dit door elk pad af te lopen en per node een lijst met 0/1 bij te houden.
Elke keer als we links gaan, voegen we een 0 toe en voor rechts 1. Voor elke fork maken we een kopie van de lijst (met elementen 0/1) en zetten er 0 of 1 achter. 


_haskell_
```haskell
data Bit = One | Zero
  deriving Show
type Code = [Bit]
type CodeMap = Map Char Code

buildCodes :: HTree -> CodeMap
buildCodes = Map.fromList . go []
  where 
  go :: Code -> HTree -> [(Char, Code)]
  go prefix tree = case tree of
    Leaf _ char -> [(char, reverse prefix)]
    Fork _ left right -> 
      go (One : prefix) left ++
      go (Zero: prefix) right
```

Voor kotlin heb ik de code aangepast (nieuw element achteraan en geen _reverse_ bij de leaf node). Ik weet niet waarom de haskell code dit anders doet.

_kotlin_
```kotlin
enum class Bit(val int: Int) {
    Zero(0),
    One(1)
}
typealias Code = List<Bit>
typealias CodeMap = Map<Char, Code>

fun buildCodes(hTree: HTree, prefix: Code = listOf()): CodeMap {
    return when (hTree) {
        is Leaf -> mapOf(hTree.char to prefix)
        is Fork -> {
            buildCodes(hTree.left,  prefix.toList() + listOf(Zero)) +
            buildCodes(hTree.right,  prefix.toList() + listOf(One))
        }
    }
}
```
Deze code is verder haast identiek. Het algoritme kent binaire codes toe aan de _chars_ op de manier die we willen. 
De Haskell code is recursief zowel in de aanroep als in de declaratie van de `go` functie. Kotlin heeft deze geneste functie niet nodig. De plus (+) operator op `List` is handig, maar maakt wel dat we Zero en One eerst in een lijst moeten zetten.

De Bit enum is niet echt nodig (een Int volstaat, evt met controle of het 0/1 is, óf je gebruikt de enum _ordinal_), maar had ik toegevoegd, om het te laten lijken op de haskell code.

### Test
Als we de volgende `main` toevoegen, kunnen we vaststellen dat de uitkomst identiek is met die van het genoemde artikel.


```kotlin
fun main() {
    val string = "Try it out with your own content."

    val freqMap = countFrequency(string)
    val tree = buildTree(freqMap)
    val codemap = buildCodes(tree)
    println(codemap.map { "'${it.key}'" to it.value.map { it.int } })
}
```

```
[
('n', [0, 0, 0]), 
('.', [0, 0, 1, 0]), 
('r', [0, 0, 1, 1]), 
('o', [0, 1, 0]), 
('y', [0, 1, 1, 0]), 
('i', [0, 1, 1, 1]), 
('u', [1, 0, 0, 0]), 
('w', [1, 0, 0, 1]), 
('T', [1, 0, 1, 0, 0]), 
('h', [1, 0, 1, 0, 1]), 
('c', [1, 0, 1, 1, 0]), 
('e', [1, 0, 1, 1, 1]), 
('t', [1, 1, 0]), 
(' ', [1, 1, 1])
]
```

### Conclusie
Qua expressieve kracht kan kotlin de vergelijking met haskell prima aan. Qua leesbaarheid is het sterker, maar dat is altijd ook een kwestie van kennisniveau en smaak.

De manier waarop is vaak vergelijkbaar maar vaak ook anders. Object inheritance in plaats van algebraic datatypes, bijvoorbeeld. De _buildTree_ in kotlin is minder functioneel en behoorlijk imperatief vanwege de while-loop.

Tot slot: ik hoop dat union types (of rust enums) in alle talen worden toegevoegd!


<div style="text-align: right">∞</div>