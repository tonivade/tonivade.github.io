---
layout: post
title:  Programando con DSLs en Java 25 (IV)
date:   2026-03-28 14:00:00
categories: programming java functional-programming dsl monad
---

En estos tres últimos artículos hemos visto como utilizando las novedades más recientes
de Java, como son los sealed interfaces, records, pattern matching and switch expressions
podemos implementar en Java Domain Specific Languages de tal manera que podemos describir
un lenguaje, sus operaciones y combinarlas para hacer programas que luego interpretamos.
Se podría decir que tenemos un mini lenguaje de programación dentro de Java.

Y siguiendo por esa senda hemos descubierto de manera natural que lo que obtenemos es
una estructura parecida a una free monad, que viene del mundo de la programación funcional
y de la categoría de teorías.

No me interesa ese lado más académico (si alguien tiene interés puede [profundizar](https://abuseofnotation.github.io/category-theory-illustrated/00_about/) más), lo
que me interesa es el lado más pragmático, y es que es algo que me permite definir mini lenguajes,
combinarlos y hacer programas complejos que utilicen esos lenguajes.

En el último artículos hemos visto todo lo que podíamos sacar factor común e implementar
una estructura de datos que hemos llamado `Program`. Este es el código fuente completo para
refrescar la memoria:

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

Para crear más operaciones necesitamos crear un punto de extensión que llamamos `Dsl`.
Luego por cada operación debíamos crear otra clase que extendiera `Dsl` e implementar
el método `handle` de esa nueva operación. Es mucho código repetitivo, ¿no podríamos
simplificarlo de alguna manera?

La respuesta a esta pregunta es sí, se puede hacer. Como `State` necesitaba almacenar
un dato en alguna parte, el método `eval` recibía un contexto de tipo `S`, y luego
en `eval` accedíamos a ese contexto. ¿No podríamos generalizar ese patrón? Podríamos
crear otra implementación de `Program` que tuviera una función que recibiera `S` y
devolviera el resultado de ejecutar la función. Algo como esto:

```java
sealed interface Program<S, T> {

  record Done<S, T>(T value) implements Program<S, T> {}

  record AndThen<S, X, T>(Program<S, X> current, Function<X, Program<S, T>> next) 
    implements Program<S, T> {}

  record Access<S, T>(Function<S, T> mapper) {}
}
```

Y luego interpretar esto sería tan sencillo como esto:

```java
default T eval(S state) {
  return switch (this) {
    case Done<S, T>(T value) -> value;
    case AndThen<S, ?, T>(var current, var next) -> {
      var value = (Object) current.eval(state);
      var nextValue = ((Function<Object, Program<S, T>>) next).apply(value);
      yield nextValue.eval(state);
    }
    case Access<S, T>(var mapper) -> mapper.apply(state);
  };
}
```

¿Qué conseguimos con esto? Pues ya no necesitamos `Dsl`, ni crear extensiones de
`Dsl` para definir nuevas operaciones. Ahora sería simplemente llamadas a métodos 
que luego se implementaran nuestro contexto.

¿Ahora cómo quedaría la cosa? Veamos como quedaría `Console`:

```java
interface Console {

  interface Service {
    void println(String line);
    String readln();
  }

  static <S extends Console.Service> Program<S, String> readln() {
    return new Program.Access<>(Console.Service::readln);
  }

  static <S extends Console.Service> Program<S, Void> println(String question) {
    return new Program.Access<>(s -> {
      s.println(question);
      return null;
    });
  }
}
```

Tenemos que definir un interfaz adicional que definiría todas las operaciones
que podríamos hacer, en este caso `Console.Service`, y una serie de métodos de ayuda
que accedieran a los métodos de `Service` a través de `Program.Access`. Para que esto 
funcione `S` tiene que implementar el interfaz `Console.Service`, y para eso
necesitamos decir que `S extends Console.Service`.

Ahora `Random`:

```java
interface Random {

  interface Service {
    int nextInt(int bound);
  }

  static <S extends Random.Service> Program<S, Integer> nextInt(int bound) {
    return new Program.Access<>(s -> s.nextInt(bound));
  }
}
```

Y por último `State`:

```java
interface State {

  interface Service {
    void setValue(int value);
    int getValue();
  }

  static <S extends Service> Program<S, Void> setValue(int value) {
    return new Program.Access<>(state -> {
      state.setValue(value);
      return null;
    });
  }

  static <S extends Service> Program<S, Integer> getValue() {
    return new Program.Access<>(Service::getValue);
  }
}
```

Con esta nueva forma de implementar las operaciones el compilador nos da menos
problemas, todo queda mucho más limpio y más sencillo de seguir.

Por último necesitaremos implementar nuestro nuevo contexto, que debe implementar 
los interfaces `Console.Service`, `Random.Service` y `State.Service`:

```java
public class Context implements Console.Service, State.Service, Random.Service {

  private int value;

  @Override
  public void setValue(int value) {
    this.value = value;
  }

  @Override
  public int getValue() {
    return value;
  }

  @Override
  public int nextInt(int bound) {
    return ThreadLocalRandom.current().nextInt(bound);
  }

  @Override
  public void println(String line) {
    IO.println(line);
  }

  @Override
  public String readln() {
    return IO.readln();
  }
}
```

Ahora la implementación de nuestro programa sería muy parecida a lo que teníamos antes
solo pequeños cambios cosméticos:

```java
class Game {

  static void main() {
    var program =
      Console.<Context>prompt("Do you want to play a game? (y/n)")
        .andThen(answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return Random.<Context>nextInt(10).andThen(State::setValue).andThen(_ -> play());
          }
          return Console.println("Bye!");
        });

    program.eval(new Context());
  }

  static Program<Context, Void> play() {
    return Console.<Context>prompt("Enter a number between 0 to 9")
      .map(Integer::parseInt)
      .andThen(number -> State.<Context>getValue().map(value -> value == number))
      .andThen(result -> {
        if (result) {
          return Console.println("YOU WIN!");
        }
        return play();
      });
  }
}
```

Todavía necesitamos ayudar al compilador para inferir correctamente los tipos,
pero podemos mejorar esto mediante algunos métodos de ayuda que faciliten la
combinación de operaciones.

Veamos por qué el compilador no es capaz de inferir los tipos, por ejemplo aquí:

```java
Random.nextInt(10).andThen(State::setValue)
```

La primera operación, `Random.nextInt` necesita un contexto de tipo `Random.Service` y
la segunda operación `State.setValue` necesita un contexto de tipo `State.Service`. El
compilador se queja, y con razón ya que ambos tipos no son compatibles entre sí. Asi que
por eso debemos ayudarle indicando que el contexto va a ser `Context` que implementa
ambos tipos: `Random.Service` y `State.Service`.

```java
Random.<Context>nextInt(10).andThen(State::setValue)
```

Para resolver esto podemos definir unas funciones de ayuda, que voy a llamar `pipe`:

```java
static <S, T, R> Program<S, R> pipe(Program<S, T> first, Function<T, Program<S, R>> next) {
  return first.andThen(next);
}
```

Con estas funciones el código del programa quedaría de esta manera:

```java
class Game {

  static void main() {
    var program =
      Program.pipe(
        Console.prompt("Do you want to play a game? (y/n)"),
        answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return Program.pipe(Random.nextInt(10), State::setValue, _ -> play());
          }
          return Console.println("Bye!");
        });

    program.eval(new Context());
  }

  static Program<Context, Void> play() {
    return Program.pipe(
      Console.prompt("Enter a number between 0 to 9"),
      s -> new Program.Done<>(Integer.parseInt(s)),
      number -> Program.pipe(State.getValue(), value -> new Program.Done<>(value == number)),
      result -> {
        if (result) {
          return Console.println("YOU WIN!");
        }
        return play();
      });
  }
}
```

Ahora ya no es necesario meter esos hints para el compilador, este ya es capaz de inferir los tipos
correctamente, aunque nos han quedado algunas cosas un poco feas como esta:

```java
s -> new Program.Done<>(Integer.parseInt(s))
```

Esto lo podemos resolver fácilmente definiendo otro método de ayuda:

```java
static <S, T, R> Function<T, Program<S, R>> lift(Function<T, R> mapper) {
  return mapper.andThen(Done::new);
}
```

Este método simplemente recibe una función que devolvía un tipo X, en su lugar devuelve otra función
que devuelve `Program<S, X>`.  Este sería el resultado:

```java
static Program<Context, Void> play() {
  return Program.pipe(
    Console.prompt("Enter a number between 0 to 9"),
    lift(Integer::parseInt),
    number -> Program.pipe(State.getValue(), lift(value -> value == number)),
    result -> {
      if (result) {
        return Console.println("YOU WIN!");
      }
      return play();
    });
}
```

Refactorizando, extrayendo algunas funciones privadas y con imports estáticos el programa
finalmente podría quedar de esta manera:

```java
class Game {

  static void main() {
    var program = pipe(
        prompt("Do you want to play a game? (y/n)"),
        answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return generateNumberAndPlay();
          }
          return println("Bye!");
        });

    program.eval(new Context());
  }

  static Program<Context, Void> play() {
    return pipe(
        readNumber(),
        Game::checkNumber,
        result -> {
          if (result) {
            return println("YOU WIN!");
          }
          return play();
        });
  }

  static Program<Context, Void> generateNumberAndPlay() {
    return pipe(
        nextInt(10),
        State::setValue,
        _ -> play());
  }

  static Program<Context, Integer> readNumber() {
    return pipe(
        prompt("Enter a number between 0 to 9"),
        lift(Integer::parseInt)
      );
  }

  static Program<Context, Boolean> checkNumber(Integer number) {
    return pipe(getValue(), lift(value -> value == number));
  }
}
```

A mi personalmente este tipo de código me resulta fácil de seguir y de entender,
y podemos interpretar este programa de muchas maneras diferentes.

La interpretación actual que tenemos utiliza la consola estándar, pero podríamos
definir otro interprete que utilizara una interfaz gráfica. O podríamos crear
un interprete mockeado que nos permitiera probar el comportamiento del programa.

Hagamos este otro refactor de tal manera que no estemos forzados a usar `Context` como
contexto. Podemos usar otra cosa chula que tiene java que es la intersección de
tipos:

```java
class Game {

  static void main() {
    game().eval(new Context());
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Void> game() {
    return pipe(
        prompt("Do you want to play a game? (y/n)"),
        answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return generateNumberAndPlay();
          }
          return println("Bye!");
        });
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Void> play() {
    return pipe(
        prompt("Enter a number between 0 to 9"),
        lift(Integer::parseInt),
        Game::checkNumber,
        result -> {
          if (result) {
            return println("YOU WIN!");
          }
          return play();
        });
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Void> generateNumberAndPlay() {
    return pipe(
        nextInt(10),
        State::setValue,
        _ -> play());
  }

  static <S extends Console.Service> Program<S, Integer> readNumber() {
    return pipe(
        prompt("Enter a number between 0 to 9"),
        lift(Integer::parseInt)
      );
  }

  static <S extends State.Service> Program<S, Boolean> checkNumber(Integer number) {
    return pipe(getValue(), lift(value -> value == number));
  }
}
```

Y ahora definamos una implementación mockeada del contexto:

```java
class MockContext implements Console.Service, Random.Service, State.Service {

  private final List<String> output = new ArrayList<>();
  private final List<String> input = new ArrayList<>();

  private int value;

  public MockContext(String... input) {
    this.input.addAll(List.of(input));
  }

  @Override
  public void setValue(int value) {
    this.value = value;
  }

  @Override
  public int getValue() {
    return value;
  }

  @Override
  public int nextInt(int bound) {
    return 2;
  }

  @Override
  public void println(String line) {
    output.addLast(line);
  }

  @Override
  public String readln() {
    return input.removeFirst();
  }
}
```

Este interprete mockeado debe implementar los tres servicios, igual que la clase `Context`.
Para mockear la salida y entrada estándar simplemente usamos un par de `ArrayList`.
Cuando se lee de la entradas estándar borramos el primer elemento de la lista `input`, y cada
vez que se escribe en la salida estándar añadimos un elemento a la lista `output`. El método
`nextInt` hacemos que devuelva un valor concreto, por ejemplo 2.

Ahora podemos escribir un test como este utilizando este mock:

```java
@Test
void happyPath() {
  var context = new MockContext("y", "1", "2");

  game().eval(context);

  var expectedOutput = List.of(
      "Do you want to play a game? (y/n)",
      "Enter a number between 0 to 9",
      "Enter a number between 0 to 9", 
      "YOU WIN!");
  assertEquals(expectedOutput, context.output);
}
```

Este test simula que el usuario introduce "y", "1" y "2" por consola, lo que significa que
acierta el número al segundo intento. Y en la salida estándar verificamos que se han escrito
los lineas esperadas.

Con librerías de tipo Mockito se podrían escribir tests mucho más elaborados y elegantes.

Y esto es todo lo que quería contar. Cuando quieres implementar DSLs y perseveras por ese
camino al final aparecen estructuras inesperadas que podemos reconocer como abstracciones
que vienen de las matemáticas. No necesitamos entender esas matemáticas para hacer uso
de esas abstracciones. Esta claro que conocer la teoría matemática subyacente ayuda y mucho
pero no es estrictamente necesario.

En el anterior articulo dije que este iba a ser mi último artículo de la serie, pero creo
que me da para hacer otro más sobre el manejo de errores. Supongamos que en este mismo
ejemplo el usuario en lugar de introducir un número, introduce una letra, ¿qué pasaría?

Evidentemente `Integer.parseInt` lanzaría una `NumberFormatException`. Con la implementación
actual de `Program` no podemos hacer nada, pero si que adaptándola un poco sí que podríamos
manejarlo. Lo veremos en el siguiente artículo.

Dejo aquí todo el código completo del ejemplo, pero en lugar the `andThen` creo que podemos
finalmente renombrarlo `flatMap`, que es lo que hace realmente:

```java
import java.util.function.Function;

sealed interface Program<S, T> {

  record Done<S, T>(T value) implements Program<S, T> {}

  record FlatMap<S, X, T>(Program<S, X> current, Function<X, Program<S, T>> next) implements Program<S, T> {}

  record Access<S, T>(Function<S, T> mapper) implements Program<S, T> {}

  default <R> Program<S, R> map(Function<T, R> mapper) {
    return flatMap(lift(mapper));
  }

  default <R> Program<S, R> flatMap(Function<T, Program<S, R>> next) {
    return new FlatMap<>(this, next);
  }

  static <S, T, R> Function<T, Program<S, R>> lift(Function<T, R> mapper) {
    return mapper.andThen(Done::new);
  }

  static <S, T> Program<S, T> access(Function<S, T> mapper) {
    return new Access<>(mapper);
  }

  static <S, T, R> Program<S, R> pipe(Program<S, T> first, Function<T, Program<S, R>> next) {
    return first.flatMap(next);
  }

  static <S, T, U, R> Program<S, R> pipe(
      Program<S, T> first, Function<T, Program<S, U>> second, Function<U, Program<S, R>> next) {
    return pipe(first, t -> pipe(second.apply(t), next));
  }

  static <S, T, U, V, R> Program<S, R> pipe(
      Program<S, T> first, Function<T, Program<S, U>> second, Function<U, Program<S, V>> third, Function<V, Program<S, R>> next) {
    return pipe(first, t -> pipe(second.apply(t), u -> pipe(third.apply(u), next)));
  }

  @SuppressWarnings("unchecked")
  default T eval(S state) {
    return switch (this) {
      case Done<S, T>(T value) -> value;
      case FlatMap<S, ?, T>(var current, var next) -> {
        var value = (Object) current.eval(state);
        var nextValue = ((Function<Object, Program<S, T>>) next).apply(value);
        yield nextValue.eval(state);
      }
      case Access<S, T>(var mapper) -> mapper.apply(state);
    };
  }
}
```

```java
import static Program.access;
import static Program.pipe;

interface Console {

  interface Service {
    void println(String line);
    String readln();
  }

  static <S extends Console.Service> Program<S, String> readln() {
    return access(Console.Service::readln);
  }

  static <S extends Console.Service> Program<S, Void> println(String question) {
    return access(s -> {
      s.println(question);
      return null;
    });
  }

  static <S extends Console.Service> Program<S, String> prompt(String question) {
    return pipe(println(question), _ -> readln());
  }
}
```

```java
import static Program.access;

interface Random {

  interface Service {
    int nextInt(int bound);
  }

  static <S extends Random.Service> Program<S, Integer> nextInt(int bound) {
    return access(s -> s.nextInt(bound));
  }
}
```

```java
import static Program.access;

interface State {

  interface Service {
    void setValue(int value);
    int getValue();
  }

  static <S extends Service> Program<S, Void> setValue(int value) {
    return access(state -> {
      state.setValue(value);
      return null;
    });
  }

  static <S extends Service> Program<S, Integer> getValue() {
    return access(Service::getValue);
  }
}
```

```java
import static Console.println;
import static Console.prompt;
import static Program.lift;
import static Program.pipe;
import static Random.nextInt;
import static State.getValue;

class Game {

  static void main() {
    game().eval(new Context());
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Void> game() {
    return pipe(
        prompt("Do you want to play a game? (y/n)"),
        answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return generateNumberAndPlay();
          }
          return println("Bye!");
        });
  }

  static <S extends Console.Service & State.Service> Program<S, Void> play() {
    return pipe(
        readNumber(),
        Game::checkNumber,
        result -> {
          if (result) {
            return println("YOU WIN!");
          }
          return play();
        });
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Void> generateNumberAndPlay() {
    return pipe(
        nextInt(10),
        State::setValue,
        _ -> play());
  }

  static <S extends Console.Service> Program<S, Integer> readNumber() {
    return pipe(
        prompt("Enter a number between 0 to 9"),
        lift(Integer::parseInt)
      );
  }

  static <S extends State.Service> Program<S, Boolean> checkNumber(Integer number) {
    return pipe(getValue(), lift(value -> value == number));
  }
}
```

```java
import java.util.concurrent.ThreadLocalRandom;

public class Context implements Console.Service, State.Service, Random.Service {

  private int value;

  @Override
  public void setValue(int value) {
    this.value = value;
  }

  @Override
  public int getValue() {
    return value;
  }

  @Override
  public int nextInt(int bound) {
    return ThreadLocalRandom.current().nextInt(bound);
  }

  @Override
  public void println(String line) {
    IO.println(line);
  }

  @Override
  public String readln() {
    return IO.readln();
  }
}
```

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static Game.game;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

class GameTest {

  @Test
  void happyPath() {
    var context = new MockContext("y", "1", "2");

    game().eval(context);

    var expectedOutput = List.of(
        "Do you want to play a game? (y/n)",
        "Enter a number between 0 to 9",
        "Enter a number between 0 to 9", "YOU WIN!");
    assertEquals(expectedOutput, context.output);
  }

  @Test
  void userEnterAnInvalidValue() {
    var context = new MockContext("y", "c");

    assertThrows(NumberFormatException.class, () -> game().eval(context));
  }

  class MockContext implements Console.Service, Random.Service, State.Service {

    private final List<String> output = new ArrayList<>();
    private final List<String> input = new ArrayList<>();

    private int value;

    public MockContext(String... input) {
      this.input.addAll(List.of(input));
    }

    @Override
    public void setValue(int value) {
      this.value = value;
    }

    @Override
    public int getValue() {
      return value;
    }

    @Override
    public int nextInt(int bound) {
      return 2;
    }

    @Override
    public void println(String line) {
      output.add(line);
    }

    @Override
    public String readln() {
      return input.removeFirst();
    }
  }
}
```
