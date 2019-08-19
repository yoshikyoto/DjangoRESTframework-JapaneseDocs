# 原文

https://www.django-rest-framework.org/tutorial/5-relationships-and-hyperlinked-apis/

（データベースなどでは）主キーを使って「関係」を表現することがあると思います。
今回はこの「関係」を、ハイパーリンクをつかって表すことで、
「凝集」と「発見可能性」を改善する方法について説明します。

# Creating an endpoint for the root of our API

今、 snippets と users のエンドポイントがあります。
しかし、これらを同時に扱えるエンドポイントはありません。
そのようなエンドポイントを作るために、とりあえず通常の function-based な view と、
`@api_view` を使います。
`snippets/views.py` に以下を追記してください。

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

注目するべき点は2つあります。
１ ．完全なURLを取得するためには REST framework の `reverse` 関数を使うこと。
2. それぞれの URL が、 `snippets/urls.py` で後ほど定義するような、わかりやすい名前で区別されていること。
です。

# Creating an endpoint for the highlighted snippets

我々の API もう一つ明らかな問題点があります。コードハイライトのエンドポイントです。

他の API と違って、この API は JSON 形式ではなくて HTML 形式でレスポンスを返したい。
REST framework で HTMl を返す方法は2つある。
テンプレートをつかて HTMl をレンダリングする方法と、
プリレンダリングされた HTML を使う方法である。
今回は後者を採用したいと思う。

