# Quickstart

簡単なAPIを作成します。adminユーザーにだけユーザーとグループの閲覧と編集が可能なAPIを作成します。

# Project Setup

tutorial という名前のDjango Projectを作成し、quickstartというアプリを作成します。

```sh
# プロジェクトのディレクトリを作成
mkdir tutorial
cd tutorial

# virtual environment を作成する
python3 -m venv env
source env/bin/activate  # On Windows use `env\Scripts\activate`

# Django と Django REST framework を作成する
pip install django
pip install djangorestframework

# プロジェクトとアプリケーションの設定をします
django-admin startproject tutorial .  # Note the trailing '.' character
cd tutorial
django-admin startapp quickstart
cd ..
```

プロジェクトのレイアウトはこのようになります。

```sh
$ pwd
<some path>/tutorial
$ find .
.
./manage.py
./tutorial
./tutorial/__init__.py
./tutorial/quickstart
./tutorial/quickstart/__init__.py
./tutorial/quickstart/admin.py
./tutorial/quickstart/apps.py
./tutorial/quickstart/migrations
./tutorial/quickstart/migrations/__init__.py
./tutorial/quickstart/models.py
./tutorial/quickstart/tests.py
./tutorial/quickstart/views.py
./tutorial/settings.py
./tutorial/urls.py
./tutorial/wsgi.py
```

プロジェクトディレクトリ野中にアプリケーションが作成されるのはちょっとおかしく見えるかもしれません。
プロジェクトのnamespaceを作ることで、外部モジュールとの名前空間の衝突を回避できます。
（quickstartの話から外れますが）

データベースをマイグレーションします。

```sh
python manage.py migrate
```

`admin` というユーザーを作成します。パスワードは `password123` とします。
後にこのユーザーを認証に利用します。

```sh
python manage.py createsuperuser --email admin@example.com --username admin
```

これで準備は完了です。ついにアプリケーションのディレクトリを開いてコードを書きます。

# Serializers

まず、Serializerを定義します。
`tutorial/quickstart/serializers.py` というモジュールを作成します。
Serializerはデータを「表示」するのに使います。

```python
from django.contrib.auth.models import User, Group
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```

`HyperlinkedModelSerializer` というのを使っているのに注意してください。
普通は主キー（Primary Key）をRDBの「関係」に使うかと思いますが、RESTの場合はハイパーリンクを使うのがいいでしょう。

# Views

次に `tutorial/quickstart/views.py` を開いてください。

```python
from django.contrib.auth.models import User, Group
from rest_framework import viewsets
from tutorial.quickstart.serializers import UserSerializer, GroupSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
```

たくさんビューを書くのではなく、共通の挙動は ViewSets と呼ばれるクラスでまとめるのが良いです。

こう書かずとも個別に書くこともできますが、 簡潔に書くよりも、
ViewSets を利用したほうがロジックが整理されます。

# URLs

では次は API のURLとの紐付けを行いましょう。 `tutorial/urls.py` を開いてください。

```python
from django.urls import include, path
from rest_framework import routers
from tutorial.quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

viewではなくて ViewSetsを使っているので、自動的にURL設定を生成することができます。
Routeクラスを作成して登録するだけです。

もし、もっとURLをカスタマイズしたい場合は、class-based view を使えば、URL設定は明示的に書くことができます。

最後に、 Browsable API のログイン・ログアウトのためのviewを含めています。
これはオプショナルですが、APIに認証をつけたり、Brewsable URLを利用する場合は非常に便利です。

# Pagination

ページネーションとは、1ページにいくつのオブジェクトを返すか、を制御する方法です。
ページネーションを有効にするには、 `tutorial/settings.py` に以下の行を追記します。

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

# Settings

`'rest_framework'` を `tutorial/settings.py` の `INSTALLED_APPS` に追加してください。

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

これで準備ができました。

# Testing our API

作ったAPIのをテストする準備ができました。下記コマンドでサーバーを動かしてみましょう。

```sh
python manage.py runserver
```

APIにアクセスできるようになったはずです。
コマンドラインをつかってアクセスすることもできますし、curlを使ってもできるはずです。

```sh
bash: curl -H 'Accept: application/json; indent=4' -u admin:password123 http://127.0.0.1:8000/users/
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin"
        },
        {
            "email": "tom@example.com",
            "groups": [                ],
            "url": "http://127.0.0.1:8000/users/2/",
            "username": "tom"
        }
    ]
}
```

httpieというコマンドラインツールを使うとこうです。

```sh
bash: http -a admin:password123 http://127.0.0.1:8000/users/

HTTP/1.1 200 OK
...
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://localhost:8000/users/1/",
            "username": "paul"
        },
        {
            "email": "tom@example.com",
            "groups": [                ],
            "url": "http://127.0.0.1:8000/users/2/",
            "username": "tom"
        }
    ]
}
```

ブラウザから直接 `http://127.0.0.1:8000/users/` にアクセスしてもよいです。

ブラウザから確認する場合は、右上のログインボタンからログインしてください。

簡単でしょう？

REST framework についてもっと深く理解したい場合は、上のメニューにある((公式のドキュメントは上部にメニューがあります))API guideを見てください。
