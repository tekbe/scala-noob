+++
date = "2017-06-05T21:10:57+02:00"
subtitle = ""
tags = []
title = "Monaden, flatMap() und die Bedeutung von for"
draft = true
+++

Was ist eine *Monade*? Diese Frage stellt man sich früher oder später bei der Beschäftigung mit Funktionaler Programmierung. Und hier ist die 1001. Antwort darauf! (Weitere Antworten sind am Ende des Artikels verlinkt.)

Um mit konkreten Beispielen anzufangen, könnte man sagen, eine Monade ist das, was den Typen `Option[T]`, `Future[T]` und `Stream[T]` gemeinsam ist.

Sie sind recht geläufige Exemplare (zur Erinnerung s. [hier](https://www.tutorialspoint.com/scala/scala_options.htm), [hier](http://docs.scala-lang.org/overviews/core/futures.html#futures) und [hier](http://www.mrico.eu/entry/scala_streams)) mit intuitiv klaren, expressiven und vorallem sehr unterschiedlichen Konzepten oder Effekten. Wir wollen sehen, wie sich vor dem Hintergrund dieser heterogenen Typen die Monade abzeichnet. 

Als erste Gemeinsamkeit kann man davon sprechen, dass alle diese Beispiele einen typisierten Berechnungskontext bereitstellen. Die Ergebnisse dieser Berechnungen liegen nicht unbedingt vor: vielleich enthält `Option[T]` keinen Wert, dauert die Berechnung im `Future[T]` noch an, oder ist der `Stream[T]` unendlich lang und produziert auf Anfrage immer weiter Daten.

## Einen Kontext erzeugen

Jede Monade ist in der Lage, ihren spezifischen Kontext zu erzeugen, was zumeist über die `apply` Methode des *Companion Object* geschieht.

~~~scala
val f = Future(someLongLastingComputation())
val opt = Option(thisCouldBeNull())
~~~

Im Falle von `Stream[T]` wird die `apply` Methode eher nicht verwendet. Zwar kann man einen `Stream[T]` um eine gegebene Sequenz von Werten herum konstruieren, im Allgemeinen sind Streams aber *lazy* und *unbegrenzt*, d.h. potentiell unendlich viele Werte werden jeweils erst bei Bedarf bereitgestellt. Man definiert deshalb bei der Konstruktion von Streams nur,
~~~scala
// unendlicher Strom von Zufallszahlen
def randomGenerator(): Stream[Double] = math.random #:: randomGenerator()
~~~

wie der jeweils nächste Wert erzeugt wird.

## Werte aus dem Kontext herausholen

Aus Sicht eines Entwickles, der sich nicht für den Kontext interessiert und &bdquo;einfach nur&ldquo; mit den Werten arbeiten möchte, erscheint der Kontext vielleicht als eine Art Container, der die gewünschten Werte enthält.

Allerdings sind die Wege zum &bdquo;Herausholen&ldquo; der Werte vielfältig und von Kontext zu Kontext unterschiedlich.

Egal, ob man den Kontext einfach zerstört,

~~~scala
// was, wenn die Berechnung länger dauert?
val x = Await.result(future, 1 second)
// was, wenn der Stream unendlich lang ist?
val y = stream.toVector
// was, wenn es den Wert nicht gibt?
val z = option.get
~~~

oder ihn auf sicherere Weise abstreift

~~~scala
future.onComplete {
  case Success(x) => handle(x)
  case Failure(t) => t.printStackTrace()
}

val y = stream.headOption
val z = option.getOrElse("default")
~~~

In jedem Fall ist gerade dann, wenn man den Kontext eliminiert, weil man sich nur für die enthaltenen Werte interessiert, die Auseinandersetzung mit dem Kontext und seinen möglichen Fehlerfällen unumgänglich.

Nun wäre es sehr umständlich, für jede Art von Kontext, in dem bestimmte Typen vorkommen, entsprechende Funktionen bereitzustellen.

~~~scala
def length(s: Future[String]): Int
def length(s: Option[String]): Int
def length(s: Stream[String]): Vector[Int]
~~~

Im Allgemeinen wäre es sogar wünschenswert, den Berechnungskontext intakt zu lassen. Man möchte ja gerade nicht auf die Berechnung eines `Future[T]` warten oder alle Werte eines `Stream[T]` materialisieren (z.B. aus einer Datei laden).

~~~scala
def length(s: Option[String]): Option[String]
def length(s: Future[String]): Future[Int]
def length(s: Stream[String]): Stream[Int]
~~~

Stattdessen reicht es (unabhängig davon, ob Werte vorliegen oder nicht) nur zu *beschreiben*, was mit ihnen geschehen soll, ohne diese Berechnung auch direkt auszuführen bzw. zu *evaluieren*.

## Funktionen in den Kontext heben

Wie wäre es also, wenn wir anstatt Werte aus dem Kontext zu holen, eine von der Art der Monade unabhängige, aber zu ihrem Typparameter passende, Funktion 

~~~scala
def length(s: String): Int
~~~

in den Kontext &bdquo;hinein heben&ldquo;? Genau das ist die erste wesentliche Eigenschaft von Monaden. Sie erlauben das *liften* von Funktionen in ihren Kontext über ihre `map` Methode und damit die Modifikation der gekapselten Berechnung.

~~~scala
trait Future[A] {
  def map[B](f: A => B): Future[B]
}

val f1: Future[String] = someAsyncComputation()
// die Berechnung von f2 beginnt, sobald die von f1 abgeschlossen ist
val f2: Future[Int] = f1.map(s => length(s))
~~~

Die Funktion `length` lässt sich unverändert auch in den Kontext von `Option[String]` oder `Stream[String]` (oder jeder anderen Monade mit Typparameter `String`) heben.

Gerade wenn man sich nicht für ihn interessiert, ist es pragmatischer, vom Kontext zu abstrahieren, als ihn zu beenden. Ob und wann eine Berechnung evaluiert wird, bleibt so die Sache der Monade.

Darüber hinaus bietet der  Kontext der Monade einen echten Mehrwert:

~~~scala
// Strom von Zufallszahlen
def randomGenerator(): Stream[Double] = math.random #:: randomGenerator()
// Zufallszahl als Würfelwurf
def diceThrow(random: Double): Int = (random * 6).toInt + 1
// Zwei unabhängige Würfel
val dice1: Stream[Int] = randomGenerator().map(r => diceThrow(r))
val dice2: Stream[Int] = randomGenerator().map(r => diceThrow(r))
~~~ 

Beide Würfel können unbeschränkt oft geworfen werden und merken sich jeweils die Ergebnisse der Würfe (*memoization*) &ndash; beides Eigenschaften des Streamkontexts!

## Gleichartige Kontexte verbinden

Das im vorherigen Abschnitt beschriebene Merkmal, eine Funktion in einen Kontext heben zu können, ist für sich genommen das eines *Funktors*. Während jede Monade ein Funktor ist, muss zum Funktor aber noch eine Eigenschaft hinzukommen, um Monade genannt zu werden.

Manchmal möchte man Berechnungen beschreiben, die sich über mehrere Kontexte erstrecken. 

~~~scala
def max(f1: Future[Int], f2: Future[Int]): Future[Int]
~~~

Man würde hier erwarten, dass der größere der beiden Werte ermittelt wird, sobald die Berechnungen der beiden Parameter abgeschlossen ist.

Auch hier will man sich nicht um die Eigenheit der Monade kümmern, sondern eine allgemeine, von der Monade unabhängige Funktion wiederverwenden.

~~~scala
def max(i1: Int, i2: Int): Int = if (i1 > i2) i1 else i2
~~~

Es sollte also möglich sein, Funktionen über Typparameter gleichartiger Monaden in einen Verbund ihrer Kontexte zu heben. Und tatsächlich ist die Kombinierbarkeit gleichartiger Kontexte das Merkmal, welches Monaden von einfachen Funktoren unterscheidet.

Die Art und Weise, wie das geschieht, mag etwas kompliziert erscheinen. Denn zunächst wird die eine Monade mittels `map` in den Kontext der anderen geliftet. Anschließend werden die nun geschachtelten Kontexte mit `flatten` eingeebnet.

~~~scala
val f1: Future[Int]
val f2: Future[Int]

val r1: Future[Future[(Int,Int)]] = f1.map(v1 => f2.map(v2 => (v1, v2)))
val r2: Future[(Int, Int)] = r1.flatten
val r3: Future[Int] = r2.map(v => max(v._1, v._2))
~~~

Man kann sich kürzer fassen, indem man das Liften der Monade mit dem Liften der Funktion `max` kombiniert.

~~~scala
val r1: Future[Future[Int]] = f1.map(v1 => f2.map(v2 => max(v1, v2)))
val r2: Future[Int] = r1.flatten
~~~

Das Liften einer Monade in eine andere und das Einebnen der geschachtelten Kontexte in einem Schritt erreicht man mit `flatMap` &ndash; einer Kombination aus `map` mit anschließendem `flatten`.

~~~scala
trait Future[A] {
  def flatMap[B](f: A => Future[B]): Future[B] = map(f).flatten
}
~~~

Das Liften einer Funktion in den kombinierten Kontext gleichartiger Monaden lässt sich nun also in einen einzigen Ausdruck fassen.

~~~scala
val r: Future[Int] = f1.flatMap(v1 => f2.map(v2 => max(v1, v2)))
~~~

Leider passt das nicht recht zu unserem intuitiven Verständnis. Zu sehr tritt die Schachtelung in den Vordergrund, die doch nur ein Mittel zum Verbinden der Monaden sein sollte.

Man kann sich vorstellen, wie schnell es unübersichtlich wird, wenn mehr als zwei Monaden im Spiel sind.

~~~scala
def max(is: Int*): Int

m1.flatMap(v1 => m2.flatMap(v2 => m3.flatMap(v3 => m4.map(v4 =>
    max(v1, v2, v3, v4)
))))
~~~

Auf jeder Ebene muss man &bdquo;`flatten`&ldquo;. Nur die innerste Monade ist nicht geschachtelt, hier reicht `map`.

## Alternative Schreibweise mit for

Man würde wohl lieber etwas schreiben wie 
~~~scala
join(m1, m2).map(v => max(v._1, v._2))
~~~
Monaden verbinden, Funktion in den resultierenden Kontext liften, fertig. Nun gibt es keine `join` Funktion in der Standardbibliothek, aber wie wäre es stattdessen mit

~~~scala
for {
  v1 <- m1
  v2 <- m2
} yield max(v1, v2)
~~~

Die geschweiften Klammern markieren den Verbund gleichartiger Monaden und nach `yield` folgt die in den resultierenden Kontext zu liftende Funktion. Das Ergebnis ist wieder von gleicher Art (also ein `Option[T]` oder `Future[T]` oder ...). 

Tatsächlich ist diese Schreibweise äquivalent zur Schachtelung von `flatMap` und `map` aus dem vorherigen Abschnitt. Mehr noch, der Scala Compiler übersetzt einen solchen `for`-Ausdruck sogar wortwörtlich in eben jenen geschachtelten Ausdruck! `yield ...` entspricht dabei dem innersten `map(...)`.

Bei der sog. *for comprehension* handelt es sich also keinesfalls um eine Schleife. Nur in Verbindung mit Monaden wie `List[T]` oder `Vector[T]` erinnert das Ergebnis als Spezialfall an etwas, für dessen Erzeugung man in C-ähnlichen Sprachen üblicherweise for-Schleifen verwendet.

~~~scala
scala> for {
     |   i <- 1 to 3
     |   j <- 1 to 3
     | } yield (i,j)

res0 = Vector((1,1), (1,2), (1,3), (2,1), (2,2), (2,3), (3,1), (3,2), (3,3))
~~~


## Suchen und Filtern und Monaden mit Null 

Scalas for comprehension geht noch über die bisher vorgestellte Struktur der Monade hinaus, indem es erlaubt, bestimmte Ergebnisse zu filtern.

~~~scala
scala> for {
     |   i <- 1 to 3
     |   j <- 1 to 3
     |   if i < j
     | } yield (i,j)

res1 = Vector((1,2), (1,3), (2,3))
~~~ 

Das wird übersetzt zu

~~~scala
(1 to 3).flatMap(i => (1 to 3).withFilter(j => j < i).map(j => (i,j)))
~~~

Nun sind *Filter* im Allgemeinen keine Eigenschaft von Monaden, aber es fällt auf, wie gut sie sich in dieses System fügen. Häufig ergibt es ja Sinn, wenn eine Monade in der Lage ist, das Fehlen, Ausbleiben oder Auslassen von Ergebnissen auszudrücken. Tatsächlich handelt es sich dann um Monaden mit einem *Nullobjekt* (für `Option[T]` wäre das beispielweise `None`, für `Stream[T]` ist es `Empty`).

Mit dem Filter `if false` bekommt man entsprechend stets das Nullelement der Monade zum Ergebnis. Umgekehrt gilt: egal welche Funktion man in das Nullelement einer Monade liftet, mit welchem anderen Kontext man es verbindet oder mit welchem Filter man es durchsucht, das Ergebnis ist stets wieder das Nullelement.

## Zwischenfazit

Als vorläufige Antwort auf die Frage vom Anfang, können wir jetzt sagen: Monaden kapseln eine bestimmte Struktur oder einen bestimmten Nebeneffekt (rekursive Datenstrukturen, I/O-Operationen, Datenströme, Parallelisierung, Ausnahmebehandlung usw.) und stellen zugleich einen Kontext bereit, der es ermöglicht, auf gleichartige Weise die aus der Struktur oder dem Effekt hervorgehenden typisierten Daten zu transformieren und zu durchsuchen, sowie mehrere Instanzen solcher Kontexte bedeutsam miteinander zu verbinden. In Scala gibt es dazu die Methoden `map`, `flatMap` und `withFilter`.

Ein weiteres anschauliches Beispiel für Monaden, diesmal nicht aus der Standardbibliothek, sind Datenbankabfragen in [Slick](http://slick.lightbend.com/).

~~~scala
val monadicInnerJoin = for {
  c <- coffees
  s <- suppliers if c.supplierId === s.id
} yield (c.name, s.name)
~~~

Das Codefragment ist der offiziellen Dokumentation entnommen. Es repräsentiert eine Datenbankabfrage in Scala Code. Das Verbinden von Monaden entspricht hier dem *join* zweier Datenbanktabellen.

Abstraktion und Pragmatismus sind hier keine Gegensätze. Selbst wer noch nie etwas von Monaden gehört hat, kann sie effektiv einsetzen. Dass sich in Scala ein Sprachkonstrukt eigens um Monaden dreht, mag ihren Stellenwert noch einmal unterstreichen. 

## Theorie

Da schon von Funktor und Nullelement die Rede war, ist es vielleicht nicht überraschend, dass Monaden eine mathematisch fundierte Struktur haben. Der Zweig der Mathematik, der sich u.a. mit Monaden beschäftigt ist die [Kategorientheorie](https://de.wikipedia.org/wiki/Kategorientheorie).

Was ist nun eine Kategorie? Um mit konkreten Beispielen anzufangen, könnte man sagen, eine Kategorie ist das, was Monaden (mit *bind* bzw. `flatMap`), Funktionen (mit *Komposition* bzw. `f(x) = g(h(x))`) und natürlichen Zahlen (mit Multiplikation, `a = b*c*d`) gemeinsam ist...

...nun, man kann (d.h. nicht ich, aber prinzipiell) hier tatsächlich rekursiv wieder einsteigen. Und das ist für Programmierer durchaus relevant. Man wird sich dann vielleicht sogar irgendwann, nämlich ein Rekursionsschritt weiter, fragen, was den mathematischen Teildisziplinen Berechenbarkeitstheorie, Logik und Kategorientheorie gemeinsam ist. Wer hier weitergehen möchte, dem sei der Kanal von [Bartosz Milewski](https://www.youtube.com/user/DrBartosz) ans Herz gelegt. Hier nur ein paar Grundbegriffe in aller Kürze: 

Kategorien beschreiben Abbildungen oder *Morphismen* zwischen Objekten. Diese Abbildungen lassen sich zu neuen Morphismen verketten. Solche *Kompositionen* sind assoziativ (`(a*b)*c = a*(b*c)`) und zu jedem Objekt gibt es einen identischen Morphismus, das *neutrale Element* oder *Einselement* (`a*1 = 1*a = a`).

Ein Objekt ist hierbei kein einzelnes Ding, sondern eine Klasse gleichartiger Strukturen (wobei die Strukturen selbst vernachlässigt werden). So hat z.B. die Kategorie natürliche Zahlen mit Multiplikation nur ein einziges Objekt, eben die natürlichen Zahlen: Jede Abbildung erfolgt *von* einer natürlichen Zahl *zu* einer natürlichen Zahl. Bei der Komposition von Funktionen, sind die Objekte die Datentypen, über die die Funktionen definiert sind. Und so wie eine Funktionskomposition `f(x) = g(h(x))` nur möglich ist, wenn der Ergebnistyp von `h` zum Eingabetyp von `g` passt, sind auch allgemein zwei Morphismen einer beliebigen Kategorie nur dann komponierbar, wenn das Endobjekt des einen mit dem Anfangsobjekt des anderen übereinstimmt. Das Anfangsobjekt einer Komposition `g•h` (&bdquo;`g` folgt auf `h`&ldquo;) entspricht dann demjenigen von `h`, das Endobjekt demjenigen von `g`.

Was eine Kategorie also ausmacht, sind, in einem Satz, 

- die Abbildungen bzw. Morphismen zwischen Objekten (Strukturen gleicher Art), 
- deren assoziative Komponierbarkeit, sowie 
- jeweils ein Begriff von Identität (ein neutraler Morphismus) für alle Objekte der Kategorie. 

Wir finden diese Aspekte bei Monaden wieder. Implementieren wir als Beispiel `Option[T]` (stark vereinfacht und zur Verdeutlichung mit `unit` und `zero`, für die es in der Standardbibliothek keine einheitlichen Bezeichnungen gibt).

~~~scala
object Option {
  // Konstruktion
  def unit[T]: T => Option[T] = x => Some(x)
  // Nullobjekt
  def zero[T]: Option[T] = None
}

case object None extends Option[Nothing]
case class Some[T](x: T) extends Option[T]

sealed trait Option[+T] {
  import Option._
  // Komposition
  def flatMap[U](f: T => Option[U]): Option[U] = this match {
    case None => this
    case Some(x) => f(x)
  }
  // Liften zurückgeführt auf flatMap und unit
  def map[U](f: T => U): Option[U] = flatMap(x => unit(f(x)))
  // Filter zurückgeführt auf flatMap, unit und zero
  def filter(p: T => Boolean): Option[T] = flatMap { x => 
    if (p(x)) unit(x) 
    else zero
  }
}
~~~

Wie man sieht, sind `map` und `filter` nicht wesentlich für Monaden, denn sie lassen sich auf die Implementierungen von `unit` (Wie werden Objekte der Kategorie konstruiert?), `flatMap` (Wie werden die Objekte aufeinander abgebildet?) und `zero` (Was ist das Nullobjekt?) zurückführen.

Instanzen von `Option[T]` mit übereinstimmenden Typparametern bilden nun die Objekte der Kategorie. Seien `X`, `Y` und `Z` drei beliebige Typen und 

~~~scala
val x: X
val m: Option[X]
def f: X => Option[Y]
def g: Y => Option[Z]
~~~

dann ist `m.flatMap(f)` ein Morphismus von `Option[X]` nach `Option[Y]` und `m.flatMap(f).flatMap(g)` die Komposition zweier Morphismen mit Anfangsobjekt `Option[X]` und Endobjekt `Option[Z]`. Desweiteren ist `m.flatMap(unit)` der *neutrale Morphismus* und `m.flatMap(_ => zero)` der *Nullmorphismus* (mit *Nullobjekt* `None`).


Es gelten nun die folgenden monadischen Gesetze:

~~~scala
// Assoziativität
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
// Identität / Neutralität
m.flatMap(unit) == m
unit(x).flatMap(f) == f(x)
m.filter(_ => true) == m
// Nullobjekt
zero.flatMap(f) == zero
m.flatMap(_ => zero) == zero
m.filter(_ => false) == zero
~~~

Davon abgeleitet gelten mit 

~~~scala
def h: X => Y
def i: Y => Z
~~~

auch diese Funktor-Gesetze:

~~~scala
// Assoziativität (Funktor)
m.map(h).map(i) == m.map(x => i(h(x)))
// Identität / Neutralität (Funktor)
m.map(x => x) == m
unit(x).map(h) == unit(h(x))
// Nullobjekt (Funktor)
zero.map(h) == zero
~~~

Damit ist `Option[T]` auch im streng mathematischen Sinn eine Monade mit Null. 

## Praxis

Die Identitätsregel des Funktors sorgt nun dafür, dass 

~~~scala
(for (x <- m) yield x) == m
~~~

gilt, denn `yield x` bedeutet nichts anderes als das Liften der Identitätsfunktion, hier also `m.map(x => x)`.

Die monadische Assoziativität bedeutet übersetzt auf die for-comprehension, dass man auf geschachtelte `for` Ausdrücke gleichartiger Monaden verzichten kann, denn

~~~scala
for {
  y <- for (x <- m; y <- f(x)) yield y
  z <- g(y)
} yield z
~~~

wird übersetzt zu 

~~~scala
m.flatMap(f).flatMap(g)
~~~

Was das Gleiche ist wie 

~~~scala
m.flatMap(x => f(x).flatMap(g))
~~~

oder in for-Schreibweise

~~~scala
for {
  x <- m
  y <- f(x)
  z <- g(y)
} yield z
~~~

## Der Typ `Monad[T]`

## Sich schließlich doch um den Kontext kümmern

## Referenzen
