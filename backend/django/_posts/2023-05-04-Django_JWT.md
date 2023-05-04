---
layout: post
title: "Django JWT"
description: >
  Django JWT
tags: [django]
sitemap: false
---

# Django 회원가입 JWT 로 구현

기존 로그인 방식으로는 세션과 JWT 를 이용한 방식이 있다. JWT에 대해서는 아래 글에서 설명하였다. 이번에는 Django에서 JWT 로그인 방식을 구현하는 지 소개한다.


[JWT](/backend/web/2023-05-04-JWT/) 

<br>

**세팅**

우선 djangorestframework--simplejwt 와 django_rest_auth 를 패키지를 Django 프로젝트에 추가시킨다.

<br>

[Getting started — Simple JWT 5.2.2.post16+gf298efa documentation](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/getting_started.html#requirements)

<br>

그리고 위 문서에도 나타나 있듯이 Django 세팅 파일에 다음과 같이 추가해주어야 인증 시 JWT를 이용하게 된다.

```python
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
```

그러면 이제 해당 패키지 내부의 있는 기능들을 하나씩 사용할 수 있다. 간단히 회원가입 시 JWT 를 발급해주는 것 부터 시작하자.

<br>
<br>

**준비 과정**

우선 발급하기에 앞서서 회원가입에 필요한 유저 아이디, 패스워드를 받는 api를 구성해야한다. 간단히… User 모델을 만들고..

```python
from email.policy import default
from importlib.metadata import requires
from django.contrib.auth.models import AbstractUser
from django.contrib.auth.models import PermissionsMixin
from django.contrib.auth.models import UserManager
from django.db import models

class User(AbstractUser, PermissionsMixin):
    username = models.CharField(
        "아이디",
        max_length=256,
        unique=True,
    )
    name = models.CharField(
        "이름",
        max_length=256,
        default="",
        blank=True,
    )

    class Meta:
        db_table = "users"
        verbose_name = "user"
        verbose_name_plural = "users"
        swappable = "AUTH_USER_MODEL"

    def __str__(self):
        return f"[{self.id}] {self.name}"
```

마이그레이션을 하고 진행하자.

<br>

이제 view를 구성해서 간단히 작성하면 다음과 같다. 이름은 Register라는 이름으로 많이 붙이는데 본인이 원하는 이름으로 잘 작성하자.

```python
class UserSignUpView(APIView):
    model = User
    queryset = User.objects.all()
    authentication_classes = []
    permission_classes = []
    serializer_class = UserSignUpSerializer

    def post(self, request):
        serializer = UserSignUpSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
        else:
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

코드를 보면 알겠지만, serializer에 맞는 데이터가 들어오면 user를 생성하도록 구성했다. 만약 invalid한 값이 있다면 400에러가 나타날 것이다. 여기서 serializer는 회원가입시 유저아이디, 패스워드를 정의하고 save 메소드도 정의해두었다.

<br>

```python
class UserSignUpSerializer(serializers.ModelSerializer):
    class Meta:
        model = get_user_model()
        fields = ("username", "name", "password")

    def save(self, request):
        user = super().save()
        user.username = self.validated_data["username"]
        user.name = self.validated_data["name"]
        user.set_password(self.validated_data["password"])
        user.save()
        return user
```

<br>

이렇게 해두면, 요청을 보낼때 값에 맞게 user가 생성된다.

![Screen Shot 2023-04-20 at 4 42 09 PM](https://user-images.githubusercontent.com/47859845/236109047-ca59501d-eaa1-4125-bae0-2b2147ef089a.png)


<br>

- 참고
    
    [What is the different between save(), create() and update () in django rest framework?](https://stackoverflow.com/questions/45100515/what-is-the-different-between-save-create-and-update-in-django-rest-fram)
    

<br>
<br>

**Token 발급**

Token은 실제 유저를 인증하기 위한 토큰이다. 회원가입이나 로그인시 해당 토큰을 서버에서 발급하면, 클라이언트 브라우저에 쿠키로 심도록 한다. 토큰을 발급하려면 암호화나 기타 등등 확인해야하는데, simplejwt 내부 코드들을 사용하면 편리하다.

 토큰을 발급하는 역할을 하는 것은 `TokenObtainPairView` 와  `TokenObtainPairSerializer` 에서 이루어진다. 두개 모두 Base에서 상속받아 사용하는데, 간단히 유저 인증을 하고, 해당 serializer에서 각각 assess, refresh 토큰을 전달한다. view는 해당 data를 반환한다.

```python
class TokenObtainPairSerializer(TokenObtainSerializer):
    token_class = RefreshToken

    def validate(self, attrs):
        data = super().validate(attrs)

        refresh = self.get_token(self.user)

        data["refresh"] = str(refresh)
        data["access"] = str(refresh.access_token)

        if api_settings.UPDATE_LAST_LOGIN:
            update_last_login(None, self.user)

        return data
```

여기까지 보면 잘 와닿지 않은데, 우리에게 필요한 것은 우리 회원가입 플로우에서 토큰을 발급하는 역할이다. 즉, user 정보를 토대로 토큰 발급하는 기능을 패키지에서 찾아볼 것이다. 위 코드를 보면 `self.get_token(self.user)` 라는 코드가 보인다. `TokenObtainSerializer` 를 확인하면 user 값을 통해 토큰을 리턴해주는 클래스 메소드가 있기 때문에 이를 이용한다.

<br>

회원가입 API View에 다음과 같이 추가한다.

```python
def post(self, request):
    serializer = self.serializer_class(data=request.data)
    print("serializer:", serializer)
    print("is_valid:", serializer.is_valid())
    print("errors:", serializer.errors)

    if serializer.is_valid():
        user = serializer.save(request=request)
        token = TokenObtainPairSerializer.get_token(user)
        refresh_token = str(token)
        access_token = str(token.access_token)
        res = Response(
            {
                "user": serializer.data,
                "message": "register successs",
                "token": {
                    "access": access_token,
                    "refresh": refresh_token,
                },
            },
            status=status.HTTP_200_OK,
        )
        res.set_cookie("access", access_token, httponly=True)
        res.set_cookie("refresh", refresh_token, httponly=True)
        return res
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

TokenObtainPairSerializer.get_token(user) 를 통해 회원가입한 user 정보를 토대로 token을 생성하고, 해당 token 값들을 Reponse로 보내준다…

![Screen Shot 2023-04-20 at 4 45 08 PM](https://user-images.githubusercontent.com/47859845/236109054-bdea232a-08af-4a8f-adfb-c380f94fa54b.png)


브라우저에서 회원가입 응답을 확인하면 위와 같이 Set-Cookie 헤더가 있는 것을 볼 수 있다. 이제 클라이언트 코드에서 해당 헤더를 토대로 브라우저 쿠키에 설정해주면 된다.

<br>

아래와 같은 코드를 통해서 쿠키값을 넣어준다.

![Screen Shot 2023-04-20 at 7 34 30 PM](https://user-images.githubusercontent.com/47859845/236109140-191b1e9f-76a4-46a7-a3b5-abd3848754d9.png)


아래와 같이 헤더에 있는 값이 지정된다.

![Screen Shot 2023-04-20 at 7 35 24 PM](https://user-images.githubusercontent.com/47859845/236109057-6cc5e45d-9216-4e57-91b4-f084ca02ce8d.png)

<br>

여기서 HttpOnly와 Secure을 지정해주는 것이 좋다. 토큰이 브라우저에 남는 것이 보안상 문제가 될 수 있기 때문에 HttpOnly를 통해서 내용을 볼수 없도록 하는 것이다. 또한, 토큰을 보낼 때 Same-site 옵션도 주는 것이 좋다.


<br>
<br>