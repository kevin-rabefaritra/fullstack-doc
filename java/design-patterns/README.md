# Design patterns

## Key points
- Reusable solutions for common software design problems.
- 3 types: creational, structural, and behavioral.

## Creational
### Singleton
- Ensure that a class has **only one instance**.
- Useful for classes that manage resources (database connection, sockets, ..).
- Provides global access point to the instance.

```
class MySingleton {
  
  static ResourceClass myResource;

  static ResourceClass getResource() {
    if (myResource == null) {
      // Initialize myResource
    }
    return myResource;
  }
}
```

### Factory
- Create objects without specifying the exact class of object that will be created.
- Location for creation of objects is centralized and its implementation can be changed easily.
- Code is more generic thus reusable.
- Used by `Calendar.getInstance()`, `NumberFormat`, or via the `valueOf()` method.

### Abstract Factory
- Same as Factory but takes advantage of inheritance to return multiple object classes.
- **Factory creates objects of a single class, Abstract Factory creates objects of related classes.**

```
class ComputerFactory {

  /**
  * Suppose we have an abstract class Computer 
  * and two subclasses AsusComputer and SamsungComputer
  **/
  static Computer getComputer(String brand, String model, int ram) {
    if (brand.equals("ASUS")) {
      return new AsusComputer(model, ram);
    }
    else if (brand.equals("Samsung")) {
      return new SamsungComputer(model);
    }
    return null;
  }

}
```

### Builder
- Creation of complex objects in a *step-by-step* manner.
- Allows multiple representations to be created.
- Used by `StringBuilder` and `StringBuffer`.

```
AsusComputer computer = AsusComputer.Builder()
    .brand("ASUS")
    .ram(8)
    .build();
```

## Structural

### Adapter
- Allows 2 incompatible interfaces to work together.
- Used by `List.of` to convert items to `List` object or `Stream.of` to convert items to `Stream` object.
- It's possible to create a *two-way* adapter.

### Bridge
- Separation of abstraction and implementation.
- `MyService` abstract class and `MyServiceImpl` actual implementation.
- Uses **dependency Injection** guidelines.

## Behavioral

### Observer
- Allows an object to notify at set of objects when its state changes.
- Known as *publish - subscribe* pattern.