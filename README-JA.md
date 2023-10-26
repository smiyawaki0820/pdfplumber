# pdfplumber

[![Version](https://img.shields.io/pypi/v/pdfplumber.svg)](https://pypi.python.org/pypi/pdfplumber) ![Tests](https://github.com/jsvine/pdfplumber/workflows/Tests/badge.svg) [![Code coverage](https://codecov.io/gh/jsvine/pdfplumber/branch/stable/graph/badge.svg)](https://codecov.io/gh/jsvine/pdfplumber/branch/stable) [![Support Python versions](https://img.shields.io/pypi/pyversions/pdfplumber.svg)](https://pypi.python.org/pypi/pdfplumber)

PDF を詳しく調査して、各テキスト文字、矩形、および線に関する詳細情報を取得します。
さらに表の抽出と視覚的なデバッグも行います。

スキャンされた PDF よりも、機械によって生成された PDF の方が最適に動作します。
[`pdfminer.six`](https://github.com/goulu/pdfminer) をベースに構築されています。

現在、[Python 3.8, 3.9, 3.10, 3.11](.github/workflows/tests.yml) で [テスト](tests/) されています。

このドキュメントの翻訳は以下で利用可能です： [中国語 (by @hbh112233abc)](https://github.com/hbh112233abc/pdfplumber/blob/stable/README-CN.md)。

__バグを報告する場合__ や機能をリクエストする場合は、[issue から投稿してください](https://github.com/jsvine/pdfplumber/issues/new/choose)。
また __質問をする場合__ や特定の PDF での支援をリクエストする場合は、[ディスカッションフォーラムを使用してください](https://github.com/jsvine/pdfplumber/discussions)。

> 👋 このリポジトリのメンテナンスは、PDF データ抽出のコンサルティングプロジェクトのために雇用することができます。
> 費用の見積もりを取得するには、[Jeremy](https://www.jsvine.com/consulting/pdf-data-extraction/)（任意のサイズや複雑さ等プロジェクト関連を担当）、または [Samkit](https://www.linkedin.com/in/samkit-jain/)（表抽出を担当）に連絡してください。

## Table of Contents

- [Installation](#installation)
- [Command line interface](#command-line-interface)
- [Python library](#python-library)
- [Visual debugging](#visual-debugging)
- [Extracting text](#extracting-text)
- [Extracting tables](#extracting-tables)
- [Extracting form values](#extracting-form-values)
- [Demonstrations](#demonstrations)
- [Comparison to other libraries](#comparison-to-other-libraries)
- [Acknowledgments / Contributors](#acknowledgments--contributors)
- [Contributing](#contributing)

## Installation

```sh
pip install pdfplumber
```

## Command line interface

### Basic example

```sh
curl "https://raw.githubusercontent.com/jsvine/pdfplumber/stable/examples/pdfs/background-checks.pdf" > background-checks.pdf
pdfplumber < background-checks.pdf > background-checks.csv
```

出力は PDF 内のすべての文字、行、矩形に関する情報を含む CSV になります。

### Options

| 引数 | 解説 |
|----------|-------------|
|`--format [出力形式]`| `csv` または `json`。`json` フォーマットはより多くの情報を返します。PDF レベルとページレベルのメタデータ、さらには辞書ネストされた属性を含みます。|
|`--pages [ページリスト]`| スペースで区切られた、`1` から始まるページのリストまたはハイフンで区切られたページ範囲。例えば、`1, 11-15` は `1、11、12、13、14、15` ページのデータを返します。|
|`--types [抽出する領域のタイプリスト]`| 選択肢は `char`、`rect`、`line`、`curve`、`image`、`annot` など。デフォルトはすべての利用可能なものになります。|
|`--laparams`| JSON 形式の文字列（e.g.`'{"detect_vertical": true}'`） を `pdfplumber.open(..., laparams=...)` に渡すためのもの。|
|`--precision [integer]`| 浮動小数点数を四捨五入する小数点以下の桁数。デフォルトは四捨五入しない。|


## Python library

### Basic example

```python
import pdfplumber

with pdfplumber.open("path/to/file.pdf") as pdf:
    first_page = pdf.pages[0]
    print(first_page.chars[0])
```

### Loading a PDF

PDF を読み込むためには、`pdfplumber.open(x)` を呼び出します。
`x` には以下の値を指定できます：

- PDF ファイルへのパス
- バイトとして読み込まれるファイルオブジェクト
- バイトとして読み込まれるファイルライクなオブジェクト

`open` メソッドは `pdfplumber.PDF` クラスのインスタンスを返します。

パスワードで保護された PDF を読み込むには、引数 `password` を渡します。
例えば `pdfplumber.open("file.pdf", password="test")` のように指定します。

`pdfminer.six` のレイアウトエンジンにレイアウト解析パラメータを設定するには、`laparams` キーワード引数を渡します。
例えば `pdfplumber.open("file.pdf", laparams={"line_overlap": 0.7})` のように指定します。

無効なメタデータの値は、デフォルトで警告として扱われます。
`pdfplumber.open` において、メタデータを解析できない場合は例外が発生します。

### The `pdfplumber.PDF` class

トップレベルの `pdfplumber.PDF` クラスは 1 つの PDF を表し、2 つの主要な属性を持ちます：

| 属性 | 解説 |
|----------|-------------|
|`.metadata`| PDF の `Info` トレーラから引き出された、メタデータの key-value ペアの辞書。通常、"CreationDate"、"ModDate"、"Producer "などが含まれる。|
|`.pages`| 読み込んだページごとに `pdfplumber.Page` インスタンスを 1 つずつ含むリスト。|

... また、次のようなメソッドも含む：

| メソッド | 解説 |
|--------|-------------|
|`.close()`| デフォルトでは、`Page` オブジェクトはレイアウトとオブジェクト情報をキャッシュし、再処理の必要性を回避します。しかし、大きな PDF を解析する場合、これらのキャッシュされた属性は多くのメモリを必要とします。このメソッドを使ってキャッシュをフラッシュし、メモリを解放することができます。(バージョン `<= 0.5.25` では、 `.flush_cache()` を使用してください)|

### The `pdfplumber.Page` class

`pdfplumber.Page` クラスは `pdfplumber` の中核をなすものである。
`pdfplumber`で行うことのほとんどはこのクラスを中心に行われます。
このクラスには主に以下の属性があります：

| 属性 | 解説 |
|----------|-------------|
|`.page_number`| ページ番号。1 ページ目は `1`、2 ページ目は `2` というように、連続したページ番号。|
|`.width`| ページ幅。|
|`.height`| ページ高さ。|
|`.objects` / `.chars` / `.lines` / `.rects` / `.curves` / `.images`| これらの属性はそれぞれリストであり、各リストはページに埋め込まれたそのようなオブジェクトごとに 1 つの辞書を含んでいます。詳細については、以下の"[Objects](#objects) "を参照してください。|

... 主なメソッドは以下の通り:

| メソッド | 解説 |
|--------|-------------|
|`.crop(bounding_box, relative=False, strict=True)`| BBox に切り取ったページを返します。`bounding_box` は `(left, top, right, bottom)` のタプルを渡します。切り取ったページは、BBox 内に部分的に含まれるオブジェクトを保持します。オブジェクトが部分的にしかボックス内に入っていない場合、その寸法は BBox に合わせてスライスされます。`relative=True` の場合、BBox はページの BBox の左上からのオフセットとして計算され、絶対位置ではありません。([Issue #245](https://github.com/jsvine/pdfplumber/issues/245)に視覚的な例と説明があります) `strict=True`（デフォルト）の場合、クロップの BBox はページの BBox 内に完全に入っていなければなりません。|
|`.within_bbox(bounding_box, relative=False, strict=True)`| `.crop` に似ていますが、BBox の *完全に内部に* 含まれるオブジェクトのみを保持します。|
|`.outside_bbox(bounding_box, relative=False, strict=True)`| `.crop`および`.within_bbox`に似ていますが、BBox の *完全に外部に* 含まれるオブジェクトのみを保持します。|
|`.filter(test_function)`| `test_function(obj)` が `True` を返す `.objects` のみを持つページのバージョンを返します。|

その他の方法については、以下のセクションで説明する：

- [Visual debugging](#visual-debugging)
- [Extracting text](#extracting-text)
- [Extracting tables](#extracting-tables)

### Objects

`pdfplumber.PDF` と `pdfplumber.Page` の各インスタンスは、[`pdfminer.six`](https://github.com/pdfminer/pdfminer.six/) の PDF 解析から派生した、いくつかのタイプの PDF オブジェクトへのアクセスを提供します。
以下のプロパティはそれぞれ、マッチするオブジェクトの Python リストを返します：

- `.chars`、それぞれが1つのテキスト文字を表しています。
- `.lines`、それぞれが1次元のラインを表しています。
- `.rects`、それぞれが1つの2次元の矩形を表しています。
- `.curves`、それぞれが`pdfminer.six`がラインや矩形として認識しない一連の接続点を表しています。
- `.images`、それぞれが画像を表しています。
- `.annots`、それぞれが1つのPDF注釈を表しています（詳細は[公式PDF仕様](https://www.adobe.com/content/dam/acom/en/devnet/acrobat/pdfs/pdf_reference_1-7.pdf)のセクション8.4を参照）
- `.hyperlinks`、それぞれがサブタイプ`Link`を持ち、`URI`アクション属性を持つ1つのPDF注釈を表しています。

各オブジェクトは、以下のプロパティを持つシンプルなPythonの`dict`として表されます：


#### `char` properties

| Property | Description |
|----------|-------------|
|`page_number`| この文字が見つかったページ番号。|
|`text`| 例："z"、"Z"、" "。|
|`fontname`| 文字のフォント名。|
|`size`| フォントサイズ。|
|`adv`| テキストの幅 * フォントサイズ * スケーリング係数に等しい。|
|`upright`| 文字が直立しているかどうか。|
|`height`| 文字の高さ。|
|`width`| 文字の幅。|
|`x0`| ページの左側から文字の左側までの距離。|
|`x1`| ページの左側から文字の右側までの距離。|
|`y0`| ページの下から文字の底までの距離。|
|`y1`| ページの下から文字の上までの距離。|
|`top`| ページの上から文字の上までの距離。|
|`bottom`| ページの上から文字の底までの距離。|
|`doctop`| ドキュメントの上から文字の上までの距離。|
|`matrix`| この文字の「現在の変換行列」。（詳細は以下を参照。）|
|`mcid`| この文字の[マークされたコンテンツ](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850)セクションID（なければ`None`）。*実験的な属性。*|
|`tag`| この文字の[マークされたコンテンツ](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850)セクションタグ（なければ`None`）。*実験的な属性。*|
|`ncs`|TKTK|
|`stroking_pattern`|TKTK|
|`non_stroking_pattern`|TKTK|
|`stroking_color`| 文字の外郭の色（つまり、ストローク）。詳細は[docs/colors.md](docs/colors.md)を参照。|
|`non_stroking_color`| 文字の内部の色。詳細は[docs/colors.md](docs/colors.md)を参照。|
|`object_type`| "char"|


__NOTE__： キャラクタの`matrix`プロパティは、[PDFリファレンス](https://ghostscript.com/~robin/pdf_reference17.pdf)(第6版)のセクション4.2.2で説明されているように、「現在の変換行列」を表します。
この行列は、キャラクタのスケール、傾き、および位置の移動を制御します。
回転はスケールとスキューの組み合わせですが、ほとんどの場合、x軸のスキューと等しいと考えることができます。
`pdfplumber.ctm`サブモジュールは、これらの計算を補助するクラス `CTM` を定義している。
例えば：

```python
from pdfplumber.ctm import CTM
my_char = pdf.pages[0].chars[3]
my_char_ctm = CTM(*my_char["matrix"])
my_char_rotation = my_char_ctm.skew_x
```

#### `line` properties

| Property | Description |
|----------|-------------|
|`page_number`| Page number on which this line was found.|
|`height`| Height of line.|
|`width`| Width of line.|
|`x0`| Distance of left-side extremity from left side of page.|
|`x1`| Distance of right-side extremity from left side of page.|
|`y0`| Distance of bottom extremity from bottom of page.|
|`y1`| Distance of top extremity bottom of page.|
|`top`| Distance of top of line from top of page.|
|`bottom`| Distance of bottom of the line from top of page.|
|`doctop`| Distance of top of line from top of document.|
|`linewidth`| Thickness of line.|
|`stroking_color`|The color of the line. See [docs/colors.md](docs/colors.md) for details.|
|`non_stroking_color`|The non-stroking color specified for the line’s path. See [docs/colors.md](docs/colors.md) for details.|
|`mcid`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section ID for this line if any (otherwise `None`). *Experimental attribute.*|
|`tag`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section tag for this line if any (otherwise `None`). *Experimental attribute.*|
|`object_type`| "line"|

#### `rect` properties

| Property | Description |
|----------|-------------|
|`page_number`| Page number on which this rectangle was found.|
|`height`| Height of rectangle.|
|`width`| Width of rectangle.|
|`x0`| Distance of left side of rectangle from left side of page.|
|`x1`| Distance of right side of rectangle from left side of page.|
|`y0`| Distance of bottom of rectangle from bottom of page.|
|`y1`| Distance of top of rectangle from bottom of page.|
|`top`| Distance of top of rectangle from top of page.|
|`bottom`| Distance of bottom of the rectangle from top of page.|
|`doctop`| Distance of top of rectangle from top of document.|
|`linewidth`| Thickness of line.|
|`stroking_color`|The color of the rectangle's outline. See [docs/colors.md](docs/colors.md) for details.|
|`non_stroking_color`|The rectangle’s fill color. See [docs/colors.md](docs/colors.md) for details.|
|`mcid`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section ID for this rect if any (otherwise `None`). *Experimental attribute.*|
|`tag`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section tag for this rect if any (otherwise `None`). *Experimental attribute.*|
|`object_type`| "rect"|

#### `curve` properties

| Property | Description |
|----------|-------------|
|`page_number`| Page number on which this curve was found.|
|`pts`| Points — as a list of `(x, top)` tuples — describing the curve.|
|`height`| Height of curve's bounding box.|
|`width`| Width of curve's bounding box.|
|`x0`| Distance of curve's left-most point from left side of page.|
|`x1`| Distance of curve's right-most point from left side of the page.|
|`y0`| Distance of curve's lowest point from bottom of page.|
|`y1`| Distance of curve's highest point from bottom of page.|
|`top`| Distance of curve's highest point from top of page.|
|`bottom`| Distance of curve's lowest point from top of page.|
|`doctop`| Distance of curve's highest point from top of document.|
|`linewidth`| Thickness of line.|
|`fill`| Whether the shape defined by the curve's path is filled.|
|`stroking_color`|The color of the curve's outline. See [docs/colors.md](docs/colors.md) for details.|
|`non_stroking_color`|The curve’s fill color. See [docs/colors.md](docs/colors.md) for details.|
|`mcid`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section ID for this curve if any (otherwise `None`). *Experimental attribute.*|
|`tag`| The [marked content](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) section tag for this curve if any (otherwise `None`). *Experimental attribute.*|
|`object_type`| "curve"|

#### Derived properties

さらに、`pdfplumber.PDF` と `pdfplumber.Page` は、いくつかの派生オブジェクトリストにアクセスできる：
`.rect_edges`（各矩形を4本の線に分解する）、 `.curve_edges`（`曲線`オブジェクトに対して同じことを行う）、 `.edges`（`.rect_edges`、`.curve_edges`、`.lines`を組み合わせたもの）である。

#### `image` properties

[To be completed.]

### Obtaining higher-level layout objects via `pdfminer.six`

`pdfplumber.open(...)`に `pdfminer.six` を扱う `laparams` パラメータを渡すと、各ページの `.objects` 辞書には `pdfminer.six` の上位レベルのレイアウトオブジェクト、例えば `"textboxhorizontal"` も含まれるようになります。


## Visual debugging

`pdfplumber`のビジュアルデバッグツールは、PDFの構造や、そこから抽出されたオブジェクトを理解するのに役立ちます。


### Creating a `PageImage` with `.to_image()`

To turn any page (including cropped pages) into an `PageImage` object, call `my_page.to_image()`. You can optionally pass *one* of the  following keyword arguments:

- `resolution`: The desired number pixels per inch. Default: `72`. Type: `int`.
- `width`: The desired image width in pixels. Default: unset, determined by `resolution`. Type: `int`.
- `height`: The desired image width in pixels. Default: unset, determined by `resolution`. Type: `int`.
- `antialias`: Whether to use antialiasing when creating the image. Setting to `True` creates images with less-jagged text and graphics, but with larger file sizes. Default: `False`. Type: `bool`.

For instance:

```python
im = my_pdf.pages[0].to_image(resolution=150)
```

From a script or REPL, `im.show()` will open the image in your local image viewer. But `PageImage` objects also play nicely with Jupyter notebooks; they automatically render as cell outputs. For example:

![Visual debugging in Jupyter](examples/screenshots/visual-debugging-in-jupyter.png "Visual debugging in Jupyter")

*Note*: `.to_image(...)` works as expected with `Page.crop(...)`/`CroppedPage` instances, but is unable to incorporate changes made via `Page.filter(...)`/`FilteredPage` instances.


### Basic `PageImage` methods

| Method | Description |
|--------|-------------|
|`im.reset()`| Clears anything you've drawn so far.|
|`im.copy()`| Copies the image to a new `PageImage` object.|
|`im.show()`| Opens the image in your local image viewer.|
|`im.save(path_or_fileobject, format="PNG", quantize=True, colors=256, bits=8)`| Saves the annotated image as a PNG file. The default arguments quantize the image to a palette of 256 colors, saving the PNG with 8-bit color depth. You can disable quantization by passing `quantize=False` or adjust the size of the color palette by passing `colors=N`.|

### Drawing methods

You can pass explicit coordinates or any `pdfplumber` PDF object (e.g., char, line, rect) to these methods.

| Single-object method | Bulk method | Description |
|----------------------|-------------|-------------|
|`im.draw_line(line, stroke={color}, stroke_width=1)`| `im.draw_lines(list_of_lines, **kwargs)`| Draws a line from a `line`, `curve`, or a 2-tuple of 2-tuples (e.g., `((x, y), (x, y))`).|
|`im.draw_vline(location, stroke={color}, stroke_width=1)`| `im.draw_vlines(list_of_locations, **kwargs)`| Draws a vertical line at the x-coordinate indicated by `location`.|
|`im.draw_hline(location, stroke={color}, stroke_width=1)`| `im.draw_hlines(list_of_locations, **kwargs)`| Draws a horizontal line at the y-coordinate indicated by `location`.|
|`im.draw_rect(bbox_or_obj, fill={color}, stroke={color}, stroke_width=1)`| `im.draw_rects(list_of_rects, **kwargs)`| Draws a rectangle from a `rect`, `char`, etc., or 4-tuple bounding box.|
|`im.draw_circle(center_or_obj, radius=5, fill={color}, stroke={color})`| `im.draw_circles(list_of_circles, **kwargs)`| Draws a circle at `(x, y)` coordinate or at the center of a `char`, `rect`, etc.|

Note: The methods above are built on Pillow's [`ImageDraw` methods](http://pillow.readthedocs.io/en/latest/reference/ImageDraw.html), but the parameters have been tweaked for consistency with SVG's `fill`/`stroke`/`stroke_width` nomenclature.

### Troubleshooting ImageMagick on Debian-based systems

If you're using `pdfplumber` on a Debian-based system and encounter a `PolicyError`, you may be able to fix it by changing the following line in `/etc/ImageMagick-6/policy.xml` from this:

```xml
<policy domain="coder" rights="none" pattern="PDF" />
```

... to this:

```xml
<policy domain="coder" rights="read|write" pattern="PDF" />
```

(More details about `policy.xml` [available here](https://imagemagick.org/script/security-policy.php).)

## Extracting text

`pdfplumber` は任意のページ（切り取られたページや派生ページも含む）からテキストを抽出することができる。
また、テキストのレイアウトを保持したり、単語や検索クエリの座標を特定したりすることもできる。
`Page` オブジェクトは以下のテキスト抽出メソッドを呼び出すことができる：


| Method | Description |
|--------|-------------|
|`.extract_text(x_tolerance=3, y_tolerance=3, layout=False, x_density=7.25, y_density=13, **kwargs)`| Collates all of the page's character objects into a single string.<ul><li><p>When `layout=False`: Adds spaces where the difference between the `x1` of one character and the `x0` of the next is greater than `x_tolerance`. Adds newline characters where the difference between the `doctop` of one character and the `doctop` of the next is greater than `y_tolerance`.</p></li><li><p>When `layout=True` (*experimental feature*): Attempts to mimic the structural layout of the text on the page(s), using `x_density` and `y_density` to determine the minimum number of characters/newlines per "point," the PDF unit of measurement. All remaining `**kwargs` are passed to `.extract_words(...)` (see below), the first step in calculating the layout.</p></li></ul>|
|`.extract_text_simple(x_tolerance=3, y_tolerance=3)`| A slightly faster but less flexible version of `.extract_text(...)`, using a simpler logic.|
|`.extract_words(x_tolerance=3, y_tolerance=3, keep_blank_chars=False, use_text_flow=False, horizontal_ltr=True, vertical_ttb=True, extra_attrs=[], split_at_punctuation=False, expand_ligatures=True)`| Returns a list of all word-looking things and their bounding boxes. Words are considered to be sequences of characters where (for "upright" characters) the difference between the `x1` of one character and the `x0` of the next is less than or equal to `x_tolerance` *and* where the `doctop` of one character and the `doctop` of the next is less than or equal to `y_tolerance`. A similar approach is taken for non-upright characters, but instead measuring the vertical, rather than horizontal, distances between them. The parameters `horizontal_ltr` and `vertical_ttb` indicate whether the words should be read from left-to-right (for horizontal words) / top-to-bottom (for vertical words). Changing `keep_blank_chars` to `True` will mean that blank characters are treated as part of a word, not as a space between words. Changing `use_text_flow` to `True` will use the PDF's underlying flow of characters as a guide for ordering and segmenting the words, rather than presorting the characters by x/y position. (This mimics how dragging a cursor highlights text in a PDF; as with that, the order does not always appear to be logical.) Passing a list of `extra_attrs`  (e.g., `["fontname", "size"]` will restrict each words to characters that share exactly the same value for each of those [attributes](#char-properties), and the resulting word dicts will indicate those attributes. Setting `split_at_punctuation` to `True` will enforce breaking tokens at punctuations specified by `string.punctuation`; or you can specify the list of separating punctuation by pass a string, e.g., <code>split_at_punctuation='!"&\'()*+,.:;<=>?@[\]^\`\{\|\}~'</code>. Unless you set `expand_ligatures=False`, ligatures such as `ﬁ` will be expanded into their constituent letters (e.g., `fi`).|
|`.extract_text_lines(layout=False, strip=True, return_chars=True, **kwargs)`|*Experimental feature* that returns a list of dictionaries representing the lines of text on the page. The `strip` parameter works analogously to Python's `str.strip()` method, and returns `text` attributes without their surrounding whitespace. (Only relevant when `layout = True`.) Setting `return_chars` to `False` will exclude the individual character objects from the returned text-line dicts. The remaining `**kwargs` are those you would pass to `.extract_text(layout=True, ...)`.|
|`.search(pattern, regex=True, case=True, main_group=0, return_groups=True, return_chars=True, layout=False, **kwargs)`|*Experimental feature* that allows you to search a page's text, returning a list of all instances that match the query. For each instance, the response dictionary object contains the matching text, any regex group matches, the bounding box coordinates, and the char objects themselves. `pattern` can be a compiled regular expression, an uncompiled regular expression, or a non-regex string. If `regex` is `False`, the pattern is treated as a non-regex string. If `case` is `False`, the search is performed in a case-insensitive manner. Setting `main_group` restricts the results to a specific regex group within the `pattern` (default of `0` means the entire match). Setting `return_groups` and/or `return_chars` to `False` will exclude the lists of the matched regex groups and/or characters from being added (as `"groups"` and `"chars"` to the return dicts). The `layout` parameter operates as it does for `.extract_text(...)`. The remaining `**kwargs` are those you would pass to `.extract_text(layout=True, ...)`. __Note__: Zero-width and all-whitespace matches are discarded, because they (generally) have no explicit position on the page. |
|`.dedupe_chars(tolerance=1)`| Returns a version of the page with duplicate chars — those sharing the same text, fontname, size, and positioning (within `tolerance` x/y) as other characters — removed. (See [Issue #71](https://github.com/jsvine/pdfplumber/issues/71) to understand the motivation.)|

## Extracting tables

pdfplumberのテーブル検出へのアプローチは、Anssi Nurminenの修士論文を大いに取り入れており、Tabulaに触発されています。次のように動作します：

1. 任意のPDFページに対して、(a) 明示的に定義されているか、または (b) ページ上の単語の配置によって暗示されるラインを見つけます。
2. 重複する、またはほとんど重複するラインを統合します。
3. これらのラインのすべての交差点を見つけます。
4. これらの交差点を頂点として使用する最も詳細な矩形（つまり、セル）のセットを見つけます。
5. 連続するセルをテーブルにグループ化します。

### Table-extraction methods

`pdfplumber.Page` objects can call the following table methods:

| Method | Description |
|--------|-------------|
|`.find_tables(table_settings={})`|Returns a list of `Table` objects. The `Table` object provides access to the `.cells`, `.rows`, and `.bbox` properties, as well as the `.extract(x_tolerance=3, y_tolerance=3)` method.|
|`.find_table(table_settings={})`|Similar to `.find_tables(...)`, but returns the *largest* table on the page, as a `Table` object. If multiple tables have the same size — as measured by the number of cells — this method returns the table closest to the top of the page.|
|`.extract_tables(table_settings={})`|Returns the text extracted from *all* tables found on the page, represented as a list of lists of lists, with the structure `table -> row -> cell`.|
|`.extract_table(table_settings={})`|Returns the text extracted from the *largest* table on the page (see `.find_table(...)` above), represented as a list of lists, with the structure `row -> cell`.|
|`.debug_tablefinder(table_settings={})`|Returns an instance of the `TableFinder` class, with access to the `.edges`, `.intersections`, `.cells`, and `.tables` properties.|

For example:

```python
pdf = pdfplumber.open("path/to/my.pdf")
page = pdf.pages[0]
page.extract_table()
```

[Click here for a more detailed example.](examples/notebooks/extract-table-ca-warn-report.ipynb)

### Table-extraction settings

By default, `extract_tables` uses the page's vertical and horizontal lines (or rectangle edges) as cell-separators. But the method is highly customizable via the `table_settings` argument. The possible settings, and their defaults:

```python
{
    "vertical_strategy": "lines", 
    "horizontal_strategy": "lines",
    "explicit_vertical_lines": [],
    "explicit_horizontal_lines": [],
    "snap_tolerance": 3,
    "snap_x_tolerance": 3,
    "snap_y_tolerance": 3,
    "join_tolerance": 3,
    "join_x_tolerance": 3,
    "join_y_tolerance": 3,
    "edge_min_length": 3,
    "min_words_vertical": 3,
    "min_words_horizontal": 1,
    "keep_blank_chars": False,
    "text_tolerance": 3,
    "text_x_tolerance": 3,
    "text_y_tolerance": 3,
    "intersection_tolerance": 3,
    "intersection_x_tolerance": 3,
    "intersection_y_tolerance": 3,
}
```

| Setting | Description |
|---------|-------------|
|`"vertical_strategy"`| Either `"lines"`, `"lines_strict"`, `"text"`, or `"explicit"`. See explanation below.|
|`"horizontal_strategy"`| Either `"lines"`, `"lines_strict"`, `"text"`, or `"explicit"`. See explanation below.|
|`"explicit_vertical_lines"`| A list of vertical lines that explicitly demarcate cells in the table. Can be used in combination with any of the strategies above. Items in the list should be either numbers — indicating the `x` coordinate of a line the full height of the page — or `line`/`rect`/`curve` objects.|
|`"explicit_horizontal_lines"`| A list of horizontal lines that explicitly demarcate cells in the table. Can be used in combination with any of the strategies above. Items in the list should be either numbers — indicating the `y` coordinate of a line the full height of the page — or `line`/`rect`/`curve` objects.|
|`"snap_tolerance"`, `"snap_x_tolerance"`, `"snap_y_tolerance"`| Parallel lines within `snap_tolerance` pixels will be "snapped" to the same horizontal or vertical position.|
|`"join_tolerance"`, `"join_x_tolerance"`, `"join_y_tolerance"`| Line segments on the same infinite line, and whose ends are within `join_tolerance` of one another, will be "joined" into a single line segment.|
|`"edge_min_length"`| Edges shorter than `edge_min_length` will be discarded before attempting to reconstruct the table.|
|`"min_words_vertical"`| When using `"vertical_strategy": "text"`, at least `min_words_vertical` words must share the same alignment.|
|`"min_words_horizontal"`| When using `"horizontal_strategy": "text"`, at least `min_words_horizontal` words must share the same alignment.|
|`"intersection_tolerance"`, `"intersection_x_tolerance"`, `"intersection_y_tolerance"`| When combining edges into cells, orthogonal edges must be within `intersection_tolerance` pixels to be considered intersecting.|
|`"text_*"`| All settings prefixed with `text_` are then used when extracting text from each discovered table. All possible arguments to `Page.extract_text(...)` are also valid here.|
|`"text_x_tolerance"`, `"text_y_tolerance"`| These `text_`-prefixed settings *also* apply to the table-identification algorithm when the `text` strategy is used. I.e., when that algorithm searches for words, it will expect the individual letters in each word to be no more than `text_[x|y]_tolerance` pixels apart.|

### Table-extraction strategies

Both `vertical_strategy` and `horizontal_strategy` accept the following options:

| Strategy | Description | 
|----------|-------------|
| `"lines"` | Use the page's graphical lines — including the sides of rectangle objects — as the borders of potential table-cells. |
| `"lines_strict"` | Use the page's graphical lines — but *not* the sides of rectangle objects — as the borders of potential table-cells. |
| `"text"` | For `vertical_strategy`: Deduce the (imaginary) lines that connect the left, right, or center of words on the page, and use those lines as the borders of potential table-cells. For `horizontal_strategy`, the same but using the tops of words. |
| `"explicit"` | Only use the lines explicitly defined in `explicit_vertical_lines` / `explicit_horizontal_lines`. |

### Notes

- Often it's helpful to crop a page — `Page.crop(bounding_box)` — before trying to extract the table.

- Table extraction for `pdfplumber` was radically redesigned for `v0.5.0`, and introduced breaking changes.


## Extracting form values

PDFファイルには、人々が記入して保存できる入力を含むフォームが含まれることがあります。
フォームフィールドの値はPDFファイルの他のテキストのように表示されますが、フォームデータは異なる方法で処理されます。
詳細を知りたい場合は、この仕様書の671ページを参照してください。

pdfplumberにはフォームデータを操作するためのインターフェースはありませんが、pdfplumberのpdfminerのラッパーを使用してアクセスすることができます。

例えば、このスニペットはフォームフィールドの名前と値を取得して、それらを辞書に保存します。

```python
import pdfplumber
from pdfplumber.utils.pdfinternals import resolve_and_decode, resolve

pdf = pdfplumber.open("document_with_form.pdf")

def parse_field_helper(form_data, field, prefix=None):
    """ appends any PDF AcroForm field/value pairs in `field` to provided `form_data` list

        if `field` has child fields, those will be parsed recursively.
    """
    resolved_field = field.resolve()
    field_name = '.'.join(filter(lambda x: x, [prefix, resolve_and_decode(resolved_field.get("T"))]))
    if "Kids" in resolved_field:
        for kid_field in resolved_field["Kids"]:
            parse_field_helper(form_data, kid_field, prefix=field_name)
    if "T" in resolved_field or "TU" in resolved_field:
        # "T" is a field-name, but it's sometimes absent.
        # "TU" is the "alternate field name" and is often more human-readable
        # your PDF may have one, the other, or both.
        alternate_field_name  = resolve_and_decode(resolved_field.get("TU")) if resolved_field.get("TU") else None
        field_value = resolve_and_decode(resolved_field["V"]) if 'V' in resolved_field else None
        form_data.append([field_name, alternate_field_name, field_value])


form_data = []
fields = resolve(pdf.doc.catalog["AcroForm"])["Fields"]
for field in fields:
    parse_field_helper(form_data, field)
```

Once you run this script, `form_data` is a list containing a three-element tuple for each form element. For instance, a PDF form with a city and state field might look like this.
```
[
 ['STATE.0', 'enter STATE', 'CA'],
 ['section 2  accident infoRmation.1.0',
  'enter city of accident',
  'SAN FRANCISCO']
]
```

## Demonstrations

- [Using `extract_table` on a California Worker Adjustment and Retraining Notification (WARN) report](examples/notebooks/extract-table-ca-warn-report.ipynb). Demonstrates basic visual debugging and table extraction.
- [Using `extract_table` on the FBI's National Instant Criminal Background Check System PDFs](examples/notebooks/extract-table-nics.ipynb). Demonstrates how to use visual debugging to find optimal table extraction settings. Also demonstrates `Page.crop(...)` and `Page.extract_text(...).`
- [Inspecting and visualizing `curve` objects](examples/notebooks/ag-energy-roundup-curves.ipynb).
- [Extracting fixed-width data from a San Jose PD firearm search report](examples/notebooks/san-jose-pd-firearm-report.ipynb), an example of using `Page.extract_text(...)`.

## Comparison to other libraries

他にもPDFから情報を抽出するための Python ライブラリはいくつかあります。
大まかな概要として、pdfplumber は以下の特徴を組み合わせることで他の PDF 処理ライブラリとは異なっています：

各 PDF オブジェクトに関する詳細な情報への簡単なアクセス
テキストとテーブルを抽出するための高レベルでカスタマイズ可能な方法
緊密に統合されたビジュアルデバッグ
クロップボックスを介したオブジェクトのフィルタリングなどの他の便利なユーティリティ関数
また、pdfplumber が提供__しない__機能を知っておくと役立ちます：

- PDFの生成
- PDF の修正
- 光学的文字認識（OCR）
- OCR 処理されたドキュメントからのテーブル抽出の強力なサポート

### Specific comparisons

- [`pdfminer.six`](https://github.com/pdfminer/pdfminer.six) provides the foundation for `pdfplumber`. It primarily focuses on parsing PDFs, analyzing PDF layouts and object positioning, and extracting text. It does not provide tools for table extraction or visual debugging.

- [`PyPDF2`](https://github.com/mstamy2/PyPDF2) is a pure-Python library "capable of splitting, merging, cropping, and transforming the pages of PDF files. It can also add custom data, viewing options, and passwords to PDF files." It can extract page text, but does not provide easy access to shape objects (rectangles, lines, etc.), table-extraction, or visually debugging tools.

- [`pymupdf`](https://pymupdf.readthedocs.io/) is substantially faster than `pdfminer.six` (and thus also `pdfplumber`) and can generate and modify PDFs, but the library requires installation of non-Python software (MuPDF). It also does not enable easy access to shape objects (rectangles, lines, etc.), and does not provide table-extraction or visual debugging tools.

- [`camelot`](https://github.com/camelot-dev/camelot), [`tabula-py`](https://github.com/chezou/tabula-py), and [`pdftables`](https://github.com/drj11/pdftables) all focus primarily on extracting tables. In some cases, they may be better suited to the particular tables you are trying to extract.


## Acknowledgments / Contributors

Many thanks to the following users who've contributed ideas, features, and fixes:

- [Jacob Fenton](https://github.com/jsfenfen)
- [Dan Nguyen](https://github.com/dannguyen)
- [Jeff Barrera](https://github.com/jeffbarrera)
- [Bob Lannon](https://github.com/boblannon)
- [Dustin Tindall](https://github.com/dustindall)
- [@yevgnen](https://github.com/Yevgnen)
- [@meldonization](https://github.com/meldonization)
- [Oisín Moran](https://github.com/OisinMoran)
- [Samkit Jain](https://github.com/samkit-jain)
- [Francisco Aranda](https://github.com/frascuchon)
- [Kwok-kuen Cheung](https://github.com/cheungpat)
- [Marco](https://github.com/ubmarco)
- [Idan David](https://github.com/idan-david)
- [@xv44586](https://github.com/xv44586)
- [Alexander Regueiro](https://github.com/alexreg)
- [Daniel Peña](https://github.com/trifling)
- [@bobluda](https://github.com/bobluda)
- [@ramcdona](https://github.com/ramcdona)
- [@johnhuge](https://github.com/johnhuge)
- [Jhonatan Lopes](https://github.com/jhonatan-lopes)
- [Ethan Corey](https://github.com/ethanscorey)
- [Shannon Shen](https://github.com/lolipopshock)
- [Matsumoto Toshi](https://github.com/toshi1127)
- [John West](https://github.com/jwestwsj)
- [David Huggins-Daines](https://github.com/dhdaines)
- [Jeremy B. Merrill](https://github.com/jeremybmerrill)

## Contributing

Pull requests are welcome, but please submit a proposal issue first, as the library is in active development.

Current maintainers:

- [Jeremy Singer-Vine](https://github.com/jsvine)
- [Samkit Jain](https://github.com/samkit-jain)
