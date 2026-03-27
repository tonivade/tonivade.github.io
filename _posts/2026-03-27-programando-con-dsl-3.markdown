---
layout: post
title:  Programando con DSLs en Java 25 (III)
date:   2026-03-27 15:00:00
categories: programming java functional-programming dsl monad
---

Después de ver estos dos últimos artículos como implementar sencillos programas
con DSLs en Java, por fin ha llegado el momento de generalizar y sacar factor
común de tal manera que no tengamos que implementar una y otra vez lo mismo.

```java
sealed interface Program<T> {
  record Done<T>(T value) implements Program<T> {}
  record AndThen<X, T>(
    Program<X> current, 
    Function<X, Program<T>> next) implements Program<T> {}
}
```

Por ahora simplemente definimos el `Done` y el `AndThen` que ya teníamos del
ejemplo anterior. Ahora necesitamos añadir un punto de extensión para poder definir 
operaciones adicionales.

```java
sealed interface Program<T> {
  record Done<T>(T value) implements Program<T> {}
  record AndThen<X, T>(
    Program<X> current, 
    Function<X, Program<T>> next) implements Program<T> {}
  non-sealed interface Dsl<T> extends Program<T> {
    T handle();
  }
}
```

Para ello añadimos otra extensión de `Program` llamada `Dsl`. Lo innovador
de esto es que usamos la palabra reservada `non-sealed` lo que significa 
que `Dsl` se puede extender y aunque `Program` esté definida como `sealed`
el compilador no se quejará. `Dsl` define además un método `handle` para
poder implementar la gestión de las diferentes operaciones específicas de
cada extensión.

Ahora necesitamos implementar los métodos `andThen` y `map`, muy parecido
a lo que ya teníamos del ejemplo anterior:

```java
default <R> Program<R> andThen(Function<T, Program<R>> next) {
  return new AndThen<>(this, next);
}

default <R> Program<R> map(Function<T, R> mapper) {
  return andThen(mapper.andThen(Done::new));
}
```

Y por último el método `eval`, que es muy similar también a lo que teníamos
del ejemplo anterior. Para el caso de `Dsl` simplemente delegamos en el método
`handle` que habíamos definido antes:

```java
default T eval() {
  return switch (this) {
    case Done<T>(T value) -> value;
    case AndThen<?, T>(var current, var next) -> {
      var value = (Object) current.eval();
      var nextValue = ((Function<Object, Program<T>>) next).apply(value);
      yield nextValue.eval();
    }
    case Dsl<T> dsl -> dsl.handle();
  };
}
```

Ahora vamos a definir las extensiones necesarias para nuestro ejemplo. Empecemos por 
`Console`:

```java
sealed interface Console<T> extends Program.Dsl<T> {

  record WriteLine(String line) implements Console<Void> {}
  record ReadLine() implements Console<String> {}

  default T handle() {
    return (T) switch (this) {
      case WriteLine(var line) -> {
        IO.println(line);
        yield null;
      }
      case ReadLine _ -> IO.readln();
    };
  }
}
```

La única diferencia a lo que teníamos antes es que `Console` extiende de `Program.Dsl`.

Ahora es el turno de `Random`:

```java
sealed interface Random<T> extends Program.Dsl<T> {

  record NextInt(int bound) implements Random<Integer> {}

  default T handle() {
    return (T) switch (this) {
      case NextInt(int bound) -> ThreadLocalRandom.current().nextInt(bound);
    };
  }
}
```

Y por último `State`:

```java
sealed interface State<T> extends Program.Dsl<T> {

  record SetValue(int value) implements State<Void> {}
  record GetValue() implements State<Integer> {}

  default T handle() { // it's not going to work
    return (T) switch (this) {
      case SetValue(int value) -> {
        context.set(value);
        yield null;
      }
      case GetValue _ -> context.get();    
    };
  }
}
```

Como se ve, algo falta, ya que para hacer funcionar `State` necesitamos el contexto
donde guardar el valor. Para ello necesitaremos pasar un estado por parámetro.

```java
sealed interface Program<S, T> {
  record Done<S, T>(T value) implements Program<S, T> {}
  record AndThen<S, X, T>(
    Program<S, X> current, 
    Function<X, Program<S, T>> next) 
      implements Program<S, T> {}
  non-sealed interface Dsl<S, T> extends Program<S, T> {
    T handle(S state);
  }
}
```

En `Program` hemos añadido otro tipo `S` que se propaga por el resto de tipos,
hasta llegar al método `handle`. Y `eval` también hay que adaptarlo:

```java
default T eval(S state) {
  return switch (this) {
    case Done<S, T>(T value) -> value;
    case AndThen<S, ?, T>(var current, var next) -> {
      var value = (Object) current.eval(state);
      var nextValue = ((Function<Object, Program<S, T>>) next).apply(value);
      yield nextValue.eval(state);
    }
    case Dsl<S, T> dsl -> dsl.handle(state);
  };
}
```

Ahora `State` ya podemos definirlo de esta manera:

```java
sealed interface State<T> extends Program.Dsl<Context, T> {

  record SetValue(int value) implements State<Void> {}
  record GetValue() implements State<Integer> {}

  default T handle(Context context) {
    return (T) switch (this) {
      case SetValue(int value) -> {
        context.set(value);
        yield null;
      }
      case GetValue _ -> context.get();
    };
  }
}
```

Y cómo quedarían Console y Random ahora? Como no necesitan estado, al menos por ahora,
definimos el parámetro `S` como `Object`.

```java
sealed interface Console<T> extends Program.Dsl<Object, T> {
  // ...
}

sealed interface Random<T> extends Program.Dsl<Object, T> {
  // ...
}
```

Podríamos pensar que ya lo tenemos todo, pero tenemos un problema todavía, y es que los
diferentes tipos de operaciones no se pueden componer entre sí, ya que los tipos de
contexto no son iguales. Por ejemplo este código no compila:

```java
new Random.NextInt(10).andThen(State.SetValue::new);
```

`Random.NextInt` tienen un contexto de tipo `Object` y `State.SetValue` de tipo `Context`.
Para resolver esto podemos crear unos métodos de ayuda para permitir esta composición. Como
por ejemplo este método:


```java
static <S> Program<S, Integer> nextInt(int bound) {
  return (Program<S, Integer>) new NextInt(bound);
}
```

El casting hay que hacerlo, ya que el compilador se queja, pero es completamente seguro.

Los métodos de ayuda para `Console` serían estos:

```java
static <S> Program<S, String> readln() {
  return (Program<S, String>) new ReadLine();
}

static <S> Program<S, Void> println(String question) {
  return (Program<S, Void>) new WriteLine(question);
}

static <S> Program<S, String> prompt(String question) {
  return (Program<S, String>) println(question).andThen(_ -> readln());
}
```

Ahora el código que antes no compilaba ya compila, aunque hay que hacer un pequeño
truco, ya que el compilador no es capaz de inferir los tipos correctamente, por lo
que esto sigue sin funcionar:

```java
nextInt(10).andThen(State.SetValue::new);
```

Pero con este pequeño truco para ayudar al compilador por fin funciona:

```java
Random.<Context>nextInt(10).andThen(State.SetValue::new);
```

Una vez hecho todo esto, el código queda de esta forma:

```java
static void main() {
  var program =
    Console.<Context>prompt("Do you want to play a game? (y/n)")
      .andThen(answer -> {
        if (answer.equalsIgnoreCase("y")) {
          return Random.<Context>nextInt(10).andThen(State.SetValue::new).andThen(_ -> play());
        }
        return Console.println("Bye!");
      });

    program.eval(new Context());
}

static Program<Context, Void> play() {
  return Console.<Context>prompt("Enter a number between 0 to 9")
    .map(Integer::parseInt)
    .andThen(number -> new State.GetValue().map(value -> value == number))
    .andThen(result -> {
      if (result) {
        return Console.println("YOU WIN!");
      }
      return play();
    });
}
```

Y ahora, por fin, ya está todo en su sitio. No está mal, ya tenemos nuestro tipo `Program`
que podremos extender para definir otras operaciones, y componerlas para hacer otros programas.
Pero todavía podemos mejorarlo, hay mucho código repetitivo que es realmente tedioso de escribir
y todavía hay que ayudar al compilador ya que no es capaz de inferir los tipos correctamente. 
Aunque eso será en el próximo (y probablemente último) artículo sobre este tema.

Dejo aquí todo el código completo del ejemplo:

```java
import java.util.function.Function;

sealed interface Program<S, T> {

  record Done<S, T>(T value) implements Program<S, T> {}

  record AndThen<S, X, T>(Program<S, X> current, Function<X, Program<S, T>> next) 
    implements Program<S, T> {}

  non-sealed interface Dsl<S, T> extends Program<S, T> {
    T handle(S state);
  }

  default <R> Program<S, R> map(Function<T, R> mapper) {
    return andThen(mapper.andThen(Done::new));
  }

  default <R> Program<S, R> andThen(Function<T, Program<S, R>> next) {
    return new AndThen<>(this, next);
  }

  @SuppressWarnings("unchecked")
  default T eval(S state) {
    return switch (this) {
      case Done<S, T>(T value) -> value;
      case AndThen<S, ?, T>(var current, var next) -> {
        var value = (Object) current.eval(state);
        var nextValue = ((Function<Object, Program<S, T>>) next).apply(value);
        yield nextValue.eval(state);
      }
      case Dsl<S, T> dsl -> dsl.handle(state);
    };
  }
}
```

```java
sealed interface Console<T> extends Program.Dsl<Object, T> {

  record WriteLine(String line) implements Console<Void> {}

  record ReadLine() implements Console<String> {}

  @SuppressWarnings("unchecked")
  static <S> Program<S, String> readln() {
    return (Program<S, String>) new ReadLine();
  }

  @SuppressWarnings("unchecked")
  static <S> Program<S, Void> println(String question) {
    return (Program<S, Void>) new WriteLine(question);
  }

  @SuppressWarnings("unchecked")
  static <S> Program<S, String> prompt(String question) {
    return (Program<S, String>) println(question).andThen(_ -> readln());
  }

  @Override
  @SuppressWarnings("unchecked")
  default T handle(Object state) {
    return (T) switch (this) {
      case WriteLine(var line) -> {
        IO.println(line);
        yield null;
      }
      case ReadLine _ -> IO.readln();
    };
  }
}
```

```java
import java.util.concurrent.ThreadLocalRandom;

sealed interface Random<T> extends Program.Dsl<Object, T> {

  record NextInt(int bound) implements Random<Integer> {}

  @SuppressWarnings("unchecked")
  static <S> Program<S, Integer> nextInt(int bound) {
    return (Program<S, Integer>) new NextInt(bound);
  }

  @Override
  @SuppressWarnings("unchecked")
  default T handle(Object state) {
    return (T) switch (this) {
      case NextInt(var bound) -> (Integer) ThreadLocalRandom.current().nextInt(bound);
    };
  }
}
```

```java
sealed interface State<T> extends Program.Dsl<Context, T> {

  record SetValue(Integer value) implements State<Void> {}

  record GetValue() implements State<Integer> {}

  @Override
  @SuppressWarnings("unchecked")
  default T handle(Context state) {
    return (T) switch (this) {
      case SetValue(var value) -> {
        state.set(value);
        yield null;
      }
      case GetValue _ -> state.get();
    };
  }
}
```

```java
class Game {

  static void main() {
    var program =
      Console.<Context>prompt("Do you want to play a game? (y/n)")
        .andThen(answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return Random.<Context>nextInt(10).andThen(State.SetValue::new).andThen(_ -> play());
          }
          return Console.println("Bye!");
        });

    program.eval(new Context());
  }

  static Program<Context, Void> play() {
    return Console.<Context>prompt("Enter a number between 0 to 9")
      .map(Integer::parseInt)
      .andThen(number -> new State.GetValue().map(value -> value == number))
      .andThen(result -> {
        if (result) {
          return Console.println("YOU WIN!");
        }
        return play();
      });
  }
}
```

```java
public class Context {

  private Integer value;

  void set(Integer value) {
    this.value = value;
  }

  Integer get() {
    return value;
  }
}
```
