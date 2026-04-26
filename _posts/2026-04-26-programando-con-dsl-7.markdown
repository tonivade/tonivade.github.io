---
layout: post
title:  Programando con DSLs en Java 25 (VII)
date:   2026-04-26 14:00:00
categories: programming java functional-programming dsl monad
---

Como ya comenté en el último artículo de esta serie, aquí estoy otra vez dando 
la lata con el mismo tema. Pero es que no lo puedo evitar y una cosa lleva a
la otra, y no puedo evitarlo. Mientras siga divirtiéndome, seguiré escribiendo 
sobre esto.

En el último artículo hablamos de como tratar la recursividad en nuestro mini
lenguaje, y llevamos a la conclusión de que si eval se implementaba de manera
iterativa podríamos escribir programas recursivos y que estos no tuvieran
problemas de desbordamiento de pila.

Para refrescar la memoria, aquí está la implementación de `eval` que desarrollamos:

```java
default Either<E, T> eval(S state) {
  Program<S, ?, ?> current = this;
  Deque<Function<Object, ? extends Program<S, ?, ?>>> failureStack = new ArrayDeque<>();
  Deque<Function<Object, ? extends Program<S, ?, ?>>> successStack = new ArrayDeque<>();

  while (true) {
    if (current instanceof Success(var value)) {
      if (successStack.isEmpty()) {
        return Either.right((T) value);
      }
      current = successStack.poll().apply(value);
    } else if (current instanceof Failure(var error)) {
      if (failureStack.isEmpty()) {
        return Either.left((E) error);
      }
      current = failureStack.poll().apply(error);
    } else if (current instanceof Access(var mapper)) {
      current = mapper.apply(state);
    } else if (current instanceof FlatMap(var next, var mapper)) {
      successStack.push((Function<Object, ? extends Program<S, ?, ?>>) mapper);
      current = next;
    } else if (current instanceof FlatMapError(var next, var mapper)) {
      failureStack.push((Function<Object, ? extends Program<S, ?, ?>>) mapper);
      current = next;
    }
  }
}
```

Y para demostrar que funcionaba pusimos dos ejemplos, un pequeño programa
que sumaba todos los número consecutivos de 1 hasta n:

```java
static Program<Void, Void, Integer> recursiveSum(int n, int sum) {
  if (n == 0) {
    return success(sum);
  }
  return suspend(() -> recursiveSum(n - 1, n + sum));
}
```

Y otro clásico, un programa que calculaba la secuencia de Fibonacci:

```java
static Program<Void, Void, Integer> fib(int n) {
  if (n < 2) {
    return success(1);
  }
  var fib2 = suspend(() -> fib(n - 2));
  var fib1 = suspend(() -> fib(n - 1));
  return zip(fib2, fib1, Integer::sum);
}
```

El primer ejemplo es un programa muy sencillo que se ejecuta de manera rápida,
cuanto más grande sea el valor de `n` más tarda, pero dentro de lo asumible.

El segundo ejemplo es un programa más complejo ya que calcula el valor para
`n - 2` y luego de el de `n - 1`. Y así sucesivamente, lo que significa que
vamos aculando una y otra vez las mismas operaciones. Por ejemplo, para `n = 10`
Se calculará lo siguiente:

- fib(8) y fib(9)
- fib(6), fib(7), fib(7) y fib(8)
- fib(4), fib(5), fib(5), fib(6), fib(5), fib(6), fib(6) y fib(7)
- fib(2), fib(3), fib(3), fib(4), fib(3), fib(4), fib(4), fib(5), fib(3), fib(4), fib(4), fib(5), fib(4), fib(5), fib(5) y fib(6)
...

No sigo porque la cosa se pone complicada. Como se ve cada vez se multiplican por 2
el número de operaciones, y el número de operaciones es proporcional al número
de la secuencia de Fibonacci que queremos calcular por lo tanto si queremos calcular
el valor cuando `n = 10` el número de operaciones necesarias serán `2^10 = 1024`.
Para `n = 20` serán `1048576` operaciones.

Digamos que este algoritmo para calcular Fibonacci no es eficiente. Usando notación
estilo [Big O](https://en.wikipedia.org/wiki/Big_O_notation) sería `O(2^n)`. En
ciencia de la computación esto nos viene a decir cuan eficiente es un algoritmo.
Por ejemplo uno que fuera `O(1)` significaría que el algoritmo se ejecuta en un tiempo
siempre constante. No importa que valor tenga `n` el algoritmo siempre tarda lo mismo.

Otros valores usuales en orden de mejor a mayor:

- `O(log n)`: orden logarítmica.
- `O(n)`: orden lineal.
- `O(n log n)`: orden lineal logarítmica.
- `O(n^2)`: orden cuadrática.
- `O(n^3)`: orden cúbica.
- `O(n!)`: orden factorial.
- `O(n^n)`: orden potencia exponencial.

Ahora bien, cómo podríamos mantener nuestro programa recursivo y hacerlo a su vez
computacionalmente eficiente. La respuesta es una técnica que se llama [memoization](https://en.wikipedia.org/wiki/Memoization)
pero básicamente se trata de cachear las respuesta de una función.




