---
title: "PKI入門 ~hashicorp vaultによるルート証明書のローテーション~"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hashicorpVault", "pki"]
published: false
---
## 背景
hashicorp vaultを使ったルート証明書のローテーションの手順を確認していく。

## 参考
https://zenn.dev/i10chu/articles/f27f0a2129707d

## ローテーション前、サーバ証明書の確認
hashicorp vaultを使ったpki構築方法は割愛します。

## 現状確認
#### 現在の証明書チェーンを確認
```
## サーバ証明書
$ openssl x509 -noout -serial -subject_hash -issuer_hash -in <サーバ証明書>
serial=6BC92AE76A60149A122C72F53D5EE7B638F31183
cfb07e43    #主体者
1e3552d1    #発行者

$ openssl x509 -noout -serial -subject_hash -issuer_hash -in <中間証明書>
serial=7557133A04B5FEFD2449176AFF49AEA464E2DF6A
1e3552d1    #主体者 -> サーバ証明書の発行者と一致
3cb54758    #発行者 

$ openssl x509 -noout -serial -subject_hash -issuer_hash -in <ルート証明書>
serial=44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5
3cb54758    #主体者 -> 中間証明書の発行者と一致
3cb54758    #発行者 -> 主体者と一致
```
#### ブラウザからのアクセス確認
ブラウザにルート証明書をインストールしているため、問題なくアクセスできる。
![](/images/vault-pki-root-rotation/https-test-page.png)

## ルート証明書のローテーション
#### 証明書のTTL決定
ここでは以下の2つの前提でTTLを決定した。
- PKIの勉強のため、頻繁にローテーションするTTLに決定する。
- 常に2つのルート証明書、中間証明書で運用することは避ける。

設定したTTL
- サーバ証明書: 1440h  (約2か月)
- 中間証明書:   5760h  (約8か月)
- ルート証明書: 17280h (約24か月)
- pkiエンジン(ルート認証局): 43200h (約5年)
- pkiエンジン(中間認証局): 17280h (約2年)

#### 新しいルート証明書を作成
```
## pkiエンジンのmax ttlを設定する。
$ vault secrets tune -max-lease-ttl="43200h" pki_root/

$ vault write pki_root/root/rotate/internal \
common_name="uws.lan" \
issuer_name="root-2025" \
ttl="17280h"

$ vault list pki_root/issuers
Keys
----
3d6c537f-d490-8c6d-3138-0c527c2f9dda    ## -> 新たに作成したルート証明書
fbe1e696-4eca-f128-f9ad-10a1eb14fee8    ## -> 以前のルート証明書

$ vault read -format=json pki_root/issuer/root-2025 \
  | jq -r '.data.certificate' | openssl x509 -noout -serial -dates
serial=61D859D4EC6AD34269B5CA04D7F1198F33033589  ## -> 証明書のシリアル
notBefore=Feb 28 17:44:40 2025 GMT
notAfter=Feb 18 17:45:10 2027 GMT

$ vault read -format=json pki_root/issuer/root-2025 | jq -r '.data.certificate' \
  | openssl x509 -noout -text | grep -A 3 "X509v3 Subject Key Identifier:"
            X509v3 Subject Key Identifier:
                B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> 主体者の識別子
            X509v3 Authority Key Identifier:
                B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> 発行者の識別子
```

#### クロスルート中間証明書を作成
```
## pkiエンジンのmax ttlを設定する。
$ vault secrets tune -max-lease-ttl="5760h" pki_int/

## 中間認証局でCSRを作成
$ vault write -format=json pki_int/intermediate/cross-sign \
 common_name="uws.lan Intermediate Authority" \
 key_ref="$(vault read pki_int/issuer/$(vault read -field=default pki_int/config/issuers) \
 | grep -i key_id | awk '{print $2}')" \
 | jq -r '.data.csr' \
 | tee cross-signed-intermediate.csr

## ルート認証局でクロス中間証明書を作成
$ vault write -format=json pki_root/issuer/root-2025/sign-intermediate \
 common_name="uws.lan Intermediate Authority" \
 csr=@cross-signed-intermediate.csr \
 ttl="5760h" \
 | jq -r '.data.certificate' | tee cross-signed-intermediate.crt

## 証明書のシリアル、主体者、発行者を確認
$ cat cross-signed-intermediate.crt \
  | openssl x509 -noout -serial -dates
serial=7485A01781184780BD365E4CA3FB40FEF9CBCC69
notBefore=Feb 28 18:26:20 2025 GMT
notAfter=Oct 26 18:26:50 2025 GMT

$ cat cross-signed-intermediate.crt \
  | openssl x509 -noout -text \
  | grep -A 3 "X509v3 Subject Key Identifier:"
            X509v3 Subject Key Identifier:
                E7:92:DE:02:36:B9:8D:E7:39:29:EC:4D:89:12:90:31:34:BC:5D:74
            X509v3 Authority Key Identifier:
                B1:A0:89:2A:ED:CD:A7:26:63:C6:BD:59:76:E3:0D:1F:DB:7D:C2:5C  ## -> ルート証明書のsubjectと一致
```

#### クロスルート中間証明書をインポート
```
$ vault write pki_int/intermediate/set-signed \
  certificate=@cross-signed-intermediate.crt
Key                 Value
---                 -----
existing_issuers    <nil>
existing_keys       <nil>
imported_issuers    [bfd69160-0473-9ebb-0025-3aad4e5be9ed]
imported_keys       <nil>
mapping             map[bfd69160-0473-9ebb-0025-3aad4e5be9ed:d8f87b5b-fb76-7873-e2db-db07983c614b]

$ vault list pki_int/issuers
Keys
----
bfd69160-0473-9ebb-0025-3aad4e5be9ed  ## -> 新しい中間証明書(クロスルート)
f67ca50d-0ebc-9e78-6487-ab7842a3429b

$ vault write pki_int/issuer/bfd69160-0473-9ebb-0025-3aad4e5be9ed \
issuer_name=xc-uws-dot-lan-intermediate
Key                               Value
---                               -----
ca_chain                          [-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx
ODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G
HQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa
fVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw
OHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0
dDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy
73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN
hqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N
pyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw
czovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2
MDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv
b3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv
VGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV
D4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7
/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa
Nv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo
nFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ
-----END CERTIFICATE-----
]
certificate                       -----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx
ODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G
HQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa
fVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw
OHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0
dDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy
73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN
hqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N
pyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw
czovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2
MDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv
b3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv
VGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV
D4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7
/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa
Nv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo
nFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ
-----END CERTIFICATE-----
crl_distribution_points           []
issuer_id                         bfd69160-0473-9ebb-0025-3aad4e5be9ed
issuer_name                       xc-uws-dot-lan-intermediate
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
leaf_not_after_behavior           err
manual_chain                      <nil>
ocsp_servers                      []
revocation_signature_algorithm    n/a
revoked                           false
usage                             crl-signing,issuing-certificates,ocsp-signing,read-only

$ vault patch pki_int/issuer/"$(v read -field=default pki_int/config/issuers)" \
  manual_chain=self,xc-uws-dot-lan-intermediate
Key                               Value
---                               -----
ca_chain                          [-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAxMTkxNTM5NTFaFw0yNTA3MjEw
MzQwMjFaMCIxIDAeBgNVBAMTF3V3cy5sYW4gSW50ZXJtZWRpYXRlIENBMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kwFaxitg/EDd0YLwYdCWCV8ALf
YLNOt3V6uIaDimoK/fJIPdFTr80SZ/li34EV3Aj3ul+tvoieZDpA4Vp9UTwb9dGJ
IFniWIxfh4yhKRLu81hX5w+9XHl9Src8NskOqpNs/TOQET5+L3dvinA4cbJGQyWJ
6uy4f6sjoRvkg21k4evGG2pZwnl1wma0roIMPzlNJ+QObvjXFF3KKXR0MpgQeYhe
b1FI1BBl6elPf1Mhr5uyK3UFydFc/dSDcVQ3Oi2L6O9aAHo/mXCrvDLvc4kj8Wbe
2z6kLX6U8QWMuAVkbzr7IO/wJj/svBQW8COZEyNmsumewivz3/06xQ2GowIDAQAB
o4HsMIHpMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW
BBTnkt4CNrmN5zkp7E2JEpAxNLxddDAfBgNVHSMEGDAWgBSBMPzhXe8vji2dsFRh
aeDF5i108TBHBggrBgEFBQcBAQQ7MDkwNwYIKwYBBQUHMAKGK2h0dHBzOi8vdmF1
bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jvb3QvY2EwPQYDVR0fBDYwNDAyoDCg
LoYsaHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lfcm9vdC9jcmww
DQYJKoZIhvcNAQELBQADggEBAJhhgiroETcke19MFlh3hVmumzvi/9d/ZyxYUgyB
89zy+FlzJIHOHm89nmiVyapuvzMREPW8MNy2Yf7s8IdeGRO4MAqtOe1SxaRDq2IN
bvZzfdEerfa3eJ6ijjl+Ji7aRm//ZjKTZp1An40ZCoF2fTO1OkRsHgZlAcVV+DlT
+UDrt351GXQFwy5LfXh80stGOHR1F/P3OExvaKIa+2Y3ftXohHP/K0VBV5SpFd4i
lYLM2f1SIR1YX6xhK6VEId7BWsE5fr6uFt4fVPkERd8QyoaqsRYhJM5gGZu2Fh2/
PFkmLkVzs+DCa9GiuH3h8+HZyhaHIPaaxZjb/1raPBSjvTI=
-----END CERTIFICATE-----
 -----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx
ODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G
HQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa
fVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw
OHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0
dDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy
73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN
hqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N
pyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw
czovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2
MDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv
b3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv
VGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV
D4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7
/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa
Nv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo
nFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ
-----END CERTIFICATE-----
]
certificate                       -----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAxMTkxNTM5NTFaFw0yNTA3MjEw
MzQwMjFaMCIxIDAeBgNVBAMTF3V3cy5sYW4gSW50ZXJtZWRpYXRlIENBMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kwFaxitg/EDd0YLwYdCWCV8ALf
YLNOt3V6uIaDimoK/fJIPdFTr80SZ/li34EV3Aj3ul+tvoieZDpA4Vp9UTwb9dGJ
IFniWIxfh4yhKRLu81hX5w+9XHl9Src8NskOqpNs/TOQET5+L3dvinA4cbJGQyWJ
6uy4f6sjoRvkg21k4evGG2pZwnl1wma0roIMPzlNJ+QObvjXFF3KKXR0MpgQeYhe
b1FI1BBl6elPf1Mhr5uyK3UFydFc/dSDcVQ3Oi2L6O9aAHo/mXCrvDLvc4kj8Wbe
2z6kLX6U8QWMuAVkbzr7IO/wJj/svBQW8COZEyNmsumewivz3/06xQ2GowIDAQAB
o4HsMIHpMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW
BBTnkt4CNrmN5zkp7E2JEpAxNLxddDAfBgNVHSMEGDAWgBSBMPzhXe8vji2dsFRh
aeDF5i108TBHBggrBgEFBQcBAQQ7MDkwNwYIKwYBBQUHMAKGK2h0dHBzOi8vdmF1
bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jvb3QvY2EwPQYDVR0fBDYwNDAyoDCg
LoYsaHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lfcm9vdC9jcmww
DQYJKoZIhvcNAQELBQADggEBAJhhgiroETcke19MFlh3hVmumzvi/9d/ZyxYUgyB
89zy+FlzJIHOHm89nmiVyapuvzMREPW8MNy2Yf7s8IdeGRO4MAqtOe1SxaRDq2IN
bvZzfdEerfa3eJ6ijjl+Ji7aRm//ZjKTZp1An40ZCoF2fTO1OkRsHgZlAcVV+DlT
+UDrt351GXQFwy5LfXh80stGOHR1F/P3OExvaKIa+2Y3ftXohHP/K0VBV5SpFd4i
lYLM2f1SIR1YX6xhK6VEId7BWsE5fr6uFt4fVPkERd8QyoaqsRYhJM5gGZu2Fh2/
PFkmLkVzs+DCa9GiuH3h8+HZyhaHIPaaxZjb/1raPBSjvTI=
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


$ vault patch pki_int/issuer/xc-uws-dot-lan-intermediate \
 manual_chain=self,"$(vault read -field=default pki_int/config/issuers)"
Key                               Value
---                               -----
ca_chain                          [-----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx
ODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G
HQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa
fVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw
OHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0
dDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy
73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN
hqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N
pyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw
czovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2
MDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv
b3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv
VGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV
D4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7
/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa
Nv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo
nFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ
-----END CERTIFICATE-----
 -----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAxMTkxNTM5NTFaFw0yNTA3MjEw
MzQwMjFaMCIxIDAeBgNVBAMTF3V3cy5sYW4gSW50ZXJtZWRpYXRlIENBMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kwFaxitg/EDd0YLwYdCWCV8ALf
YLNOt3V6uIaDimoK/fJIPdFTr80SZ/li34EV3Aj3ul+tvoieZDpA4Vp9UTwb9dGJ
IFniWIxfh4yhKRLu81hX5w+9XHl9Src8NskOqpNs/TOQET5+L3dvinA4cbJGQyWJ
6uy4f6sjoRvkg21k4evGG2pZwnl1wma0roIMPzlNJ+QObvjXFF3KKXR0MpgQeYhe
b1FI1BBl6elPf1Mhr5uyK3UFydFc/dSDcVQ3Oi2L6O9aAHo/mXCrvDLvc4kj8Wbe
2z6kLX6U8QWMuAVkbzr7IO/wJj/svBQW8COZEyNmsumewivz3/06xQ2GowIDAQAB
o4HsMIHpMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW
BBTnkt4CNrmN5zkp7E2JEpAxNLxddDAfBgNVHSMEGDAWgBSBMPzhXe8vji2dsFRh
aeDF5i108TBHBggrBgEFBQcBAQQ7MDkwNwYIKwYBBQUHMAKGK2h0dHBzOi8vdmF1
bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jvb3QvY2EwPQYDVR0fBDYwNDAyoDCg
LoYsaHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lfcm9vdC9jcmww
DQYJKoZIhvcNAQELBQADggEBAJhhgiroETcke19MFlh3hVmumzvi/9d/ZyxYUgyB
89zy+FlzJIHOHm89nmiVyapuvzMREPW8MNy2Yf7s8IdeGRO4MAqtOe1SxaRDq2IN
bvZzfdEerfa3eJ6ijjl+Ji7aRm//ZjKTZp1An40ZCoF2fTO1OkRsHgZlAcVV+DlT
+UDrt351GXQFwy5LfXh80stGOHR1F/P3OExvaKIa+2Y3ftXohHP/K0VBV5SpFd4i
lYLM2f1SIR1YX6xhK6VEId7BWsE5fr6uFt4fVPkERd8QyoaqsRYhJM5gGZu2Fh2/
PFkmLkVzs+DCa9GiuH3h8+HZyhaHIPaaxZjb/1raPBSjvTI=
-----END CERTIFICATE-----
]
certificate                       -----BEGIN CERTIFICATE-----
MIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx
ODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G
HQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa
fVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw
OHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0
dDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy
73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN
hqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N
pyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw
czovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2
MDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv
b3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv
VGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV
D4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7
/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa
Nv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo
nFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ
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

```

#### サーバ証明書を発行してみる
```
$ vault write -format=json pki_int/issue/uws-dot-lan \
  common_name="test-rhel9.uws.lan" ttl="72h" > test-rhel9.2.json

## issuing_caはまだ現行の中間証明書
## ca_chainに新規作成した中間証明書(クロスルート中間証明書)が確認できる
$ cat test-rhel9.2.json
{
  "request_id": "b2c80dcb-f098-5fec-18dd-3a9fac2a5506",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "ca_chain": [
      "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\nBQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAxMTkxNTM5NTFaFw0yNTA3MjEw\nMzQwMjFaMCIxIDAeBgNVBAMTF3V3cy5sYW4gSW50ZXJtZWRpYXRlIENBMIIBIjAN\nBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kwFaxitg/EDd0YLwYdCWCV8ALf\nYLNOt3V6uIaDimoK/fJIPdFTr80SZ/li34EV3Aj3ul+tvoieZDpA4Vp9UTwb9dGJ\nIFniWIxfh4yhKRLu81hX5w+9XHl9Src8NskOqpNs/TOQET5+L3dvinA4cbJGQyWJ\n6uy4f6sjoRvkg21k4evGG2pZwnl1wma0roIMPzlNJ+QObvjXFF3KKXR0MpgQeYhe\nb1FI1BBl6elPf1Mhr5uyK3UFydFc/dSDcVQ3Oi2L6O9aAHo/mXCrvDLvc4kj8Wbe\n2z6kLX6U8QWMuAVkbzr7IO/wJj/svBQW8COZEyNmsumewivz3/06xQ2GowIDAQAB\no4HsMIHpMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW\nBBTnkt4CNrmN5zkp7E2JEpAxNLxddDAfBgNVHSMEGDAWgBSBMPzhXe8vji2dsFRh\naeDF5i108TBHBggrBgEFBQcBAQQ7MDkwNwYIKwYBBQUHMAKGK2h0dHBzOi8vdmF1\nbHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jvb3QvY2EwPQYDVR0fBDYwNDAyoDCg\nLoYsaHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lfcm9vdC9jcmww\nDQYJKoZIhvcNAQELBQADggEBAJhhgiroETcke19MFlh3hVmumzvi/9d/ZyxYUgyB\n89zy+FlzJIHOHm89nmiVyapuvzMREPW8MNy2Yf7s8IdeGRO4MAqtOe1SxaRDq2IN\nbvZzfdEerfa3eJ6ijjl+Ji7aRm//ZjKTZp1An40ZCoF2fTO1OkRsHgZlAcVV+DlT\n+UDrt351GXQFwy5LfXh80stGOHR1F/P3OExvaKIa+2Y3ftXohHP/K0VBV5SpFd4i\nlYLM2f1SIR1YX6xhK6VEId7BWsE5fr6uFt4fVPkERd8QyoaqsRYhJM5gGZu2Fh2/\nPFkmLkVzs+DCa9GiuH3h8+HZyhaHIPaaxZjb/1raPBSjvTI=\n-----END CERTIFICATE-----",
      "-----BEGIN CERTIFICATE-----\nMIIDtjCCAp6gAwIBAgIUdIWgF4EYR4C9Nl5Mo/tA/vnLzGkwDQYJKoZIhvcNAQEL\nBQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAyMjgxODI2MjBaFw0yNTEwMjYx\nODI2NTBaMCkxJzAlBgNVBAMTHnV3cy5sYW4gSW50ZXJtZWRpYXRlIEF1dGhvcml0\neTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMtZMBWsYrYPxA3dGC8G\nHQlglfAC32CzTrd1eriGg4pqCv3ySD3RU6/NEmf5Yt+BFdwI97pfrb6InmQ6QOFa\nfVE8G/XRiSBZ4liMX4eMoSkS7vNYV+cPvVx5fUq3PDbJDqqTbP0zkBE+fi93b4pw\nOHGyRkMliersuH+rI6Eb5INtZOHrxhtqWcJ5dcJmtK6CDD85TSfkDm741xRdyil0\ndDKYEHmIXm9RSNQQZenpT39TIa+bsit1BcnRXP3Ug3FUNzoti+jvWgB6P5lwq7wy\n73OJI/Fm3ts+pC1+lPEFjLgFZG86+yDv8CY/7LwUFvAjmRMjZrLpnsIr89/9OsUN\nhqMCAwEAAaOB7DCB6TAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAd\nBgNVHQ4EFgQU55LeAja5jec5KexNiRKQMTS8XXQwHwYDVR0jBBgwFoAUsaCJKu3N\npyZjxr1ZduMNH9t9wlwwRwYIKwYBBQUHAQEEOzA5MDcGCCsGAQUFBzAChitodHRw\nczovL3ZhdWx0MDEudXdzLmxhbjo4MjAwL3YxL3BraV9yb290L2NhMD0GA1UdHwQ2\nMDQwMqAwoC6GLGh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jv\nb3QvY3JsMA0GCSqGSIb3DQEBCwUAA4IBAQB6dD5u6Jf7AKDYAiKb+IbH/fvGCtDv\nVGrOwF5qcMd12ooT18qQbV0rKk1WSjEL0KwzzEPSwxGfs3oHW3XNIU1bv5WvJMmV\nD4Y+fsbnw+dj1cWrnwXCVWe/hjKKn3SygK5ih2OEP8njWBnLiob0GxTGty7vPMd7\n/rQ6dLC2qLOi80D3grig2H3vrypCKy7uZ2gqhzxCTQ/kfPynXwpsRTM/EJc4LDYa\nNv162nF6ouzBlEfVWP8natGHlBaCWj8FlMGc3t5j4FtWEVEqoB7t+zc0EwpQdMQo\nnFI0wRyIB6taljf60NZzsq9kLA4dcNk74DdHNq+wCOl1FCVwBjdB0TGJ\n-----END CERTIFICATE-----"
    ],
    "certificate": "-----BEGIN CERTIFICATE-----\nMIID5zCCAs+gAwIBAgIUW8gnSOG6YBci61d4xB/DTZ1wmAEwDQYJKoZIhvcNAQEL\nBQAwIjEgMB4GA1UEAxMXdXdzLmxhbiBJbnRlcm1lZGlhdGUgQ0EwHhcNMjUwMzAy\nMDUyNDM2WhcNMjUwMzA1MDUyNTA1WjAdMRswGQYDVQQDExJ0ZXN0LXJoZWw5LnV3\ncy5sYW4wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDBCRR3IojIpDCb\nhRaBG44M5iaQI/R9b1cKjbAd5emUTPyXSb1LGTM+zMxe91ZyJ6kfbNegHzmbbHRv\nMdsT//oYylOutFkj8oUqhu7vIYJRqEx0AJFCYQQ0V3BPi1jopDlBxsWuFSevA/Zh\nxap25tVBkxiD9uj2gHzEJNT/JBw5D5mna2W1brArQpspHKGgT00HQkb/vgKaCe1x\nAVEFBOYn3KNnFYrKIgiXel3MY73NjhdsrNHVhYEO44eJyFLyAi0LZqduo0d2w4fZ\niWL4KIqIXlI2QLhCx3YUAOkAkLbj8GnkWvMM1khOmGyHbMd7M7w2d2Sa+vJyZgWg\ncn2l78JnAgMBAAGjggEYMIIBFDAOBgNVHQ8BAf8EBAMCA6gwHQYDVR0lBBYwFAYI\nKwYBBQUHAwEGCCsGAQUFBwMCMB0GA1UdDgQWBBR1nAyPNiVWhKD71X+g66zr7v4A\nBDAfBgNVHSMEGDAWgBTnkt4CNrmN5zkp7E2JEpAxNLxddDBGBggrBgEFBQcBAQQ6\nMDgwNgYIKwYBBQUHMAKGKmh0dHBzOi8vdmF1bHQwMS51d3MubGFuOjgyMDAvdjEv\ncGtpX2ludC9jYTAdBgNVHREEFjAUghJ0ZXN0LXJoZWw5LnV3cy5sYW4wPAYDVR0f\nBDUwMzAxoC+gLYYraHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lf\naW50L2NybDANBgkqhkiG9w0BAQsFAAOCAQEAi8OJvrjFQyKg8y+lR6TAfpMK1J3/\nvPqMlHKk/5IOlRozU/59/YmwT7L2u3Df0bKlAEplqdOBDeOnXPyqNMxRrUBpsePl\n4/IowLxYA/KGendoRbd5aubqTqR66idrn4JGhiB3UwqcsaFTHGHb22HvDGOSyHdz\n2y6yjVelaKrmQ+96WEjpuuWbQV4iu8BllJ2b5J8jugbeFOVxPyhZnzna1iASK9ZI\nIrh11LxgIU7MiKhXH3zm59TQQia+X49Ew6pDZ7KI48qIiYOZo/OzbcQ4QEWYNP8b\nMLTslYUntMmXvLFClmu5r8Bii4U12cUWcm9H23qvE/qqKbVjan0v33F3uA==\n-----END CERTIFICATE-----",
    "expiration": 1741152305,
    "issuing_ca": "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIUdVcTOgS1/v0kSRdq/0mupGTi32owDQYJKoZIhvcNAQEL\nBQAwEjEQMA4GA1UEAxMHdXdzLmxhbjAeFw0yNTAxMTkxNTM5NTFaFw0yNTA3MjEw\nMzQwMjFaMCIxIDAeBgNVBAMTF3V3cy5sYW4gSW50ZXJtZWRpYXRlIENBMIIBIjAN\nBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kwFaxitg/EDd0YLwYdCWCV8ALf\nYLNOt3V6uIaDimoK/fJIPdFTr80SZ/li34EV3Aj3ul+tvoieZDpA4Vp9UTwb9dGJ\nIFniWIxfh4yhKRLu81hX5w+9XHl9Src8NskOqpNs/TOQET5+L3dvinA4cbJGQyWJ\n6uy4f6sjoRvkg21k4evGG2pZwnl1wma0roIMPzlNJ+QObvjXFF3KKXR0MpgQeYhe\nb1FI1BBl6elPf1Mhr5uyK3UFydFc/dSDcVQ3Oi2L6O9aAHo/mXCrvDLvc4kj8Wbe\n2z6kLX6U8QWMuAVkbzr7IO/wJj/svBQW8COZEyNmsumewivz3/06xQ2GowIDAQAB\no4HsMIHpMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW\nBBTnkt4CNrmN5zkp7E2JEpAxNLxddDAfBgNVHSMEGDAWgBSBMPzhXe8vji2dsFRh\naeDF5i108TBHBggrBgEFBQcBAQQ7MDkwNwYIKwYBBQUHMAKGK2h0dHBzOi8vdmF1\nbHQwMS51d3MubGFuOjgyMDAvdjEvcGtpX3Jvb3QvY2EwPQYDVR0fBDYwNDAyoDCg\nLoYsaHR0cHM6Ly92YXVsdDAxLnV3cy5sYW46ODIwMC92MS9wa2lfcm9vdC9jcmww\nDQYJKoZIhvcNAQELBQADggEBAJhhgiroETcke19MFlh3hVmumzvi/9d/ZyxYUgyB\n89zy+FlzJIHOHm89nmiVyapuvzMREPW8MNy2Yf7s8IdeGRO4MAqtOe1SxaRDq2IN\nbvZzfdEerfa3eJ6ijjl+Ji7aRm//ZjKTZp1An40ZCoF2fTO1OkRsHgZlAcVV+DlT\n+UDrt351GXQFwy5LfXh80stGOHR1F/P3OExvaKIa+2Y3ftXohHP/K0VBV5SpFd4i\nlYLM2f1SIR1YX6xhK6VEId7BWsE5fr6uFt4fVPkERd8QyoaqsRYhJM5gGZu2Fh2/\nPFkmLkVzs+DCa9GiuH3h8+HZyhaHIPaaxZjb/1raPBSjvTI=\n-----END CERTIFICATE-----",
    "private_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAwQkUdyKIyKQwm4UWgRuODOYmkCP0fW9XCo2wHeXplEz8l0m9\nSxkzPszMXvdWciepH2zXoB85m2x0bzHbE//6GMpTrrRZI/KFKobu7yGCUahMdACR\nQmEENFdwT4tY6KQ5QcbFrhUnrwP2YcWqdubVQZMYg/bo9oB8xCTU/yQcOQ+Zp2tl\ntW6wK0KbKRyhoE9NB0JG/74CmgntcQFRBQTmJ9yjZxWKyiIIl3pdzGO9zY4XbKzR\n1YWBDuOHichS8gItC2anbqNHdsOH2Yli+CiKiF5SNkC4Qsd2FADpAJC24/Bp5Frz\nDNZITphsh2zHezO8NndkmvrycmYFoHJ9pe/CZwIDAQABAoIBAHKN3ONGTz4ikeX4\n+P3tSENHYaMwcyrtJA5TPyy+//rOJSfyzq7+aXbfOnkw9tAP0UGg6eVQInOlzQMf\n5w7bXaPQjhCjXjMC/Rvbr3ehvyCOa7B7lbh6snANY80QuNZ2frQWLcG9NCucgl5L\nW3nsSqn7jRTjNiTy4xfTc8NlvontOsU5qN80yComw2X9DfgzTOG60r7d4lTsv1KS\nfIMHu1sr/hY6c4Aov9hDYV37SRLohkexz2L6I/n2p4ucPgqOIiKZwTc8r/6WcH/v\nHnpp6xA8/v5//noDeYQlgRkOtYE3dxfsxwsBBpz9b2kSJEsFPgNPgtcf9GXWKQFJ\nHLtJ6wECgYEA9wwTsYIZ5Q5i10gGhGb2H1uhQq0XhX4wnlIdNbwRURNP8DCrlLHq\nyV8WmdRth87vv2P47ius2etjeDsD27Q0AFS41U/H01pPsWeefzig6Kon3orbfwp9\nawDQUr+6yl7qb6GLbnJcskpiUKm1Qa8unGKZ5eke1q32MoyMUUX5h2ECgYEAyAfs\nANAqAyzIt3/gI6qb1k2OZiAKjv6GKNDkRUUvTtCE2HryEpDUK3C2ljnBIyDcygnc\nT5UJPMR8mcj2lgCCcm3GGP4XTh9pZBTEU9/pipQl3pHjkk4aeCIteMzbMwDy9zyz\n+/tkiclP9iwdii+xVm5UThLP8ozWuiXDRhHTRscCgYBfBwU4PXwqcJsyhiEDovs/\nWqawGBa5Ia4f6CQWPE5I6m3QTVhirQFMDkiKSX0MRVxROWpSavhlJrcvUzwLschi\n7DPg0Xxi3xVSfzIna6fxdyo43x7JQka19y0q91cpatMwt2oDxPfFGPmyX2U6a+E5\nBHCAUGitWWMfVJLQ3GK8YQKBgH7Njs1BKLDUjfTNSoAxohJrHc8dlrPpI5DyQxKq\ndf/nbZ9x6MzeJLHZBNYcjJPBPFWThKaqWq27/STb4X1bm1YAwqiLQqjSftPj2kU1\nV23y1kLOhs3zVxI60EqYyof9nQgf4hTl22kBRgBPHPbBnxCkZisL/+jJYUGluLFN\nkXp3AoGBAKCYh8+4VuA/Mw8CLLSc9t32h0h/7IbCZVSTiSTszZ19p+TyCfGIAUaX\ne2YtEOn4hSdZ+8T/gBlBi6K8oAWE0dM/7C7yfEFG0Ki9TlsoeeyG6RQwRJNYKx71\npMg6pzp0YIGwJJ4wGw+LAVzhCGwESi0Jg1d17I4eG82DHueb9sM0\n-----END RSA PRIVATE KEY-----",
    "private_key_type": "rsa",
    "serial_number": "5b:c8:27:48:e1:ba:60:17:22:eb:57:78:c4:1f:c3:4d:9d:70:98:01"
  },
  "warnings": null,
  "mount_type": "pki"
}

## jsonファイルから作成した証明書チェーン
$ cat cur-root-cur-int-new-int-new-test-rhel9.crt
## サーバ証明書
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
#### 発行したサーバ証明書を新ルート証明書で検証する
```
PS C:\Users\techn> cd cert:\CurrentUser\Root
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan
[Subject]
  CN=uws.lan

~~略~~

[Thumbprint]
  5BB2E4A48ECAEEB92FCB91E9C1708D100A999795

PS Cert:\CurrentUser\Root> Get-ChildItem 5BB2E4A48ECAEEB92FCB91E9C1708D100A999795 | Format-List
Subject      : CN=uws.lan
Issuer       : CN=uws.lan
Thumbprint   : 5BB2E4A48ECAEEB92FCB91E9C1708D100A999795
FriendlyName :
NotBefore    : 2025/03/01 土曜日 2:44:40
NotAfter     : 2027/02/19 金曜日 2:45:10
Extensions   : {System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, S
               ystem.Security.Cryptography.Oid...}

## 接続出来た
PS Cert:\CurrentUser> curl.exe --ssl-no-revoke https://test-rhel9.uws.lan
<html>
    <body>
        <div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
            ago-shi test page.
        </div>
    </body>
</html>

PS Cert:\CurrentUser> echo $?
True
```

#### 発行したサーバ証明書を旧ルート証明書で検証する
```
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan
[Subject]
  CN=uws.lan

[Issuer]
  CN=uws.lan

[Serial Number]
  44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5

[Not Before]
  2024/07/09 火曜日 0:26:48

[Not After]
  2025/07/09 水曜日 0:27:18

[Thumbprint]
  0EC62BB61D1BD4129C9AB9FCAECF641BB224AEF0

PS Cert:\CurrentUser\Root> curl.exe --ssl-no-revoke https://test-rhel9.uws.lan
<html>
    <body>
        <div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
            ago-shi test page.
        </div>
    </body>
</html>

## 接続できた
PS Cert:\CurrentUser\Root> echo $?
True
```

#### 発行したサーバ証明書に対し証明書チェーンを旧中間証明書のみ、旧ルート証明書で検証する
```
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan

[Subject]
  CN=uws.lan

[Issuer]
  CN=uws.lan

[Serial Number]
  44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5

[Not Before]
  2024/07/09 火曜日 0:26:48

[Not After]
  2025/07/09 水曜日 0:27:18

[Thumbprint]
  0EC62BB61D1BD4129C9AB9FCAECF641BB224AEF0

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

#### 発行したサーバ証明書に対し証明書チェーンを旧中間証明書のみ、新ルート証明書で検証する
```
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan

[Subject]
  CN=uws.lan

[Issuer]
  CN=uws.lan

[Serial Number]
  61D859D4EC6AD34269B5CA04D7F1198F33033589

[Not Before]
  2025/03/01 土曜日 2:44:40

[Not After]
  2027/02/19 金曜日 2:45:10

[Thumbprint]
  5BB2E4A48ECAEEB92FCB91E9C1708D100A999795

PS Cert:\CurrentUser\Root> curl.exe --ssl-no-revoke https://test-rhel9.uws.lan
curl: (35) schannel: next InitializeSecurityContext failed: Unknown error (0x80096004) - 証明書の署名を検証できません。

## 接続に失敗
PS Cert:\CurrentUser\Root> echo $?
False
```

## ルート認証局のデフォルトルート証明書を変更
```
$ vault write pki_root/root/replace default=root-2025
Key                              Value
---                              -----
default                          3d6c537f-d490-8c6d-3138-0c527c2f9dda
default_follows_latest_issuer    false
```

## 旧ルート証明書の利用権限変更
```
## 現在の利用権限を確認
$ vault read pki_root/issuer/fbe1e696-4eca-f128-f9ad-10a1eb14fee8 \
  | tail -1
usage                             crl-signing,issuing-certificates,ocsp-signing,read-only

## 利用権限の変更
v write pki_root/issuer/uws-admin issuer_name="uws-admin" usage=read-only,crl-sig
ning | tail -1
usage                             crl-signing,read-only
```

## 中間認証局のデフォルト中間証明書を変更
```
## 現在のデフォルトを確認
$ vault read pki_int/issuer/default | tail -11
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

## 変更する中間証明書(クロスルート証明書)を確認
$ vault list pki_int/issuers
Keys
----
bfd69160-0473-9ebb-0025-3aad4e5be9ed  ## -> クロスルート証明書
f67ca50d-0ebc-9e78-6487-ab7842a3429b

$ vault read pki_int/issuer/bfd69160-0473-9ebb-0025-3aad4e5be9ed | tail -11
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

## デフォルトをクロスルート証明書へ変更
$ vault write pki_int/root/replace default=xc-uws-dot-lan-intermediate
Key                              Value
---                              -----
default                          bfd69160-0473-9ebb-0025-3aad4e5be9ed
default_follows_latest_issuer    false

$ vault read pki_int/issuer/default | tail -11
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
```

## 旧中間証明書の利用権限変更
```
$ vualt read pki_int/issuer/int-20250222 | tail -11
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

$ vault write pki_int/issuer/int-20250222 \
> issuer_name=int-20250222 usage=read-only,crl-signing | tail -11
crl_distribution_points           []
issuer_id                         f67ca50d-0ebc-9e78-6487-ab7842a3429b
issuer_name                       int-20250222
issuing_certificates              []
key_id                            d8f87b5b-fb76-7873-e2db-db07983c614b
leaf_not_after_behavior           err
manual_chain                      <nil>
ocsp_servers                      []
revocation_signature_algorithm    n/a
revoked                           false
usage                             crl-signing,read-only
```

## 古いルート証明書で新しい中間証明書で発行したサーバ証明書を検証する
```
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan

[Subject]
  CN=uws.lan

[Issuer]
  CN=uws.lan

[Serial Number]
  44EECAD591CD00D4F53A3C0A881CBBB5EEE44FF5

[Not Before]
  2024/07/09 火曜日 0:26:48

[Not After]
  2025/07/09 水曜日 0:27:18

[Thumbprint]
  0EC62BB61D1BD4129C9AB9FCAECF641BB224AEF0

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
PS Cert:\CurrentUser\Root>
```

## 新しいルート証明書で新しい中間証明書で発行したサーバ証明書を検証する
```
PS Cert:\CurrentUser\Root> Get-ChildItem | Select-String uws.lan

[Subject]
  CN=uws.lan

[Issuer]
  CN=uws.lan

[Serial Number]
  61D859D4EC6AD34269B5CA04D7F1198F33033589

[Not Before]
  2025/03/01 土曜日 2:44:40

[Not After]
  2027/02/19 金曜日 2:45:10

[Thumbprint]
  5BB2E4A48ECAEEB92FCB91E9C1708D100A999795

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