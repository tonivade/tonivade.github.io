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

Empezaremos por definir una nueva extensión para `Program` que vamos a llamar `Memoized`:

```java
sealed interface Program<S, E, T> {

  record Success<S, E, T>(T value) implements Program<S, E, T> {}

  record Failure<S, E, T>(E error) implements Program<S, E, T> {}

  record FlatMap<S, E, X, T>(Program<S, E, X> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record FlatMapError<S, X, E, T>(Program<S, X, T> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record Access<S, E, T>(Function<S, Program<S, E, T>> mapper) implements Program<S, E, T> {}

  record Memoized<S, E, T>(Program<S, E, T> program) implements Program<S, E, T> {}
}
```

Esto hace que tengamos que adaptar el método `eval`. Primero necesitaremos una estructura
de datos para guardar los resultados cacheados.

```java
Map<Program<S, ?, ?>, Either<?, ?>> cache = new IdentityHashMap<>();
```

Vamos a usar [IdentityHashMap](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/IdentityHashMap.html) 
que es un caso un tanto especial de un `HashMap`. Esta implementación
no cumple con el contrato de `equals` y `hashCode`, y basa su implementación en `equals`. Para
nuestro caso funcionará perfectamente.

Como el resultar de evaluar un programa puede ir bien o mal, necesitaremos guardar en la cache
objetos de tipo `Either`.

Ahora necesitamos soportar el caso de la clase `Memoized`. Lo primero que tenemos que hacer
es comprobar si está en la caché. En caso de que ya exista en la caché, actualizamos el valor 
de `current` y de esa manera en la siguiente iteración se evaluará. Si no está en la cache
la cosa se complica, ya que en todavía no se ha evaluado todavía el programa, por lo tanto
cómo gestionamos esto?

```java
  // ...
  } else if (current instanceof Memoized(var next)) {
    if (cache.containsKey(next)) {
      current = cache.get(next).fold(Program::success, Program::failure);
    } else {
      // TBC
    }
  }
  //...
```

Podemos aprovechar el stack para de esta manera cuando el programa que queremos
cachear se ejecute, de esa manera actualizamos la cache. Tendremos dos casos
cuando todo va bien y cuando algo va mal. Y por último actualizamos `current` 
para que se evalúe en la siguiente iteración del bucle, y en ese momento,
actualizar la caché. 

```java
  // ...
  } else if (current instanceof Memoized(var next)) {
    if (cache.containsKey(next)) {
      current = cache.get(next).fold(Program::failure, Program::success);
    } else {
      successStack.push(value -> {
        cache.put(next, Either.right(value));
        return success(value);
      });
      failureStack.push(error -> {
        cache.put(next, Either.left(error));
        return failure(error);
      });
      current = next;
    }
  }
  //...
```

Volvamos entonces a nuestro programa, necesitaremos otra cache para en este caso
generar la misma instancia de `Program` para cada valor de `n`.

```java
static final Map<Integer, Program<Void, Void, Integer>> cache = new HashMap<>();
```

Definimos otra función que use la caché. Usaremos el método `computeIfAbsent`, lo
que significa que si no existe en la cache, se ejecutará la lambda que se pasa por
parámetro. En nuestro caso, llamará al método `fib` y lo wrapeará en un Memoized.
De esta forma este programa se ejecutará solo una vez, no importa las veces que se
llame.

```java
static Program<Void, Void, Integer> fibMemoized(int n) {
  return cache.computeIfAbsent(n, _ -> new Memoized<>(fib(n)));
}
```

Ahora `fib` cambia levemente ya que tiene que llamar a `fibMemoized` para que
obtenga una instancia del programa memoizado.

```java
static Program<Void, Void, Integer> fib(int n) {
  if (n < 2) {
    return success(1);
  }
  var fib2 = suspend(() -> fibMemoized(n - 2));
  var fib1 = suspend(() -> fibMemoized(n - 1));
  return zip(fib2, fib1, Integer::sum);
}
```

De esta forma tenemos nuestro programa que calcula la secuencia de Fibonacci
computacionalmente eficiente, y usando la notación Big O, pasaríamos de un
`O(2^n)` a `O(n)`, es decir que es linealmente proporcional al valor de `n`
lo cual es mucho más óptimo.

Ahora la cuestión, ¿cómo podemos generalizar esto? Necesitaremos una función
que use una hashMap para cachear los resultados:

```java
static <S, E, T, R> Function<T, Program<S, E, R>> memoize(Function<T, Program<S, E, R>> program) {
  Map<T, Program<S, E, R>> cache = new HashMap<>();
  return t -> cache.computeIfAbsent(t, program.andThen(Memoized::new));
}
```

Aplicándolo a nuestro ejemplo tendríamos esto:

```java
static Function<Integer, Program<Void, Void, Integer>> fibMemoized = Program.memoize(n -> {
  if (n < 2) {
    return success(1);
  }
  var fib2 = suspend(() -> fibMemoized.apply(n - 2));
  var fib1 = suspend(() -> fibMemoized.apply(n - 1));
  return zip(fib2, fib1, Integer::sum);
});
```

Esto tiene un aspecto estupendo, pero lamentablemente esto no funciona en Java. El
compilador se queja de que el valor de `fibMemoized` no está definido por lo que no
podemos volver a llamar a `fibMemoized` desde la definición de `fibMemoized`.

Una pena, pero hay una forma muy sencilla de arreglarlo, si en lugar de una lambda
creamos una clase anónima que implemente `Function` todo vuelve a funcionar:

```java
static Function<Integer, Program<Void, Void, Integer>> fibMemoized = Program.memoize(
    new Function<Integer, Program<Void, Void, Integer>>() {
      @Override
      public Program<Void, Void, Integer> apply(Integer n) {
        if (n < 2) {
          return success(1);
        }
        var fib2 = suspend(() -> fibMemoized.apply(n - 2));
        var fib1 = suspend(() -> fibMemoized.apply(n - 1));
        return zip(fib2, fib1, Integer::sum);
      }
    });
```

Es un poco más verboso, pero a mi me vale.