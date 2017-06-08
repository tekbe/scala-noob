+++
date = "2017-06-05T21:10:57+02:00"
subtitle = ""
tags = []
title = "Monaden, flatMap() und die Bedeutung von for"
draft = true
+++
Was ist eigentlich eine Monade? Diese Frage stellt man sich früher oder später bei der Beschäftigung mit Scala. Eine Antwort darauf wäre, eine Monade ist das, was den Typen `Option[T]`, `Future[T]` und `Stream[T]` gemeinsam ist.

Diese Beispiele für Monaden sind nicht nur sehr geläufig, sie repräsentieren auch intuitiv klare, expressive und vorallem sehr unterschiedliche Konzepte. Wir wollen sehen, wie sich vor diesem heterogenen Hintergrund das gemeinsame Kontept der Monade abzeichnet. 

Als erste Gemeinsamkeit kann man davon sprechen, dass alle Beispiele einen typisierten Berechnungskontext bereitstellen.

## Werte aus dem Kontext herausholen

Aus Sicht eines Entwickles, dem der Kontext egal ist und der einfach nur mit den Werten arbeiten möchte, erscheint der Kontext vielleicht als eine Art Container, der die gewünschten Werte enthält.

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

Stattdessen reicht es meist &ndash; unabhängig davon, ob Werte vorliegen oder nicht &ndash; nur zu *beschreiben*, was mit ihnen geschehen soll, ohne diese Berechnung auch direkt auszuführen (zu *evaluieren*).     

## Funktionen in den Kontext heben

Wie wäre es also, wenn wir anstatt Werte aus dem Kontext zu holen, eine von der Art des Kontext unabhängige (aber zum Typparameter des Kontext passende) Funktion 

~~~scala
def length(s: String): Int
~~~

in den Kontext &bdquo;hinein heben&ldquo;? Genau das ist die erste wesentliche Eigenschaft von Monaden. Sie erlauben das *liften* von Funktionen in ihren Kontext über die Funktion `map` und damit die Modifikation der gekapselten Berechnung.

~~~scala
trait Future[A] {
  def map[B](f: A => B): Future[B]
}

def someAsyncComputation: Future[String]
val f: Future[Int] = someAsyncComputation.map(s => length(s))
~~~

Die Funktion `length` lässt sich unverändert in jede beliebige Monade heben, dessen Typparameter String ist. Es ist also bedeutend einfacher vom Kontext zu abstrahieren, als ihn zu beenden.
