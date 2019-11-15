# Alexaスキル開発からリリースまでに経験したアレコレ

## 自己紹介

- 名乗るほどの者ではありませんが
- 工藤 高志
- 1994年4月よりソフトウェア開発業務
- 2009年9月 [アイピーロジック株式会社](https://www.iplogic.co.jp/) 設立メンバー。現職
- 主にEC-CUBE向け大量メール配信システム [PostCarrier for EC-CUBE](https://www.postcarrier.jp/) の開発保守
- [SUPER CLASSIC 様](https://superclassic.jp) Amazon Pay インテグレーションを担当

### 趣味など
- 自転車カレー部 クロスバイクに乗ってカレーを食べます
- カレーは基本毎日食べます。興味ありましたらtwitterを #自転車カレー部 で検索
- 数年おきに痩せたり太ったり 半年で20kg太り、今年3月から11kg痩せた
- 筋トレはさぼりぎみ BP85kg, DL100kg, SQ100kg まだザコ

## 経緯

- SUPER CLASSIC 様への Amazon Pay 導入から、弊社で EC-CUBE Amazon Pay プラグインを開発
- EC-CUBE向け Alexaプラグイン を開発、無事リリース

| リリース日 | 成果物 | 公開URL |
| -------- |-------|--------|
| 11/11 | EC-CUBEプラグイン | [Alexaカスタムスキル(4.0系)プラグイン](https://www.ec-cube.net/products/detail.php?product_id=1968) |
| 11/14 | Alexaスキル | [ミラショーン オンラインストア](https://www.amazon.co.jp/dp/B081GGCVDG/) |

- SMAPI を利用したスキル登録・審査申請機能
- Buyer ID を利用したECサイト会員との紐付け。ECサイトで AmazonPay 購入経験または Login With Amazon ボタンでログイン済ならば、Buyer ID で紐付けられる。
- Buyer ID API にサンドボックス環境は存在しない。セラーセントラルの本番環境にスキルIDを登録する必要がある。

スキル審査は、合格まで3回が目安

| 審査結果 | 審査回数 | 合否 |
| -------- |--------|----|
| 10/25 | 1 | × |
| 10/31 | 2 | × |
| 11/5 | 3 | ○ |

- 11/11 にミラショーン様スキル審査を申請
- *先週の記憶が曖昧*

## 本日お伝えしたいこと

- SDK は必要か？ 問題なし。今回の開発言語は PHP
- Alexaエンドポイントは json 入出力の Web API
- ダイアログモデルを活用すべし
- VUI設計が一番大事。技術的なハードルはそれはそれほど高くない

## SDK は必要か？

- 公式サポートSDKが存在するのは Node.js, Python or Java
- インテグレート先の EC-CUBE は PHP で実装。公式SDKは存在しない
- Node.js のサンプルや [応答の形式](https://developer.amazon.com/ja/docs/custom-skills/request-and-response-json-reference.html#response-format) などを見て試してみた

```php
<?php

// 応答の形式
// https://developer.amazon.com/ja/docs/custom-skills/request-and-response-json-reference.html#response-format

$response = [
    "version" => "1.0",
    "response" => [
        "outputSpeech" => [
            "type" => "PlainText",
            "text" => "PHPで実装したエンドポイントです"
        ],
        "shouldEndSession" => true,
    ],
];

header('Content-Type: application/json; charset=utf-8');
echo json_encode($response);

```

- SDK の主な機能は、入力jsonのパース、出力jsonの組み立て、Alexaリクエストの検証
- [Alexaから送信されたリクエストを手動で検証する](https://developer.amazon.com/ja/docs/custom-skills/host-a-custom-skill-as-a-web-service.html#manually-verify-request-sent-by-alexa)
- PHPで実装された非公式SDK https://github.com/maxbeckers/amazon-alexa-php
- 弊社でAmazon Pay対応を追加した物 https://github.com/iplogic-support/amazon-alexa-php

```php
    /**
     * Alexa端末からのリクエストを捌くエンドポイント
     *
     * @Route("/alexa/endpoint", name="endpoint", methods={"POST","GET"})
     *
     * @param HttpRequest $httpRequest
     */
    public function endpoint(HttpRequest $httpRequest)
    {
        try {
            $alexaRequest = Request::fromAmazonRequest($httpRequest->getContent(), $_SERVER['HTTP_SIGNATURECERTCHAINURL'], $_SERVER['HTTP_SIGNATURE']);
            // Request validation
            $validator = new RequestValidator();
            $validator->validate($alexaRequest);
        } catch (\Throwable $e) {
            // BadRequest.
            return new Response(Response::$statusTexts[400], Response::HTTP_BAD_REQUEST, ['Content-Type' => 'text/plain']);
        }

        // add handlers to registry
        $requestHandlerRegistry = new RequestHandlerRegistry([
            $this->launchRequestHandler,
            $this->pointRequestHandler,
            // handlers...
            $this->sessionEndedRequestHandler,
            $this->missingRequestHandler,
        ]);

        // handle request
        $requestHandler = $requestHandlerRegistry->getSupportingHandler($alexaRequest);
        $alexaResponse  = $requestHandler->handleRequest($alexaRequest);

        // return JSON RESPONSE
        $jsonResponse = new JsonResponse($alexaResponse);
        return $jsonResponse->send();
    }
```

## Alexaエンドポイントは json 入出力の Web API

- Alexa AmazonPay 決済の json の実際
- Setup/Charge API
- ペアで利用する。Setupで準備、Chargeでオーソリ/キャプチャを行う
- [セットアップ](https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#setup-1)
- *token* にセッションを特定する任意の値を設定すると、Connections.Response で受け取る事ができる

```json
{
	"version": "1.0",
	"response": {
		"outputSpeech": {
			"type": "PlainText",
			"text": "セットアップを実行します"
		},
		"shouldEndSession": true,
		"directives": [
			{
				"type": "Connections.SendRequest",
				"name": "Setup",
				"payload": {
					"@type": "SetupAmazonPayRequest",
					"@version": "2",
					"sellerId": "YOUR-SELLER-ID",
					"countryOfEstablishment": "JP",
					"ledgerCurrency": "JPY",
					"checkoutLanguage": "ja_JP",
					"billingAgreementAttributes": {
						"@type": "BillingAgreementAttributes",
						"@version": "2",
						"sellerNote": "PHPからのセットアップ疎通テスト",
						"sellerBillingAgreementAttributes": {
							"@type": "SellerBillingAgreementAttributes",
							"@version": "2",
							"storeName": "キューブショップ"
						}
					},
					"needAmazonShippingAddress": false,
					"sandboxMode": true,
					"sandboxCustomerEmailId": "takashi@iplogic.co.jp"
				},
				"token": "UNIQUE-TOKEN"
			}
		]
	}
}
```

- [セットアップの応答要素](https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#%E5%BF%9C%E7%AD%94%E8%A6%81%E7%B4%A0)

```json
{
  "version": "1.0",
  ...省略...
  "request": {
    "type": "Connections.Response",
    "name": "Setup",
    "requestId": "amzn1.echo-api.request.XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "timestamp": "2019-11-14T19:02:06Z",
    "locale": "ja-JP",
    "status": {
      "code": "200",
      "message": "OK"
    },
    "payload": {
      "billingAgreementDetails": {
        "releaseEnvironment": "SANDBOX",
        "billingAgreementStatus": "OPEN",
        "creationTimestamp": "2019-08-09T06:05:54.745Z",
        "billingAgreementId": "C03-XXXXXXX-YYYYYYY",
        "checkoutLanguage": "ja_JP"
      }
    },
    "token": "UNIQUE-TOKEN"
  }
}
```

- [チャージ](https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#charge-1)

```json
{
    "version": "1.0",
    "response": {
        "outputSpeech": {
            "type": "PlainText",
            "text": "サンドボックスでチャージを実行します"
        },
        "shouldEndSession": true,
        "directives": [
            {
                "type": "Connections.SendRequest",
                "name": "Charge",
                "payload": {
                    "@type": "ChargeAmazonPayRequest",
                    "@version": "2",
                    "sellerId": "YOUR-SELLER-ID",
                    "billingAgreementId": "C03-XXXXXXX-YYYYYYY",
                    "paymentAction": "AuthorizeAndCapture",
                    "authorizeAttributes": {
                        "@type": "AuthorizeAttributes",
                        "@version": "2",
                        "authorizationReferenceId": "test-5dcda4aecaf1d",
                        "authorizationAmount": {
                            "@type": "Price",
                            "@version": "2",
                            "amount": "300",
                            "currencyCode": "JPY"
                        },
                        "transactionTimeout": 0,
                        "sellerAuthorizationNote": ""
                    },
                    "sellerOrderAttributes": {
                        "@type": "SellerOrderAttributes",
                        "@version": "2",
                        "sellerOrderId": "1",
                        "storeName": "TestStoreName",
                        "customInformation": "テスト用カスタム情報",
                        "sellerNote": "テスト用の販売事業者のメモ"
                    }
                },
                "token": "UNIQUE-TOKEN"
            }
        ]
    }
}
```

- [チャージの応答要素](https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#%E5%BF%9C%E7%AD%94%E8%A6%81%E7%B4%A0-1)

```json
{
  "version": "1.0",
  ...省略...
  "request": {
    "type": "Connections.Response",
    "name": "Charge",
    "requestId": "amzn1.echo-api.request.XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "timestamp": "2019-11-14T19:02:09Z",
    "locale": "ja-JP",
    "status": {
      "code": "200",
      "message": "OK"
    },
    "payload": {
      "authorizationDetails": {
        "authorizationAmount": {
          "amount": "300.00",
          "currencyCode": "JPY"
        },
        "capturedAmount": {
          "amount": "300.00",
          "currencyCode": "JPY"
        },
        "softDescriptor": "AMZ*IPLOGIC　デモサイト",
        "expirationTimestamp": "2019-12-14T19:02:08.347Z",
        "idList": [
         "S03-XXXXXXX-YYYYYYY-CZZZZZZ"
        ],
        "softDecline": false,
        "authorizationStatus": {
          "lastUpdateTimestamp": "2019-11-14T19:02:08.347Z",
          "reasonCode": "MaxCapturesProcessed",
          "state": "Closed"
        },
        "authorizationFee": {
          "amount": "0.00",
          "currencyCode": "JPY"
        },
        "captureNow": true,
        "sellerAuthorizationNote": "",
        "creationTimestamp": "2019-11-14T19:02:08.347Z",
        "amazonAuthorizationId": "S03-XXXXXXX-YYYYYYY-AZZZZZZ",
        "authorizationReferenceId": "test-5dcda4aecaf1d"
      },
      "amazonOrderReferenceId": "S03-XXXXXXX-YYYYYYY"
    },
    "token": "UNIQUE-TOKEN"
  }
}
```

## ダイアログモデルを活用すべし

スキルを実装する前に理解しておきたいドキュメント
- [ダイアログモデルを使用してあいまい応答を管理する](https://developer.amazon.com/ja/docs/custom-skills/use-a-dialog-model-to-manage-ambiguous-responses.html)
- [Dialogインターフェースのリファレンス](https://developer.amazon.com/ja/docs/custom-skills/dialog-interface-reference.html)

### あるインテントでダイアログ機能を使うには？

インテントに、少なくとも下記のどれか一つが定義されていること
- インテントの確認プロンプト
- スロットの確認プロンプト
- 必須のスロットとプロンプト
- スロット検証ルールとプロンプト

### ダイアログインターフェイス

- Dialog.ConfirmIntent 全てのスロット値を収集した状態での確認
- Dialog.ConfirmSlot スロット値を確認する
- Dialog.ElicitSlot スロット値を収集する発話。旅予約なら、「目的地は？」「いつ出発しますか？」
- Dialog.Delegate オートデリゲート(対話の制御を完全にAlexaに任せる)

### tips
- はい/いいえ の二択の応答を期待する場面では、Dialog.ConfirmIntent, Dialog.ConfirmSlot を使用するようにした
- スロットのないインテントでダイアログを使う場合は、対話モデルでダミーのインテント確認プロンプトを定義すれば良い
- Dialog.Delegate は原則利用しない。エンドポイント側で会話の流れを制御できなくなるため
- Dialog.Delegate 以外のダイアログインターフェイスでも他のインテントを渡すことができる
- 対話モデルから余計なインテントは削除しておく
- 対話モデルに登録されているインテントは、何の脈絡もなく起動されうる。どのタイミングで起動されても落ちないように。

## VUI設計が一番大事

- 絶対に理解しておきたい [音声デザインガイド](https://developer.amazon.com/ja/designing-for-voice/)

## 全てに感謝を
