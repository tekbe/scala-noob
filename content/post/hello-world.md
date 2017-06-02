+++
date = "2017-06-02T16:11:20+02:00"
subtitle = ""
tags = []
title = "hello world"
draft = true
+++

first hugo page, yeah!

### some scala code here:

{{< highlight scala >}}
val data = Map("a"->"Null", "b"->"12", "c"->"23", "d"->"", "e"->"apple", "f"->"pear", "g"->"banana", "h"->null)

val replaced = data.map {
  case (k@("a"|"d"|"h"), v) => (k, "0.0")
  case x => x
}
{{< /highlight >}}

