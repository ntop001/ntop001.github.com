---
layout: post
title: "Java的一些冷知识"
description: ""
category: 
tags: []
---

### What's the reason I can't create generic array types in Java?

private T[] elements = new T[initialCapacity];

It's because Java's arrays (unlike generics) contain, at runtime, information about its component type. So you must know the component type when you create the array. Since you don't know what T is at runtime, you can't create the array.

[link](http://stackoverflow.com/questions/2927391/whats-the-reason-i-cant-create-generic-array-types-in-java)

### Synthetic Class in Java

For example, When you have a switch statement, java creates a variable that starts with a $. If you want to see an example of this, peek into the java reflection of a class that has a switch statement in it. You will see these variables when you have at least one switch statement anywhere in the class.

To answer your question, I dont believe you are able to access(other than reflection) the synthetic classes.

[link](http://stackoverflow.com/questions/399546/synthetic-class-in-java)

### 
