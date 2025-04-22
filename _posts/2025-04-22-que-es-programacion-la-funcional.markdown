---
layout: post
title:  "¿Qué es la programación funcional?"
date:   2025-04-22 22:00:00
categories: commitconf java programming
---

En la última edición de la #commitconf tuve la oportunidad de dar una charla. Estaba centrada en cómo utilizar las últimas novedades del lenguaje Java para desarrollar DSLs. Mi intención inicial era simplemente ver cómo sacar partido de estas técnicas para hacer el código mucho más expresivo, pero mientras preparaba la charla me topé con algo que ciertamente fue una sorpresa para mi y que me pareció super interesante.

Intentando desarrollar un DSL sencillo para hacer programas que escribieran y leyeran desde consola llegué como resultado a implementar una mónada. Es algo que surge de manera natural, ya que una mónada nos permite exactamente eso, secuenciar operaciones que dependen una de otra y componerlas de manera que realicen algo útil, es decir, una lógica de negocio.

¿Por qué creo que es interesante? En los últimos años me he interesado bastante por la programación funcional, pero siempre me topaba con la misma barrera: la cantidad de conceptos que en mi humilde entendimiento eran completamente incompresibles. Palabras exóticas como functor, mónada, transformación natural, tipos de órden superior, etc... hacían que todo pareciera mucho más complejo de lo que podría ser necesario. No digo que todo eso no sirva, estoy seguro de que si dominas todo eso, el resultado debe ser espectacular, pero pensando en el programador medio que ha programado toda su vida en algún lenguaje orientado a objetos (como yo), resulta intimidante. Así que ver que era posible implementar una mónada de manera natural sin necesidad de utilizar ningún tipo de concepto extraño es algo que me llenó de excitación, y pensé que era algo que tenía que compartir.

Y es que la programación funcional, en mi experiencia, es una herramienta súper potente. Te obliga a razonar sobre lo que estás haciendo. No basta con tirar líneas de código una detrás de otra sin ton ni son, con la programación funcional tienes que pensar cuidadosamente qué es lo que estás haciendo, cómo lo estás haciendo, cómo vas a probar que tu programa es correcto, definir unas operaciones básicas para, haciendo uso de ellas, crear programas más complejos. Y es algo que creo es muy valioso aprender.

Puedo entender que tal vez defraudé un poco las expectativas que pudieron hacerse los asistentes a mi charla, pero creo que también es comprensible que durante el proceso de preparación las cosas fluyan de una manera diferente a la que pensaste inicialmente. Espero que haya merecido la pena.
