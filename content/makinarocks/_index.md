+++
title = "Makinarocks"
outputs = ["Reveal"]
+++

{{% section %}}
# Soundhub
---

# Soundhub: 소개
- **개요**: 여러 사람들이 음악 트랙을 공유하고, 믹스할 수 있는 앱
  - 음악 트랙을 Posting 하고, 
  - 그 트랙에 어울리는 또 다른 트랙을 Comment 의 형식으로 공유하면,
  - Post 창작자가 원하는 조합으로 음원을 믹스할 수 있다.
- **팀**: Backend 2, iOS 1, Android 1
- **맡은 파트**: DB 구조 설계, 유저 파트, 배포 등
- **데모 영상**: https://www.youtube.com/watch?v=fukEqcsbkJA 
---
# Soundhub: 구현된 핵심 기능

- **유저 관련**: 
  - 소셜로그인, 로그인 관련 다양한 이슈 해결
  - 이메일 인증, 비밀번호 변경 메일
  - Follow
  - ~~비실시간 메세지, 검색~~
- **음악 Posting 관련**:
  - Post, Comment 작성, 트랙 믹스
  - Post 좋아요

---
# Soundhub: 사용된 기술
- **웹 앱 구현**: 
  - Python, Django, Django REST Framework, 
  - Postgresql, Celery
  - TDD
- **배포**: 
  - uWSGI, NginX 
  - Docker, AWS EB, AWS S3, AWS RDS, 
  - Travis CI, Sentry
{{% /section %}}

---
{{% section %}}
# RESTful API

---
# URL 구조
- `"/"` 또는  `"/home"`: 메인 페이지 관련 url
- `"/user"`: 유저 관련 url
- `"/post"`: Post 관련 url
- `"/message"`: 메세지 관련 url
- `"/search"`: 검색 관련 url

---
# URL 구조: `/home`
- `/home/` : (get) 인기순으로 정렬된 user, post 리스트; 최신순으로 정렬된 post 리스트
- `/home/genre/<genre>` : 해당 장르별 user, post 리스트
- `/home/instrument/<instrument>` : 해당 악기별 user, post 리스트

---
# URL 구조: `/user`
- `/user` : (get) 유저 리스트
- `/user/login` : (post) 로그인
- `/user/google_login` : (post) 구글 로그인
- `/user/facebook_login` : (post) 페이스북 로그인
- `/user/logout` : (post) 로그아웃
- `/user/signup` : (get) 회원가입 관련 특수한 케이스 처리, (post) 회원가입
- `/user/activate` : (get) 이메일 인증
- `/user/password` : (get) 이메일 인증 후 패스워드 변경 (post) 변경 전 패스워드 확인
- `/user/<pk>` :  (get) 해당 유저 조회, (post/patch) 수정, (delete) 삭제
- `/user/<pk>/follow` : (post) follow toggle
- `/user/<pk>/profile_img` : (get) 프로필 이미지 조회, (post/patch) 수정, (delete) 삭제

---
# URL 구조: `/post`
- `/post` : (get) Post 리스트
- `/post/<pk>` : (get) Post 조회, (post/patch) 수정, (delete) 삭제 
- `/post/<pk>/mix` : (patch) 트랙 믹스(Post 트랙과 Comment 트랙)
- `/post/<pk>/like` : (post) 해당 Post like 토글
- `/post/<pk>/comments` : (get) Comment 리스트
- `/post/comment/<pk>` : (get) Comment 조회, (post/patch) 수정, (delete) 삭제


{{% /section %}}

---
{{% section %}}
# 특수한 기능

---
# 특수한 기능
- Activation Key 를 이용한 이메일 인증 
- 확률 기반 Stale User 처리 
- 회원가입/로그인 관련 예외상황 처리 

---
# 이메일 인증: Activation Key
1. 회원가입 요청시, 유저 생성
2. 유저 생성시 activation key 생성
3. 생성된 activation key 정보를 담은 activation link 가 있는 메일 발송
4. activation link에 접속하면
5. 링크 get parameter 에 담긴 activation key 를 통해 유저 특정
6. 해당 유저 activation

---
# 이메일 인증: Activation Key

```python{}
class Signup(APIView):
    def get(self, request):
        ...

    def post(self, request):
        ...

        serializer = SignupSerializer(data=request.data)
        if serializer.is_valid():
            # user 생성, 반환
            user = serializer.save()
            # activation key 생성
            activation_key_info = ActivationKeyInfo.objects.create(user=user)
            # 인증 메일 발송
            send_verification_mail.delay(
                activation_key=activation_key_info.key,
                recipient_list=[user.email],
            )
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@celery_app.task
def send_verification_mail(activation_key, recipient_list):
    """
    activation key 를 담은 activation_link 를 recipient 에게 보냄

    :param activation_key: activation key 의 기능을 수행하는 40자 문자열
    :param recipient_list: 수신자 이메일 목록 list 객체
    :return: send_mail 함수 반환 값
    """

    scheme = 'https://'
    host = 'soundhub.che1.co.kr'
    activation_link = scheme + host + reverse('user:activate') + f'?activation_key={activation_key}'

    subject = '[Soundhub] Email Verification'
    message = ''
    html_message = f'Verify your email to login Soundhub: <a href="{activation_link}">activation link</a>'
    from_email = 'joo2theeon@gmail.com'
    return send_mail(
        subject=subject,
        message=message,
        html_message=html_message,
        from_email=from_email,
        recipient_list=recipient_list,
    )


class ActivationKeyInfoManager(models.Manager):
    def create(self, user):
        """
        1. stale user 삭제
        2. activation key & expired_at 자동 생성

        :param user: ActivationKeyInfo 와 연결될 유저 객체
        :return: 생성된 ActivationKeyInfo 객체
        """
        ...

        # activation key 생성을 위한 무작위 문자열
        # user 마다 unique 한 값을 가지게 하기 위해 user.email 첨가
        random_string = str(random()) + user.email
        # sha1 함수로 영문소문자 또는 숫자로 이루어진 40자의 해쉬토큰 생성
        activation_key = hashlib.sha1(random_string.encode('utf-8')).hexdigest()
        # activation key 유효기간 2일
        expires_at = timezone.now() + timezone.timedelta(days=2)
        # activation key 생성
        activation_key_info = ActivationKeyInfo(
            user=user,
            key=activation_key,
            expires_at=expires_at,
        )
        activation_key_info.save()

        return activation_key_info


# Email Verification 에 사용되는 Activation key 정보를 담고 있는 클래스
# User class 와 one to one 으로 연결
class ActivationKeyInfo(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    key = models.CharField(max_length=40, blank=True)
    expires_at = models.DateTimeField()  # key 만료 기한

    objects = ActivationKeyInfoManager()

    def __str__(self):
        return f'user:{self.user.nickname}'

    def refresh(self):
        """
        간단하게 ActivationKeyInfo 를 새로고침 하기 위한 함수
        """
        # activation key 생성을 위한 무작위 문자열
        # user 마다 unique 한 값을 가지게 하기 위해 user.email 첨가
        random_string = str(random()) + self.user.email
        # sha1 함수로 영문소문자 또는 숫자로 이루어진 40자의 해쉬토큰 생성
        activation_key = hashlib.sha1(random_string.encode('utf-8')).hexdigest()
        # activation key 유효기간 2일
        expires_at = timezone.now() + timezone.timedelta(days=2)

        self.key = activation_key
        self.expires_at = expires_at
        self.save()

    def refresh_expires_at(self):
        expires_at = timezone.now() + timezone.timedelta(days=2)
        self.expires_at = expires_at
        self.save()
```

---
# Stale User 처리

- 이메일 인증 방식의 문제: stale user 가 생긴다
- 대표적인 방법: 비동기 처리로, 서버 접속률이 낮을 때 주기적으로 실행
- 비동기 처리의 단점: 
  - stale user 의 급작스러운 증가에 대응할 수 없고
  - stale user 가 없을 때도 돌아간다
- 새로운 방법: User 가 생성될 때 일정 확률로 처리 
  - 비동기 처리의 단점을 극복할 뿐 아니라, 구현도 쉽다 

---
# Stale User 처리

```python{}
class ActivationKeyInfoManager(models.Manager):
    def create(self, user):
        """
        1. stale user 삭제
        2. activation key & expired_at 자동 생성

        :param user: ActivationKeyInfo 와 연결될 유저 객체
        :return: 생성된 ActivationKeyInfo 객체
        """
        # A.K.I manager 를 생성할 때마다 stale user 를 삭제하는 것은 매우 비효율적이다. 
        # 따라서, 5%의 확률로 삭제한다.
        if randint(1, 20) == 1:
            # stale user 삭제 비동기 처리
            delete_staleuser.delay()
        ...

        return activation_key_info
        

@celery_app.task
def delete_staleuser():
    """
    A.K.I 가 생성된 유저를 조회하면서 stale user 를 삭제하는 메서드
    (stale user: 회원가입 후 이메일 인증을 하지 않은 채 오래 지난 유저)

    :return: None
    """
    # 이 함수가 users.models 에서 쓰이기 때문에, global 선언은 로드될 때 충돌을 야기한다.
    from users.models import ActivationKeyInfo

    for aki in ActivationKeyInfo.objects.all():
        if aki.user.is_active is False and aki.expires_at < timezone.now():
            aki.delete()
```

---
# 회원가입/로그인 관련 예외 처리

- stale user 재로그인시 바로 인증 메일 발송
- 지메일로 일반 회원가입 후, 구글 소셜 로그인을 한 경우 바로 로그인
- 소셜로그인 후 일반 회원가입


{{% /section %}}

---
{{% section %}}

# DB 구조

---
# DB 구조
![](img/db_structure.png)

---
# DB 구조: Many To Many

별도의 Relation 을 만들고, Foreign Key 로 연결

```python{}
# 팔로우 관계 모델
class Relationship(models.Model):
    from_user = models.ForeignKey(
        User, on_delete=models.CASCADE,
        related_name='following_set'
    )
    to_user = models.ForeignKey(
        User, on_delete=models.CASCADE,
        related_name='follower_set'
    )
    related_date = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'{self.from_user.nickname} is following {self.to_user.nickname}'


class PostLike(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    liked_date = models.DateTimeField(auto_now_add=True, blank=True, null=True)

    def __str__(self):
        return f'{self.author} liked {self.post}'

    class Meta:
        ordering = ['-liked_date']


class Message(models.Model):
    from_user = models.ForeignKey(settings.AUTH_USER_MODEL,
                                  null=True,
                                  related_name='sent_msgs',
                                  on_delete=models.SET_NULL)
    to_user = models.ForeignKey(settings.AUTH_USER_MODEL,
                                related_name='received_msgs',
                                null=True,
                                on_delete=models.SET_NULL)
    content = models.CharField(max_length=255, blank=True, null=True)
    read_date = models.DateTimeField(default=None, null=True)
    sent_deleted = models.BooleanField(default=False)
    inbox_deleted = models.BooleanField(default=False)
    created_date = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'from: {self.from_user}, to: {self.to_user}, content: {self.content}'
```

---
# DB 구조: One To One

다중속성의 경우, 따로 Relation 을 만든다  

```python{}
# Email Verification 에 사용되는 Activation key 정보를 담고 있는 클래스
# User class 와 one to one 으로 연결
class ActivationKeyInfo(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    key = models.CharField(max_length=40, blank=True)
    # key 만료 기한
    expires_at = models.DateTimeField()

    objects = ActivationKeyInfoManager()

    def __str__(self):
        return f'user:{self.user.nickname}'

    def refresh(self):
        """
        간단하게 ActivationKeyInfo 를 새로고침 하기 위한 함수
        :return: None
        """
        # activation key 생성을 위한 무작위 문자열
        # user 마다 unique 한 값을 가지게 하기 위해 user.email 첨가
        random_string = str(random()) + self.user.email
        # sha1 함수로 영문소문자 또는 숫자로 이루어진 40자의 해쉬토큰 생성
        activation_key = hashlib.sha1(random_string.encode('utf-8')).hexdigest()
        # activation key 유효기간 2일
        expires_at = timezone.now() + timezone.timedelta(days=2)

        self.key = activation_key
        self.expires_at = expires_at
        self.save()

    def refresh_expires_at(self):
        expires_at = timezone.now() + timezone.timedelta(days=2)
        self.expires_at = expires_at
        self.save()


# 페이스북 유저의 Facebook User ID 를 저장하기 위한 모델
# Facebook Backend 에 사용된다
class FacebookUserInfo(models.Model):
    facebook_user_id = models.CharField(max_length=50, unique=True)
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)

```

{{% /section %}}

---
{{% section %}}
# Celery, 비동기 처리
---

## mail 발송 로직

`soundhub/utils/tasks/mail.py`
```python{}
from django.core.mail import send_mail
from django.urls import reverse

from config import celery_app


__all__ = (
    'send_verification_mail',
    'send_confirm_readmission_mail',
    'send_verification_mail_after_social_login',
    'send_password_reset_mail',
)

@celery_app.task
def send_verification_mail(activation_key, recipient_list):
    """
    activation key 를 담은 activation_link 를 recipient 에게 보냄

    :param activation_key: activation key 의 기능을 수행하는 40자 문자열
    :param recipient_list: 수신자 이메일 목록 list 객체
    :return: send_mail 함수 반환 값
    """

    scheme = 'https://'
    host = 'soundhub.che1.co.kr'
    activation_link = scheme + host + reverse('user:activate') + f'?activation_key={activation_key}'

    subject = '[Soundhub] Email Verification'
    message = ''
    html_message = f'Verify your email to login Soundhub: <a href="{activation_link}">activation link</a>'
    from_email = 'joo2theeon@gmail.com'
    return send_mail(
        subject=subject,
        message=message,
        html_message=html_message,
        from_email=from_email,
        recipient_list=recipient_list,
    )


@celery_app.task
def send_confirm_readmission_mail(recipient_list):
    """
    아직 이메일 인증이 이루어지지 않은 메일로 새로운 가입 요청이 왔을 때 보내주는 알림 메일

    :param recipient_list: 리스트 타입 수신자 목록
    :return: send_mail 함수 반환 값
    """
    subject = '[Soundhub] Confirmation of Readmission'
    message = "Someone tried to signup with your email."
    from_email = 'joo2theeon@gmail.com'
    return send_mail(
        subject=subject,
        message=message,
        from_email=from_email,
        recipient_list=recipient_list,
    )


@celery_app.task
def send_verification_mail_after_social_login(data, recipient_list):
    """
    소셜로그인을 했다가 일반 회원가입을 다시 요청한 변태 가입자를 위한 안내 메일

    :param data: 회원 가입에 필요한 정보 dictionary 객체
    :param recipient_list: 수신자 email list 객체
    :return: send_mail 함수 반환 값
    """

    scheme = 'https://'
    host = 'soundhub.che1.co.kr'

    # data 에 전달된 값을 get parameter 로 재구성
    # params = '?key=value&key=value&' 형태
    params = '?'
    for key, value in data.items():
        params = params + f'{key}={value}&'

    # 완성된 회원가입 링크

    signup_link = scheme + host + reverse('user:signup') + params[:-1]  # params[:-1]은 맨 뒤에 &를 떼는 로직

    subject = '[Soundhub] Email Verification (Signup Soundhub after social login)'
    message = ''  # 메세지는 필수 필드
    html_message = f'Verify your email to login Soundhub: <a href="{signup_link}">signup link</a>'
    from_email = 'joo2theeon@gmail.com'
    return send_mail(
        subject=subject,
        message=message,
        html_message=html_message,
        from_email=from_email,
        recipient_list=recipient_list,
    )


@celery_app.task
def send_password_reset_mail(data, recipient_list):
    """
    비밀번호가 변경되었음을 알리는 메일. 본인이 한 것이 아닐 경우, 다시 변경할 수 있는 링크가 걸려있다.

    :param data: 비밀번호 변경에 필요한 정보 dictionary 객체
    :param recipient_list: 수신자 email list 객체
    :return: send_mail 함수 반환 값
    """
    scheme = 'http://'
    # host = 'soundhub-dev.ap-northeast-2.elasticbeanstalk.com'
    host = 'localhost:8000'

    # data 에 전달된 값을 get parameter 로 재구성
    # params = '?key=value&key=value&' 형태
    params = '?'
    for key, value in data.items():
        params = params + f'{key}={value}&'

    # 완성된 패스워드 변경 링크
    reset_password_link = scheme + host + reverse('user:password') + params[:-1]  # params[:-1]은 맨 뒤에 &를 떼는 로직

    subject = '[Soundhub] Reset Password Link  (Signup Soundhub after social login)'
    message = ''  # 메세지는 필수 필드
    html_message = f'Verify your email to reset password in Soundhub: ' \
                   f'<a href="{reset_password_link}">reset password link</a>'
    from_email = 'joo2theeon@gmail.com'
    return send_mail(
        subject=subject,
        message=message,
        html_message=html_message,
        from_email=from_email,
        recipient_list=recipient_list,
    )
```

---

## stale user 제거 로직
`soundhub/utils/tasks/staleuser.py`
```python{}
@celery_app.task
def delete_staleuser():
    """
    A.K.I 가 생성된 유저를 조회하면서 stale user 를 삭제하는 메서드
    (stale user: 회원가입 후 이메일 인증을 하지 않은 채 오래 지난 유저)

    :return: None
    """
    # 이 함수가 users.models 에서 쓰이기 때문에, global 선언은 로드될 때 충돌을 야기한다.
    from users.models import ActivationKeyInfo

    for aki in ActivationKeyInfo.objects.all():
        if aki.user.is_active is False and aki.expires_at < timezone.now():
            aki.delete()
```
{{% /section %}}

---
{{% section %}}
# 배포 관련 기술 

---
# 배포 관련 기술: 요약
- uWSGI, Nginx 사용 
- Deploy, Local 용 설정 파일 분리
- Docker + AWS EB 로 배포 자동화
- AWS S3 static file 저장
- AWS RDS 데이터베이스 저장
- Travis CI 로 빌드 테스트

---
#  settings 파일 분리

```{}
.config_secrets/
    settings_common.json
soundhub/
    mysite/
      settings/
        __init__.py
        base.py
        local.py
        debug.py
```

- secret 정보 분리 (django secret key, encryption key 등)
- deploy 설정과 local 설정 분리 

---
# Docker

Dockerfile.base 

```docker{}
# Dockerfile.base
FROM        ubuntu:16.04
MAINTAINER  joo2theeon@gmail.com

RUN         apt-get -y update
RUN         apt-get -y dist-upgrade
RUN         apt-get install -y python-pip git vim

# ffmpeg install
RUN         apt-get -y update
RUN         apt-get -y install ffmpeg
RUN         apt-get -y install libavcodec-ffmpeg-extra56

# pyenv
RUN         apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev

RUN         curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
ENV         PATH /root/.pyenv/bin:$PATH
RUN         pyenv install 3.6.3

# zsh
RUN         apt-get install -y zsh
RUN         wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true
RUN         chsh -s /usr/bin/zsh

# pyenv settings
RUN         echo 'export PATH="/root/.pyenv/bin:$PATH"' >> ~/.zshrc
RUN         echo 'eval "$(pyenv init -)"' >> ~/.zshrc
RUN         echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc

# pyenv virtualenv
RUN         pyenv virtualenv 3.6.3 app

# uWGSI install
RUN         /root/.pyenv/versions/app/bin/pip install uwsgi

# Nginx install
RUN         apt-get -y install nginx

# supervisor install
RUN         apt-get -y install supervisor


# rabbitMQ install
RUN         apt-get -y install rabbitmq-server

# python requirements install
ENV         LANG C.UTF-8
COPY        requirements.txt /srv/requirements.txt
RUN         /root/.pyenv/versions/app/bin/pip install -r /srv/requirements.txt
```

---
# Docker

Dockerfile

```docker{}
# Dockerfile
FROM        base
MAINTAINER  joo2theeon@gmail.com

ENV         LANG C.UTF-8
ENV         DJANGO_SETTINGS_MODULE config.settings.deploy

WORKDIR     /srv/app
COPY        . /srv/app

# pyenv virtualenv
RUN         pyenv local app
RUN         /root/.pyenv/versions/app/bin/pip install -r /srv/app/requirements.txt

# Nginx
RUN         cp /srv/app/.config/nginx/nginx.conf /etc/nginx/
RUN         cp /srv/app/.config/nginx/soundhub.conf /etc/nginx/sites-available/
RUN         rm -rf /etc/nginx/sites-enabled/*
RUN         ln -sf /etc/nginx/sites-available/soundhub.conf /etc/nginx/sites-enabled/

# log dir
RUN         mkdir -p /var/log/uwsgi/app
RUN         mkdir -p /var/log/celery/app

# manage.py
WORKDIR     /srv/app/soundhub
RUN         /root/.pyenv/versions/app/bin/python /srv/app/soundhub/manage.py collectstatic --noinput
RUN         /root/.pyenv/versions/app/bin/python /srv/app/soundhub/manage.py migrate --noinput

# supervisor
RUN         cp /srv/app/.config/supervisor/* /etc/supervisor/conf.d/
CMD         supervisord -n

# port
EXPOSE      80
```

---
# AWS S3
`django_storages` 사용
```python{}
# soundhub/config/settings/deploy.py
INSTALLED_APPS = [..., 'storages', ...]

# AWS S3 Access
config_secret = json.loads(open(CONFIG_SETTINGS_COMMON_FILE).read())
AWS_ACCESS_KEY_ID = config_secret['aws']['access_key_id']
AWS_SECRET_ACCESS_KEY = config_secret['aws']['secret_access_key']
AWS_STORAGE_BUCKET_NAME = config_secret['aws']['s3_bucket_name']
AWS_S3_SIGNATURE_VERSION = 's3v4'
AWS_S3_REGION_NAME = 'ap-northeast-2'

# S3 저장소 설정
DEFAULT_FILE_STORAGE = 'config.storages.MediaStorage'
STATICFILES_STORAGE = 'config.storages.StaticStorage'
MEDIAFILES_LOCATION = 'media'
STATICFILES_LOCATION = 'static'

MEDIA_URL = f'https://s3.ap-northeast-2.amazonaws.com/{AWS_STORAGE_BUCKET_NAME}/media/'
STATIC_URL = f'https://s3.ap-northeast-2.amazonaws.com/{AWS_STORAGE_BUCKET_NAME}/static/'
```

```python{}
# soundhub/config/storages.py
from django.conf import settings
from storages.backends.s3boto3 import S3Boto3Storage

class MediaStorage(S3Boto3Storage):
    location = settings.MEDIAFILES_LOCATION

class StaticStorage(S3Boto3Storage):
    location = settings.STATICFILES_LOCATION
```

---
# AWS RDS

`.config_secrets/settings_common.json`

```json{}
"databases": {
  "postgresql": {
    "default": {
      "ENGINE": "django.db.backends.postgresql",
      "HOST": "soundhub.czxnimwgemge.ap-northeast-2.rds.amazonaws.com",
      "PORT": "5432",
      "NAME": "soundhub",
      "USER": "joo2theeon",
      "PASSWORD" : "********"
    }
  }
}
```

`soundhub/config/settings/deploy.py`
```python{}
config_secret = json.loads(open(CONFIG_SETTINGS_COMMON_FILE).read())
DATABASES = config_secret['databases']['postgresql']
```

---
# Travis CI

```yaml{}
language: python
python:
  - 3.6
  - nightly
services:
  - postgresql
before_install:
  - openssl aes-256-cbc -K $encrypted_604cc47e123f_key -iv $encrypted_604cc47e123f_iv -in secrets.tar.enc -out secrets.tar -d
  - tar xvf secrets.tar
install:
  - pip install -r requirements.txt
before_script:
  - cd soundhub
  - psql -c 'create database travis_ci_test;' -U postgres
script:
  - python manage.py test
```

---
# Thanks

---
# Sentry

```Python{}
# Application definition
INSTALLED_APPS = [
    ...
    'raven.contrib.django.raven_compat',
    ...
]

RAVEN_CONFIG = {
    'dsn': 'https://e01ad1cebe374afba306dd30c0c95aec:f188b0f719e24508946c5ab72b4de3c8@sentry.io/259770',
    # If you are using git, you can also automatically configure the
    # release based on the git info.
    'release': raven.fetch_git_sha(os.path.abspath(os.pardir)),
}
```

{{% /section %}}

---
{{% section %}}
# Thanks
{{% /section %}}