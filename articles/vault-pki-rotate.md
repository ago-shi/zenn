---
title: "hashicorp Vault CA証明書のローテーション"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hashicorpVault", "pki"]
published: false
---
## この記事の目的
- hashicorp Vault pkiエンジンで構築した中間CAの証明書を更新する。


## 対応の経緯
自製の更新shellに問題があってサーバ証明書が失効していた。
手動でサーバ証明書を更新しようとしたがエラー。
中間CA証明書も失効していた。
中間CA証明書を手動更新し、復旧するところからやる。

構成
root CA  pki_root
|
中間CA    pki_int

root CA証明書の確認
2025/7/8までなのでrootも更新しておきたい。が、まずは中間CAの復旧を優先する。
```
$ vault read pki_root/cert/ca -format=json | jq -r '.data.certificate' > rootCA.crt
$ openssl x509 -noout -subject -dates -in rootCA.crt
subject=CN = uws.lan
notBefore=Jul  8 15:26:48 2024 GMT
notAfter=Jul  8 15:27:18 2025 GMT
```

中間CA証明書の確認
有効期限切れ。これを更新する。
```
$ openssl x509 -noout -subject -dates -in pki_int_ca.crt
subject=CN = uws.lan Intermediate CA
notBefore=Jul  9 04:06:49 2024 GMT
notAfter=Jan  7 16:07:19 2025 GMT
```

中間CAのCSRと証明書の作成
```
# 中間CAのCSR作成
$ vault write -format=json pki_int/intermediate/generate/internal \
> common_name="uws.lan Intermediate CA" \
> issuer_name="uws-intermediate-admin" \
> | jq -r '.data.csr' > pki_intermediate.csr

# 中間CA証明書の作成  (ここではTTLは3か月)
$ vault write -format=json pki_root/root/sign-intermediate \
> issuer_ref="uws-admin" \
> csr=@pki_intermediate.csr \
> format=pem_bundle \
> ttl="2160h" \
> | jq -r '.data.certificate' > pki_intermediate.cert.pem
```

まだ中間CA証明書が期限切れ。更新されていない。
```
$ openssl x509 -noout -subject -dates -in intermediate_ca.crt
subject=CN = uws.lan Intermediate CA
notBefore=Jul  9 04:06:49 2024 GMT
notAfter=Jan  7 16:07:19 2025 GMT
```

issuerは登録されているが、使っている中間CA証明書が古いままのようだ。
```
$ vault list pki_int/issuers
Keys
----
3f04ec8d-fa7a-1c68-af77-ecac46126529
abad205e-2f99-be98-ec4d-5cfb2db73b6f
baa33091-6fd5-2cb8-256d-a6938dc0ade5
c1231ab9-4f01-0687-758c-76acfca139f4
f67ca50d-0ebc-9e78-6487-ab7842a3429b
# 何回か中間CA証明書をインポートしたのでたくさん出てくるｗ
```

ここが参考になる。
https://zenn.dev/i10chu/articles/f27f0a2129707d

rootのローテーションは例示されているが中間CAが見つからない。
https://developer.hashicorp.com/vault/tutorials/pki/pki-engine?productSlug=vault&tutorialSlug=secrets-management&tutorialSlug=pki-engine

2個上のサイトの以下部分を対応すれば中間CAのdefault issuerが変えられる！
```
$ vault write pki_int/root/replace default=xc-example-dot-com-intermediate
# default=にissuersのkeyを指定してもOK↓

$ vault write pki_int/root/replace default=f67ca50d-0ebc-9e78-6487-ab7842a3429b
Key                              Value
---                              -----
default                          f67ca50d-0ebc-9e78-6487-ab7842a3429b
default_follows_latest_issuer    false

$ openssl x509 -noout -subject -dates -in pki_int.ca.crt
subject=CN = uws.lan Intermediate CA
notBefore=Jan 19 15:39:51 2025 GMT
notAfter=Jul 21 03:40:21 2025 GMT
```
サーバ証明書を作ってみる。
```
[root@vault01 intermediate]# v write -format=json pki_int/issue/uws-dot-lan \
> common_name="vault01.uws.lan" \
> alt_names="DNS:vault01,IP:192.168.1.69" > vault01.json
Error writing data to pki_int/issue/uws-dot-lan: Error making API request.

URL: PUT https://127.0.0.1:8200/v1/pki_int/issue/uws-dot-lan
Code: 400. Errors:

* cannot satisfy request, as TTL would result in notAfter of 2025-02-23T15:51:38.619885369Z that is beyond the expiration of the CA certificate at 2025-01-07T16:07:19Z
```

roleのissuer_refを更新しないといけない。
以下コマンドでissuer_refを明示しないと、issuer_ref=defaultになる。
```
$ vault write pki_int/roles/uws-dot-lan ttl="720h" max_ttl="2160h" allow_ip_sans=true key_type=rsa key_bits=2048 allowed_domains="uws.lan" allow_subdomains=true
[root@vault01 intermediate]# v write pki_int/roles/uws-dot-lan ttl="720h" max_ttl="2160h" allow_ip_
sans=true key_type=rsa key_bits=2048 allowed_domains="uws.lan" allow_subdomains=true
Key                                   Value
---                                   -----
allow_any_name                        false
allow_bare_domains                    false
allow_glob_domains                    false
allow_ip_sans                         true
allow_localhost                       true
allow_subdomains                      true
allow_token_displayname               false
allow_wildcard_certificates           true
allowed_domains                       [uws.lan]
allowed_domains_template              false
allowed_other_sans                    []
allowed_serial_numbers                []
allowed_uri_sans                      []
allowed_uri_sans_template             false
allowed_user_ids                      []
basic_constraints_valid_for_non_ca    false
client_flag                           true
cn_validations                        [email hostname]
code_signing_flag                     false
country                               []
email_protection_flag                 false
enforce_hostnames                     true
ext_key_usage                         []
ext_key_usage_oids                    []
generate_lease                        false
issuer_ref                            default
key_bits                              2048
key_type                              rsa
key_usage                             [DigitalSignature KeyAgreement KeyEncipherment]
locality                              []
max_ttl                               2160h
no_store                              false
not_after                             n/a
not_before_duration                   30s
organization                          []
ou                                    []
policy_identifiers                    []
postal_code                           []
province                              []
require_cn                            true
server_flag                           true
signature_bits                        256
street_address                        []
ttl                                   720h
use_csr_common_name                   true
use_csr_sans                          true
use_pss                               false
```
サーバ証明書を再作成してみる。->成功！！