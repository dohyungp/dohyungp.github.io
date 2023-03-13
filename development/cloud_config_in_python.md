---
layout: default
title: Python에서 Spring Cloud Config 값 받아오다가 고통받은 썰
---

Spring Cloud Config는 아래와 같이 recursive하게 값을 덮어씌우는 템플리팅을 지원한다.

```yaml
my-key: ${my-value.url.domain}
```

근데 이걸 Python에서 처리하자니 답이 없다.

근데 기본 모듈 중에 `string.Template`을 써보면 예쁘게 되긴 할 것 같아서 써보기로 했다.

```py
from string import Template


class SpringConfigTemplate(Template):
    """Spring Cloud Config 템플릿"""

    idpattern = r"(?a:[_.-a-z][_.-a-z0-9]*)"

val = "${my-value.url.domain}"
template = SpringConfigTemplate(val)
val = template.safe_substitute(config_obj)
```

이러니까 좀 편하게 처리가 가능하다. 이걸 while-loop으로 특정조건이 걸릴 때 break하게 하기만 하면 쉽게 nested한 값도 처리가 된다.

근데 아래 같은 케이스는 또 답이 없다.

```yaml
my-key: ${my.domain.host}:${my.url.port:8080}
```

생각나면 추가해야겠다. 지금은 그냥 replace하는걸로.
