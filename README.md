<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [使い方](#使い方)
    - [インベントリの設定](#インベントリの設定)
    - [ansible.cfgの設定](#ansiblecfgの設定)
    - [Ansible Vaultのパスワードファイルの作成と暗号化文字列の更新](#ansible-vaultのパスワードファイルの作成と暗号化文字列の更新)
    - [GitLabとGitLab Runner間の自己署名証明書の問題の回避策](#gitlabとgitlab-runner間の自己署名証明書の問題の回避策)
    - [実行](#実行)
    - [GitLabの初回アクセス時のrootパスワードの変更](#gitlabの初回アクセス時のrootパスワードの変更)
    - [Towerでの自己署名証明書の許可](#towerでの自己署名証明書の許可)
    - [GitLab Runnerのインストール](#gitlab-runnerのインストール)

<!-- markdown-toc end -->



使い方
=========


インベントリの設定
------------

GitLab用にCentOS 7サーバーが1台、
Ansible Tower（以下単に"Tower"と呼ぶ）用にRHEL 8サーバーが1台、
それぞれ必要です。

`hosts.example`を各自コピーし編集して使います。
カレントディレクトリに`./hosts`としてコピーし
同ディレクトリ内で`ansible-playbook`コマンドを実行するのであれば、
同ディレクトリのansible.cfgが読み込まれてデフォルトのインベントリとみなされ
`-i`オプションの指定が省略できます。

他に、`~/.ansible/hosts`にコピーすれば、
実行ディレクトリに拘わらず`-i`オプションを省略できます。


ansible.cfgの設定
--------------

`ansible-playbook`コマンドを実行するディレクトリに`ansible.cfg`があれば
それが読み込まれます。

カレントディレクトリに同ファイルがなければ`~/.ansible.cfg`が、
それもなければ`/etc/ansible/ansible.cfg`が使われます。


Ansible Vaultのパスワードファイルの作成と暗号化文字列の更新
--------------

現在、以下の情報をAnsible Vaultを使って暗号化して保存しています。

- RHELのサブスクリプションのパスワード
- Towerインストール時の各パスワード3つ（Tower, PostgreSQL, RabbitMQ）
- TowerのライセンスのJSONオブジェクト

これにより秘密を知られることなくGitリポジトリで保存・公開できます。

Ansible Vaultでの暗号化や復号化には別のパスワードが必要で、
通常は必要になるたびに一時ファイルに保存してそれを使用します。
(Towerでの実行時にはTower内に保存できる。)

以下の例では`/tmp/my-password.txt`にパスワードが保存されているとします。
ファイル中の最後に改行が含まれないよう、以下のようなコマンドで作成します。

```
echo -n '<your_password>' > /tmp/my-password.txt
```

ただし、この場合はシェルの履歴にパスワードが残る可能性に注意してください。

別の方法として、`vi`で編集し最後に付いてしまう改行を`tr -d '\n'`のフィルターを
用いて実行時に削除するというのがあります。

現在ファイルに保存されている暗号化文字列は私自身のものですので、
別の使用者は自身で暗号化文字列を更新して保存する必要があります。

ファイル`/tmp/my-value.txt`に暗号化したい元の文字列があるとして、
以下のコマンドで標準出力に暗号化文字列を表示できますので、
これをコピペします。


```
cat /tmp/my-value.txt | tr -d '\n' | \
  ansible-vault encrypt_string --vault-password-file /tmp/my-password.txt \
  --stdin-name subscription_password
```

最後の`subscription_passowrd`は変数名で、適宜変更して下さい。

実際に変更が必要なファイルと変数のリストは以下の通りです。
"*"を付けたものは暗号化が必要です。

- roles/common/vars/RedHat_8.yml
  - subscription_username
  - subscription_password *
  - subscription_pool_id
- roles/tower/defaults/main.yml
  - admin_password *
  - pg_password *
  - rabbitmq_password *
  - tower_license *

最後の`tower_lisense`にはTowerのライセンスのJSONオブジェクトを格納します。
このとき、下記のように`"eula_accepted": true`が含まれていないと
ライセンス登録時にエラーになりますので、無い場合は追加してください。


```
{
  ...
  "license_key": "...",
  "eula_accepted": true,
  ...
}
```

GitLabとGitLab Runner間の自己署名証明書の問題の回避策
----------------

後でGitLab Runnerをインストールするときに、
GitLab Runnerが自分をGitLabの証明書を正しく検証できないとエラーになります。
これを回避するために、GitLabの自己署名証明書をGitLab Runner側にコピーする
作業を（後で）行います。
それに備えて、GitLabの証明書のサブジェクトが正しいものになるように
deploy_gitlab.ymlで下記の変数を定義しています。

    gitlab_external_url: "https://{{ inventory_hostname }}/"
    gitlab_self_signed_cert_subj: "/C=JP/ST=Tokyo/L=Shibuya/O=Consulting/CN={{ inventory_hostname }}"

これらの変数を未定義のままGitLabをインストールした場合は、
再インストールするか、下記のタスクで行っているように`openssl`コマンドで再作成し配置し直します。

https://github.com/geerlingguy/ansible-role-gitlab/blob/2.5.2/tasks/main.yml#L62


実行
----------------

GitLabにはAnsible Galaxyにある
[geerlingguy](https://galaxy.ansible.com/geerlingguy/gitlab)
を再利用しており、まず以下のコマンドでロールをダウンロードします。
(Towerでの実行時には自動でダウンロードされる。)

```
ansible-galaxy role install -r roles/requirements.yml -p roles
```

その後、以下のコマンドで全体をデプロイします。

```
ansible-playbook [-i /your/hosts] --vault-password-file /tmp/my-password.txt site.yml
```

`-i`オプションによるインベントリの指定は前述のように省略可能です。

`site.yml`では2つのPlaybookをインポートしているだけで、両者は独立しているので、
それぞれを別のターミナルで並列に実行することもできます。


```
# ターミナル1
ansible-playbook deploy_gitlab.yml
# ターミナル2
ansible-playbook --vault-password-file /tmp/mypassword.txt deploy_tower.yml

```

GitLabの場合は、OSがCentOSならば暗号化文字列は使わないため
パスワードファイルの指定は要りません。

何度か繰り返し実行する場合は、`-t`や`--skip-tags`オプションを用いて
タグのついた特定部分のみを実行するのが便利です。



GitLabの初回アクセス時のrootパスワードの変更
----------------

GitLabの管理者のパスワードはファイルで指定するのではなく、
`root/5iveL!fe`で固定で、ブラウザでの初回アクセス時に任意のものに変更します。

なお、Towerの管理ユーザは`admin`、
パスワードは前述の`admin_password`変数でしていしたものです。


Towerでの自己署名証明書の許可
-----------------

GitLabが自己署名証明書を用いているため、そのままではTowerがそこから
リポジトリをクローンするときにエラーになってしまいます。
これを回避するために、"SETTINGS / JOBS / EXTRA ENVIRONMENT VARIABLES"
に以下の設定を行います。

```
{
 "GIT_SSL_NO_VERIFY": "True"
}
```

GitLab Runnerのインストール
------------------

GitLabでCI/CDを行うには、GitLab本体とは別にGitLab Runnerをインストールします。
GitLab Runnerではビルド中に危険な操作（`rm -rf /`など）も行い得るため、
GitLab本体とは別のサーバにインストールすることが強く推奨されています。

ここではGalaxy Hubにあるriemers.ansible-gitlab-runnerを使います。
GitLabの画面上でRegistration Tokenを取得する必要があるため、
一旦GitLabのインストールを終えて、その後GitLab Runnerをインストールします。

GitLabにrootでログインし、下記のURLにアクセスしてURLとRegistration Tokenをメモします。

https://gitlab.example.com/admin/runners


また、GitLabの自己署名証明書をGitLab Runnerにあらかじめコピーしておきます。

```
scp root@gitlab.example.com:/etc/gitlab/ssl/gitlab.crt \
    root@gitlab-runner.example.com:/etc/gitlab-runner/certs/gitlab.example.com.crt
```

（これに相当する処理もdeploy_gitlab-runner.ymlに書いてあるが、
うまく動かなかったので現在はコメントアウトしている。）

さらに、このPlaybookではREADME.mdの例と同様のDocker Excecutorの設定をそのまま行っていますが、
エラーが発生するためIssue #59で報告されている回避策を適用します。

https://github.com/riemers/ansible-gitlab-runner

https://github.com/riemers/ansible-gitlab-runner/issues/59#issuecomment-531559554

その後以下のようにURLとRegistration Tokenを指定してdeploy_gitlab-runner.ymlを実行します。

```
ansible-playbook deploy_gitlab-runner.yml \
  -e 'gitlab_runner_coordinator_url=<GitLabのURL>'
  -e 'gitlab_runner_registration_token=<Registration_Token>'
```
