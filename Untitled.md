Five questions, answer all of them, and I'll give feedback after. These use your professor's definitions: the root is level 0, and height = the number of edges on the longest root-to-leaf path.

---

**Q1 · Terminology**

```text
          A
        /   \
       B     C
      / \     \
     D   E     F
          \
           G
```

a) Which nodes are leaves?  
**d, g, f**
b) What's the height of the tree?  
**4**
c) What are the ancestors of G?  
**E, B, A**
d) What are the descendants of B?  
**D, E, G**
e) Is this tree **full**? **Complete**? **Balanced**? Give a reason for each.
**Idk what a full tree means or complete, but its balanced because the balance factor is not more than 1. Meaning there is at most 1 more node on the right side of B than its left.**

---

**Q2 · Counting nodes**

a) What's the maximum number of nodes in a binary tree of height 3?  
8
b) What's the maximum number of nodes at level 4?
8

---

**Q3 · Array-based binary tree**

```python
arr = [50, 30, 70, 20, 40, None, 80]
```

a) What are the left and right children of 70?  
20 and 40
b) What's the parent of 40?  
70
c) Give one pro and one con of storing a binary tree in an array. Why does it work best for complete trees?
It doesn't require pointers and just needs math to calculate the location. 

---

**Q4 · BST insertion**

Insert these into an empty BST, in this order:

```text
15, 8, 20, 3, 10, 25, 12
```

a) Draw the final tree.  
i cant draw on PC rn but ill just answer the next questions
b) What's its height?  
4
c) Which nodes does `find(12)` visit, in order?
15,18,10,12

---

**Q5 · BST find code**

Fill in the four blanks:

```python
def _find(self, node, value):
    if node.value is None:                     # (1)
        return False
    if node.value == value:
        return True
    if node.value < value:
        return self._find(________, value)   # (2)
    return self._find(________, value)       # (3)
```

(4) What's the Big O of `find` on a balanced BST, and on a worst-case BST? What kind of insert order creates the worst case?