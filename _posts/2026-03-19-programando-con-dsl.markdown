---
layout: post
title:  Programando con DSLs en Java
date:   2026-03-19 21:00:00
categories: programming java functional-programming dsl monad
---

El año pasado estuve en la commit-conf dando una charla sobre programar con DSLs en Java. La idea
era usar las nuevas funcionalidades del lenguaje como los records, sealed classes and interfaces o
el pattern matching para llevar a Java a una nueva dimensión mucho más cercana a la programación
funcional.

El paradigma funcional tiene muchas cosas buenas, la inmutabilidad, las funciones puras, aislar
los side effects. Todo ello hace que el código sea mucho más fácil de mantener y de entender, 
menos dolores de cabeza depurando el código intentando descubrir qué hilo está mutando tu clase 
y hace que tu programa falle a las 4 de la mañana.

Pero, el problema de la programación funcional, al menos como yo lo veo, es que viene acompañada 
con un montón de conceptos complejos que vienen de la teoría de categorías (una rama de las 
matemáticas que se podría decir que son las matemáticas de las matemáticas) y que honestamente 
asustan, y mucho.

Efectivamente, me refiero a las monadas y otros palabros bizarros como los tipos de datos algebraicos, 
functor, monoide, etc... Pero al fin y al cabo, ¿qué problema hay? Una mónada es un monoide en la 
categoría de los endofuntores. ¿No está claro? (perdón, es una broma casi obligatoria en estos momentos).

Mi charla trataba precisamente de eso, introducir a los programadores en conceptos de la programación
funcional de una manera que resultara una consecuencia inevitable y sin usar palabras intimidantes. 
Y este articulo trata de plasmar lo que conté en esa charla.

Me saltaré la introducción e iré directo al grano.

Imaginemos que queremos escribir un programa muy sencillo, que:

1. Pregunte al usuario cómo se llama.
1. Imprima por pantalla un saludo usando el nombre del usuario.
1. Fin.

```
=> What's your name?
<= Toni
=> Hello Toni!
```

En programación tradicional (direct style) en Java simplemente haríamos esto:

```java
void main() {
  IO.println("What's your name?");
  var name = IO.readln();
  IO.println("Hello " + name + "!");
}
```

Esto es trivial, pero podríamos decir que este programa tiene algunos problemas, cómo por ejemplo,
¿cómo probaríamos que el programa es correcto?

Podríamos simplemente capturar la salida y entrada estandar, pero IMO no es una solución demasiado
elegante.

Podríamos crear un interfaz `Console` para abstraernos de la interacción con la entrada y salida
estandar. Testearlo sería trivial usando mocks.

Pero demos otro paso más y definamos un DSL `Console` con dos operaciones `ReadLine` y `WriteLine`.
Esto lo haríamos con un sealed interface y dos records. Fácil y sencillo.

Hay que tener en cuenta que cuando definimos un DSL ya no estamos ejecutando el programa directamente,
sino haciendo una descripción del mismo pero usando una estructura de datos. Luego este "programa"
habrá que interpretarlo.

```java
sealed interface Console {
  record WriteLine(String line) implements Console {}
  record ReadLine() implements Console {}
}
```

Ahora podríamos intentar combinarlo todo de una manera un tanto ingenua:

```java
var program = List.of(
    new WritleLine("What's your name?"),
    new ReadLine(),
    new WriteLine("Hello " + name + "!")
  );
```

Está claro que esto no va funcionar ya que la salida de `ReadLine` tiene que recibirla la siguiente 
operación.

Podríamos intentar apañarlo de alguna manera, pero intentemos con una técnica que está muy extendida en
el mundo de la programación, que es el [Continuation Passing Style](https://en.wikipedia.org/wiki/Continuation-passing_style).

```java
sealed interface Console {
  record WriteLine(String line, Console next) implements Console {}
  record ReadLine(Function<String, Console> next) implements Console {}
}
```

Cada operación tiene una referencia a la siguiente operación. En el caso de `ReadLine` next representa 
‘qué hacer después’, pero como depende del input del usuario, lo modelamos como una función.

Ahora nuestro programa ahora tendría este aspecto:

```java
void main() {
  var program = 
    new WriteLine("What's your name?", 
      new ReadLine(
        name -> new WriteLine("Hello " + name + "!", null)));
}
```

Ahora esto ya tiene más sentido, no? La primera operación `WriteLine` que pregunta el nombre del 
usuario tiene una referencia a la siguiente operación, que es leer el nombre del usuario. `ReadLine` 
tiene como referencia a la siguiente operación que realmente es una función que recibe como entrada 
lo que el usuario ha escrito por consola. Finalmente la última operación toma el valor introducido
por el usuario y finalmente pinta por consola el saludo. El último, `WriteLine` como no hay más 
operaciones, simplemente pasamos un `null`.

Cómo evaluamos el programa? Implementémoslo:

```java
default String eval() {
  return switch (this) {
    case WriteLine(var line, var next) -> {
      IO.println(line);
      yield next.eval();
    }
    case ReadLine(var next) -> {
      var line = IO.readln();
      yield next.apply(line).eval();
    }
    case null -> null;
  };
}
```

Aquí usamos pattern matching, switch expressions, y ya que anteriormente habíamos usado un 
sealed interface, no necesitamos definir un `default` case.

Hay que tener en cuenta que seguimos llamando directamente a `IO.println` y `IO.readln`, por lo que
tendremos el mismo problema para probar nuestro programa, pero podríamos definir otro método eval 
para testing y probar que nuestro programa es correcto.

Continuemos. Creo que podríamos hacerlo mejor, esta vez en lugar de referenciar las operaciones 
directamente (como en CPS), vamos a separar las operaciones de la forma de combinarlas:

```java
sealed interface Console {
  record WriteLine(String line) implements Console {}
  record ReadLine() implements Console {}
  record AndThen(
    Console current, Function<String, Console> next) implements Console {}
}
```

Como he dicho, en lugar de tener una referencia en cada operación a la siguiente, tenemos una
operación específica para combinar dos operaciones. Ahora nuestro programa tendría este aspecto:

```java
void main() {
  var program = 
    new AndThen(
      new AndThen(
        new WriteLine("What's your name?"), 
        _ -> new ReadLine()), 
        name -> new WriteLine("Hello " + name + "!"));
}
```

Esta claro que esto es mucho más dificil de entender así a primera vista, pero podemos arreglarlo 
fácilmente definiendo una función `andThen`:

```java
Console andThen(Function<String, Console> next) {
  return new AndThen(this, next);  
}
```

Ahora nos queda algo así, creo que es mucho más fácil de entender de esta manera:

```java
void main() {
  var program = 
    new WriteLine("What's your name?")
      .andThen(_ -> new ReadLine())
      .andThen(name -> new WriteLine("Hello " + name + "!"));  
}
```

Y ahora necesitamos una función para evaluar nuestro programa:

```java
default String eval() {
  return switch (this) {
    case WriteLine(var line) -> {
      IO.println(line);
      yield null;
    }
    case ReadLine _ -> IO.readln();
    case AndThen(var current, var next) -> next.apply(current.eval()).eval();
  };
}
```

`AndThen` es ahora el pegamento de nuestro DSL que nos permite combinar una operación con otra. Vale,
pero ¿qué significa esto exactamente? Pues que hemos creado nuestra propia mónada de una manera
intuitiva, no es más que una forma estructurada de encadenar operaciones donde cada paso puede 
depender del anterior.

Ahora bien, todavía esto es algo muy **ad hoc** para nuestro pequeño programa, pero en el 
siguiente artículo seguiremos profundizando en el tema sacando factor común y poder combinar 
diferentes DSLs en un mismo programa.