+++
title = "Markdown 语法测试"
date = 2021-11-28
+++

## Splliter

--------

## Headings

# First Level

## Second Level

### Third Level

#### Fourth Level

##### Fifth Level

###### Sixth Level

## Fonts

**bold**

*itallic*

***bold-italic*** 

~~deleted~~

## Quote

> Nvidia: Fuck you!
>
> -- Linus Torvalds

> This is a Quote
>> Inner a quote
>>> And inner

## Image inclusion

![dove](/favicon.png)

## hyperlink

[ArchLinux](https://archlinux.org)

## Lists

- item1
  1. ordered item1
  2. ordered item2
  3. ordered item3
- item2
- item3

## Table

| name | gender | age |
| ---- | ------ | --- |
| dove | Male   | ??  |
| foo  | Male   | 18  |
| bar  | Female | 19  |

## Syntax Highlighting

```js
function foo(l) {
  let sum = 0;
  for (let item of l) {
    sum += l;
  }
  return sum;
}
console.log(foo([1, 2, 3]));
```

```python
print(sum([1, 2, 3]))
```

```c
#include <stdio.h>

int foo(int *l) {
  ing sum = 0;
  for (int i = 0; i < sizeof(l); i++) {
    sum += *l + i;
  }
  return sum;
}

int main() {
  int *l = {1, 2, 3};
  printf("%d\n", foo(l));
}
```

