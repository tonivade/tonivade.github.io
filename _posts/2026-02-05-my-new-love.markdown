---
layout: post
title:  My new love
date:   2026-02-05 18:00:00
categories: programming java clojure scala lisp
---

Llevo un tiempo trabajando en mi propia implementación de [Make a Lisp](https://github.com/kanaka/mal). 
La idea era usar Java 25 sacandole partido a todas las novedades del lenguage. He usado sealed interfaces, 
pattern matching, switch mejorados, virtual threads... Y la verdad es que me lo estoy pasando pipa.
Aquí está el [enlace](https://github.com/tonivade/mal/tree/impl/java25/impls/java25) al proyecto.

Siempre me ha interesado mucho el mundo de los compiladores e interpretes, y este proyecto te permite implementar
un interprete sencillo de Lisp. Te guía paso a paso, añadiendo nuevas funcionalidades poco a poco.
Hay un set de tests que te ayudan a ir verificando que todo funciona como se espera. Y cómo último paso, 
ejecuta una implementación del interprete, implementada a su vez en el mismo lenguage, dentro de tu 
propio interprete. Una cosa así como en la película de Origen, pero con interpretes. Mola mucho.

Esto me ha llevado a experimentar con Lisp y he descubierto que a pesar de que al principio me parecía
un lenguaje marciano, ahora me resulta super divertido programar en este lenguaje. La sensación es
como si estuvieras juntando piezas. Una encaja con la siguiente y a su vez con la siguiente,
la verdad es que resulta muy entretenido. Otra cosa sorprendente es que a pesar de ser un lenguaje
muy pequeño, con muy pocas directivas básicas, se pueden hacer muchísimas cosas.

De tal modo que la última edición de [Advent Of Code](https://adventofcode.com/2025/) la he implementado 
en el interprete que he desarrollado. Ha sido muy divertido y he aprendido un montón de cosas nuevas.
Aquí están las [soluciones](https://github.com/tonivade/adventofcode-2025/tree/main/mal) por si alguien tiene interes.

Y finalmente, esto me ha llevado a [Clojure](https://clojure.org/). Para los despistados es una versión de Lisp pero en Java. 
Tiene completa interoperabilidad con Java por lo que puedes llamar a métodos de Java desde Clojure.

Asi que me temo que Scala se cae del podio de mis lenguajes favoritos y asciende a la primera posición Clojure.

Scala parece que está en claro declive. La versión 3 prometía mucho pero en lugar de 
subir en popularidad, año a año cae un poco más. Scala es un lenguaje magnífico, pero me temo que
está encaminado a dejar de ser relevante. Clojure tampoco es que sea super popular, pero es lo
que pasa cuando te enamoras, no antiendes a razones.

Me acaba de llegar un libro de Clojure que compré, y mi intención es darle caña durante los próximos días.