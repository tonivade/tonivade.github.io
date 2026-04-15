---
layout: post
title:  Diversión con Transducers
date:   2026-03-08 13:00:00
categories: programming java clojure streams transducers
---

Llevo un tiempo trabajando con clojure, ya escribí otro articulo hablando de lo mucho que me ha
sorprendido este lenguaje, y me he encontrado con un concepto nuevo llamado Transducers. Estos son
una optimización cuando por ejemplo en clojure nos encontramos con algo como esto:

```clojure
user => (filter even? (map #(+ % 2) [1 2 3 4 5 6 7 8 9]))
(4 6 8 10)
```

También se puede escribir de esta otra manera, tal vez sea así más facil de ver:

```clojure
user => (->> [1 2 3 4 5 6 7 8 9]
  (map #(+ % 2))
  (filter even?))
(4 6 8 10)
```

Esto es un ejemplo trivial, pero sirve para ilustrar lo que quiero decir. Tenemos una lista, luego 
después de `map` se crea otra, y al ejecutar `filter` otra lista más. Es decir, se generan listas
intermedias con cada operación.

Esto cuando el número de elementos de la lista es pequeño pues no es relevante, pero según el número
de elementos aumenta, el rendimiento cae y cae.

En clojure tienen una cosa que se llaman [transducers](https://clojure.org/reference/transducers), 
son una función de orden superior, que reciben otra función y generan otra función combinando las dos 
operaciones. En clojure tenemos la función `comp` que combina ambas operaciones, creando algo parecido 
a un pipeline:

```clojure
(comp (map #(+ % 2)) (filter even?))
```

Aplicandolo en nuestro ejemplo tenemos:

```clojure
user => (into []
  (comp
    (map #(+ % 2))
    (filter even?))
  [1 2 3 4 5 6 7 8 9])
(4 6 8 10)
```

Este código solo genera una lista nueva con el resultado de aplicar el pipeline que hemos definido.

En Java tenemos los streams que hacen algo similar, al crear un stream, podemos crear un pipeline
combinando operaciones con `map`, `flatMap`, `filter`... pero hasta que no se llama lo que es
una operación final (ej. `collect`, `distinct`, `count`, etc..) no se ejecuta nada. La idea de esto es 
reducir el número de objetos intermedios, igual que los transducers.

Y como no, me he puesto a implementar estos transducers en Java y aplicarlos a mi librería purefun,
que justamente echaba en falta algo parecido a esto.

Primero de todo tenemos que definir lo que es un `Reducer`. Esto es simplemente una función que recibe
un acumulador y un elemento y devuelve un nuevo acumulador. Tan sencillo como esto:
  
```java
@FunctionalInterface
public interface Reducer<A, E> {

  A apply(A accumulator, E element);
  
}
```

Por otra parte necesitaremos el `Transducer` propiamente dicho, que es otra función que recibe
un `Reducer` y devuelve otro `Reducer`.

```java
@FunctionalInterface
public interface Transducer<A, I, O> {

  Reducer<A, I> apply(Reducer<A, O> reducer);

}
```

Ahora podemos definir una función para combinar dos transducers:

```java
  static <A, I, O, R> Transducer<A, I, R> chain(Transducer<A, I, O> t1, Transducer<A, O, R> t2) {
    return reducer -> t1.apply(t2.apply(reducer));
  }
```

Y ahora ya podemos definir `map` y `filter`:

```java
  static <A, I, O> Transducer<A, I, O> map(Function<I, O> mapper) {
    return reducer -> 
      (accumulator, element) -> reducer.apply(accumulator, mapper.apply(element));
  }

  static <A, I> Transducer<A, I, I> filter(Predicate<I> predicate) {
    return reducer -> 
      (accumulator, element) -> predicate.test(element) ? 
        reducer.apply(accumulator, element) : accumulator;
  }
```

Ahora necesitamos una función para aplicar todo esto y generar un resultado. La función recibe
el valor inicial del acmulador, el pipeline en sí, un reducer para añadir elementos al acumulador y
finalmente la lista original con la entrada.

```java
  static <A, I, O> A transduce(
      A init, Transducer<A, I, O> pipeline, Reducer<A, O> appender, Iterable<I> input) {
    var reducer = pipeline.apply(appender);
    var acc = init;
    for (I value : input) {
        acc = reducer.apply(acc, value);
    }
    return acc;
  }
```

Como combinamos todo esto?

```java
  var list = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9);

  Transducer<List<Integer>, Integer, Integer> pipeline =
      chain(map(x -> x + 2), filter(x -> x % 2 == 0));

  var result = transduce(
    new ArrayList<Integer>(), pipeline, (a, e) -> { a.add(e); return a; }, list);

  println(result);
```

El resultado es igual al que obteníamos con clojure: `[4, 6, 8, 10]`.

Podemos simplificarlo un poco creando un método `toList`:

```java
  static <I, O> List<O> toList(Transducer<List<O>, I, O> pipeline, Iterable<I> input) {
    return transduce(new ArrayList<>(), pipeline, (a, e) -> { a.add(e); return a; }, input);
  }
```

Y queda algo así:

```java
  var list = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9);

  var result = toList(chain(map(x -> x + 2), filter(x -> x % 2 == 0)), list);

  println(result);
```

Si lo comparamos con los streams de Java vemos que queda algo muy parecido.

```java
  toList(chain(map(x -> x + 2), filter(x -> x % 2 == 0)), list);
  list.stream().map(x -> x + 2).filter(x -> x % 2 == 0).toList();
```

En mi librería queda de esta manera, que es totalmente equivalente a los Streams:

```java
  listOf(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .pipeline().map(x -> x + 2).filter(x -> x % 2 == 0).toImmutableList();
```

Lo que más me gusta de los transducers es que son mucho más fácil de entender que los streams, para
la mayor parte de los programadores son una especie de caja negra. Además, son mucho más potentes,
se pueden implementar todos los métodos que existen en los streams (map, flatMap, filter, etc...), 
y también se pueden implementar window functions que no se han incluido en Java hasta la llegada de
los gathers, con los transducers son triviales de implementar.

Si tenéis curiosidad, aquí está el enlace a mi librería [purefun](https://github.com/tonivade/purefun).
