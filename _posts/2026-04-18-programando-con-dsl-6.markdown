---
layout: post
title:  Programando con DSLs en Java 25 (VI)
date:   2026-04-18 22:00:00
categories: programming java functional-programming dsl monad
---

Si habéis estado atentos a los otro artículos, supongo que os habrá llamado la
atención una cosa, y es que la implementación del método `eval` era recursiva.
Volvamos a ver el código de la última versión del método:

```java
@SuppressWarnings("unchecked")
default Either<E, T> eval(S state) {
  return switch (this) {
    case Success<S, E, T>(T value) -> Either.right(value);
    case Failure<S, E, T>(E error) -> Either.left(error);
    case FlatMap<S, E, ?, T>(var current, var next) -> {
    var result = current.eval(state); // <- recursion here
    yield result.fold(
      _ -> (Either<E, T>) result,
      value -> {
        var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(value);
        return nextValue.eval(state);// <- recursion here
      });
    }
    case FlatMapError<S, ?, E, T>(var current, var next) -> {
    var result = current.eval(state); // <- recursion here
    yield result.fold(
      error -> {
        var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(error);
        return nextValue.eval(state); // <- recursion here
      },
      _ -> (Either<E, T>) result);
    }
    case Access<S, E, T>(var mapper) -> Either.right(mapper.apply(state));
  };
}
```

Para el ejemplo con el que estábamos trabajando esta implementación es perfectamente
válida. Además aquí usamos muchas de las nuevas mejoras del lenguaje Java, pattern
matching, improved switch expressions. Pero si tuviéramos un programa más complejo
que fuera a su vez recursivo, pues tendríamos un problema.