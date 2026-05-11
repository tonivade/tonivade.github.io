---
layout: post
title:  Test unitarios con JBang
date:   2026-05-11 22:00:00
categories: programming java jbang unit-testing
---

Llevo estos días preparando la charla que voy a dar en la commit-conf este año y me he
encontrado con una cosa que no conocía y que me ha parecido bastante interesante: es posible
ejecutar test unitarios JUnit con JBang.

Si no conocéis [JBang](https://www.jbang.dev/) os recomiendo que le echéis un vistazo. Este
es un ejemplo sacado de la documentación oficial.

```java
///usr/bin/env jbang "$0" "$@" ; exit $?

//JAVA 25+
//DEPS com.github.lalyos:jfiglet:0.0.8

import com.github.lalyos.jfiglet.FigletFont;

void main() throws Exception {
  IO.println(FigletFont.convertOneLine("Hello JBang!"));
}
```

Si instaláis jbang, copiáis este código en un fichero llamado hello.java, luego lo podéis
ejecutar con el comando `jbang hello.java`. Y esto es lo que sacaría por consola:

```
  _   _          _   _                 _   ____                            _ 
 | | | |   ___  | | | |   ___         | | | __ )    __ _   _ __     __ _  | |
 | |_| |  / _ \ | | | |  / _ \     _  | | |  _ \   / _` | | '_ \   / _` | | |
 |  _  | |  __/ | | | | | (_) |   | |_| | | |_) | | (_| | | | | | | (_| | |_|
 |_| |_|  \___| |_| |_|  \___/     \___/  |____/   \__,_| |_| |_|  \__, | (_)
                                                                   |___/     
```

Si le dais permiso de ejecución al fichero `chmod +x hello.java`, se puede ejecutar 
directamente como si fuera cualquier otro comando `./hello.java`.

Todo está muy chulo, pero se pueden hacer cosas todavía más chulas. Como por ejemplo
ejecutar tests unitarios.

```
//DEPS org.assertj:assertj-core:3.27.7
//DEPS org.junit.jupiter:junit-jupiter:6.0.3

//JAVA 25+
//SOURCES Program.java,Either.java

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.function.Function;

import org.junit.jupiter.api.Test;

public class ProgramTest {

  @Test
  void shouldCalculateFibSequenceRecursive() {
    var program = fibMemoized.apply(20);

    var result = program.eval(null);

    assertEquals(Either.right(10946), result);
  }

  @Test
  void shouldCalculateFibSequenceNonRecursive() {
    var result = fib(20);

    assertEquals(10946, result);
  }

  static Function<Integer, Program<Void, Void, Integer>> fibMemoized = Program.memoize(
      new Function<Integer, Program<Void, Void, Integer>>() {
        @Override
        public Program<Void, Void, Integer> apply(Integer n) {
          if (n < 2) {
            return Program.success(1);
          }
          var fib2 = Program.suspend(() -> fibMemoized.apply(n - 2));
          var fib1 = Program.suspend(() -> fibMemoized.apply(n - 1));
          return Program.zip(fib2, fib1, Integer::sum);
        }
      });

  static int fib(int n) {
    if (n < 2) {
      return 1;
    }
    int a = 1, b = 1;
    for (int i = 2; i <= n; i++) {
      int temp = a + b;
      a = b;
      b = temp;
    }
    return b;
  }
}
```

Aquí tenemos un test unitario creado para la librería en la que hemos estado trabajando
en mis últimos artículos.

Añadimos con la directiva `//DEP` las dependencias a junit y assertj usando las coordenadas
de maven, de una manera muy similar a como hacemos en gradle.

Con `//SOURCES` podemos añadir otros fichero java para que se compilen junto a la clase de
test.

La directiva `//JAVA 25+` lo que hace jbang es ver si tienes instalado una jdk compatible
con la versión que necesitas. Si usas [sdkman](https://sdkman.io/), jbang buscará entre todas las versiones
de java que tengas una compatible.

Ahora bien, ¿cómo los ejecutamos?

Primero tenemos que compilar los tests:

```shell
jbang build ProgramTest.java
```

Esto generará un fichero jar con esta estructura:

```
     0 Mon May 11 23:33:58 CEST 2026 META-INF/
    79 Mon May 11 23:33:58 CEST 2026 META-INF/MANIFEST.MF
  1653 Mon May 11 23:33:58 CEST 2026 Either$Left.class
  1661 Mon May 11 23:33:58 CEST 2026 Either$Right.class
  2329 Mon May 11 23:33:58 CEST 2026 Either.class
     0 Mon May 11 23:33:58 CEST 2026 META-INF/maven/
     0 Mon May 11 23:33:58 CEST 2026 META-INF/maven/group/
  2302 Mon May 11 23:33:58 CEST 2026 META-INF/maven/group/pom.xml
  1911 Mon May 11 23:33:58 CEST 2026 Program$Access.class
  1710 Mon May 11 23:33:58 CEST 2026 Program$Failure.class
  2229 Mon May 11 23:33:58 CEST 2026 Program$FlatMap.class
  2264 Mon May 11 23:33:58 CEST 2026 Program$FlatMapError.class
  1764 Mon May 11 23:33:58 CEST 2026 Program$Memoized.class
  1710 Mon May 11 23:33:58 CEST 2026 Program$Success.class
 12389 Mon May 11 23:33:58 CEST 2026 Program.class
  2511 Mon May 11 23:33:58 CEST 2026 ProgramTest$1.class
  1923 Mon May 11 23:33:58 CEST 2026 ProgramTest.class
```

Ahora ya podemos ejecutar el test, lo podemos hacer de esta forma:

```shell
jbang junit@junit-team execute --class-path `jbang info classpath ProgramTest.java` --scan-classpath
```

El resultado será el siguiente:

```
Thanks for using JUnit! Support its development at https://junit.org/sponsoring

╷
├─ JUnit Platform Suite ✔
├─ JUnit Jupiter ✔
│  └─ ProgramTest ✔
│     ├─ shouldCalculateFibSequenceRecursive() ✔
│     └─ shouldCalculateFibSequenceNonRecursive() ✔
└─ JUnit Vintage ✔

Test run finished after 112 ms
[         4 containers found      ]
[         0 containers skipped    ]
[         4 containers started    ]
[         0 containers aborted    ]
[         4 containers successful ]
[         0 containers failed     ]
[         2 tests found           ]
[         0 tests skipped         ]
[         2 tests started         ]
[         0 tests aborted         ]
[         2 tests successful      ]
[         0 tests failed          ]
```

Cuando un test falla este es el resultado:

```
Thanks for using JUnit! Support its development at https://junit.org/sponsoring

╷
├─ JUnit Platform Suite ✔
├─ JUnit Jupiter ✔
│  └─ ProgramTest ✔
│     ├─ shouldCalculateFibSequenceRecursive() ✘ expected: <Right[right=109460]> but was: <Right[right=10946]>
│     └─ shouldCalculateFibSequenceNonRecursive() ✔
└─ JUnit Vintage ✔

Failures (1):
  JUnit Jupiter:ProgramTest:shouldCalculateFibSequenceRecursive()
    MethodSource [className = 'ProgramTest', methodName = 'shouldCalculateFibSequenceRecursive', methodParameterTypes = '']
    => org.opentest4j.AssertionFailedError: expected: <Right[right=109460]> but was: <Right[right=10946]>
       org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:158)
       org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:139)
       org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:201)
       org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:184)
       org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:179)
       org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1188)
       ProgramTest.shouldCalculateFibSequenceRecursive(ProgramTest.java:21)

Test run finished after 140 ms
[         4 containers found      ]
[         0 containers skipped    ]
[         4 containers started    ]
[         0 containers aborted    ]
[         4 containers successful ]
[         0 containers failed     ]
[         2 tests found           ]
[         0 tests skipped         ]
[         2 tests started         ]
[         0 tests aborted         ]
[         1 tests successful      ]
[         1 tests failed          ]
```

Y no solo eso podemos ejecutar tests, podemos también ejecutar micro benchmarks
del tipo [JMH](https://github.com/openjdk/jmh). Por ejemplo:

```java
///usr/bin/env jbang "$0" "$@" ; exit $?

//DEPS org.openjdk.jmh:jmh-core:1.37
//DEPS org.openjdk.jmh:jmh-generator-annprocess:1.37

//SOURCES ProgramTest.java,Program.java,Either.java

//JAVA 25+

//JAVAC_OPTIONS -processor org.openjdk.jmh.generators.BenchmarkProcessor

package test;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.*;
import org.openjdk.jmh.runner.options.*;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class Bench {

  private String text = "hello";

  @Benchmark
  public Integer iterative() {
    return ProgramTest.fib(20);
  }

  @Benchmark
  public Either<Void, Integer> recursive() {
    return ProgramTest.fibMemoized.apply(20).eval(null);
  }

  public static void main(String[] args) throws Exception {
    Options opt = new OptionsBuilder()
      .include(Bench.class.getSimpleName())
      .forks(1)
      .build();

    new Runner(opt).run();
  }
}
```

Si le damos permiso de ejecución al archivo `chmod +x Bench.java`, y lo ejecutamos
`./Bench.java` se ejecutará el benchmark.

Aquí lo único especial es la directiva `//JAVAC_OPTIONS`, que la necesitamos añadir
ya que JMH genera código a partir de las anotaciones y hay añadir explícitamente
el gestor de anotaciones para genere todo el código necesario.

```
[jbang] Building jar for Bench.java...
WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
WARNING: sun.misc.Unsafe::objectFieldOffset has been called by org.openjdk.jmh.util.Utils (file:/home/vant/.m2/repository/org/openjdk/jmh/jmh-core/1.37/jmh-core-1.37.jar)
WARNING: Please consider reporting this to the maintainers of class org.openjdk.jmh.util.Utils
WARNING: sun.misc.Unsafe::objectFieldOffset will be removed in a future release
# JMH version: 1.37
# VM version: JDK 25.0.3, OpenJDK 64-Bit Server VM, 25.0.3+9-LTS
# VM invoker: /home/vant/.sdkman/candidates/java/25.0.3-tem/bin/java
# VM options: <none>
# Blackhole mode: compiler (auto-detected, use -Djmh.blackhole.autoDetect=false to disable)
# Warmup: 5 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: test.Bench.iterative

# Run progress: 0,00% complete, ETA 00:03:20
# Fork: 1 of 1
WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
WARNING: sun.misc.Unsafe::objectFieldOffset has been called by org.openjdk.jmh.util.Utils (file:/home/vant/.m2/repository/org/openjdk/jmh/jmh-core/1.37/jmh-core-1.37.jar)
WARNING: Please consider reporting this to the maintainers of class org.openjdk.jmh.util.Utils
WARNING: sun.misc.Unsafe::objectFieldOffset will be removed in a future release
# Warmup Iteration   1: 1,514 ns/op
# Warmup Iteration   2: 1,743 ns/op
# Warmup Iteration   3: 1,754 ns/op
# Warmup Iteration   4: 1,671 ns/op
# Warmup Iteration   5: 1,859 ns/op
Iteration   1: 1,750 ns/op
Iteration   2: 1,815 ns/op
Iteration   3: 1,708 ns/op
Iteration   4: 1,725 ns/op
Iteration   5: 1,657 ns/op


Result "test.Bench.iterative":
  1,731 ±(99.9%) 0,224 ns/op [Average]
  (min, avg, max) = (1,657, 1,731, 1,815), stdev = 0,058
  CI (99.9%): [1,507, 1,955] (assumes normal distribution)


# JMH version: 1.37
# VM version: JDK 25.0.3, OpenJDK 64-Bit Server VM, 25.0.3+9-LTS
# VM invoker: /home/vant/.sdkman/candidates/java/25.0.3-tem/bin/java
# VM options: <none>
# Blackhole mode: compiler (auto-detected, use -Djmh.blackhole.autoDetect=false to disable)
# Warmup: 5 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: test.Bench.recursive

# Run progress: 50,00% complete, ETA 00:01:40
# Fork: 1 of 1
WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
WARNING: sun.misc.Unsafe::objectFieldOffset has been called by org.openjdk.jmh.util.Utils (file:/home/vant/.m2/repository/org/openjdk/jmh/jmh-core/1.37/jmh-core-1.37.jar)
WARNING: Please consider reporting this to the maintainers of class org.openjdk.jmh.util.Utils
WARNING: sun.misc.Unsafe::objectFieldOffset will be removed in a future release
# Warmup Iteration   1: 2282,098 ns/op
# Warmup Iteration   2: 2462,642 ns/op
# Warmup Iteration   3: 2353,671 ns/op
# Warmup Iteration   4: 2388,340 ns/op
# Warmup Iteration   5: 2407,009 ns/op
Iteration   1: 2411,544 ns/op
Iteration   2: 2308,276 ns/op
Iteration   3: 2444,991 ns/op
Iteration   4: 2417,737 ns/op
Iteration   5: 2324,341 ns/op


Result "test.Bench.recursive":
  2381,378 ±(99.9%) 234,824 ns/op [Average]
  (min, avg, max) = (2308,276, 2381,378, 2444,991), stdev = 60,983
  CI (99.9%): [2146,554, 2616,202] (assumes normal distribution)


# Run complete. Total time: 00:03:20

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

NOTE: Current JVM experimentally supports Compiler Blackholes, and they are in use. Please exercise
extra caution when trusting the results, look into the generated code to check the benchmark still
works, and factor in a small probability of new VM bugs. Additionally, while comparisons between
different JVMs are already problematic, the performance difference caused by different Blackhole
modes can be very significant. Please make sure you use the consistent Blackhole mode for comparisons.

Benchmark        Mode  Cnt     Score     Error  Units
Bench.iterative  avgt    5     1,731 ±   0,224  ns/op
Bench.recursive  avgt    5  2381,378 ± 234,824  ns/op
```

Como hemos podido ver, jbang es una herramienta estupenda. Puedes usarlo como
un gestor de paquetes, para crear scripts en Java, ejecutar tests unitarios
y micro benchmarks.