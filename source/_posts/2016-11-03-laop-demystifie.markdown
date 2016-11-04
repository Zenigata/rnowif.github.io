---
layout: post
title: "L'AOP démystifié"
date: 2016-11-03 08:55:27 +0200
comments: true
categories: ["Design Pattern", "Java", "AOP"]
---

L'_Aspect Oriented Programming_ est un paradigme de programmation qui permet de traiter les préoccupations transverses (ou _cross-cutting concerns_ en anglais) – tels le _logging_, le cache ou les transactions – séparément du code métier.

L'AOP repose généralement sur l'utilisation de librairies ou de _frameworks_ qui rendent ce type de programmation assez obscur, voire difficile à comprendre. Mais il n'en est rien. Explications.

<!-- more -->

L'ensemble du code montré dans cet article est accessible sur ce dépôt [GitHub](https://github.com/rnowif/demo-aop).

## Pourquoi l'AOP ?

Tout d'abord, partons d'une classe qui implémente la [suite de Fibonacci](https://fr.wikipedia.org/wiki/Suite_de_Fibonacci) et dont on veut estimer le temps d'exécution.
De manière naïve, il serait possible de calculer et logger ce temps directement dans la méthode ([voir le commit](https://github.com/rnowif/demo-aop/tree/63660763cdb1f1d97e3ba63e9c5a271a60d53a99)):

```java
public class FibonacciCalculatorImpl implements FibonacciCalculator {

    private static final Logger LOG = LoggerFactory.getLogger(FibonacciCalculatorImpl.class);

    @Override
    public int calculate(int n) {
        StopWatch watch = new StopWatch(Clock.systemDefaultZone());
        watch.start();
        LOG.info("Start calculation for number {}", n);
        int result = n;
        if (n > 1) {
            result = calculate(n-1) + calculate(n-2);
        }
        LOG.info("End calculation for number {} ({} ms)", n, watch.elapsed());
        return result;
    }
}
```

Il y a deux problèmes majeurs dans cette implémentation : le _tangling_ et le _scattering_.

### _Code tangling_

Le _code tangling_ signifie que le code imbrique à tort différentes notions. Ici, la mesure du temps et le code métier. On pourrait en imaginer d'autres comme la gestion des transactions, le cache ou le _monitoring_.

Ceci est problématique car le code métier – qui apporte la vraie valeur ! – est noyé dans du code de « plomberie » qui peut finir par prendre beaucoup plus de place que le code initial.

### _Code scattering_

Le _code scattering_ signifie que le code non métier sera sans nul doute répété dans beaucoup de méthodes. On voit alors émerger un grand nombre de duplications de code qui le rendent très difficile à maintenir.

## Le pattern _Proxy_

La première amélioration évidente qu'il est possible d'apporter est d'extraire la logique de calcul du temps dans une classe spécifique, qui va implémenter la même interface que `FibonacciCalculatorImpl` et déléguer l'appel de la méthode `calculate` à une instance de `FibonacciCalculatorImpl`. Il s'agit du pattern _Proxy_ ([voir le commit](https://github.com/rnowif/demo-aop/tree/a4d00f2b5b0a2cb28196a738ece906560546bda5)).

```java
public class TimedFibonacciCalculator implements FibonacciCalculator {

    private static final Logger LOG = LoggerFactory.getLogger(FibonacciCalculatorImpl.class);

    private final FibonacciCalculator calculator;

    public TimedFibonacciCalculator(FibonacciCalculator calculator) {
        this.calculator = calculator;
    }

    @Override
    public int calculate(int n) {
        StopWatch watch = new StopWatch(Clock.systemDefaultZone());
        watch.start();
        LOG.info("Start calculation for number {}", n);
        int result = calculator.calculate(n); // Delegate calculation
        LOG.info("End calculation for number {} ({} ms)", n, watch.elapsed());
        return result;
    }
}
```

L'utilisation de cette classe se fait comme suit :

```java
FibonacciCalculator calculator = new TimedFibonacciCalculator(new FibonacciCalculatorImpl());
LOG.info("10th fibonacci number : {}", calculator.calculate(10));
```

Grâce à ce _pattern_, le problème du _code tangling_ est réglé mais pas du tout celui du _code scattering_. En effet, la classe `TimedFibonacciCalculator` n'est pas vraiment réutilisable car elle est fortement couplée à l'interface `FibonacciCalculator`.

## Externalisation de la logique de chronométrage

Pour résoudre le _code scattering_, il faut extraire la logique de mesure du temps et lui permettre d'appeler n'importe quelle méthode ([voir le commit](https://github.com/rnowif/demo-aop/tree/fb545b1a3af8e34b37e8f93f4abf97b57ae75f33)). Pour cela, on peut définir une interface fonctionnelle générique qui contient une méthode `invoke`.

```java
@FunctionalInterface
public interface InvocationPoint<T> {

    T invoke() throws Exception;
}
```

Cette interface pourra alors être injectée dans une classe réutilisable, permettant de mesurer le temps nécessaire à l'exécution de la méthode `invoke` :

```java
public class TimedInvocation {

    private static final Logger LOG = LoggerFactory.getLogger(TimedInvocation.class);

    public <T> T invoke(InvocationPoint<T> invocationPoint) {
        StopWatch watch = new StopWatch(Clock.systemDefaultZone());
        watch.start();
        LOG.info("Start calculation");
        T result = invocationPoint.invoke();
        LOG.info("End calculation ({} ms)", watch.elapsed());
        return result;
    }
}
```

Le _proxy_ ne sert donc plus que de passe-plat pour appeler cette classe (l'interface `InvocationPoint` peut aisément être implémentée par un lambda) :

```java
public class TimedFibonacciCalculator implements FibonacciCalculator {

    private final FibonacciCalculator calculator;

    public TimedFibonacciCalculator(FibonacciCalculator calculator) {
        this.calculator = calculator;
    }

    @Override
    public int calculate(int n) {
        return new TimedInvocation().invoke(
                () -> calculator.calculate(n)
        );
    }
}
```

On remarque ici que le message de log est plus générique : il n'est plus possible de tracer l'index de la suite que l'on veut calculer. En effet, ce log peut désormais permettre de mesurer le temps d'exécution de n'importe quelle méthode.

## Génération automatique de _proxy_

Le _tangling_ et le _scattering_ sont éliminés mais il faut tout de même créer une classe de _proxy_ pour chaque méthode que l'on souhaite chronométrer. Le mécanisme de _JDKProxy_, inclus directement dans le JDK, permet de créer des _proxys_ dynamiques à partir d'une interface (`FibonacciCalculator`) et d'une implémentation de `InvocationHandler` ([voir le commit](https://github.com/rnowif/demo-aop/tree/88dda77b942cc9370762d3e2efb79b6b9c0c6aeb)). Celle-ci contient une méthode `invoke(Object proxy, Method method, Object[] args): Object` qui prend en paramètre le proxy, la méthode qui a été appelée ainsi que ses arguments, et doit retourner le résultat de l'appel de la méthode.

Le plus simple des `InvocationHandler`, qui ne fait strictement rien, peut s'écrire comme ceci :

```java
public class SimpleInvocationHandler implements InvocationHandler {

    private final Object target;

    public SimpleInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return method.invoke(target, args);
    }
}
```

La création du proxy devient donc :

```java
FibonacciCalculator calculator = (FibonacciCalculator) Proxy.newProxyInstance(
  FibonacciCalculator.class.getClassLoader(),
  new Class[] {FibonacciCalculator.class},
  new SimpleInvocationHandler(new FibonacciCalculatorImpl())
);
```

Dans notre cas, le handler va devoir calculer le temps d'exécution de la méthode. Il s'écrit ainsi :

```java
public class TimedInvocationHandler implements InvocationHandler {

    private final Object target;

    public TimedInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return new TimedInvocation().invoke(
                () -> method.invoke(target, args)
        );
    }
  }
}
```

## Activer ou désactiver la mesure du temps

Pour faire comme les pros _(sic)_, il est possible d'activer ou de désactiver la mesure du temps sur une méthode en y ajoutant une annotation `@Timed` qui servira de marqueur.

```java
public class FibonacciCalculatorImpl implements FibonacciCalculator {

    @Timed
    @Override
    public int calculate(int n) {
        int result = n;
        if (n > 1) {
            result = calculate(n-1) + calculate(n-2);
        }
        return result;
    }
}
```

Dans le _handler_, il suffit alors d'utiliser la réflection pour déterminer si la méthode est annotée ou non, et auquel cas le temps d'exécution est mesuré :

```java
public class TimedInvocationHandler implements InvocationHandler {
    private static final Class<Timed> ANNOTATION_CLASS = Timed.class;

    private final Object target;

    public TimedInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (isTimed(method)) {
            return new TimedInvocation().invoke(
                    () -> method.invoke(target, args)
            );
        }

        // Method not annotated, just invoke target
        return method.invoke(target, args);
    }

    private boolean isTimed(Method method) throws NoSuchMethodException {
        // Is timed if the interface method or the target method is annotated
        return method.getDeclaredAnnotation(ANNOTATION_CLASS) != null
                || target.getClass().getDeclaredMethod(method.getName(), method.getParameterTypes()).getDeclaredAnnotation(ANNOTATION_CLASS) != null;
    }
}
```

Pour rappel, une annotation n'est qu'un marqueur. La magie n'est pas dans l'annotation mais dans le code qui l'utilise !

## Conclusion

### À propos de l'AOP

Cet article a, de manière assez rudimentaire, permis d'implémenter un mécanisme d'AOP sans avoir recours à un quelconque _framework_. Ce faisant, il a permis de comprendre certains aspects et limitations de l'AOP :

- Pour utiliser l'AOP avec des interfaces, il est possible d'utiliser le mécanisme de _JDKProxy_, inclus directement dans le JDK.
- Pour utiliser l'AOP sans interface, le proxy dynamique doit étendre la classe à proxifier (il est possible d'utiliser une librairie comme [CGLib](https://github.com/cglib/cglib) pour faire ça).
- L'AOP ne peut pas fonctionner sur des méthodes ou des classes privées ou finales. En effet, la méthode doit être définie dans une interface ou pouvoir être surchargée.
- Si la classe à proxifier appelle directement une méthode de sa propre classe, l'AOP ne s'applique pas. Ici, la méthode `calculate` est récursive. Cependant, la mesure du temps ne s'opère que si la méthode est appelée à travers le proxy. Donc, seul l'appel de la méthode par « l'extérieur » est chronométré.

### AspectJ et Spring AOP

[AspectJ](http://www.eclipse.org/aspectj/) est un _framework_ qui permet de définir de manière très précise les méthodes à proxifier pour y apporter du comportement. Il est également possible de ne définir du comportement qu'avant ou après l'exécution de la méthode. Cependant, le mécanisme sous-jacent est exactement le même que celui décrit dans cet article. Spring, quant à lui, utilise abondamment l'AOP grâce à AspectJ pour enrichir les _beans_ avec du comportement transverse (via des annotations comme `@Transactional` ou `@Cacheable`). Il va parcourir l'ensemble des _beans_ du contexte d'application et les remplacer par des _proxys_ dynamiques. Ainsi, les _beans_ que l'application manipule ne sont pas les implémentations brutes, mais des _proxys_.

### Note finale

L'AOP n'est pas magique, ni très complexe. Cependant il est fortement utilisé dans les _frameworks_ de type Spring. Une bonne compréhension de ses mécanismes et de son implémentation permet de faciliter le dévelopement et le debug des applications d'entreprise.
