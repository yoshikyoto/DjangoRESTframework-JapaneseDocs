# Tutorial 2: Requests and Responses

# Request objects

REST framework は HttpRequest　を拡張した Request クラスがある。
Request　クラスはより柔軟にリクエストをパースできる。
Request クラスの中で最も重要なのは `request.data` である。
これは `request.POST` のようなものであるが、もっと Web API にとって使いやすいものになっている。

```python
request.POST  # POST された form-data しかハンドリングしていない
request.data  # POST, PUT, PATCH など任意のものをハンドリングできる
```

# Response objects

REST framework は TemplateResponse を拡張した Response クラスがある。
これは描画不要なコンテントを扱えたり、クライアントが要求した content type を正しく返せたりする機能を持つ。

# Status codes

view で、数値で HTTP ステータスコードを扱うのはあまり読みやすいとは言えない。
また、エラーの際のステータスコードが間違っていたとしても気づかない。
REST framework は `HTTP_400_BAD_REQUEST` のような定数を `status` モジュールで提供している。
これにより明確にステータスコードが何を表しているのかがわかる。
数字でステータスコードを扱うよりはわかりやすいはずだ。

# Wrapping API views

REST framework は API　の view を実装するときに使える2つのラッパーがある。

1. function-based view で利用できる　`@api_view` デコレータ
2. class-based view で利用できる `APIView`

このラッパーは view にリクエスト内容を `Request` のインスタンスとして渡したり、
`Response` オブジェクトにコンテンツネゴシエーションなどのコンテキストを追加してくれたりする。

また、このラッパーは `405 Method Not Allowed` などのレスポンスを適切にかえしてくれる機能もある。
あるいは、 `ParseError` をハンドリングしてくれる機能もある。
`ParseError` はリクエストがおかしくて `request.data` へのアクセスがエラーになった場合に出る例外である。

# Pulling it all together

では、今紹介したコンポーネントを view で使ってみましょう。

もはや `JSONResponse` クラスは `views.py` には必要ありません。
`JSONResponse` を消していきましょう。

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

これで view はちょっと改善されました。
ちょっと簡潔になりましたし、動作も Forms に近くなってきました。
ステータスコードに定数を使いましたが、これによりよりわかりやすくなりましたね。

単体の Snippet を表示する view は以下の通りです。

```python
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

Django の view のような見た目になって馴染みやすくなりましたね。

リクエストやレスポンスの content type どうだということは気にしなくて良くなりました。
`request.data` は json のリクエストでもハンドリングできますし、他の形式のリクエストでもいけます。
戻り値の Response オブジェクトも同様です。
我々が何も気にしなくても、 REST framework が自動的に適切な content type のものを返してくれます。

# Adding optional format suffixed to our URLs

我々がレスポンスを（jsonなどの形式に）フォーマットしなくても良くなったので、
API のエンドポイントの末尾でフォーマットを指定できるようにしましょう。
URL の末尾にフォーマットをつけることで、どのようなフォーマットなのか見てわかるようになります。
例えば、  http://example.com/api/items/4.json のようなエンドポイントです。

view に `format` 引数を追加しましょう。

```python
def snippet_list(request, format=None):
```

```python
def snippet_detail(request, pk, format=None):
```

そして、 `snippets/urls.py` で、定義されているURLに `format_suffix_patterns` を追加しましょう。

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.snippet_list),
    path('snippets/<int:pk>', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

このURLの拡張子は特につける必要はないのですが、これをつけることで、シンプルかつわかりやすくレスポンスのフォーマットを指定できます。

# How's it looking

では、 Tutorial 1 でやったようにコマンドラインから API をテストしてみましょう。
ちゃんと動いていれば、変なリクエストを送った時のエラーハンドリングもいい感じになっているはずです。

下記の様な方法で Snippet のリストを取得できます。

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

`Accept` ヘッダーを利用してレスポンスのフォーマットを制御できます。

```sh
http http://127.0.0.1:8000/snippets/ Accept:application/json  # JSON をリクエスト
http http://127.0.0.1:8000/snippets/ Accept:text/html         # HTML をリクエスト
```

末尾の拡張子でも指定できます。

```sh
http http://127.0.0.1:8000/snippets.json  # JSON 拡張子
http http://127.0.0.1:8000/snippets.api   # Browsable API の拡張子
```

リクエストについても `Content-Type` ヘッダで制御できます。

```sh
# form data 形式で POST
http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
  "id": 3,
  "title": "",
  "code": "print(123)",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# JSON で POST
http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

{
    "id": 4,
    "title": "",
    "code": "print(456)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

`--debug` オプションを　`http` コマンドにつけると、リクエストヘッダのリクエストタイプがわかります。

http://127.0.0.1:8000/snippets/ に Web ブラウザでアクセスしても API にアクセスできます。

# Browsability 

API はレスポンスの形式をリクエスト内容によって決めるので、
ブラウザで見た時はデフォルトで HTML 形式のレスポンスを返します。
これにより、 Web ブラウザで閲覧可能な API （Borwsable API と呼ばれていたもの）
の実現が可能になっています。

Web ブラウザで閲覧可能な API は非常に使いやすいです。
開発にも役に立ちます。
開発者にとってもっとも敷居の低いく、かつ求められていた API の動作確認方法です。

この Browsable API について詳細に知りたい場合や、もっとカスタマイズしたい場合は、
Brewsable API の章を見てください。

# What's next?

Tutorial 3 では、 class-based view を扱い、
voew のコード量を減らしていく方法を考えます。