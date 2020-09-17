---
layout: default
title: Mathjax가 적용되는지 테스트해보자
---

## Mathjax가 적용되는지 테스트해보자

수알못이나 가-끔 블로그할 때 공식을 쓸 일이 있을까 싶어 Mathjax를 추가해봤습니다. 이 전에 Gatsby.js로 블로그를 만들었으나 배포 난이도로 꾸준한 블로깅이 어려워 이번에는 정말 minimal로 하기위해 원래 default.html에 아래 코드만 추가해 Mathjax가 동작하도록 세팅했습니다.

```
<script type="text/x-mathjax-config">
      MathJax.Hub.Config({
          TeX: {
            equationNumbers: {
              autoNumber: "AMS"
            }
          },
          tex2jax: {
            inlineMath: [ ['$','$'], ["\\(","\\)"] ],
            displayMath: [ ['$$','$$'], ["\\[","\\]"] ],
            processEscapes: true
        }
      });
      MathJax.Hub.Register.MessageHook("Math Processing Error",function (message) {
            alert("Math Processing Error: "+message[1]);
          });
      MathJax.Hub.Register.MessageHook("TeX Jax - parse error",function (message) {
            alert("Math Processing Error: "+message[1]);
          });
    </script>
    <script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML"></script> 
```

이렇게 세팅한 결과,

**inline tex** 
```
This formula $f(x) = x^2$ is an example.
```

This formula $f(x) = x^2$ is an example.

**display block**
```
$$
\lim_{x\to 0}{\frac{e^x-1}{2x}}
\overset{\left[\frac{0}{0}\right]}{\underset{\mathrm{H}}{=}}
\lim_{x\to 0}{\frac{e^x}{2}}={\frac{1}{2}}
$$
```

$$
\lim_{x\to 0}{\frac{e^x-1}{2x}}
\overset{\left[\frac{0}{0}\right]}{\underset{\mathrm{H}}{=}}
\lim_{x\to 0}{\frac{e^x}{2}}={\frac{1}{2}}
$$

가 가능해졌습니다 🙌