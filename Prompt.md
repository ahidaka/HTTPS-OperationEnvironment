# HTTPS-OperationEnvironment

## 最初の質問

< こんばんは。私は現在、自体で固定IPアドレスに Linux サーバーを数台立ち上げて、Apache2 を使用して、いくつかの独自ドメインの Homepage を運用しています。DNS (BIND)や、Postfix のインターネットサービスも運用しています。オンラインショップも運用しているので、Let's Encrypt の証明書サービスを利用して、https web サーバーを2台、個別・各専用に立ち上げています。この様な運用で、これまで大きな問題はありませんでした。一方で Apache2 のVirtualHost 機能を利用して、15個の独自ドメインの web サーバーを 15個、運用しています。これらは古くからの運用なので、いずれも port 80番 だけの、http サーバーでの運用です。あなたと相談したいのは、これら 15個ある httpサーバーの https 化運用です。私が取得している Global IPアドレスは8個（3ビット）で、そのうち2個は 前述のhttps web サーバー 2台専用で使用しています。従って使える Global IPアドレスはあと僅かです。Apache2 のVirtualHost の様な仕組みで、残りの15個 の独自ドメインを 1台のLinux ホストマシンでhttps 化運用する方法があれば教えてください。サーバー用のPCは多数運用しているので複数台のマシンを使用したり、あるいは VM 仮想環境を利用することは可能です。この件で検討する前に必要な情報があれば、指摘をお願いします。

> できます。結論から言うと、**HTTPS でも Apache の名前ベース VirtualHost（1台・1つのGlobal IP）で 15ドメインを同居運用** できます。
鍵になるのは SNI（Server Name Indication） です。いまの主要ブラウザ／OS はほぼ全て SNI 対応なので、**「1つのIP + 443番」** で複数ドメインをTLS終端できます。

---

## 方式の選択肢（現実的な順）

### A. 1台のApacheで 15ドメインをそのままHTTPS化（おすすめ）

- 1台のLinuxサーバに 443/tcp を開ける
- Apache の :443 に ドメインごとの <VirtualHost *:443> を作る（今の :80 と同じ発想）
- 証明書は ドメインごとに Let’s Encrypt（または後述のSAN/ワイルドカード）

**長所:** 構成がシンプル、追加IP不要  
**注意:** 古いクライアント（超古いAndroid等）でSNI非対応だと該当ドメインに繋がらない（現代のWeb用途なら多くの場合無視できる）

### B. フロントにリバースプロキシ（nginx / HAProxy）を置いて、裏にApache/VMをぶら下げる

- 公開側は1台（または冗長化）で **TLS終端だけ担当**
- 裏側はVM/別マシンに http のまま配置しても良い

**長所:** 既存の各VM/既存Apache構成を崩しにくい、段階移行しやすい  
**短所:** 1段増える（ログ/ヘッダ/ACL等の調整が必要）

### C. ワイルドカード証明書（DNS-01必須）で運用を簡単化

- *.example.com みたいに「同一親ドメイン配下」が多いなら強い
- ただし DNS-01（TXTレコード）での認証が必要（BIND運用なら自動化もしやすい）

---
## A方式（Apache単体）の実装イメージ

### 1) Apache側の前提

- mod_ssl 有効化
- 443開放（FW/NAT含む）
- ServerName が正しく引ける（DNSは既に運用されているのでOKのはず）

Debian/Ubuntu系だと典型は：
```
sudo a2enmod ssl headers http2 rewrite
sudo a2ensite default-ssl   # 使うなら
sudo systemctl reload apache2
```

### 2) 80番は「証明書更新」と「HTTPSリダイレクト」に使う（推奨）

Let’s Encrypt の HTTP-01 を使うなら **80番は残す** のが運用楽です（常時公開がイヤならDNS-01へ）。

/etc/apache2/sites-available/example.com.conf（例）

```apache
<VirtualHost *:80>
  ServerName example.com
  ServerAlias www.example.com
  DocumentRoot /var/www/example.com

  # まずは動作確認後に最終的にHTTPSへリダイレクト
  RewriteEngine On
  RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
  RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>
```

### 3) 443番はドメインごとに VirtualHost

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName example.com
  ServerAlias www.example.com
  DocumentRoot /var/www/example.com

  SSLEngine on
  SSLCertificateFile      /etc/letsencrypt/live/example.com/fullchain.pem
  SSLCertificateKeyFile   /etc/letsencrypt/live/example.com/privkey.pem

  # セキュリティ・基本
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"  # 本当に問題ないのを確認してから推奨
  SSLUseStapling on

  # 必要ならHTTP/2（問題が出る環境も稀にあるので段階導入でもOK）
  Protocols h2 http/1.1

  ErrorLog ${APACHE_LOG_DIR}/example.com-ssl-error.log
  CustomLog ${APACHE_LOG_DIR}/example.com-ssl-access.log combined
</VirtualHost>
</IfModule>
```

---
## Let’s Encrypt 証明書の取り方（代表例）

### 一番ラク：certbot + Apacheプラグイン

- Debian/Ubuntuならだいたいこれが早いです。

```bash
sudo certbot --apache -d example.com -d www.example.com
```

15ドメイン分まとめてやるなら、ドメインを並べてOK（ただし長くなる）：


```bash
sudo certbot --apache \
  -d a.com -d www.a.com \
  -d b.com -d www.b.com \
  -d c.com -d www.c.com
```

更新は通常タイマーで自動化されます（certbot renew）。更新後Apache reloadが必要な場合はフックで：

```bash
sudo certbot renew --deploy-hook "systemctl reload apache2"

```

### DNS-01（80/443を触りたくない・閉じたい場合）

- BIND運用なら、TSIG + nsupdate で自動化できる可能性が高いです。
- この場合は certbot より acme.sh を使う人も多いです（手元DNSの自動更新がやりやすい）。
---

## 運用上の注意点（ハマりどころ）

- **SNI非対応クライアント**：ごく古い端末向けサービスが含まれるなら、例外ドメインだけ専用IPにする等を検討。
- **HSTS**：一度入れると戻しづらいので、全サイトが安定してから（または短い max-age から）。
- **証明書の方式**
  - ドメインごとに個別証明書：分かりやすい・独立性高い
  - SAN（1枚に複数FQDN）：枚数が減るが、更新/失効の影響範囲が広くなる
  - ワイルドカード：DNS-01が必須、同一ゾーン配下が多いと超便利
- **ログ**：VirtualHostごとにログ分割すると後が楽。
- **HTTP→HTTPS**：リダイレクトは .well-known/acme-challenge/ を除外（HTTP-01運用時）。
---

### 検討前に “あると一気に詰められる情報”

質問はこのまま進められます。その上で、設計を最短で確定させるのに効くのは次です（分かる範囲でOK）：

**1. Apacheのバージョン / OS（Ubuntu/Debian系か等）**  
**2.** 15ドメインのうち
  - それぞれ独立ドメイン（例: a.com, b.net…）が中心？
  - それとも **xxx.example.com** のようなサブドメイン中心？
3. 各ドメインのDNSは **あなたのBINDで権威運用** ですか？（TXT追加を自動化できるかが変わる）
4. 443番は現状どこかで使用中ですか？（NAT/FW含む）
5. 既存2台のHTTPS専用サーバは「分離が必須」なのか（決済系/要件/負荷）、「統合してもOK」なのか
---
もし「15個は全部独立ドメインで、BINDも自前、ApacheはUbuntu」みたいな状況なら、**A方式（Apache単体 + certbot）で即日運用** が一番スムーズだと思います。
上の5点のうち分かるところだけ教えてくれたら、あなたの現状に合わせて **推奨構成（個別証明書/SAN/ワイルドカード）と具体コマンド、VirtualHost雛形、更新フック、移行手順（段階リリース）** まで一気に書きます。

< ありがとう。まさしく知りたかった回答です。certbot で更新管理が出来るというのも、望んでいました。SNI非対応クライアントは今回無視してください。証明書は、ドメインごとに個別証明書を望みます。ログもドメインごとに分割します。> 1. Apacheのバージョン今回の https 化をきっかけに、新規に一台専用の Web サーバーを Ubuntu 24.03 （実体は Windows 10 Hyper-V のゲストOS）で立ち上げる予定です。Apacheのバージョンは、Ubuntu 24.03 同梱のものとなります。> 2. 15ドメインのうち 15ドメイン は全て、a.com, b.jp の様な独立ドメインです。> 3. ドメインのDNSは 全て私のBINDで権威運用ですかこの相談に出てくる全てのドメインは、現在所有の2台のDNS で管理していて、全て私が管理者です。2台とも Ubuntu 22.04.5 で運用中ですが、今回のWebサーバー運用変更を機会に、Ubuntu 24.03 に更新します。その更新した Ubuntu 24.03 の BIND を利用する予定です。> 4. 443番は現状どこかで使用中ですか？自由に使える Global IP アドレスが残り3個あるので、そのうち1個を今回、これら移行の15台専用に割り当てる予定です。この使用するIPアドレスには、NAT/FW などはありません。> 5. 既存2台のHTTPS専用サーバは「分離が必須」なのか（決済系/要件/負荷）、「統合してもOK」なのかこられは現在オンラインショップ決済と、https ブログに使用しているので、当面は別IPアドレスで、分離運用の予定です。統合は考えていません。それではよろしくお願いします。

> ありがとうございます。状況、とてもよく分かりました。
結論としては、あなたの構成は **「Apache単体 + SNI + certbot + 個別証明書」** の王道パターンにぴったりで、Ubuntu 24.04（※表記は 24.04 LTS だと思います）であれば、特別な回避策は一切不要です。

> 以下は、あなたの前提（自前BIND／独立15ドメイン／専用IP1個／個別証明書／ログ分離）に最適化した、**ほぼそのまま実運用に入れる設計＋移行手順** です。
---

## 全体構成（確定案）

- 新サーバー：Ubuntu 24.04 LTS + Apache 2.4.x
- IP：新しい Global IP 1個
- ポート：80, 443
- 方式：Apache name-based VirtualHost + SNI
- 証明書：Let's Encrypt（certbot）
- 証明書単位：ドメインごとに個別
- 認証方式：まず HTTP-01（将来DNS-01へも容易に移行可）
- DNS：あなたのBINDでAレコード切替
- ログ：VirtualHostごとに完全分離

## 構築ステップ（実運用レベル）
---

### ① OS / Apache / certbot 準備**

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y apache2 certbot python3-certbot-apache
```

Apache確認：

```bash
apache2 -v
# Apache/2.4.58 (Ubuntu) など
```

必要モジュール：

```bash
sudo a2enmod ssl headers rewrite http2
sudo systemctl restart apache2
```

必要モジュール：
```bash
sudo a2enmod ssl headers rewrite http2
sudo systemctl restart apache2
```

FWがあるなら：
```bash
sudo ufw allow 80
sudo ufw allow 443
```

---

### ② ディレクトリ設計（おすすめ）**  
例：

```css
/var/www/
   ├── a.com/
   │     └── public_html
   ├── b.jp/
   │     └── public_html
```

```bash
sudo mkdir -p /var/www/a.com/public_html
sudo mkdir -p /var/www/b.jp/public_html

sudo chown -R www-data:www-data /var/www
sudo chmod -R 755 /var/www
```

---
### ③ HTTP VirtualHost（まずは80番だけ）

/etc/apache2/sites-available/a.com.conf

```apache
<VirtualHost *:80>
    ServerName a.com
    ServerAlias www.a.com
    DocumentRoot /var/www/a.com/public_html

    ErrorLog ${APACHE_LOG_DIR}/a.com-error.log
    CustomLog ${APACHE_LOG_DIR}/a.com-access.log combined
</VirtualHost>
```
有効化：

```bash
sudo a2ensite a.com.conf
sudo systemctl reload apache2
```

この状態で 

http://a.com が 新サーバーに向くようDNSを一時的に切替して疎通確認。

---
### ④ certbotで証明書取得（Apache連携）

```bash
sudo certbot --apache -d a.com -d www.a.com
```

成功すると自動で：
- 443 VirtualHost 作成
- 証明書パス設定
- 80→443リダイレクト設定（聞かれる）

が入ります。

生成される典型構成：

```apache
<VirtualHost *:443>
    ServerName a.com
    ServerAlias www.a.com
    DocumentRoot /var/www/a.com/public_html

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/a.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/a.com/privkey.pem

    ErrorLog ${APACHE_LOG_DIR}/a.com-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/a.com-ssl-access.log combined
</VirtualHost>
```
これを15ドメイン分、同じ要領で作成。

---

### ⑤ HTTPS強制（acmeを除外）

certbot任せでもOKですが、明示する場合：

```apache
<VirtualHost *:80>
    ServerName a.com
    ServerAlias www.a.com

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>
```
---
### ⑥ 証明書の自動更新確認


```bash
sudo systemctl list-timers | grep certbot
```
テスト：

```bash
sudo certbot renew --dry-run
```

必要なら Apache reload hook：
```bash
sudo certbot renew --deploy-hook "systemctl reload apache2"
```

---

### 既存環境からの安全な移行手順

あなたの運用経験を前提に、事故が起きにくいやり方です。

✔ Step 1

新サーバー構築
→ /etc/hosts で一部ドメインだけローカル検証

✔ Step 2

DNSを1ドメインずつ切替
→ HTTP確認 → certbot → HTTPS化

✔ Step 3

全HTTPS動作確認
→ 旧サーバーは HTTP 301で新サーバーへ転送（SEO・ブックマーク保護）

✔ Step 4

数週間安定後、旧HTTPサーバー停止

---

## SSL/TLS 運用品質向上（推奨）

### Mozilla推奨設定（中庸）

```bash
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLOpenSSLConfCmd DHParameters "/etc/ssl/certs/dhparam.pem"
SSLHonorCipherOrder on
```

- #### HSTS（慎重に）

```apache
Header always set Strict-Transport-Security "max-age=31536000"
```
（最初は max-age=86400 推奨）

---

## あなたの構成で“非常に相性が良い”将来拡張

あなたはBINDも完全管理されているので、将来：

- ✅ ワイルドカード証明書
- ✅ 80番完全閉鎖（DNS-01）
- ✅ acme.sh + nsupdate による完全自動化
- ✅ BIND更新も Ansible 化

などがすぐ可能です。

---

## まとめ（あなたのケースの最適解）

✔ 1IPで15ドメインHTTPS可能  
✔ Apacheの通常VirtualHostと全く同じ感覚  
✔ certbotで完全自動更新  
✔ ログも証明書も完全分離  
✔ 既存HTTPSサーバーと共存可能  
✔ 自前DNS運用はむしろ有利  

もしご希望があれば

- certbot実行順序テンプレ
- VirtualHost雛形一括生成スクリプト
- BIND側のDNS-01自動化例
- 大量ドメイン向けApacheディレクトリ設計
- HTTPS移行チェックリスト（実務用）

も書きます。

---
この構成、正直に言って「かなり理想的な自前HTTPS運用環境」です。
ここまで揃っている個人運用はなかなかありません🙂

実装フェーズでログやconfを貼っていただければ、レビューもします。


