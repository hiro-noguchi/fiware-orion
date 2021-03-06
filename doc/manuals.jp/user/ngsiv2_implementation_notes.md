# <a name="top"></a>NGSIv2 実装ノート (NGSIv2 Implementation Notes)

* [禁止されている文字](#forbidden-characters)
* [通知のカスタムペイロードデコード](#custom-payload-decoding-on-notifications)
* [カスタム通知を無効にするオプション](#option-to-disable-custom-notifications)
* [カスタム通知の変更不可能なヘッダ](#non-modifiable-headers-in-custom-notifications)
* [エンティティ・ロケーションの属性に制限](#limit-to-attributes-for-entity-location)
* [通知の従来の属性フォーマット](#legacy-attribute-format-in-notifications)
* [日時サポート](#datetime-support)
* [ユーザ属性または組み込み名前と一致するメタデータ](#user-attributes-or-metadata-matching-builtin-name)
* [サブスクリプション・ペイロードの検証](#subscription-payload-validations)
* [`actionType` メタデータ](#actiontype-metadata)
* [`noAttrDetail` オプション](#noattrdetail-option)
* [通知スロットリング](#notification-throttling)
* [異なる属性型間の順序付け](#ordering-between-different-attribute-value-types)
* [初期通知](#initial-notifications)
* [Oneshot サブスクリプション](#oneshot-subscriptions)
* [レジストレーション](#registrations)
* [`POST /v2/op/notify` でサポートされない `keyValues`](#keyvalues-not-supported-in-post-v2opnotify)
* [廃止予定の機能](#deprecated-features)

このドキュメントでは、Orion Context Broker が [NGSI v2 仕様](http://telefonicaid.github.io/fiware-orion/api/v2/stable/)で行った具体的な実装について考慮する必要があるいくつかの考慮事項について説明します。

<a name="forbidden-characters"></a>
## 禁止されている文字

NGSIv2 仕様の "フィールド構文の制限" セクションから :

> 上記のルールに加えて、NGSIv2 サーバの実装では、クロス・スクリプト注入攻撃を避けるために、それらのフィールドまたは他のフィールドに構文上の制限を追加することができます。

Orion に適用される追加の制限事項は、マニュアルの [禁止されている文字](forbidden_characters.md)のセクションに記載されているものです。

[トップ](#top)

<a name="custom-payload-decoding-on-notifications"></a>
## 通知でのカスタムペイロードデコード

禁止された文字の制限のために、Orion は発信カスタム通知に追加のデコード・ステップを適用します。これについては、このマニュアルの [このセクション](forbidden_characters.md#custom-payload-special-treatment)で詳しく説明します。

[トップ](#top)

<a name="option-to-disable-custom-notifications"></a>
## カスタム通知を無効にするオプション

Orion は、`-disableCustomNotifications` [CLI パラメータ](../admin/cli.md)を使用してカスタム通知を無効にするように設定できます。

この場合 :

* `httpCustom` が `http` として解釈されます。すなわち、`url` を除くすべてのサブフィールドは無視されます。
* マクロ置換 `${...}` は実行されません。

[トップ](#top)

<a name="non-modifiable-headers-in-custom-notifications"></a>
## カスタム・通知の変更不可能なヘッダ

カスタム・通知で次のヘッダを上書きすることはできません :

* `Fiware-Correlator`
* `Ngsiv2-AttrsFormat`

そのような試み (例えば `"httpCustom": { ... "headers": {"Fiware-Correlator": "foo"} ...}`) は無視されます。

[トップ](#top)

<a name="limit-to-attributes-for-entity-location"></a>
## エンティティ・ロケーションの属性に制限

NGSIv2 仕様の "エンティティの地理空間プロパティ" のセクションから :

> クライアントアプリケーションは、(適切な NGSI アトリビュート型を提供することによって) ジオスペース・プロパティを伝えるエンティティ属性を定義する責任があります。通常これは `location` という名前のついたエンティティ属性ですが、エンティティに複数の地理空間属性が含まれているユース・ケースはありません。たとえば、異なる粒度レベルで指定された場所、または異なる精度で異なる場所の方法によって提供された場所です。それにもかかわらず、空間特性には、バックエンド・データベースによって課せられたリソースの制約下にある特別なインデックスが必要であることは注目に値します。したがって、実装では、空間インデックスの制限を超えるとエラーが発生する可能性があります。これらの状況で推奨される HTTP ステータス・コードは `413` です。リクエスト・エンティティが大きすぎます。また、レスポンス・ペイロードで報告されたエラーは、`NoResourcesAvailable` である必要があります。

Orion の場合、その制限は1つの属性です。

[トップ](#top)

<a name="legacy-attribute-format-in-notifications"></a>
## 通知の従来の属性フォーマット

NGSIv2 仕様の `attrsFormat` で述べている値とは別に、Orion は、NGSIv1 形式の通知を送信するために、`legacy` 値もサポートしています。このようにして、ユーザは NGSIv1 レガシー通知レシーバを使用した NGSIv2 サブスクリプション (フィルタリングなど) の拡張の恩恵を受けることができます。

NGSIv1 は非推奨であることに注意してください。したがって、`legacy` 通知形式をもう使用しないことを推奨します。

[トップ](#top)

<a name="datetime-support"></a>
## 日時のサポート

NGSIv2 仕様の "Special Attribute Types" セクションから :

> DateTime : 日付を ISO8601 形式で識別します。これらの属性は、より大きい、より小さい、より大きい、等しい、より小さい、等しいおよび範囲のクエリ演算子で使用できます。

次の考慮事項は、属性の作成/更新時間または `q` と `mq` フィルタで使用される時に考慮しなければならない :

* Datetimes は、date, time, timezone の指定子で構成され、次のいずれかのパターンで表されます :
    * `<date>`
    * `<date>T<time>`
    * `<date>T<time><timezone>`
    * フォーマット `<date><timezone>` は許可されていないことに注意してください。ISO8601 によると : *"タイムゾーン指定子が必要な場合、それは結合された日付と時刻に従います"*
* `<date>` ついては、それがパターンに従わなければならない : `YYYY-MM-DD`
    * `YYYY`: year (4桁)
    * `MM`: month (2桁)
    * `DD`: day (2桁)
* これについて `<time>` は、[ISO8601 仕様](https://en.wikipedia.org/wiki/ISO_8601#Times)に記述されているパターンのいずれかに従わなければならない :
    * `hh:mm:ss.sss` または `hhmmss.sss`。現時点では、Orion は内部的には `.00` として保存されますが、マイクロ秒 (またはより小さい解像度) を含む時間を処理することができます。ただし、これは将来変更される可能性があります ([関連する問題](https://github.com/telefonicaid/fiware-orion/issues/2670)を参照)
    * `hh:mm:ss` または `hhmmss` です
    * ``hh:mm` または `hhmm`。この場合、秒は `00` に設定されます
    * `hh`。この場合、分と秒は `00` に設定されます
    * もし `<time>` 省略された場合、時、分、秒が `00` に設定されます
* `<timezones>` については、[ISO8601 仕様](https://en.wikipedia.org/wiki/ISO_8601#Time_zone_designators)に記述されているパターンのいずれかに従わなければならない :
    * `Z`
    * `±hh:mm`
    * `±hhmm`
    * `±hh`
* ISO8601 は、*" 時間表現で UTC 関係情報が与えられていない場合、その時間は現地時間であると想定される "* と規定している。ただし、クライアントとサーバが異なるゾーンにある場合、これはあいまいです。したがって、この曖昧さを解決するために、時間帯指定子が省略されている場合、Orion は常にタイムゾーン `Z` を想定します

Orion は常にフォーマット `YYYY-MM-DDThh:mm:ss.ssZ` を使用して日時属性/メタデータを提供します。クライアント/レシーバが任意のタイムゾーンで実行されている可能性があるため、UTC/Zulu タイムゾーンを使用していることに注意してください。これは将来変更される可能性があります ([関連する問題](https://github.com/telefonicaid/fiware-orion/issues/2663)を参照)。

属性とメタデータの型としての文字列 "ISO8601" もサポートされています。この効果は、"DateTime" を使用した場合と同じです。

[トップ](#top)

<a name="user-attributes-or-metadata-matching-builtin-name"></a>
## ユーザ属性または組み込み名前と一致するメタデータ

(このセクションの内容は、`dateExpires` 属性を除くすべての組み込み関数に適用されます。`dateExpires` に関する特定の情報については、[一時的なエンティティ](transient_entities.md)のドキュメントを参照してください)。

まず、**NGSIv2 組み込み属性やメタデータと同じ名前のものを使用しないことを強く推奨します**。実際、NGSIv2 仕様では、これを禁止しています。仕様の "属性名の制限" と "メタデータ名の制限" のセクションを参照してください。

しかし、もし、あなたがそのような属性やメタデータを (おそらくレガシーの理由により) 持っていたとしても、以下の点を考慮に入れてください :

* NGSIv2 組み込みの属性および/またはメタデータと同じ名前のものを作成/更新することができます。Orion は、そうするでしょう
* ユーザ定義の属性および/またはメタデータは、GET リクエストまたはサブスクリプションで明示的に宣言する必要なく表示されます。 たとえば、エンティティ E1 に値 "2050-01-01" を持つ、`dateModified` 属性を作成した場合、`GET /v2/entities/E1` がそれを取得します。`?attrs=dateModified` を使う必要はありません。
* (GET オペレーションまたは通知にレスポンスして) レンダリングされると、明示的に宣言されていても、ユーザ定義の属性/メタデータが組み込み関数より優先されます。たとえば、エンティティ E1 に値 "2050-01-01" を持つ `dateModified` 属性を作成し、`GET /v2/entities?attrs=dateModified` をリクエストすると、"2050-01-01" が得られます。
* しかし、フィルタリング (すなわち `q` または `mq`) は組み込み関数の値に基づいています。 たとえば、エンティティ E1 に値 "2050-01-01" を持つ `dateModified` 属性を作成し、`GET /v2/entities?q=dateModified>2049-12-31` をリクエストした場合、エンティティは取得されません。"2050-01-01" は "2049-12-31" よりも大きいですが、エンティティを変更した日付 (2018年か2019年のいずれかの日付) は "2049-12-31" より大きくなりません。これは何らかの形で矛盾していることに注意してください (つまり、ユーザ定義はレンダリングでは優先されますがフィルタリングでは優先されません)、将来変更される可能性があります。

[トップ](#top)

<a name="subscription-payload-validations"></a>
## サブスクリプション・ペイロードの検証

Orion が NGSIv2 サブスクリプション・ペイロードで実装する特定の検証は、次のとおりです :

* **description**: オプション (最大長1024)
* **subject**: 必須
    * **entities**: 必須
        * **id** or **idPattern**: そのうちの1つは必須です (ただし、同時に両方は許可されません)。id は ID の NGSIv2 制限に従わなければなりません。idPattern は空ではなく、有効な正規表現でなければなりません
        * **type** or **typePattern**: 任意です (ただし、両方同時に許可されません)。type は ID の NGSIv2 制限に従う必要があります。型は空であってはいけません。typePattern は有効な正規表現で、空ではない必要があります
    * **condition**: オプション (但し、存在する場合は内容を持たなければなりません。つまり `{}` は許可されていません)
        * **attrs**: オプション (ただし、存在する場合はリストでなければなりません。空リストも許されます)
        * **expression**: オプション (ただし、存在する場合は内容を持たなければなりません。つまり {} は許可されていません)
            * **q**: オプション (ただし、存在する場合は空でなければなりません。つまり `""` は許可されていません)
            * **mq**: オプション (ただし、存在する場合は空でなければなりません。つまり `""` は許可されていません)
            * **georel**: オプション (ただし、存在する場合は空でなければなりません。つまり `""` は許可されていません)
            * **geometry**: オプション (ただし、存在する場合は空でなければなりません。つまり `""` は許可されていません)
            * **coords**: オプション (ただし、存在する場合は空でなければなりません。つまり `""` 許可されていません)
* **notification**:
    * **http**: `httpCustom` が省略された場合は存在しなければならず、そうでなければ禁止されています
        * **url**: 必須 (有効な URL である必要があります)
    * **httpCustom**: `http` が省略されている場合は存在し、そうでない場合は禁止されていなければなりません
        * **url**: 必須 (空でなければなりません)
        * **headers**: オプション (ただし、存在する場合はコンテンツが必要です。つまり `{}` は許可されていません)
        * **qs**: オプション (ただし、存在する場合はコンテンツが必要です。つまり `{}` は許可されていません)
        * **method**: オプション (存在する場合は有効な HTTP メソッドでなければなりません)
        * **payload**: オプション (空の文字列を使用できます)
    * **attrs**: オプション (ただし、存在する場合はリストでなければなりません。空リストも許可されます)
    * **metadata**: オプション (ただし、存在する場合はリストでなければなりません。空リストも許可されます)
    * **exceptAttrs**: オプションです (ただし、`attrs` も使用されている場合は存在できません。存在する場合は空でないリストでなければなりません)
    * **attrsFormat**: オプション (存在する場合は有効な `attrs` 形式のキーワードでなければなりません)
* **throttling**: オプション (整数でなければなりません)
* **expires**: オプション (日付または空の文字列 "" でなければなりません)
* **status**: オプション (有効なステータス・キーワードである必要があります)

[トップ](#top)

<a name="actiontype-metadata"></a>
## `actionType` メタデータ

NGSIv2 仕様のセクション "組み込みメタデータ" から `actionType` メタデータ関連 :

> その値はリクエストオペレーションの型によって異なります : 更新のための `update`, 作成のための `append`, 削除のための `delete`。その型は常に Text です。

現在の Orion の実装では、"update (更新)" と "append (追加)" がサポートされています。[この問題](https://github.com/telefonicaid/fiware-orion/issues/1494)が完了すると、"delete (削除)" のケースがサポートされます。

[トップ](#top)

<a name="noattrdetail-option"></a>
## `noAttrDetail` オプション

URI param `options` の値 `noAttrDetail` は、NGSIv2 型のブラウジング・クエリ (`GET /v2/types` および `GET /v2/types/<type>`) が、属性型の詳細を提供しないようにするために使用されます。使用すると、各属性名に関連付けられた `types` リストが `[]` に設定されます。

このオプションを使用すると、Orion はこれらのクエリをはるかに迅速に解決します。特に、それぞれが異なる型の多数の属性の場合は、これは、ユース・ケースに属性型の詳細が必要ない場合に非常に便利です。場合によっては、`noAttrDetails` オプションで30秒から 0.5 秒の節約が検出されました。

[トップ](#top)

<a name="notification-throttling"></a>
## 通知スロットリング

サブスクリプション・スロットリングに関する NGSIv2 仕様から :

> throttling : 2つの連続した通知の間に経過する必要のある秒単位の最小限の時間。オプションです。

Orion がこれを実装する方法は、保護期間のスロットル中に通知を破棄することです。したがって、あまりにも近づいてしまえば、通知が失われる可能性があります。ユース・ケースがこのように通知を失うことをサポートしていない場合は、スロットリングを使用しないでください。

さらに、Orion はローカルでスロットリングを実装します。multi-CB 構成では、最終通知の尺度が各 Orion ノードに対してローカルであることを考慮してください。各ノードは DB と定期的に同期してより新しい値を取得しますが ([ここ](../admin/perf_tuning.md#subscription-cache)ではこれ以上)、特定のノードに古い値があることがありますので、スロットリングは100％正確ではありません。

[トップ](#top)

<a name="ordering-between-different-attribute-value-types"></a>
## 異なる属性値型間の順序付け

NGISv2 仕様 "Ordering Results" セクションから :

> エンティティのリストを取得するオペレーションでは、順序付けの結果を得る際に条件として使用される属性またはプロパティを `orderBy` URI パラメータで指定できます

これは、各型が他の型に関してどのように順序づけられるかの実装アスペクトです。Orion の場合、基本的な実装(MongoDB)で使用されているものと同じ基準を使用します。詳細については、[次のリンク](https://docs.mongodb.com/manual/reference/method/cursor.sort/#ascending-descending-sort)を参照してください。

最低から最高まで :

1. Null
2. Number
3. String
4. Object
5. Array
6. Boolean

[Top](#top)

<a name="initial-notifications"></a>
## 初期通知

NGSIv2 仕様では、サブスクリプションの対象となるエンティティの更新に基づいて、特定のサブスクリプションに対応する通知をトリガするルールを "サブスクリプション" セクションで説明しています。そのような定期的な通知以外にも、Orion は、また、サブスクリプションの作成/更新時時に初期通知を送信することがあります。

初期通知は、新しい URI パラメータオプション `skipInitialNotification`
を使用して設定できます。
例えば、`POST /v2/subscriptions?options=skipInitialNotification` です。

[初期通知](initial_notification.md) について、ドキュメントで詳細を
確認してください。

[トップ](#top)

<a name="oneshot-subscriptions"></a>
## Oneshot サブスクリプション

Orionは、NGSIv2 仕様のサブスクリプション用に定義された `status` 値の他に、
`oneshot`を使用することもできます。
[Oneshot サブスクリプションのドキュメント](oneshot_subscription.md)で詳細を確認してください

[Top](#top)

<a name="registrations"></a>
## レジストレーション

Orion は、次の点を除いて、NGSIv2 仕様に記載されているレジストレーション管理を実装しています。

* `PATCH /v2/registration/<id>` は実装されていません。したがって、レジストレーションを直接更新することはできません。つまり、レジストレーションを削除して再作成する必要があります。[この issue](https://github.com/telefonicaid/fiware-orion/issues/3007)についてはこちらをご覧ください
* `idPattern` および `typePattern` は実装されていません。
* 唯一の有効な `supportedForwardingMode` は `all` です。他の値を使用しようとすると、501 Not Implemented エラー応答で終了します。[この issue](https://github.com/telefonicaid/fiware-orion/issues/3106) についてはこちらをご覧ください
* `dataProvided` 内での `expression` フィールドはサポートされていません。フィールドは単に無視されます。これについては [この issue](https://github.com/telefonicaid/fiware-orion/issues/3107) を見てください。
* `status` での `inactive` 値はサポートされていません。つまり、フィールドは正しく格納され/取得されますが、値が `inactive` の場合でもレジストレーションは常にアクティブです。これについては、[この issue](https://github.com/telefonicaid/fiware-orion/issues/3108) を見てください

NGSIv2 仕様によると :

> NGSIv2 サーバ実装は、コンテキスト情報ソースへのクエリまたは更新転送を実装することができます

Orion がこのような転送を実装する方法は次のとおりです。

Orion は、NGSIv2 仕様に含まれていない追加フィールド `legacyForwarding` を `provider` に実装しています。`legacyForwarding` の値が `true` の場合、そのレジストレーションに関連する転送リクエストに、NGSIv1 ベースのクエリ/更新が使用されます。NGSIv1 は推奨されていませんが、当面は、NGSIv2 ベースの転送が定義されていないため、([この issue](https://github.com/telefonicaid/fiware-orion/issues/3068) を参照)、唯一有効なオプションは常に `"legacyForwarding": true` を使用することです。そうでなければ、結果は、501 Not Implemented エラーのレスポンスになります。

[Top](#top)

<a name="keyvalues-not-supported-in-post-v2opnotify"></a>
## `POST /v2/op/notify` でサポートされない `keyValues`

現在の Orion の実装は、`POST /v2/op/notify` オペレーションの `keyValues` オプションをサポートしていません。それを使用しようとすると、400 Bad Request エラーが発生します。

[Top](#top)


<a name="deprecated-features"></a>
## 廃止予定の機能

安定版の NGSIv2 仕様の変更を最小限に抑えようとしていますが、最後にいくつかの変更が必要でした。したがって、現在の NGSIv2 stable 仕様には現れない機能が変更されていますが、Orion は後方互換性を維持するために([非推奨の機能](../deprecated.md)として)サポートしています。

特に、`options` のパラメータ内の `dateCreated` 及び `dateModified` の使用はまだサポートされています (安定した RC-2016.05 で導入され、RC-2016.10 で除去されました)。例えば `options=dateModified` です。ただし、代わりに `attrs` の使用することを強くお勧めします (すなわち `attrs=dateModified,*`)。

[トップ](#top)
