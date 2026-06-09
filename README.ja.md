# Crystal Architecture

[English](./README.md) | 日本語

**Crystal Architecture** は、Python プロジェクトを成長可能で、読みやすく、再利用しやすく保つためのモジュール設計方針です。

局所的な配置ルールと一方向の依存関係によって、プロジェクト全体を結晶のように一貫した構造へ整理します。

目的は、コードベースが大きくなっても、ファイルツリー・モジュール名・型・公開 API から責務を読み取れる状態を保ち、認知負荷を可能な限り下げることです。

## コンセプト

Crystal Architecture は、次の考え方を重視します。

* ファイルツリーを見るだけで、プロジェクト全体の機能を把握できること
* ファイルパスから、クラス・関数・型・定数の責務が分かること
* 公開 API と内部実装の境界が明確であること
* モジュール間の依存関係が一方向に保たれること
* 名前・型・ファイル構造によって、できるだけ多くの意味を表現すること
* コメントや docstring は、コードだけでは表現できない重要な情報に限定すること

Crystal Architecture は、フレームワークではありません。

特定の実行時ライブラリを導入するものではなく、Python パッケージを設計・整理・保守するための、意見を持ったアーキテクチャガイドラインです。

## 基本原則

### 1. 1 ファイル 1 公開要素

1 つのファイルには、公開するクラス・関数・型・定数を 1 つだけ定義します。

```txt
user/
  _models/
    user.py          # User
  _types/
    user_id.py       # UserId
  _helpers/
    create_user.py   # create_user
```

プライベートな補助関数や内部実装は、同じファイルに含めても構いません。

このルールにより、ファイル名と公開要素の対応が明確になり、ファイルツリーから責務を読み取りやすくなります。

### 2. 機能はサブパッケージ単位で実装する

トップレベルパッケージに直接機能を実装せず、機能ごとのサブパッケージに分割します。

```txt
src/
  example/
    user/
    project/
    storage/
```

トップレベルパッケージは、プロジェクト全体の名前空間として扱います。

実際の機能は、`example.user` や `example.storage` のようなサブパッケージに配置します。

### 3. 内部実装ディレクトリを明示する

サブパッケージ内の内部実装ディレクトリには、原則として `_` prefix を付けます。

```txt
user/
  __init__.py
  _helpers/
  _models/
  _schemas/
  _services/
  _types/
  _utils/
```

これにより、公開 API と内部実装の境界が分かりやすくなります。

外部から利用される要素は、サブパッケージの `__init__.py` から明示的に export します。

### 4. 公開 API はサブパッケージのトップレベルに集約する

サブパッケージの利用者は、内部ディレクトリに直接依存せず、トップレベルから公開 API を import します。

```python
from example.user import User, UserId, create_user
```

内部実装のパスは、必要に応じて変更できるようにします。

```python
# Avoid
from example.user._models.user import User
from example.user._helpers.create_user import create_user
```

### 5. 依存関係は一方向に保つ

サブパッケージ内のモジュールは、下位レイヤーにのみ依存できます。

```txt
_helpers
  ↓
_models / _services / _operations / _exceptions
  ↓
_settings / _constants
  ↓
_schemas / _views
  ↓
_enums / _types / _utils
```

逆方向の依存は避けます。

同じレイヤー内で依存が必要な場合も、循環しないように一方向に保ちます。

### 6. 名前・型・パスで意味を表現する

コメントや docstring に頼りすぎず、可能な限り名前・型・ファイルパスで意味を表現します。

```python
from typing import TypeAlias

AgentName: TypeAlias = str
```

```python
agent_name: AgentName
```

コメントや docstring は、コード・型・名前だけでは表現できない重要な情報に限定します。

## 標準的なファイル構成

サブパッケージは、次のような構造を基本とします。

```txt
example/user/
  __init__.py
  _i18n.py
  _settings.py
  _constants/
    default_role.py
  _enums/
    user_status.py
  _exceptions/
    user_not_found_error.py
  _helpers/
    create_user.py
  _models/
    user.py
  _operations/
    normalize_user_name.py
  _schemas/
    user_config.py
  _services/
    user_service.py
  _types/
    user_id.py
  _utils/
    generate_user_id.py
  _views/
    user_view.py
```

各ディレクトリの役割は次の通りです。

| Directory     | Role                               |
| ------------- | ---------------------------------- |
| `_helpers`    | サブパッケージの機能を外部に提供する関数               |
| `_models`     | ビジネスロジックを含むデータモデル                  |
| `_services`   | クラスベースのビジネスロジック                    |
| `_operations` | 関数ベースの内部ロジック                       |
| `_schemas`    | ビジネスロジックを含まない内部データ構造               |
| `_views`      | インターフェイスでやり取りするデータ構造               |
| `_settings`   | Pydantic Settings などの設定            |
| `_constants`  | 定数                                 |
| `_exceptions` | 例外クラス                              |
| `_enums`      | Enum                               |
| `_types`      | TypeAlias、TypeVar、TypedDict などの型定義 |
| `_utils`      | 純粋でステートレスな再利用可能ユーティリティ             |
| `_i18n.py`    | 国際化対応の文字列定義                        |

すべてのディレクトリを常に作る必要はありません。

必要になった責務だけを追加します。

## `__init__.py` の役割

`__init__.py` は、サブパッケージの公開 API を定義する場所です。

```python
from ._helpers.create_user import create_user
from ._models.user import User
from ._types.user_id import UserId

__all__ = [
    "User",
    "UserId",
    "create_user",
]
```

遅延 import が必要な場合は、`__getattr__` を使って公開 API を遅延解決できます。

```python
from importlib import import_module
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ._helpers.create_user import create_user
    from ._models.user import User
    from ._types.user_id import UserId

__all__ = [
    "User",
    "UserId",
    "create_user",
]

def __getattr__(name: str) -> object:
    if name not in __all__:
        raise AttributeError(f"module {__name__} has no attribute {name}")

    module_map = {
        "User": "._models.user",
        "UserId": "._types.user_id",
        "create_user": "._helpers.create_user",
    }

    globals()[name] = getattr(import_module(module_map[name], __name__), name)
    return globals()[name]
```

## プラグインパターン

プラグインパターンを使う場合は、抽象化層と実装層を分離します。

```txt
example/storage/
  __init__.py
  _settings.py
  _helpers/
    create_storage.py
  _models/
    base_storage.py
  _types/
    storage.py
    storage_name.py

example/storage_impl/
  local/
    __init__.py
    _models/
      local_storage.py
  s3/
    __init__.py
    _settings.py
    _helpers/
      create_s3_storage.py
    _models/
      s3_storage.py
```

`example.storage` は安定した抽象化層です。

`example.storage_impl` は、具体的な実装を追加するための拡張領域です。

実装クラスは import path によって動的に解決し、利用者は抽象化層に依存します。

## 同期・非同期の両対応

同期版と非同期版の両方を提供する場合は、公開 API を分けつつ、共通ロジックを `_core` に集約します。

```txt
example/client/
  __init__.py
  asyncio.py
  _settings.py
  _core/
    _helpers/
    _schemas/
  _sync/
    _helpers/
  _async/
    _helpers/
```

同期版は次のように利用します。

```python
from example.client import get_client
```

非同期版は次のように利用します。

```python
from example.client.asyncio import get_client
```

## Pydantic を使う場合

Entity として Pydantic モデルを使う場合、`Field(description=...)` よりも、IDE の補完で確認しやすいフィールド docstring を優先します。

ただし、クラス名・フィールド名・型情報だけで意味が分かる場合は、docstring 自体を省略します。

Crystal Architecture では、ドキュメント自動生成よりも、コードを書くとき・読むときの体験を重視します。

## テスト構成

テストコードも、対象のサブパッケージと同じファイル構成にします。

```txt
src/
  example/
    user/
      _helpers/
        create_user.py

tests/
  example/
    user/
      _helpers/
        test_create_user.py
```

テストは関数形式で実装します。

原則として line coverage 100% を目指します。

ただし、型チェック回避や実行環境上ほぼ到達不能な分岐については、実装側に `# pragma: no cover` を付け、無理にテストしません。

## 適用に向いているプロジェクト

Crystal Architecture は、次のようなプロジェクトに向いています。

* 長期的に保守する Python パッケージ
* 複数人で開発するライブラリ
* SDK
* CLI ツール
* 業務アプリケーションのコアパッケージ
* プラグイン拡張を持つプロジェクト
* 同期・非同期 API の両方を提供するプロジェクト

## 適用しなくてもよいケース

次のようなケースでは、Crystal Architecture は過剰になる場合があります。

* 単発スクリプト
* 小さな PoC
* 使い捨ての実験コード
* ファイル数が非常に少ない個人用ツール

Crystal Architecture は、すべての Python コードに適用するためのルールではありません。

成長・再利用・保守を前提としたコードベースのための設計方針です。

## ドキュメント

詳細なドキュメントは `docs/` に追加予定です。

```txt
docs/
  en/
  ja/
```

主なドキュメント:

* Philosophy
* Principles
* Module Architecture
* Dependency Rules
* Public API
* Plugin Pattern
* Pydantic
* Sync and Async
* Testing
* Comments and Docstrings
* Glossary

## Examples

実装例は `examples/` に配置します。

```txt
examples/
  basic-package/
  plugin-package/
  sync-async-package/
```

## Templates

再利用可能な雛形は `templates/` に配置します。

```txt
templates/
  python-package/
```

将来的には、`copier`、`cookiecutter`、CLI、import rule checker などへ発展させることができます。

## License

このプロジェクトは OSS として公開することを想定しています。

ライセンスは、リポジトリの `LICENSE` を参照してください。

## Philosophy

Programming is Elegant.
