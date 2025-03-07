---
title: "PKI入門 ~hashicorp vault PKIでのルート証明書のローテーション~"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hashicorpVault", "pki"]
published: true
---
## 背景
クロスサイン証明書を使ったルート証明書のローテーションをhashicorp vaultでやる。
クロスサイン証明書を使うと新旧どちらか一方のルート証明書を信頼していれば認証を通すことが出来る。
[PKI入門 ~証明書のローテーション~](https://zenn.dev/aa54/articles/pki-basic-rotation)を参照。

hashicorp vaultによるクロスサイン証明書を使ったローテーション方法は以下が参考になるが、個人的に詰まったところがあったため解説を追加したいと思う。
https://developer.hashicorp.com/vault/tutorials/pki/pki-engine
https://zenn.dev/i10chu/articles/f27f0a2129707d

## ルート証明書のローテーション
※ローテーションに特化して書いているため、PKIの構築は先に挙げたサイト等を参考にしてください。

#### 1. 証明書のTTL決定
初めに証明書のTTLを決定しておく。ここでは以下の2つの前提でTTLを決定した。
- PKIの勉強のため、頻繁にローテーションするTTLに決定する。
- 常に2つのルート証明書、中間証明書で運用することは避ける。

設定したTTLは以下。
- サーバ証明書: 1440h  (約2か月)
- 中間証明書:   5760h  (約8か月)
- ルート証明書: 17280h (約24か月)

:::message
詰まったポイント
証明書作成時にTTLを指定しても、指定より短いTTLになることがあります。
vaultはエンジン単位、role単位のTTL設定が存在し、これが優先されている可能性あります。設定を確認してみましょう。
今回、エンジンには以下のTTLを設定することにします。
- ルート認証局のPKIエンジン: 43200h (約5年)
- 中間認証局のPKIエンジン: 17280h (約2年)
:::

事前にPKIエンジンのTTLを設定しておく。
```bash
## ルートCAのPKIエンジンのmax ttlを設定
$ vault secrets tune -max-lease-ttl="43200h" pki_root/

## 中間CAのPKIエンジンのmax ttlを設定
$ vault secrets tune -max-lease-ttl="5760h" pki_int/
```

#### 2. 新しいルート証明書を作成
vault上に構築したルートCA(pki_root/)上に新しいルート証明書を作成する。
```bash
## 新しいルート証明書(root-2025)を作成する。
$ vault write pki_root/root/rotate/internal \
common_name="uws.lan" issuer_name="root-2025" ttl="17280h"
```

ルートCAの発行者を確認するとkeyが増えていることが分かる。
```bash
$ vault list pki_root/issuers
Keys
----
3d6c537f-d490-8c6d-3138-0c527c2f9dda    ## -> 新たに作成したルート証明書
fbe1e696-4eca-f128-f9ad-10a1eb14fee8    ## -> 以前のルート証明書
```
証明書の主体者(subject)と発行者(issuer)を確認。ルート証明書なのでsubjectとissuerが同じ。
```bash
$ vault read -format=json pki_root/issuer/root-2025 | jq -r '.data.certificate' \
  | openssl x509 -noout -text | grep -A 3 "X509v3 Subject Key Identifier:"
      X509v3 Subject Key Identifier:
          B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> 主体者の識別子
      X509v3 Authority Key Identifier:
          B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> 発行者の識別子
```
旧ルート証明書のsubjectとissuerを確認してみると、新ルート証明書とは異なることが分かる。
```bash
$ vault read -field=certificate pki_root/issuer/fbe1e696-4eca-f128-f9ad-10a1eb14fee8 \
 | openssl x509 -noout -text | grep -A 3 "X509v3 Subject Key Identifier:"
      X509v3 Subject Key Identifier:
          81:30:FC:E1:5D:EF:2F:8E:2D:9D:B0:54:61:69:E0:C5:E6:2D:74:F1
      X509v3 Authority Key Identifier:
          81:30:FC:E1:5D:EF:2F:8E:2D:9D:B0:54:61:69:E0:C5:E6:2D:74:F1
```

#### 3. クロスサイン証明書を作成
中間CA(pki_int/)で証明書署名要求(CSR)を発行する。
CSRには現在の中間証明書と同じ公開鍵情報を記載する様にする。これがクロスサイン証明書の元になる。
```bash
## CSR作成
## key_refに現在の中間証明書のkey IDを指定する。
$ vault write -format=json pki_int/intermediate/cross-sign \
 common_name="uws.lan Intermediate Authority" \
 key_ref="$(vault read pki_int/issuer/$(vault read -field=default pki_int/config/issuers) \
 | grep -i key_id | awk '{print $2}')" \
 | jq -r '.data.csr' \
 | tee cross-signed-intermediate.csr
```
ルートCAでクロスサイン証明書を作成する。
発行者(issuer)は新しいルート証明書と同じroot-2025を指定する。
```bash
## 新しい中間証明書(クロスサイン証明書)を発行。
$ vault write -format=json pki_root/issuer/root-2025/sign-intermediate \
 common_name="uws.lan Intermediate Authority" csr=@cross-signed-intermediate.csr \
 ttl="5760h" \
 | jq -r '.data.certificate' | tee cross-signed-intermediate.crt
```
証明書のissuerを確認すると、発行者がルート証明書の主体者と一致していることが分かる。
```bash
$ cat cross-signed-intermediate.crt | openssl x509 -noout -text \
  | grep -A 3 "X509v3 Subject Key Identifier:"
      X509v3 Subject Key Identifier:
          E7:92:DE:02:36:B9:8D:E7:39:29:EC:4D:89:12:90:31:34:BC:5D:74
      X509v3 Authority Key Identifier:
          B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> ルート証明書のsubjectと一致
```
クロスサイン証明書を中間CAにインポートする。
```bash
$ vault write pki_int/intermediate/set-signed certificate=@cross-signed-intermediate.crt
Key                 Value
---                 -----
existing_issuers    <nil>
existing_keys       <nil>
imported_issuers    [bfd69160-0473-9ebb-0025-3aad4e5be9ed]
imported_keys       <nil>
mapping             map[bfd69160-0473-9ebb-0025-3aad4e5be9ed:d8f87b5b-fb76-7873-e2db-db07983c614b]
```
中間CAに新しいissuerが出来たことが確認できる。
```bash
$ vault list pki_int/issuers
Keys
----
bfd69160-0473-9ebb-0025-3aad4e5be9ed  ## -> 新しい中間証明書(クロスサイン証明書)
f67ca50d-0ebc-9e78-6487-ab7842a3429b
```
新しいissuerに名前をつけておく。(xc-uws-dot-lan-intermediateとした。)
```bash
$ vault write pki_int/issuer/bfd69160-0473-9ebb-0025-3aad4e5be9ed \
 issuer_name=xc-uws-dot-lan-intermediate
Key                               Value
---                               -----
~~略~~
crl_distribution_points           []
issuer_id                         bfd69160-0473-9ebb-0025-3aad4e5be9ed
issuer_name                       xc-uws-dot-lan-intermediate
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
~~略~~
```
:::message
詰まったポイント
[参考サイト](https://developer.hashicorp.com/vault/tutorials/pki/pki-engine#step-9-create-a-cross-signed-intermediate)では、オプション扱いになっているが、次の手順はやっておいた方が良い。
これをやっておかないと、サーバ証明書作成時に返される標準出力に、旧中間証明書か新中間証明書のどちらか片方しか出力されない。
:::
```bash
## 現在の中間証明書に対して新中間証明書をチェーンする。
$ vault patch pki_int/issuer/"$(v read -field=default pki_int/config/issuers)" \
manual_chain=self,xc-uws-dot-lan-intermediate
```
:::details example stdout
Key                               Value
---                               -----
ca_chain                          [-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
 -----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
]
certificate                       -----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
crl_distribution_points           []
issuer_id                         f67ca50d-0ebc-9e78-6487-ab7842a3429b
issuer_name                       int-20250222
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
leaf_not_after_behavior           err
manual_chain                      [f67ca50d-0ebc-9e78-6487-ab7842a3429b bfd69160-0473-9ebb-0025-3aad4e5be9ed]
ocsp_servers                      []
revocation_signature_algorithm    n/a
revoked                           false
usage                             crl-signing,issuing-certificates,ocsp-signing,read-only
:::
```bash
## 新中間証明書に対して旧中間証明書をチェーンする。
$ vault patch pki_int/issuer/xc-uws-dot-lan-intermediate \
 manual_chain=self,"$(vault read -field=default pki_int/config/issuers)"
```
:::details example stdout
Key                               Value
---                               -----
ca_chain                          [-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
 -----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
]
certificate                       -----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
crl_distribution_points           []
issuer_id                         bfd69160-0473-9ebb-0025-3aad4e5be9ed
issuer_name                       xc-uws-dot-lan-intermediate
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
leaf_not_after_behavior           err
manual_chain                      [bfd69160-0473-9ebb-0025-3aad4e5be9ed f67ca50d-0ebc-9e78-6487-ab7842a3429b]
ocsp_servers                      []
revocation_signature_algorithm    n/a
revoked                           false
usage                             crl-signing,issuing-certificates,ocsp-signing,read-only
:::

試しにサーバ証明書を発行してみると、標準出力に返されるca_chainに新旧中間証明書がエントリされていることが分かる。
```bash
$ vault write -format=json pki_int/issue/uws-dot-lan \
  common_name="test-rhel9.uws.lan" ttl="72h" > test-rhel9.2.json
```
:::details example stdout
```bash
$ cat test-rhel9.2.json
{
  "request_id": "b2c80dcb-f098-5fec-18dd-3a9fac2a5506",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "ca_chain": [
      ## 新・旧中間証明書でチェーンされている
      "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
      "-----BEGIN CERTIFICATE-----\nMIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----"
    ],
    "certificate": "-----BEGIN CERTIFICATE-----\nMIID5zCCAs+gAwIBAgIUW8gnSOG6YBci61d4xB/DTZ1wmAEwDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
    "expiration": 1741152305,
    "issuing_ca": "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
    "private_key": "-----BEGIN RSA PRIVATE KEY-----\n~~略~~\n-----END RSA PRIVATE KEY-----",
    "private_key_type": "rsa",
    "serial_number": "5b:c8:27:48:e1:ba:60:17:22:eb:57:78:c4:1f:c3:4d:9d:70:98:01"
  },
  "warnings": null,
  "mount_type": "pki"
}
```
:::
#### 4. ルート認証局のデフォルトissuerを変更
中間証明書を発行するときに新しいissuer(root-2025)で発行する様にルートCAの設定を変更する。
```bash
$ vault write pki_root/root/replace default=root-2025
Key                              Value
---                              -----
default                          3d6c537f-d490-8c6d-3138-0c527c2f9dda
default_follows_latest_issuer    false
```

##### 5. ルートCAの旧issuerの権限変更
ルートCAの旧issuerを使って証明書を発行できないように権限を変更する。
```bash
## 現在の利用権限を確認
$ vault read pki_root/issuer/fbe1e696-4eca-f128-f9ad-10a1eb14fee8 | tail -1
usage   crl-signing,issuing-certificates,ocsp-signing,read-only

## 利用権限の変更
$ vault write pki_root/issuer/uws-admin issuer_name="uws-admin" usage=read-only,crl-signing | tail -1
usage   crl-signing,read-only
```

#### 6. 中間CAのデフォルトissuerを変更
リーフ証明書※を発行するときに新しいissuer(xc-uws-dot-lan-intermediate)で発行する様に中間CAの設定を変更する。
※サーバ証明書やクライアント証明書のこと
```bash
## 現在のデフォルトを確認
$ vault read pki_int/issuer/default | tail -11
crl_distribution_points           []
issuer_id                         f67ca50d-0ebc-9e78-6487-ab7842a3429b  ## -> default issuer id
issuer_name                       int-20250222
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
~~略~~

## issuerを確認
$ vault list pki_int/issuers
Keys
----
bfd69160-0473-9ebb-0025-3aad4e5be9ed  ## -> クロスサイン証明書に紐づくissuer id
f67ca50d-0ebc-9e78-6487-ab7842a3429b  ## -> 現在のdefault issuer id

## xc-uws-dot-lan-intermediateをdefault issuerに設定
$ vault write pki_int/root/replace default=xc-uws-dot-lan-intermediate
Key                              Value
---                              -----
default                          bfd69160-0473-9ebb-0025-3aad4e5be9ed
default_follows_latest_issuer    false
```

#### 7.旧中間証明書の利用権限変更
中間CAの旧issuerを使って証明書を発行できないように権限を変更する。
ここまででルート証明書のローテーションは完了。
```bash
$ vualt read pki_int/issuer/int-20250222 | tail -1
usage   crl-signing,issuing-certificates,ocsp-signing,read-only

$ vault write pki_int/issuer/int-20250222 issuer_name=int-20250222 usage=read-only,crl-signing | tail -1
usage   crl-signing,read-only
```

## 証明書チェーンの設定方法
:::message
詰まったポイント
ルート証明書をローテーションしたら、旧ルート証明書の有効期限が到来するまでは証明書チェーンに新旧中間証明書を設定する必要があります。設定しないとクロスサイン証明書の効果がないです。
:::

証明書チェーンは一つのファイル内にサーバ証明書 -> 旧中間証明書 -> クロスサイン証明書と記述すれば良い。
サーバ証明書は一番最初に記述する(必須)。旧中間とクロスサインはどちらが先でも問題ない。
```bash
$ vault write -format=json pki_int/issue/uws-dot-lan \
  common_name="test-rhel9.uws.lan" ttl="72h" > test-rhel9.2.json
```
:::details cat test-rhel9.2.json
```json
{
  "request_id": "b2c80dcb-f098-5fec-18dd-3a9fac2a5506",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "ca_chain": [
      ## 新・旧中間証明書でチェーンされている
      "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
      "-----BEGIN CERTIFICATE-----\nMIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----"
    ],
    "certificate": "-----BEGIN CERTIFICATE-----\nMIID5zCCAs+gAwIBAgIUW8gnSOG6YBci61d4xB/DTZ1wmAEwDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
    "expiration": 1741152305,
    "issuing_ca": "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\n~~略~~\n-----END CERTIFICATE-----",
    "private_key": "-----BEGIN RSA PRIVATE KEY-----\n~~略~~\n-----END RSA PRIVATE KEY-----",
    "private_key_type": "rsa",
    "serial_number": "5b:c8:27:48:e1:ba:60:17:22:eb:57:78:c4:1f:c3:4d:9d:70:98:01"
  },
  "warnings": null,
  "mount_type": "pki"
}
```
:::
```bash
## リーフ証明書をファイルにリダイレクト
$ cat test-rhel9.2.json | jq -r '.data.certificate' > cert.chain.crt
## 旧中間証明書をファイルに追記書き
$ cat test-rhel9.2.json | jq -r '.data.ca_chain[0]' >> cert.chain.crt
## クロスサイン証明書をファイルに追記書き
$ cat test-rhel9.2.json | jq -r '.data.ca_chain[1]' >> cert.chain.crt
```
こうなっていればOK.
:::details cat cert.chain.crt
```bash
## リーフ証明書
-----BEGIN CERTIFICATE-----
MIID5zCCAs+gAwIBAgIUW8gnSOG6YBci61d4xB/DTZ1wmAEwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
## 中間証明書(現行)
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
## 中間証明書(新(クロス))
-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
```
:::

## 検証
[PKI入門 ~証明書のローテーション~](https://zenn.dev/aa54/articles/pki-basic-rotation)で紹介した通りに、新旧ルート証明書のどちらかがあれば認証が通るのか検証してみる。

#### 新旧ルート証明書の内容
旧ルート証明書のシリアル、subject、issuerを確認。
```bash
$ vault read -field=certificate pki_root/issuer/fbe1e696-4eca-f128-f9ad-10a1eb14fee8 \
  | openssl x509 -noout -serial -subject -issuer
serial=44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5
subject=CN=uws.lan
issuer=CN=uws.lan
```
新ルート証明書のシリアル、subject、issuerを確認。
```bash
$ vault read -field=certificate pki_root/issuer/root-2025 \
  | openssl x509 -noout -serial -subject -issuer
serial=61D859D4EC6AD34269B5CA04D7F1198F33033589
subject=CN=uws.lan
issuer=CN=uws.lan
```

#### リーフ証明書の証明書チェーン確認
旧中間CAで発行したリーフ証明書とチェーン
```vim:old certificate chain
-----BEGIN CERTIFICATE-----
MIID5zCCAs+gAwIBAgIUL2cGtcUEaw7JEgB1yVuLRD93gxowDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
```
新中間CAで発行したリーフ証明書とチェーン
```vim: new certificate chain
-----BEGIN CERTIFICATE-----
MIID5zCCAs+gAwIBAgIUW8gnSOG6YBci61d4xB/DTZ1wmAEwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
~~略~~
-----END CERTIFICATE-----
```

#### 検証1. 旧ルート証明書で新しい中間証明書で発行したリーフ証明書を検証
エンドユーザ端末で証明書のインストール状況を確認する。
```powershell
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan
[Subject]
  CN=uws.lan
[Issuer]
  CN=uws.lan
[Serial Number]
  44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5   ## -> 旧ルート証明書がインストールされている。
[Not Before]
  2024/07/09 火曜日 0:26:48
[Not After]
  2025/07/09 水曜日 0:27:18
[Thumbprint]
  0EC62BB61D1BD4129C9AB9FCAECF641BB224AEF0
```
旧リーフ証明書が登録されたWebサーバにアクセス。-> 接続成功
```powershell
PS Cert:\CurrentUser\Root> curl.exe --ssl-no-revoke https://test-rhel9.uws.lan
<html>
    <body>
        <div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
            ago-shi test page.
        </div>
    </body>
</html>
## 接続出来た
PS Cert:\CurrentUser\Root> echo $?
True
```
#### 検証2. 新しいルート証明書で新しい中間証明書で発行したサーバ証明書を検証
```powershell
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan
[Subject]
  CN=uws.lan
[Issuer]
  CN=uws.lan
[Serial Number]
  61D859D4EC6AD34269B5CA04D7F1198F33033589  ## -> 新ルート証明書
[Not Before]
  2025/03/01 土曜日 2:44:40
[Not After]
  2027/02/19 金曜日 2:45:10
[Thumbprint]
  5BB2E4A48ECAEEB92FCB91E9C1708D100A999795
```
新リーフ証明書が登録されたwebサーバへ接続する。-> 接続成功
```powershell
PS Cert:\CurrentUser\Root> curl.exe --ssl-no-revoke https://test-rhel9.uws.lan
<html>
    <body>
        <div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
            ago-shi test page.
        </div>
    </body>
</html>
## 接続出来た
PS Cert:\CurrentUser\Root> echo $?
True
```

以上