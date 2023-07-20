# Automatic test oracle generation for SE/CT

## Task 1:
> Can we automatically compare two random objects using equals method?
If not, try to come up with an example when equals will not work.

Assume we have objects A and B, let them be instances of:

```java
...
public class Point {
    protected int x;
    protected int y;
    public Clazz(int x_, int y_) {
        x = x_;
        y = y_;
    }
};

Point A = new Point(0, 0);
Point B = new Point(0, 0);
A.equals(B); // we want it to return true, but it will not, because .equals() method
             // is called from Object class (if not overrided),
             // there it is implemented as comparing by "==" sign.
...
```


## Task 2:

>Do you have any alternative ideas of how we can compare two random objects for equality? Explain their pros and cons.

1. **Overriding `.equals()` method**:
    
    We can override `.equals()` method as follows
    
    ```java
    ...
    public class Point {
        ...
        @Override
        public boolean equals(Object o) {
            if (o instanceof Point that) {
                return (this.x == that.x && this.y == that.y);
            }
            return false;
        }
    }
    ...
    
    A.equals(B); // now it returns true
    ```
    
    Now it will work as expected when calling `A.equals(B)`, also if `A` and `B` are instances of different classes that are not comparable, it will return `false` (as expected).
    
    But what if we have inheritance:
    
    ```java
    ...
    
    public class ColorPoint extends Point {
        protected int c;
        public ColorPoint(int x_, int y_, int c_) {
            x = x_;
            y = y_;
            c = c_;
        }
    
        public boolean equals(Object o) {
            if (o instanceof ColorPoint that) {
                return this.x == that.x && this.y == that.y && this.c == that.c;
            }
            return false;
        }
    }
    
    Point A = new Point(0, 0);
    ColorPoint C = new ColorPoint(0, 0, 255);
    
    A.equals(C); // might be true
    C.equals(A); // always false
    ...
    ```
    
    Now we have a problem with laws that should hold: here we violated symmetric law.
    
    Better way would be to compare objects by layers, because objects of derived classes that do not introduce new fields still can be compared to some of their parent classes instances. To implement that we write:
    
    ```java
    ...
    
    public class Point() {
        ...
        public boolean equals(Object o) {
            if (o instanceof Point that) {
                return (that.canEqual(this)) && (this.x == that.x) && (this.y == that.y);
                //      ^^^^          ^^^^  <- order here matters
            }
            return false;
        }
    
        public boolean canEqual(Object other) {
            return (other instanceof Point);
        }
    }
    
    public class ColoredPoint {
        ...
        public boolean equals(Object o) {
            if (o instanceof ColorPoint that) {
                return (that.canEqual(this)) && (this.c == that.c) && super.equals(that);
            }
            return false;
        }
        public boolean canEqual(Object other) {
            return (other instanceof ColorPoint); // only comparing to the same class here
        }
    }
    ...
    ```
    
    By introducing `.canEqual()` method we fixed the problem that we had before, and now can compare objects "by layers".
2. **Implementing inteface `Comparable<T>`**:

    If we need some class `A` be able to compare to some class `B`, then we can write:  
    
    ```java
    public class A implements Comparable<B> {
        int compareTo(Apple o) {
            ...
        }
    }
    ```
    
    Here we must also be able to set an ordering on the elements, which makes the implementation a bit harder. On the other hand, it allows us to automatically sort collections of elements with type `T` if `T implements Comparable<T>`.

3. **Implementing interface `Comparator<T>`**:

    Similar to the previous way, but now comparator itself is a stand-alone separate class, so it can store some state, if we need it for some reason. It also can be passed to sorting functions or data structures. It also can be passed to the Stream API `sorted` method.
    
    We can write class that implements `Comparator<T>` as such:
    ```java
    class Comp implements Comparator<T> {
        @Override
        public int compare(T o1, T o2) {
            ...
        }
    }
    ```
4. **Object serialization**:

We can also compare two random objects by serializing them and then comparing the resulting byte sequences. This seems to be the most expensive and inefficient way of comparing, and I don't seem to come up with the use case when it might be applicable...

## Task 3:

> Given a representation of a binary tree, write a function that will determine whether two binary trees have similar
contents (for some definition of similarity, do not forget to also explain how you defined similarity).

1. The code below returns `true` if trees are the same in structure and element values, `false` otherwise. Time complexity `O(n)`, where `n` - total number of nodes in the tree.
   
   ```java
   class BinaryTree {
       int value;
       BinaryTree left;
       BinaryTree right;
   
       static boolean contentsSimilar(BinaryTree lhv, BinaryTree rhv) {
           if (lhv == null && rhv == null) {
               return true;
           }
   
           if (lhv == null || rhv == null) {
               return false;
           }
   
           return lhv.value == rhv.value &&
                  contentsSimilar(lhv.left, rhv.left) &&
                  contentsSimilar(lhv.right, lhv.right);
       }
   
       // You can consider that these methods are implemented
       // and you can use them if needed
       boolean contains(int value);
       boolean add(int value);
       boolean remove(int value);
       int size();
   }
   ```

2. The code below returns `true` if trees have the same multi-sets of values. Time complexity: `O(n * h)`, when `n` - total number of nodes in the tree and `h` - the height of the tree.
   
   ```java
   import java.util.ArrayList;
   
   class BinaryTree {
      int value;
      BinaryTree left;
      BinaryTree right;
   
   
      static boolean contentsSimilar(BinaryTree lhv, BinaryTree rhv) {
         List<Integer> removedElements = new ArrayList<>();
         boolean isSimilar = checkContentsSimilar(lhv, rhv, removedElements);
   
         // put elements back
         for (int value : removedElements) {
            rhv.add(value);
         }
   
         return isSimilar;
      }
   
      private static boolean checkContentsSimilar(BinaryTree lhv, BinaryTree rhv, List<Integer> removedElements) {
         if (lhv == null) {
            return true;
         }
   
         int value = lhv.value;
         if (rhv.contains(value)) {
            // remove element because BST might have more than 1 node with the same values
            rhv.remove(value);
            removedElements.add(value);
            return checkContentsSimilar(lhv.left, rhv, removedElements) &&
                    checkContentsSimilar(lhv.right, rhv, removedElements);
         }
   
         return false;
      }
   
      // You can consider that these methods are implemented
      // and you can use them if needed
      boolean contains(int value);
      boolean add(int value);
      boolean remove(int value);
      int size();
   }
   ```