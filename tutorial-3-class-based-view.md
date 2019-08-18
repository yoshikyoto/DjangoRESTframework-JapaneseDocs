
API の view は function based view より class-based view で書いたほうが良いです。
class-based view にすることで共通部分を使い回せるので、DRY ((Don't Repeat Yourself))
なコードになります。

# Rewriting our API using class-based views

We'll start by rewriting the root view as a class-based view. All this involves is a little bit of refactoring of views.py.

では class-based view を使って今までの view を書き換えてみましょう。
`views.py` をリファクタリングしていきます。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

かなり良くなりました。以前のコードとかなり似てはいるのですが、 
HTTP メソッドごとの分離がはっきりしました。
他のクラスも修正していきましょう。

```python
class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

こちらもかなり見やすくなりました。
こちらももともとの function-based view と似ていますね。

class-based views に変更したことに合わせて、
`snippets/urls.py` も修正していきましょう。

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

以上でリファクタリングは完了です。
サーバーを起動して正しく動いているか確認してみましょう。

# Using mixins

class-based view を使うと、共通動作がまとめやすくなります。

今回のような create/retrieve/update/delete の操作は moedel が裏にある view の場合、
大体同じようなものになります。
このような共通の実装は REST framework のミックスインクラスに実装されています。

ミックスインを利用してどのように共通部分をまとめるかを説明します。
`views.py` を修正してください。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

何が起こったのでしょうか。
`GenericAPIView`, `ListModelMixin`, `CreateModelMixin` から view ができています。

ベースクラスは view の基本的な機能だけを提供します。
ミックスインクラスは `list()` や `create()` を提供します。
これらのメソッドを利用して `get` や `post` を適切に実装します。
とてもシンプルになりました。

```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

こちらも同様です。
`GenericAPIView` で基本的な機能が提供されていて、
ミックスインの機能で、 `.retrieve()`, `.update()`, `.destroy()` を実装しています。

# Using generic class-based views

ミックスインクラスを使ってコード量を少なくできましたが、更に減らすことができます。
REST framework には、すでにミックスインが使われた generic view が用意されています。
`views.py` を編集してみましょう。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

おお、めっちゃ簡単になりました。
コードはすごくきれいになり、 Django っぽくなりました。

次の Tutorial 4 では、 API で認証と権限をどう扱うかについて説明します。
