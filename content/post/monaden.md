+++
date = "2017-06-05T21:10:57+02:00"
subtitle = ""
tags = []
title = "Monaden, flatMap() und die Bedeutung von for"
draft = true
+++
Was ist eine *Monade*? Diese Frage stellt man sich früher oder später bei der Beschäftigung mit Scala. Eine Antwort darauf wäre, eine Monade ist das, was den Typen `Option[T]`, `Future[T]` und `Stream[T]` gemeinsam ist.

Diese Beispiele für Monaden sind nicht nur sehr geläufig (zur Erinnerung s. [hier](https://www.tutorialspoint.com/scala/scala_options.htm), [hier](http://docs.scala-lang.org/overviews/core/futures.html#futures) und [hier](http://www.mrico.eu/entry/scala_streams)), sie repräsentieren auch intuitiv klare, expressive und vorallem sehr unterschiedliche Konzepte. Wir wollen sehen, wie sich vor dem Hintergrund dieser heterogenen Typen das gemeinsame Konzept der Monade abzeichnet. 

Als erste Gemeinsamkeit kann man davon sprechen, dass alle diese Beispiele einen typisierten Berechnungskontext bereitstellen. Die Ergebnisse dieser Berechnungen liegen nicht unbedingt vor: die `Option[T]` enthält vielleicht keinen Wert, die Berechnung im `Future[T]` dauert noch an, oder der `Stream[T]` ist unendlich lang und produziert auf Anfrage immer weiter Daten.

## Werte aus dem Kontext herausholen

Aus Sicht eines Entwickles, der sich nicht für den Kontext interessiert und &bdquo;einfach nur&ldquo; mit den Werten arbeiten möchte, erscheint der Kontext vielleicht als eine Art Container, der die gewünschten Werte enthält.

Allerdings sind die Wege zum &bdquo;Herausholen&ldquo; der Werte vielfältig und von Kontext zu Kontext unterschiedlich.

Egal, ob man den Kontext einfach zerstört,

~~~scala
// was, wenn die Berechnung länger dauert?
val x = Await.result(future, 1 second)
// was, wenn der Stream unendlich lang ist?
val y = stream.toVector(0)
// was, wenn es den Wert nicht gibt?
val z = option.get
~~~

oder ihn auf sicherere Weise abstreift

~~~scala
future.onComplete {
  case Success(x) => handle(x)
  case Failure(t) => t.printStackTrace()
}

val y = stream.head // was, wenn der Stream leer ist?
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

Stattdessen reicht es &ndash; unabhängig davon, ob Werte vorliegen oder nicht &ndash; nur zu *beschreiben*, was mit ihnen geschehen soll, ohne diese Berechnung auch direkt auszuführen (zu *evaluieren*).     

## Funktionen in den Kontext heben

Wie wäre es also, wenn wir anstatt Werte aus dem Kontext zu holen, eine von der Art des Kontext unabhängige (aber zum Typparameter des Kontext passende) Funktion 

~~~scala
def length(s: String): Int
~~~

in den Kontext &bdquo;hinein heben&ldquo;? Genau das ist die erste wesentliche Eigenschaft von Monaden. Sie erlauben das *liften* von Funktionen in ihren Kontext über ihre `map` Funktion und damit die Modifikation der gekapselten Berechnung.

~~~scala
trait Future[A] {
  def map[B](f: A => B): Future[B]
}

val f1: Future[String] = someAsyncComputation()
// f2 wird berechnet, sobald die Berechnung von f1 abgeschlossen ist
val f2: Future[Int] = f1.map(s => length(s))
~~~

Die Funktion `length` lässt sich unverändert auch in den Kontext von `Option[String]` oder `Stream[String]` (oder jeder anderen Monade mit Typparameter `String`) heben. 

Gerade wenn man sich nicht für ihn interessiert, ist es pragmatischer, vom Kontext zu abstrahieren, als ihn zu beenden. Ob und wann eine Berechnung evaluiert wird, bleibt so die Sache der Monade. 

## Gleichartige Kontexte verbinden

Das im vorherigen Abschnitt beschriebene Merkmal, eine Funktion in einen Kontext heben zu können, ist für sich genommen das eines *Funktors*. Während jede Monade ein Funktor ist, muss zum Funktor noch eine Eigenschaft hinzukommen, um Monade genannt zu werden.

Manchmal möchte man Berechnungen beschreiben, die sich über mehrere Kontexte erstrecken. 

~~~scala
def add(f1: Future[Int], f2: Future[Int]): Future[Int]
~~~

Man würde hier erwarten, dass die Addition ausgeführt wird, sobald die Berechnungen der beiden Parameter abgeschlossen ist.

~~~scala
def max(o1: Option[Int], o2: Option[Int]): Option[Int]
~~~

Wie ist es hier? Möchte man als Ergebnis `None`, wenn einer der beiden Parameter `None` ist, oder erwartet man dann den jeweils anderen Parameter als Ergebnis?

Geht man wieder von Funktionen aus, die nur über die Typparameter der Monade definiert sind,

~~~scala
def add(i1: Int, i2: Int) = i1 + i2
def max(i1: Int, i2: Int) = if (i1 > i2) i1 else i2
~~~

haben wir einen Hinweis auf die Anwort: die Funktionen lassen sich nur anwenden, wenn beide Parameter gegeben sind.



