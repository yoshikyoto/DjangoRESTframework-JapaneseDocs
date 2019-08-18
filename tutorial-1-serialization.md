# Introduction

このチュートリアルでは、 Web API に着目しながら簡単な pastebin ((そういうWebサービスがあるようです。ググってみてください。)) を作っていきます。
途中で、 REST framework によって作られたいろいろなコンポーネントについて説明します。
そして、どのように互いのコンポーネントが動作しているかを理解することで、全体が理解できるようになります。

このチュートリアルはかなり深いところまでやりますので、
読む前にクッキーと好きなビールを用意しておくとよいでしょう。
もし概要だけ知りたい場合は quickstart を読んでください。

**注意:** このチュートリアルのコードは https://github.com/encode/rest-framework-tutorial で見られます。
実際に動作しているものは https://restframework.herokuapp.com/ で見られます。

# Setting up a new environment

一番最初に、新しい仮想環境を用意しましょう。venvを使います。
仮想環境はプロジェクトの設定を他のプロジェクトと分けて管理できるため非常に便利です。

```sh
python3 -m venv env
source env/bin/activate
```

仮想環境を有効にした状態で、必要なパッケージをインストールします。

```sh
pip install django
pip install djangorestframework
pip install pygments  # コードのシンタックスハイライトのために使います
```

**注意:** 仮想環境を終了したい場合は、 `deactivate` コマンドで可能です。
venv のドキュメントも合わせてお読みください。
https://docs.python.org/3/library/venv.html

# Getting started

ではコードを各準備をしましょう。新しいプロジェクトを作成します。

```sh
cd ~
django-admin startproject tutorial
cd tutorial
```

プロジェクトを作成したら、 Web API を作成するためのアプリケーションを作成します。
((Django は一つの project の中に複数の app があるという構成になっています。詳細は Django のドキュメントを読んでください。))

```sh
python manage.py startapp snippets
```

`INSTALLED_APPS` に `rest_framework` と、先程新しく作った `snippets` を追加します。
`tutorial/settings.py` を編集します。

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
]
```

これで準備ができました。

# Creating a model to work with （モデルの作成）

このチュートリアルでは、シンプルなモデルである `Snippet` を作成するところから始めます。
`Snippet`　モデルはコードスニペットを保存するために使います。
`snippets/models.py` を編集します。 
メモ: 良いコードにはコメントが含まれています。
このチュートリアルに対応するリポジトリのコードにもコメントが書かれています。
しかしこのドキュメント内では、コードのみやすさのためにコメントは消してあります。

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ['created']
```

まだデータベースをマイグレーションしていない場合は、
Snippetモデルに対応するマイグレーションも作成し、マイグレーションしてください。

```sh
python manage.py makemigrations snippets
python manage.py migrate
```

# Creating a Serializer class

Web API を作るタメニ、 Snippet インスタンスを `json` のような形式にシリアライズしたり、
デシリアライズできる必要があります。
Django REST framework では Serializer を定義することでこれが可能になります。
Django の forms と似たようなものです。
`snippets` ディレクトリニ　`serializers.py` のようなファイルを作成し、下記のようにします。

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        バリデーションされたデータを使って Snippet オブジェクトを作成します
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        バリデーションされたデータを使って Snippet を更新し、 Snippet インスタンスを返します
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

一番上に定義されているのは、シリアライズ/デシリアライズされるフィールドです。
`create()` と `update()` メソッドは「どうやってインスタンスが作られるか」、
「どうやって編集するか」を定義しています。
これらは、 `serializer.save()` が呼ばれたときに利用されます。

Serializer は Django の `Form` クラスに似ています。
`required` `max_length` `default` のようなフィールドのバリデーションフラグを持ちます。

フィールドのフラグは、シリアライザがどのように表示するかについて決定します。
`'base_template': 'textarea.html'}` は Django の `Form` クラスの
`widget=widgets.Textarea` と同じです。
これは Browsable API の場合でも有用です。詳しくは後に出てきます。

本当は `ModelSerializer` クラスを利用することで、 簡単に save ができます。
これは後ほど出てきます。しかし今はわかりやすさのため、このようにシリアライザを定義しています。

# Working wuth Serializers

次に行く前に、Serializerクラスの使い方に慣れておきます。
Django shell に入ってください。

```sh
python manage.py shell
```

そうしたら、いくつかコードをimportして、2つのコードスニペットを作成してみましょう。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```

Snippetインスタンスが作成されましたので、シリアライズしてみましょう。

```python
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}
```

これで、モデルインスタンスをPythonのデータに変換しました。最後にこれを json でレンダリングします。

```python
content = JSONRenderer().render(serializer.data)
content
# b'{"id": 2, "title": "", "code": "print(\\"hello, world\\")\\n", "linenos": false, "language": "python", "style": "friendly"}'
```

デシリアライズも同様です。まずstreamをPythonのデータに変換して

```python
import io

stream = io.BytesIO(content)
data = JSONParser().parse(stream)
```

これをモデルインスタンスに変換します。

```python
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

API が form のように動作すると言いましたが、このように実際にシリアライザを利用してみると、
 form に似ていることがわかるでしょう。
 
モデルインスタンスだけでなくQuerySetもシリアライズできます。 
やり方は簡単で、シリアライザの引数に `many=True` を指定するだけです。

```python
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', ''), ('code', 'print("hello, world")'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

# Using ModelSerializers

先程の `SnippetSerializer` は `Snippet` モデルにある多くの情報を置き換えています。
もっと簡単にかけたら良いのでは、と思うかもしれません。

Django に `Form` と `ModelForm` があるように、 
REST framework にも `Serializer`と `ModelSerializer` があります。

先程のシリアライザを `ModelSerializer` を使って書き換えてみましょう。
`snippets/serializers.py` を再び開いて、 `SnippetSerializer` を以下のように書き換えます。

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
```

このシリアライザはこれを書くだけで、シリアライズするのに必要な情報を取得できるのです。
`python manage.py shell` で Django shell を開いてみて下記コマンドを試してみましょう。

```sh
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```

`ModelSerializer` はなにも不思議なことはしていません。
次のようなシリアライザを作成するショートカットのようなものです。

* 必要に応じて自動的にフールドを更新したりできる
* `create()` と `update()` のデフォルトの実装がされている

# Writing regular Django views using our Serializer

では Serializer を利用してどのようにして API を実装していくかを見ていきましょう。
とりあえず、 REST framework の機能を何も使わない普通の Django の view を書いてみましょう。

`snippets/views.py` を編集します。

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

今存在している Snippet をすべて表示する API と、 Snippet を作成する API の view を作成します。

```python
@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

`csrf_exempt` を書くことで、この view に対する POST には CSRF トークンが不要になります。
これは通常やってはいけません。
とくに REST framework を利用している場合はなおさらです。
今回だけ CSRF トークンを不要にします。

一つの Snippet を返す view も用意します。操作をやり直したりするために、更新と削除も用意します。

```python
@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

最後に、この view たちを使うために、 `snippets/urls.py` を作成します。

```python
from django.urls import path
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>/', views.snippet_detail),
]
```

`snippets/urls.py` を有効にするために、  `tutorial/urls.py` をこのようにします。

```python
from django.urls import path, include

urlpatterns = [
    path('', include('snippets.urls')),
]
```

今回とくに気にしていないエッジケースがあるので注意してください。
もし変な `json` を送ったり、 view でハンドリングできないメソッドを送ったりすると、 
500 "server error" を返します。

# Testing our first attempt at a Web API

そうしたらサーバーを起動してみましょう。

Django shell を終了させるには

```sh
quit()
```

そして Django サーバーを立ち上げましょう。

```sh
python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

もう一つターミナルを開いて、サーバーをテストしましょう。

curl でも httpie を使ってもいいです。
Httpie Python で書かれた使いやすいHTTPクライアントです。
インストールしてみましょう。

pip を使ってインストールできます。

```sh
pip install httpie
```

Snippet のリストを取得してみましょう。

```sh
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print(\"hello, world\")\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```

ID を指定して Snippet を見ることができます。

```sh
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print(\"hello, world\")\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

Web　ブラウザで URL にアクセスしても同じ json のレスポンスが見られると思います。

# Where are we nou

ここまでで、  Django の標準の view を利用して、 
Django の Forms のように、シリアライズを使えるようになったはずです。

この API は特に特別なことをしていない API です。
json のレスポンスで、エラーハンドリングもしていません。
しかし、 Web API の機能は持っています。

チュートリアルの part 2 でこれらをより良くする方法について学んでいきます。
