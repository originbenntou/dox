# Cognito

## 目的

以下の認証フローの仕様をCognitoを用いて実現するにはどうすればいいか説明する

### 目指してる認証シーケンス

https://github.com/aws-samples/cognito-custom-authentication/blob/main/docs/auth_sequence.png

### 認証の仕様

- EmailとPassword入力
- 認証開始
	- EmailまたはPasswordが違う場合エラーレスポンス
	- Email宛に確認コードを送付
- 確認コード入力
- 認証完了

## Cognitoとは

ウェブおよびモバイルアプリの認証、承認、およびユーザー管理機能を提供します。
ユーザーは、ユーザー名とパスワードを使用して直接サインインするか、Facebook、Amazon、Google、Apple などのサードパーティーを通じてサインインできます。
（IDフェデレーション）

## ユーザープールとIDプール

- ユーザープール
	- ユーザーID管理とログイン情報の管理（IDトークン等の発行）を行い、今アクセスしてきているのは○○というユーザーであるということを管理する、いわゆる認証。
- IDプール
	- ユーザーに応じたAWSの一時クレデンシャルキーの発行を行い、どのユーザーがAWSリソースのどこにアクセスできるのか管理する、いわゆる認可。

## アプリケーションクライアント

- パブリック
	- ネイティブアプリケーション、ブラウザアプリケーション、またはモバイルデバイスアプリケーション。Cognito API リクエストは、クライアントのシークレットで信頼されていないユーザーシステムから実行されます。
- シークレット
	- クライアントのシークレットを安全に保存できるサーバー側のアプリケーション。Cognito API リクエストは中央サーバーから実行されます。

### 認証と認可

- 認可
	- 認証は 「あなたは誰ですか？」 を確認すること
	- AuthN
	- パスポートは本人確認に用いられつつ（認証）、これにより渡航する権利を得られる（認可）
- 認証
	- 認可は 「あなたには、リソースにアクセスする権限がありますか？」 を確認すること
	- AuthZ
	- その人は、家の鍵を持っていたので家に入ることができた（認可）
		- 鍵を持っていた人がその家の住人であるとは限らない

### 認証の3要素

- What you are：相手自身の特徴を確認
	- 例）指紋認証や顔認証
- What you have：相手が持っているものを確認
	- 例）免許証やパスポート
- What you know：相手が知っていることを確認
	- 例）email と password

## ユーザープールが提供するユーザー操作

- サインアップ
- Emailまたは電話番号による確認（オンオフ切替可）
- サインイン
- パスワード忘れ対応
- パスワード変更
- ユーザー属性変更
- サインアウト

### ユーザー操作・API対応表

| 機能 | CLI | Amplify |
| ---- | ---- | ---- |
| サインアップ | [sign-up](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/sign-up.html) | [signUp](https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js/#sign-up) |
| Emailまたは電話番号による確認 | [confirm-sign-up](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/confirm-sign-up.html) | [confirmSignUp](confirmSignUp) |
| Emailまたは電話番号による確認（再送） | [resend-confirmation-code](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/resend-confirmation-code.html) | [resendConfirmationCode](https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js/#re-send-sign-up-confirmation-code) |
| サインイン | [initiate-auth](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/initiate-auth.html) | [signIn](https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js/#sign-in) |
| パスワード忘れ対応（コード送信） | [forgot-password](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/forgot-password.html) | [forgotPassword](https://docs.amplify.aws/lib/auth/manageusers/q/platform/js/#forgot-password) |
| パスワード忘れ対応（コード入力） | [confirm-forgot-password](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/confirm-forgot-password.html) | [forgotPasswordSubmit](https://docs.amplify.aws/lib/auth/manageusers/q/platform/js/#forgot-password) |
| パスワード変更 | [change-password](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/change-password.html) | [changePassword](https://docs.amplify.aws/lib/auth/manageusers/q/platform/js/#change-password) |
| ユーザー属性変更 | - | - |
| サインアウト | [global-sign-out](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/global-sign-out.html) | [signOut](https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js/#sign-out) |

ユーザー属性まわりのAPIは省略、使ってない
強いて言うならログイン確認に Auth.currentAuthenticatedUser を使っている（が、API施行回数が増えるので推奨というわけでもない）

## Lambdaトリガーによるカスタマイズ

- カスタム認証フロー
	- 認証チャレンジの定義
	- 認証チャレンジの作成
	- 認証チャレンジレスポンスの確認
- サインイン
	- サインイン前
	- サインイン後
	- トークン生成前
- サインアップ
	- サインアップ前
	- 確認後
	- ユーザー移行
- メッセージ
	- カスタムメッセージ

## カスタム認証

### 認証フロー

- USER_PASSWORD_AUTH（ADMIN_USER_PASSWORD_AUTH）
	- SRPプロトコルを使用せず、パスワード平文を送って認証する。非推奨。
- USER_SRP_AUTH
	- SRPプロトコルに基づいた方法でチャンレンジレスポンス認証する。元のパスワードを通信路に流さず、サーバー側でパスワードが不明な状態になるため安全。
- CUSTOM_AUTH
	- 認証時にLambda関数がトリガーされ、追加の認証など認証のカスマイズができる。
- REFRESH_TOKEN_AUTH
	- 更新トークンから新しいトークンをもらう。

### SRPプロトコル

（いい感じに要約）

https://medium.com/swlh/what-is-secure-remote-password-srp-protocol-and-how-to-use-it-70e415b94a76

- サーバーはパスワードと同等の情報 (ハッシュ) を保存する必要がない
- ネットワーク経由でパスワードを送信する必要なく、盗聴者や中間者は攻撃を実行するための意味のある情報を取得できない

__YOU CAN'T LEAK PASSWORD IF YOU DON'T STORE PASSOWRD__

### パスワードハッシュ

ハッシュ化されたパスワードは、シンプルさとセキュリティの間の適切なトレードオフを提供しますが、非常に機密性の高い情報またはシステムの場合、独自の欠点があります。

- ハッシュ化されたパスワードを安全に処理および保存するには、信頼できるサーバーが必要でした (パスワードをログに記録しないことを約束します 😀)
- 大規模なパスワード辞書と侵害されたデータベースを持つ攻撃者は、ユーザーのパスワードを特定できます
- 攻撃者は、クライアントとサーバー間の通信を傍受 (MITM攻撃) し、パスワードを取得することができます。

### チャレンジレスポンス

「チャレンジ」と呼ばれる一度しか使わない乱数とパスワードをハッシュ関数で計算します。その計算結果が「レスポンス」（メッセージダイジェストとも呼ぶ）となり、このレスポンスをネットワーク経由でサーバに渡して照合して、認証結果を出します。

ハッシュ化された値は不可逆式なので、盗聴されても元のパスワードに戻すことができないので、セキュリティが高くなります。

https://itmanabi.com/challenge-response/

- 利用者がサーバにアクセスを要求する
- サーバ側でチャレンジを作成し、利用者側に送る
- チャレンジを受け取った利用者側は、入力したパスワードとチャレンジを使ってレスポンスを生成し、サーバ側へ「レスポンスA」を送る
- サーバ側でもデータベースに格納されているパスワードとチャレンジを使って「レスポンスB」を生成する
- サーバ側で「レスポンスA」と「レスポンスB」を照合する
- 認証結果を利用者側に返す
- 認証結果がOKだったら、サービスの利用が開始される

※SRPは既存のチャレンジレスポンス技術に対してセキュリティとデプロイメントの両方が持つ利点を提供し、セキュアなパスワード認証が必要な場合に理想的な完全互換品となります。

## USER_SRP_AUTH

https://d1.awsstatic.com/webinars/jp/pdf/services/20200630_AWS_BlackBelt_Amazon%20Cognito.pdf#page=34

## CUSTOM_AUTH

（いい感じの要約）

https://d1.awsstatic.com/webinars/jp/pdf/services/20200630_AWS_BlackBelt_Amazon%20Cognito.pdf#page=38

- 実現できる内容の例
	- Cognito とは別方式の MFA 認証を行う
	- 月１回は 秘密の質問を追加で確認する
	- 人からのアクセスである事を検証するため CAPTCHA を確認する
	- パスワードレス認証を行う

### ユーザー操作・API対応表

| 機能 | CLI | Amplify |
| ---- | ---- | ---- |
| サインイン（チャレンジレスポンス） | [respond-to-auth-challenge](https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/respond-to-auth-challenge.html) | [sendCustomChallengeAnswer](sendCustomChallengeAnswer) |

チャレンジレスポンス認証のセッションは**秒 その間何度でもsendAnswerができる

## トークン

- IDトークン
	- 認証されたユーザーの属性情報が含まれる。
	- APIGatewayのCognitoオーソライザーでトークン検証にも使われる。
- アクセストークン
	- Cognitoユーザープール（自身のユーザー属性）にアクセスする際に検証される。
- リフレッシュトークン
	- IDトークン、またはアクセストークンの有効期限が切れた際に、リフレッシュトークンをCognitoに渡すことで新しいIDトークンとアクセストークンを返却してくれる。
