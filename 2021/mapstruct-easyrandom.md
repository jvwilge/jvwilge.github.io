Unit testing your MapStruct mapper for omitted parameters - EasyRandom to the rescue
====

MapStruct is sometimes a bit blunt regarding duplicate parameter names. Sometimes you don't even notice a parameter name is duplicate (with nested objects for example). This can result in an incorrect mapping and can take a lot of debugging time. In this article I'll show you how you can add a safety net to your mappers.


------------------


How can this problem occur
---

We recently ran into an issue where a MapStruct mapper produced an incorrect result. The cause was a parameter name that was used on a nested object and a parameter on the mapper itself.

Let's say we want have a Person with a field name and age, A standalone String called name and a PersonDto with a name and age. We create a Mapper to map the standalone String and the Person to a PersonDto.

```java
@Mapper
public interface PersonMapper {

    PersonDto toPersonDto(String name, Person person);

}
```

Which name will occur on the PersonDto?
If we go into the generated code (one of the reasons to use MapStruct is that you can easily view the generated code) we'll see that Person.name is used. This is clearly not our intention, why would we add a name field otherwise?

Imagine someone adds the name field on Person after the Mapper is running in production, do you think you will catch this mistake?

Since this mistake was easy to make and can be hard to spot I tried to figure out how we can prevent these kind of issues.

Naming conventions
---

I'll stick to the MapStruct naming. So the input parameters I'll call source and the output/result target.

The concept
---

What I basically wanted to do is create a random object for all the source parameters of the mapping. Since creating manual objects is tedious and requires more maintenace. So we need to set a bunch of random parameters and even set the deeply nested. The target object we'call the reference object.
Now we'll call the Mapper one time for every parameter with one parameter set to null. The new target object should be different to the reference object otherwise we have a parameter that doesn't do anything (like the Person.name example I mentioned earlier).

So with this concept ready we have to take a step back and prepare a few things first.

<h2>Check your MapStruct configuration first</h2>

Before you start testing it is a good idea to make sure the <code>mapstruct.unmappedTargetPolicy</code> is set to <a href="https://mapstruct.org/documentation/stable/reference/html/#configuration-options" target="_blank"/>ERROR</a>: "any unmapped target property will cause the mapping code generation to fail". This is super annoying at first, but will result in more reliable code and a higher development speed.

<h2>Make sure all our mappers are tested</h2>

When you add a new mapper to your code base you of course want it tested. A first solution would be scan for the @Mapper annotation and initialize the Mappers. This fails with nested mappers and in other ways since some mappers have parameters in the constructor.

Our solution is to count the number of Mappers and initialize them manually and verify the count corresponds with the number of manual initializations.
For this we use <a href="https://www.archunit.org/" target="_blank">ArchUnit</a>.

There are probably some nicer solutions for this (like depending on the autowiring of Spring with MapStruct), but for now this will suffice.

```java
@Test
void testMappers() {

	// we have some built other ArchUnit-checks that make sure we can rely on this filter
    final long mapperCount = new ClassFileImporter().importPackages("com.demo.service.mapper")
            .stream()
            .filter(javaClass -> javaClass.getSimpleName().endsWith("Mapper"))
            .count();

    final List<Object> mappers = List.of(ACCOUNT_MAPPER, PERSON_MAPPER, INVOICE_MAPPER);

    assertThat(mappers)
            .withFailMessage("Scanned number of mappers doesn't match the provided amount")
            .hasSize(Long.valueOf(mapperCount).intValue());

    for (Object mapper : mappers) {
        test(mapper, easyRandom);
    }
}
```

Setting random values with EasyRandom
---

As you might've guessed from the title we use <a href="https://github.com/j-easy/easy-random" target="_blank">EasyRandom</a> to randomize parameters. This a nice library to randomize objects. It is quite hard to do this manually and if there's something the library doesn't support it is easy to extend.

There are quite some articles on how to use EasyRandom so I'll just show you what I did and if you want to now the nitty gritty details Google is your friend.

With inheritance you might run into trouble. You'll probably encounter an error like :

```
- Unable to create a random instance of type class com.demo.Person
- Unable to create a random instance of type interface io.vavr.collection.Seq
```

Let's say we have an abstract Person class with multiple implementations. When using the method described in this article you have to pick an implementation. In this example we'll use the Customer:
```java
EasyRandomParameters parameters = new EasyRandomParameters()
    .randomize(Person.class, () -> easyRandom.nextObject(Customer.class))
```

When you want to leave it up to EasyRandom you can set the setScanClasspathForConcreteTypes to true.

Another thing that can happen is that the constructor of an object expects something to be valid (a count of something for example). You can solve this with a static value :
```java
.randomize(FieldPredicates.named("totalCount").and(inClass(PersonDto.class)), () -> 0)
```

Creating the test code
---

Now the preparations are done we can write the test code.

```java
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.List;

import org.jeasy.random.EasyRandom;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Fail.fail;

public abstract class AbstractAllMapperTest {

    /**
     * Call every public method in Mapper, save the value as a reference value and call the method
     * over and over again each time with a different parameter missing. The result should differ
     * from the reference value.<br/>
     */
    public void test(final Object mapper, final EasyRandom easyRandom) {

        // we only check the methods in the concrete class, so this doesn't work with inheritance, (we don't want the methods of Object.class for example)
        final Method[] declaredMethods = mapper.getClass().getDeclaredMethods();

        Arrays.stream(declaredMethods)
                .filter(method -> Modifier.isPublic(method.getModifiers()))
                .forEach(method1 -> callAndIterateOverParams(method1, mapper, easyRandom));
    }

    private void callAndIterateOverParams(final Method method, final Object mapper, final EasyRandom easyRandom) {
        final Object[] parameters = Arrays.stream(method.getParameters())
                .map(parameter -> easyRandom.nextObject(parameter.getType()))
                .toArray();

        final Object reference;
        try {
            reference = method.invoke(mapper, parameters);
        } catch (Exception e) {
            fail(mapper.getClass().getSimpleName() + "#" + method.getName() + " failed ", e);
            throw new RuntimeException("unreachable");
        }

        for (int i = 0; i < parameters.length; i++) {
            try {
                final Object[] copiedArray = new Object[parameters.length];
                System.arraycopy(parameters, 0, copiedArray, 0, parameters.length);
                copiedArray[i] = null;
                final Object result = method.invoke(mapper, copiedArray);

                assertThat(result)
                        .withFailMessage("Invoking [%s#%s] without parameter [%s] (%d) has the same result as the reference object, " +
                                "did you map the field?", method.getDeclaringClass(), method.getName(), method.getParameters()[i].getName(), i)
                        .isNotEqualTo(reference);
            } catch (Exception e) {
				//ignoring exception
            }
        }
    }
}
```

Since there are usually multiple Mappers in a project I started with an abstract class you can extend from.
The first method test is where you pass the <b>implementation</b> of your Mapper. PersonMapperImpl for example. The second parameter is your EasyRandom configuration. new EasyRandomParameters() for simple cases, but you'll soon discover that you want some configuration here.
This method scans all the public methods in the Mapper and calls callAndIterateOverParams for each method.

CallAndIterateOverParams initializes the reference object and then iterates over all the parameters of the method. In each iteration one parameter is set to null and compared to the reference object. If it differs we're safe and otherwise chances are there is a mapping error.

Example output
---

Now everything is set up we can test the example I mentioned in the introduction:

```
Invoking [class com.demo.PersonMapperImpl#toPersonDto] 
without parameter [name] (0) 
has the same result as the reference object, 
did you map the field?
```

Now it is immediately clear what might be wrong.

Next level
---

The first thing we ran into was that this method of testing can be a bit brittle, so we needed an option to skip a method. We added a method methodsToSkip that returns a list of method names as String and called it from a filter between the existing filters in the test method. For us this works, but when there is method overloading you have to tune this part.

```java
.filter(method -> methodsToSkip().stream().noneMatch(methodName -> methodName.equals(method.getName())))
```

Collections are another issue. The current test code creates an empty collections and this won't cover all the mapping errors.
To fix this we have to extend the part where easyRandom.nextObject is called.

```java
if (ClassUtils.isAssignable(parameter.getType(), Collection.class)) {
    return createRandomCollection(easyRandom, parameter);
}
return easyRandom.nextObject(parameter.getType());
```

This allows us to create a typed collection :

```java
private Iterable<Object> createRandomCollection(final EasyRandom easyRandom, final Parameter parameter) {
    final Type parameterizedType = parameter.getParameterizedType();
    if (parameterizedType.getClass().toString().equals("class sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl")) {
        final Object collectionEntry = createRandomCollectionEntry(easyRandom, parameterizedType);
        if (parameter.getType().equals(List.class)) {
            return List.of(collectionEntry);
        } else {
            throw new NotImplementedException("parameter type not yet implemented: " + parameter.getType());
        }
    }
    throw new NotImplementedException("parameterizedType not yet implemented: " + parameterizedType.getClass());
}
```

This looks (and probably is) a bit dirty. I'm still looking for a cleaner solution, but for now this will suffice.

Finally the createRandomCollectionEntry-method :

```java
private Object createRandomCollectionEntry(final EasyRandom easyRandom, final Type parameterizedType) {
final Type[] result = (Type[]) ReflectionTestUtils.invokeGetterMethod(parameterizedType, "actualTypeArguments");

    try {
        // the split is needed for abstract classes ("? extends" is showed in the String)
        final String[] typeNames = result[0].getTypeName().split(" ");
            return easyRandom.nextObject(Class.forName(typeNames[typeNames.length - 1]));
    } catch (ClassNotFoundException e) {
       e.printStackTrace();
    }
    throw new IllegalStateException("hmm, this should not happen");
}
```

Conclusion
---

Note that this method only works if you have equals and hashCode methods on your objects. This is of course quite obvious, but when preparing a demo without Lombok I found out the hard way ;-)

This construction might be a bit brittle, so make sure there is an escape hatch. The danger of course is that some people will take the easy way out, so be sure your code review process is in order (I even added it to my checklist I check a few times a month).

In our project quite a few errors were caught with a very small amount of false positives, so it might be useful for you project too.

For my own taste there is a little too much reflection and dirty constructions, but it is a quite useful addition to our code base.

Libraries used
---

[MapStruct](https://mapstruct.org/)

[ArchUnit](https://www.archunit.org/)

[Easy-Random](https://github.com/j-easy/easy-random)

> First published on Aug 31, 2021 at [jvwilge.github.io](http://jvwilge.github.io)
> 
> tags : mapstruct, easy-random, java
> 
> language: en