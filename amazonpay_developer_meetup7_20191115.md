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

アマゾンジャパン様のご依頼で、EC-CUBE向けAlexaプラグインを開発、無事リリースできました。

| リリース日 | 成果物 | 公開URL |
| -------- |-------|--------|
| 11/11 | EC-CUBEプラグイン | [Alexaカスタムスキル(4.0系)プラグイン](https://www.ec-cube.net/products/detail.php?product_id=1968) |
| 11/14 | Alexaスキル | [ミラショーン オンラインストア](https://www.amazon.co.jp/dp/B081GGCVDG/) |

先週の記憶が曖昧

## 本日お伝えしたいこと

- SDK は必要か？ 問題なし。今回の開発言語は PHP
- Alexaエンドポイントは json 入出力の Web API
- ダイアログモデルを活用すべし
- VUI設計が一番大事。技術的なハードルはそれはそれほど高くない

## SDK は必要か？
- 公式サポートSDKが存在するのは Node.js, Python or Java
- インテグレート先の EC-CUBE は PHP で実装。公式SDKは存在しない

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

https://github.com/maxbeckers/amazon-alexa-php
https://github.com/iplogic-support/amazon-alexa-php

## Alexa での AmazonPay 決済

https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#setup-1

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
					"sandboxCustomerEmailId": "takashi@crazyhacks.net"
				},
				"token": "UNIQUE-TOKEN"
			}
		]
	}
}
```

https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#%E5%BF%9C%E7%AD%94%E8%A6%81%E7%B4%A0

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

https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#charge-1

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

https://developer.amazon.com/ja/docs/amazon-pay-alexa/amazon-pay-apis-for-alexa.html#%E5%BF%9C%E7%AD%94%E8%A6%81%E7%B4%A0-1

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

あるインテントでダイアログ機能を使うには？
インテントに、少なくとも下記のどれか一つが定義されていること
- インテントの確認プロンプト
- スロットの確認プロンプト
- 必須のスロットとプロンプト
- スロット検証ルールとプロンプト

Dialog.Delegate
Dialog.ElicitSlot
Dialog.ConfirmSlot
Dialog.ConfirmIntent

## VUI設計が一番大事

- 絶対に理解しておきたい [音声デザインガイド](https://developer.amazon.com/ja/designing-for-voice/)
