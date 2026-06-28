---
layout: post
title:  Programando con DSLs en Java 25 (VIII)
date:   2026-06-27 14:00:00
categories: programming java functional-programming dsl monad
---

Aquí vuelvo a la carga con el mismo tema, ya voy por la octava entrega, y posiblemente haya alguna
más. Y es que una cosa me lleva a la otra. El hecho de que el DSL que hemos estado implementado
sea declarativo, lo que significa que no se ejecuta en ese mismo momento, sino que es una estructura
de datos que describe el programa, esto nos permite hacer cosas que no se pueden hacer normalmente.

Hoy vamos a ver como implementar programación asíncrona, es decir, ejecutar partes de nuestro
programa en diferentes hilos y finalmente combinar los resultados para generar una respuesta.

La manera de implementarlo es realmente simple, sólo necesitamos añadir un nuevo caso en la de
definición de `Program`.

```java
sealed interface Program<S, E, T> {

  // ...
  record Forked<S, E, T>(Program<S, E, T> program) 
    implements Program<S, E, CompletableFuture<Either<E, T>>> {}
}
```

Como se puede ver, wrapeamos una instancia de `Program` en una nueva instancia que devuelve como
resultado un `CompletableFuture`. A su vez, ese `CompletableFuture` devuelve un `Either<E, T>`. La
chicha está ahora en el método `eval` que es donde se evaluará ese `Forked`.


```java
sealed interface Program<S, E, T> {

  default Either<E, T> eval(S state) {
    Program<S, ?, ?> current = this;
    Deque<Function<Object, ? extends Program<S, ?, ?>>> failureStack = new ArrayDeque<>();
    Deque<Function<Object, ? extends Program<S, ?, ?>>> successStack = new ArrayDeque<>();
    Map<Program<S, ?, ?>, Program<S, ?, ?>> cache = new IdentityHashMap<>();

    while (true) {
      //...
      } else if (current instanceof Forked(var next)) {
        var future = CompletableFuture.supplyAsync(
            () -> next.eval(state), ForkJoinPool.commonPool());
        current = success((CompletableFuture<Either<Object, Object>>) future);
      }
    }
  }
}
```

En la implementación de `Forked` simplemente llamamos el método `eval` del `Program` previamente
wrapeado pasando el mismo estado, pero en este caso, lo ejecutaremos de manera asíncrona creando
un `CompletableFuture` que eventualmente generará el resutaldo de ejecutar ese programa. Es imprecindible
hacer ese casting para dejar al compilador tranquilo.

Ya casi lo tenemos, es una implementación muy sencilla que no require nada especial.

En capitulos anteriores implementamos el método `zip`, que ejecuta dos programas y combina los resultados
de los ambos programas, pues podríamos hacer lo mismo, pero en este caso una versión asíncrona
que ejecute dos programas de manera asíncrona y luego combine ambos resultados una vez que los
futuros se hayan resuelto.

Este nuevo combinador lo llamaremos `parZip`.

```java
sealed interface Program<S, E, T> {

  static <S, E, T, U, R> Program<S, E, R> parZip(
      Program<S, E, T> first, Program<S, E, U> second, BiFunction<T, U, R> mapper) {
    return zip(first.fork(), second.fork(), (t, u) -> Either.zip(t.join(), u.join(), mapper))
        .flatMap(Program::from);
  }

  default Program<S, E, CompletableFuture<Either<E, T>>> fork() {
    return new Forked<>(this);
  }

  static <S, E, T> Program<S, E, T> from(Either<E, T> value) {
    return value.fold(Program::failure, Program::success);
  }
}
```

Como ya hemos descrito, este método tiene una firma similar al método `zip` ya implementado, pero
evidentemente la implementación es completamente diferente.

Primero llamamos a un método de ayuda que hemos llamado `fork`, que simplemente wrapea la instancia
actual en un `Forked`.

Después llamamos al método `zip` pasando los dos programas ya forkeados. Como hemos dicho antes
el resultado de ejecutar un `Forked` va a ser un `CompletableFuture` asi que para combinar los
resultados tendremos que llamar a los métodos `join` de cada `CompletableFuture`.

Como el resultado de llamar a `join` va a ser un `Either`, si queremos combinar los resultados de
ambos programas, necesitaremos a su vez llamar al método `Either.zip`, este método es muy sencillo
de implementar basandonos en el método `fold` previamente ya implementado en capitulos
anteriores.

```java
sealed interface Either<L, R> {

  static <L, A, B, C> Either<L, C> zip(Either<L, A> a, Either<L, B> b, BiFunction<A, B, C> mapper) {
    return a.fold(Either::left,
        r1 -> b.fold(Either::left,
            r2 -> right(mapper.apply(r1, r2))));
  }
}
```

Como último paso necesitamos volver a transformar el resultado que es un `Either` a un `Program`,
por lo que llamamos a `flatMap` llamando al método `Program.from`. Este método simplemente crea un
`Program.success` o un `Program.failure` dependiendo de si `Either` es left o right.

Ahora que tenemos ya toda la infraestructura preparada, necesitaremos probarlo. Este test
simplemente define dos programas `p1` y `p2`. Para definirlo usamos el método `Program.suspend`, que
implementamos en capitulos anteriores. Dentro de `suspend` simplemente hacemos un sleep durante un
determinado tiempo, para `p1`, 5 segundos y para `p2`, 3 segundos. Por otra parte, `p1` devuelve
un `String` y `p2` un `Integer`.

```java
class ProgramTest {

  @Test
  void shouldExecInParallel() {
    var p1 = suspend(() -> {
      sleep(5);
      return success("hello");
    });
    var p2 = suspend(() -> {
      sleep(3);
      return success(1);
    });

    var result = parZip(p1, p2, Tuple::new).eval(null);

    assertEquals(right(new Tuple<>("hello", 1)), result);
  }

  record Tuple<A, B>(A a, B b) {}

  private void sleep(int seconds) {
    // ...
  }
}
```

Ahora ya podemos llamar a `parZip` pasando ambos programas, y para combinarlos llamamos a `Tuple::new`.
`Tuple` es simplemente un record que usamos para este test para combinar ambos resultados. Por último
necesitamos llamar a `eval`. No necesitamos un estado concreto, por lo que simplemente pasamos `null`
como parámetro.

Ahora ya tenemos en `result` el resultado de evaluar nuestro programa. Verificamos que el resultado
es el esperado, pero ahora, cómo podemos comprobar que todo esto se ha ejecutado de manera asíncrona.
Bueno, como he dicho antes, he insertado unos `sleep` diferentes para luego poder comprobar que la
el tiempo requerido para la ejecución de ambos programas debe ser inferior a la suma de la ejecución
de ambos métodos en caso de que se ejecutaran secuencialmente, que sería 8 segundos. Por lo que
para comprobar que ambos programas se han ejecutado de manera asíncrona, simplemente necesitamos
verificar que el tiempo de ejecución total será el tiempo máximo empleado por cada uno de esos programas,
que como es lógico, será de 5 segundos.

Para comprobarlo, podemos aprovecharnos de la naturaleza declarativa de nuestro DSL y definir un
metodo llamado `timed` que devuelva no solo el resultado sino el tiempo empleado en la ejecución del
programa.

```java
sealed interface Program<S, E, T> {

  default Program<S, E, ElapsedTime<T>> timed() {
    return pipe(
        suspend(() -> success(System.nanoTime())),
        start -> map(result -> new ElapsedTime<>(Duration.ofNanos(System.nanoTime() - start), result))
      );
  }

  record ElapsedTime<T>(Duration duration, T value) {}
}
```

Este nuevo método devuelve un objeto llamado `ElapsedTime` que contiene ambos valores, el resultado
de la ejecución y el tiempo empleado durante su ejecución.

La implementación es muy sencilla, simplemente obtenemos el tiempo antes de la ejecución, y luego
en el siguiente paso se llama al método `map` que nos permite acceder al resultado de la ejecución
del programa actual y justo en ese punto crear un ElapsedTime restando al tiempo actual el tiempo
inicial.

Pequeño apunte, aquí usamos `System.nanoTime`, porque para este tipo de cosas es mejor usar este 
método a `System.currentTimeMillis` que es más [preciso](https://stackoverflow.com/questions/351565/system-currenttimemillis-vs-system-nanotime#351571).

Volviendo a lo que nos ocupa, el test ahora quedaría de esta forma:

```java
class ProgramTest {

  @Test
  void shouldExecInParallel() {
    var p1 = suspend(() -> {
      sleep(5);
      return success("hello");
    });
    var p2 = suspend(() -> {
      sleep(3);
      return success(1);
    });

    var result = parZip(p1, p2, Tuple::new).timed().getOrElseThrow();

    assertEquals(new Tuple<>("hello", 1), result.value());
    assertEquals(5, result.duration().getSeconds());
  }
}
```

Finalmente podemos comprobar que el tiempo de ejecución es de alrededor de 5 segundos.

Para este último cambio, he implementado otro método nuevo en `getOrElseThrow` que básicamente
es mucho más conveniente para este tipo de casos donde no tenemos un estado y el posible error
generado no nos interesa. `getOrElseThrow` devolverá el resultado de la ejecución en caso
de ir todo bien, o lanzará una excepción `NoSuchElementException`. El código es trivial
y no lo pongo aquí.

Como deberes podríais tratar de implementar un método `sleep` que reciviera un `Duration` y que devolviera
un `Program<S, E, Void>`. Luego podríamos usar ese método `sleep` y combinarlo con suspend para crear un método
`delay`. El test quedaría de esta manera:

```java
class ProgramTest {

  @Test
  void shouldExecInParallel() {
    var p1 = delay(Duration.ofSeconds(5), () -> "hello");
    var p2 = delay(Duration.ofSeconds(3), () -> 1);

    var result = parZip(p1, p2, Tuple::new).timed().getOrElseThrow();

    assertEquals(new Tuple<>("hello", 1), result.value());
    assertEquals(5, result.duration().getSeconds());
  }
}
```