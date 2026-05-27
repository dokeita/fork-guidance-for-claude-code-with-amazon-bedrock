# Conventional Commits 1.0.0

> 出典: https://www.conventionalcommits.org/ja/v1.0.0/

## 概要

Conventional Commits の仕様はコミットメッセージのための軽量の規約です。明示的なコミット履歴を作成するための簡単なルールを提供します。この規則に従うことで自動化ツールの導入を簡単にします。コミットメッセージで機能追加・修正・破壊的変更などを説明することで、この規約は [SemVer](http://semver.org/lang/ja/) と協調動作します。

コミットメッセージは次のような形にする必要があります:

```
<型>[任意 スコープ]: <タイトル>

[任意 本文]

[任意 フッター]
```

あなたのライブラリの利用者に意図を伝えるために、コミットは以下の構造化された要素を持ちます：

1. **fix:** 型 `fix` を持つコミットはコードベースのバグにパッチを当てます (SemVer の `PATCH` に相当)。
2. **feat:** 型 `feat` を持つコミットはコードベースに新しい機能を追加します (SemVer の `MINOR` に相当)。
3. **BREAKING CHANGE:** フッターに `BREAKING CHANGE:` が書かれているか型/スコープの直後に `!` が追加されているコミットは API の破壊的変更を導入します (SemVer の `MAJOR` に相当)。`BREAKING CHANGE` は任意の型のコミットに含めることができます。
4. `fix:` や `feat:` 以外の型も許されています。たとえば `build:`, `chore:`, `ci:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:` などを推奨しています。
5. `BREAKING CHANGE: <タイトル>` 以外のフッターが与えられるかもしれません。

## 例

### タイトルおよび破壊的変更のフッターを持つコミットメッセージ

```
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending other config files
```

### 破壊的変更を目立たせるために `!` を持つコミットメッセージ

```
feat!: send an email to the customer when a product is shipped
```

### スコープおよび `!` を持つコミットメッセージ

```
feat(api)!: send an email to the customer when a product is shipped
```

### 本文を持たないコミットメッセージ

```
docs: correct spelling of CHANGELOG
```

### スコープを持つコミットメッセージ

```
feat(lang): add polish language
```

### 複数段落からなる本文と複数のフッターを持ったコミットメッセージ

```
fix: prevent racing of requests

Introduce a request id and a reference to latest request. Dismiss
incoming responses other than from latest request.

Remove timeouts which were used to mitigate the racing issue but are
obsolete now.

Reviewed-by: Z
Refs: #123
```

## 仕様

1. コミットは `feat` や `fix` などの型から始まり (MUST)、その後ろにはスコープ (OPTIONAL) と `!` (OPTIONAL) が続き、その後ろにコロンとスペース (REQUIRED) が続く。
2. コミットがアプリケーションやライブラリに新しい機能を追加するとき、型 `feat` が使われなければならない (MUST)。
3. コミットがアプリケーションのためのバグ修正を行うとき、型 `fix` が使われなければならない (MUST)。
4. スコープを型の後ろに記述してもよい (MAY)。スコープは、コードベースのセクションを記述する括弧で囲まれた名詞にしなければならない (MUST)。例: `fix(parser):`。
5. 型/スコープの後ろのコロンとスペースの直後にタイトルが続かなければならない (MUST)。タイトルはコード変更の短かい要約である。
6. 短いタイトルの後ろにより長いコミットの本文を追加してもよい (MAY)。本文はタイトルの下の 1 行の空行から始めなければならない (MUST)。
7. コミットの本文は自由な形式であり、改行で区切られた複数の段落で構成することができる (MAY)。
8. ひとつ以上のフッターを、本文の下の 1 行の空行に続けて書くことができる (MAY)。それぞれのフッターは、ひとつの単語トークン、それに続く `:<space>` か `<space>#` によるセパレータ、そして文字列の値から構成されなければならない (MUST)。
9. フッターのトークンは空白の代わりに `-` を使わなければならない (MUST)。例: `Acked-by`。例外として `BREAKING CHANGE` がある。
10. フッターの値にはスペースと改行を含めることができる (MAY)。
11. 破壊的変更は、コミットの型/スコープの接頭辞か、フッターによって明示されなければならない (MUST)。
12. 破壊的変更がフッターとして含まれる場合は、大文字の `BREAKING CHANGE` の後ろにコロンとスペース、そしてタイトルを続けなければならない (MUST)。
13. 破壊的変更が型/スコープの接頭辞として含まれる場合は、`:` の直前に `!` を用いて明示されねばならない (MUST)。
14. `feat` と `fix` 以外の型を使うことができる (MAY)。例: `docs: updated ref docs.`。
15. Conventional Commits を構成する情報の単位は、大文字の `BREAKING CHANGE` を除いて、実装は大文字と小文字を区別してはならない (MUST NOT)。
16. フッターのトークンにおいて `BREAKING-CHANGE` は `BREAKING CHANGE` と同じトークンとして解釈されなければならない (MUST)。

## 何故 Conventional Commits を使うのか

- 変更履歴 (CHANGELOG) を自動的に生成できる。
- semantic version 単位で自動的に履歴をまとめられる (コミットの型に基づく)。
- チームメイトや一般のユーザー、およびその他の利害関係者へ変更の内容を伝えることができる。
- ビルドや公開の処理をトリガーできる。
- より構造化されたコミット履歴を調査できるようにすることで、人々がプロジェクトに貢献しやすくなる。

---

License: [Creative Commons - CC BY 3.0](https://creativecommons.org/licenses/by/3.0/)
