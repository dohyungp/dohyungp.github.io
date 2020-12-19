---
layout: default
title: 밑바닥부터 시작하는 딥러닝 3 STEP 10까지 읽으면서
---

## 밑바닥부터 시작하는 딥러닝 3 STEP 10까지 읽으면서

코드를 조금씩 리팩토링하는게 매력적인 책이라 공부하면서 정리된 코드를 남겨봅니다(일부 입맛에 맞게 고쳤습니다)


```py
def as_array(x):
    if np.isscalar(x):
        return np.array(x)

    return x


class Variable:

    def __init__(self, data):
        if data is not None:
            if not isinstance(data, (np.ndarray, np.generic)):
                raise TypeError(f'{type(data)} is not supported')
        self.data = data
        self.grad = None
        self._function = None

    @property
    def function(self):
        return self._function

    @function.setter
    def function(self, function):
        self._function = function

    def backward(self):
        if self.grad is None:
            self.grad = np.ones_like(self.data)
        funcs = [self._function]
        while funcs:
            f = funcs.pop()
            x, y = f.input, f.output
            x.grad = f.backward(y.grad)

            if x.function is not None:
                funcs.append(x.function)
        return


class Function:

    def __call__(self, input):
        x = input.data
        y = self.forward(x)
        output = Variable(as_array(y))
        output.function = self
        self.input = input
        self.output = output

        return output
```

```py
x = Variable(np.array(.5))
y = square(exp(square(x)))
y.backward()
print(x.grad)
```