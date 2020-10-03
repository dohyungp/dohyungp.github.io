
---
layout: default
title: DRF(Django Rest Framework)에서 IP로 Permission 지정하고 싶을 때
---

## DRF(Django Rest Framework)에서 IP로 Permission 지정하고 싶을 때

가끔 DRF로 코드를 짜다보면 외부 시스템과 Webhook으로 통신할 일이 있습니다. 이럴 때 외부 시스템을 자사 시스템에 로그인 시킬 수도 없으니 IP로 Permission을 지정하는게 한가지 방법이 될 것입니다. 이럴 때 가장 쉬운 방법은 애초에 settings.py의 `ALLOWED_HOSTS = []`에 `[]`를 허용하는 호스트들만 쓰는 방법이 있지만 특정 endpoint 혹은 그 중에서도 특정 object 접근 제한을 시켜야하는 경우가 더 많을 것입니다.

따라서 DRF에서는 아래와 같이 간단한 방식으로 접근 제한을 줄 수 있습니다.

`permissions.py`라는 이름으로 파일을 하나 만들어주고,

```py
from rest_framework import permissions

ALLOWED_IP_ADDRESSES = ['127.0.0.1']

class IPBasedPermission(permissions.BasePermission):
    def has_permission(self, request, view):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip_address = x_forwarded_for.split(',')[0]
        else:
            ip_address = request.META.get('REMOTE_ADDR')

        return ip_address in ALLOWED_IP_ADDRESSES

    def has_object_permission(self, request, view):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip_address = x_forwarded_for.split(',')[0]
        else:
            ip_address = request.META.get('REMOTE_ADDR')

        return ip_address in ALLOWED_IP_ADDRESSES
```

이렇게 간단하게 만들 수 있고 필요 시 `ALLOWED_IP_ADDRESSES`를 model로 만들어 관리하는 방법도 있겠습니다. 물론 반대로 `BLACK_LIST_IP_ADDRESSES`같은 반대케이스도 가능하겠습니다(공식가이드에서는 비슷한 가이드가 실제로 있습니다).
