
# em vs rem

## em

- 对于标签的font-size来说，1em等于父标签的font-size;
- 对于标签的尺寸(width, height)来说，1em等于自身的font-size；

```
<html>
    <h1></h1>
    <p></p>
    ...
</html>   

html {
    font-size: 100%; // 100% means default, the default value is 16px;
}

h1 {
    font-size: 2em; // 1em = 1 * 16px(它的父标签的font-size), so 2em = 32px; 
    margin-top: 1em; // 32px, 因为它自己的font-size已经是32px了;
}

p {
    font-size: 1em; // 16px;
    margin-top: 1em; // 16px;
}

```

## rem

- It is a unit of typography equal to the root font-size. This means 1rem is always equal to the font-size defined in <html>.
- 即1rem永远等于html标签的font-size;

```
h1 {
  font-size: 2rem; /* 32px */
  margin-bottom: 1rem; /* 1rem = 16px */
}

p {
  font-size: 1rem;
  margin-bottom: 1rem; /* 1rem = 16px */
}
```

参考文档：
https://zellwk.com/blog/rem-vs-em/