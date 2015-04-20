### What's the reason I can't create generic array types in Java?

private T[] elements = new T[initialCapacity];

It's because Java's arrays (unlike generics) contain, at runtime, information about its component type. So you must know the component type when you create the array. Since you don't know what T is at runtime, you can't create the array.

[link](http://stackoverflow.com/questions/2927391/whats-the-reason-i-cant-create-generic-array-types-in-java)
