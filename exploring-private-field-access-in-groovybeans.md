# Exploring private field access in GroovyBeans
> 22/02/2012

Trying to follow the [Pragmatic Programmer's tip](http://pragprog.com/the-pragmatic-programmer) to learn at least one new programming language every year, I've started learning Groovy. Groovy is a dynamic programming language targeting the Java Virtual Machine. Groovy's syntax is (almost) a superset of Java's, so Java veterans can start experimenting with Groovy using its familiar Java-like syntax and after understanding the basics they can lean toward the powerful dynamic and functional features of Groovy, which are not present in Java.

Groovy comes with [GroovyBeans](http://groovy.codehaus.org/Groovy+Beans), an alternative to JavaBeans. GroovyBeans simplify the JavaBeans model in automatic generation of getters and setters, among others. Let's have this simple Groovy class with two properties and one private field defined as follows:

```groovy
class Employee {
    def firstName
    def lastName
    private salary

    String toString() {
        return "$firstName $lastName: $salary"
    }
}


def employee1 = new Employee()
employee1.setFirstName("Lukas")
employee1.setLastName("Sembera")
//employee1.setSalary(1000) //fail - no setSalary(value) has been generated
println employee1
```

In Groovy, if we don't specify any access modifier, Groovy automatically creates a property (i.e. a field with corresponding getter and setter). Hence here, in the `Employee` class, the getter and setter are generated for `firstName` and `lastName`, whereas for `salary` they are not, because `salary` is defined as private field.

Nothing surprising so far, string `"Lukas Sembera: null"` is printed to the console.

Let's try a short modification of the code above and try to access the private `salary` field directly:

```groovy
def employee2 = new Employee()
employee2.setFirstName("Lukas")
employee2.setLastName("Sembera")
employee2.salary = 1000
println employee2
```

This code successfully compiles and produces a very surprising result: `"Lukas Sembera: 1000"`. How is this possible? IntelliJ IDEA displays a short warning that "Access to 'salary' exceeds its access rights", but the code works! Field `salary` is defined as private and I'm accessing it from outside of the class, so why doesn't it fail? When I saw it for the first time, it was a shocker, I couldn't believe my eyes and started looking for errors in my code. After a short googling, however, I've found out that it's a [well-known bug](http://jira.codehaus.org/browse/GROOVY-1875) in the current Groovy implementation, which is somehow related to the way how closures work.


An interesting situation also arises when we use it in combination with another powerful Groovy feature: named constructor parameters. Having the class `Employee` defined as above, we can create new instances like this:

```groovy
def employee3 = new Employee(salary:2000, firstName: "Lukas")
println employee3
```

I don't know if this is a feature or a consequence of the bug, but I consider such behavior as highly undesirable. From my (conservative) point of view, private methods should remain private. Of course, in Java it's also possible to access private fields from outside the object using the reflection API (and thanks to frameworks like Spring of Hibernate it's even a normal practice). In Groovy, however, the access in entirely unrestricted and can easily lead to accidental modifications and unpredicted results, especially in code involving third-party libraries. Groovy, in the strictest sense, is not an object oriented language because it doesn't support encapsulation, which is the fundamental concept of OOP.

I was wondering how to deal with this unfortunate bug. I will definitely continue using Groovy because it is a wonderful, expressive language with great Java interoperability and powerful application frameworks, such as [Grails](http://grails.org/) or [Griffon](http://griffon.codehaus.org/). The first option is just make everything public and keep in mind that nothing is secret and all the data can be changed anytime. Alternatively, you can create some naming conventions for private fields and obey these rules in the code. For example, the class Employee could be rewritten as follows:

```groovy
class Employee {
    def firstName
    def lastName
    private _salary

    String toString() {
        return "$firstName $lastName: $_salary"
    }
}
```

Now it's obvious that all properties prefixed with underscore are private and should not be touched from outside the object. I will probably go this way in my applications, until the bug is finally fixed.

Just as a final note, I'd add that it's not a good idea to take advantage of this bug in a production code because it's not a language concept. It's just a bug and once it's fixed, all the code relying on the access to private fields will be broken.
