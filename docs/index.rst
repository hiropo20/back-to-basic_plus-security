実施環境
========

-  事前にラボ環境へのInviteを行っておりますので、メールをご確認ください
-  利用するコマンド： kubectl git , jq , sudo, curl
-  NGINX Trialライセンスの取得、ラボ実施ユーザのHome Directoryへ配置

座学資料
========

このラボはNGINX Plusのインストールから各種設定を行っていただけます。

NGINX Plusの基本的な動作や仕様について紹介しております。

| 基本的な解説資料は以下を参照してください。
| (このラボはこのセミナーでご紹介した内容を一部抜粋しております)

セミナー資料
-----
以下の資料を参考にご覧いただけます。一部内容を変更してラボの内容としております


`これから始めるNGINX技術解説～セキュリティ編 <https://www.slideshare.net/Nginx/nginx-back-to-basics-part-3-security-japanese-version>`__



Webinar(プレゼンテーション・デモ)
-----

`これから始めるNGINX技術解説～セキュリティ編 <https://www.nginx.co.jp/resources/webinars/nginx-back-to-basic-3-security-jp//>`__



ラボ環境 (UDF(Unified Demonstration Framework)) コンポーネントへの接続
======================================================================

| 弊社が提供するLAB環境を使って動作を確認いただきます。
| ラボ環境を起動する等、一部ブラウザを使って操作します。
| Google
  ChromeがSupportブラウザとなります。その他ブラウザでは正しく動作しない場合があることご了承ください。
| 参照：\ `UDF Supported Browsers and
  Clients <https://help.udf.f5.com/en/articles/3470266-supported-browsers-and-clients>`__


Windows Jump HostへのRDP接続
----------------------------


Windows Jump HostからCLIの操作を行う場合、以下タブからRDP Clientファイルをダウンロードいただき接続ください

   .. image:: ./media/udf_jumpbox.png
      :width: 200

.. NOTE::
   | RDPのUser名、パスワードはDETAILSをクリックし、GeneralのタブのCredentialsの項目を参照ください
   | ``user`` でログインしてください 

   - .. image:: ./media/udf_jumpbox_loginuser.png
       :width: 200
    
   - .. image:: ./media/udf_jumpbox_loginuser2.png
       :width: 200
   
Windows Jump Hostへログインいただくと、SSH
Clientのショートカットがありますので、そちらをダブルクリックし
``ubuntu01`` へ接続ください

   - .. image:: ./media/putty_icon.jpg
      :width: 50

   - .. image:: ./media/putty_menu.jpg
      :width: 200


Linux Hostへの接続 (Jump Host を利用しない場合)
-----------------------------------------------


| ``ubuntu01`` へのSSH接続は、Jump Host経由
または、SSH鍵認証を用いて接続可能です。SSH鍵の登録手順は以下を参照ください
| **SSH鍵を登録頂いていない場合、SSHはグレーアウトします** `UDF LAB SSH鍵登録マニュアル <https://github.com/hiropo20/partner_nap_workshop_secure/blob/main/UDF_SSH_Key.pdf>`_
| (こちらの手順が必要となる場合、本マニュアルを閲覧可に変更します)


NGINX Plus のインストール
====

1. NGINX Plus、アドオンモジュールのインストール (15min)
----

こちらの手順「 `NGINX Plusのインストール (15min) <https://f5j-nginx-plus-lab1.readthedocs.io/en/latest/#nginx-plus-15min>`__」を参考に、NGINX Plus、アドオンモジュールをインストールをインストールしてください

手順の内容に加えて、必要となるモジュールをインストールしてください

::

   sudo apt-get install nginx-plus-module-njs


インストールしたパッケージの情報を確認いただけます


::

   # dpkg-query -l | grep njs
   
   ii  nginx-plus-module-njs              25+0.7.0-1~focal                      amd64        NGINX Plus njs dynamic modules

ラボの実施（作成中）
====

必要なパッケージの取得

::

   git clone https://github.com/hiropo20/back-to-basic_plus-security.git


(参考) バックエンドのアプリケーション＆ELKのデプロイ手順
====

ELKのセットアップ
----

| ページに記載する手順に従ってELKをセットアップします
| 参考：\ `NGINX App Protect ELK Dashboard <https://github.com/f5devcentral/nap-dos-elk-dashboards>`__


.. NOTE::
   同時に複数の環境からDocker Imageを取得する場合、Registory側の仕様によりImageの取得が失敗する場合があります。
   その場合には一旦時間を開けて実行していただくか、別のタイミングでローカルのレジストリに登録するなどの対応を実施ください。

UDF環境では ``docker_host`` にログインし手順を実行します

必要なパッケージの取得

::

   git clone https://github.com/f5devcentral/f5-waf-elk-dashboards.git
   git clone https://github.com/f5devcentral/nap-dos-elk-dashboards.git


取得したファイルの確認

::

   $ ls -lrt
   total 0
   drwxrwxr-x. 7 centos centos 155 Jan  6 01:29 f5-waf-elk-dashboards
   drwxrwxr-x. 6 centos centos 195 Jan  6 01:29 nap-dos-elk-dashboards


NAP DoSで利用するLogstashの設定ファイルをNAP WAFのディレクトリへコピー

::

   cp nap-dos-elk-dashboards/logstash/conf.d/apdos-logstash.conf f5-waf-elk-dashboards/logstash/conf.d/


以下のコマンドを実行しファイルを作成

::

   cat << EOF > f5-waf-elk-dashboards/logstash/pipelines.yml
   - pipeline.id: napwaf
     path.config: "/opt/logstash/config/30-waf-logs-full-logstash.conf"

   - pipeline.id: napdos
     path.config: "/opt/logstash/config/apdos-logstash.conf"
   EOF


| 今回のサンプルで利用するELKでは複数のPiplineを利用するため、追加のSyslog Portが必要になります。
| 以下の通り ``docker-compose.yaml`` ファイルの内容を修正します

::

   cp f5-waf-elk-dashboards/docker-compose.yaml f5-waf-elk-dashboards/docker-compose.yaml-bak
   cat << EOF > f5-waf-elk-dashboards/docker-compose.yaml
   version: "2.4"
   services:
     elasticsearch:
      image: sebp/elk:793
      restart: always
      volumes:
         - ./logstash/pipelines.yml:/opt/logstash/config/pipelines.yml:ro
         - ./logstash/conf.d/30-waf-logs-full-logstash.conf:/opt/logstash/config/30-waf-logs-full-logstash.conf:ro
         - ./logstash/conf.d/apdos-logstash.conf:/opt/logstash/config/apdos-logstash.conf:ro
         - elk:/var/lib/elasticsearch
      ports:
         - 9200:9200/tcp
         - 5601:5601/tcp
         - 5144:5144/tcp
         - 5261:5261/tcp
         - 5561:5561/udp
   volumes:
     elk:
   EOF

変更内容の確認

::

   diff -u f5-waf-elk-dashboards/docker-compose.yaml-bak f5-waf-elk-dashboards/docker-compose.yaml


ELKの実行

::

   cd f5-waf-elk-dashboards
   docker-compose -f docker-compose.yaml up -d

以下が出力されることを確認する

::

   ※ docker-compose の出力結果
   Creating f5-waf-elk-dashboards_elasticsearch_1 ... done

   $ docker ps
   CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                                                                                                                                                                                                                                 NAMES
   3b5bb60d2d35   sebp/elk:793   "/usr/local/bin/star…"   3 minutes ago   Up 2 minutes   0.0.0.0:5144->5144/tcp, :::5144->5144/tcp, 0.0.0.0:5261->5261/tcp, :::5261->5261/tcp, 0.0.0.0:5601->5601/tcp, :::5601->5601/tcp, 5044/tcp, 9300/tcp, 9600/tcp, 0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:5561->5561/udp, :::5561->5561/udp   f5-waf-elk-dashboards_elasticsearch_1


起動したELKのコンテナでbashを開く

::
   docker exec -it f5-waf-elk-dashboards_elasticsearch_1 /bin/bash
   
   root@3b5bb60d2d35:/#

Pluginを設定する(ELKのbash上で行う)

::
   # logstash の停止
   service logstash stop
   # logstash pluginのinstall
   /opt/logstash/bin/logstash-plugin install logstash-output-syslog
   /opt/logstash/bin/logstash-plugin install logstash-input-syslog
   /opt/logstash/bin/logstash-plugin install logstash-input-tcp
   /opt/logstash/bin/logstash-plugin install logstash-input-udp

   ※ 各インストールコマンドの最後に Installation successful が表示されることを確認してください

logstashの設定ファイルが配置されていることを確認します。

::
   cat /etc/logstash/conf.d/apdos-logstash.conf

ファイルが存在しない場合、一度コンテナのbashから抜け、ターミナルからファイルを読み込みます
その他エラーについては `こちらの手順を参照してください <https://github.com/f5devcentral/nap-dos-elk-dashboards#deploying-elk-stack>`__

::
   # コンテナのbashから抜ける
   root@3b5bb60d2d35:/# exit

   # host上で以下コマンドを実行
   cd ~/nap-dos-elk-dashboards
   ls | grep apdos_mapping.json
   curl -XPUT "http://localhost:9200/app-protect-dos-logs"  -H "Content-Type: application/json" -d  @apdos_mapping.json
   ※実行結果サンプル
   {"acknowledged":true,"shards_acknowledged":true,"index":"app-protect-dos-logs"}[centos@ip-10-1-1-5 nap-dos-elk-dashboards]$


正しく追加されたことを確認

::
   # cd ~/nap-dos-elk-dashboards
   curl -s -XGET "http://localhost:9200/_cat/indices" | grep app-protect
   ※実行結果サンプル
   yellow open app-protect-dos-logs           Gqkh0O2VSVuRFBkbCzuJUA 1 1 0   0    208b    208b

Geo Fieldの更新

::
   # cd ~/nap-dos-elk-dashboards
   curl -XPOST "http://localhost:9200/app-protect-dos-logs/_mapping"  -H "Content-Type: application/json" -d  @apdos_geo_mapping.json

App Protect DoS の DashboardをImport

::
   # cd ~/nap-dos-elk-dashboards
   KIBANA_CONTAINER_URL=http://localhost:5601
   jq -s . kibana/apdos-dashboard.ndjson | jq '{"objects": . }' | \
    curl -k --location --request POST "$KIBANA_CONTAINER_URL/api/kibana/dashboards/import" \
        --header 'kbn-xsrf: true' \
        --header 'Content-Type: text/plain' -d @- \
        | jq

App Protect WAF のDashboardをImport

::
   cd ~/f5-waf-elk-dashboards
   jq -s . kibana/false-positives-dashboards.ndjson | jq '{"objects": . }' | curl -k --location --request POST "$KIBANA_CONTAINER_URL/api/kibana/dashboards/import"     --header 'kbn-xsrf: true'     --header 'Content-Type: text/plain' -d @-     | jq
   jq -s . kibana/overview-dashboard.ndjson | jq '{"objects": . }' | curl -k --location --request POST "$KIBANA_CONTAINER_URL/api/kibana/dashboards/import"     --header 'kbn-xsrf: true'     --header 'Content-Type: text/plain' -d @-     | jq

再度ELKのbashを開く

::
   docker exec -it f5-waf-elk-dashboards_elasticsearch_1 /bin/bash

logstashを起動

::
   # 起動
   service logstash start

   ※出力結果サンプル
   logstash started.

   # 確認
   service logstash status

   ※出力結果サンプル
   logstash is running


ELKは起動に時間がかかります。以下のコマンドを実行し想定した結果となることを確認します。

.. NOTE::

   $ docker exec -it  $(docker ps -a -f name=f5-waf-elk-dashboards_elasticsearch_1  -q) ps -aef
   UID        PID  PPID  C STIME TTY          TIME CMD
   root         1     0  0 01:48 ?        00:00:00 /bin/bash /usr/local/bin/start.s
   root        13     1  0 01:48 ?        00:00:00 /usr/sbin/cron
   elastic+   191     1  1 01:48 ?        00:01:48 /opt/elasticsearch/jdk/bin/java
   elastic+   215   191  0 01:48 ?        00:00:00 /opt/elasticsearch/modules/x-pac
   logstash   305     1  2 01:48 ?        00:02:13 /usr/bin/java -Xms1g -Xmx1g -XX:
   kibana     327     1  1 01:48 ?        00:01:21 /opt/kibana/bin/../node/bin/node
   root       330     1  0 01:48 ?        00:00:00 tail -f /var/log/elasticsearch/e
   root       518     0  0 03:37 pts/0    00:00:00 ps -aef

   $ docker logs  $(docker ps -a -f name=f5-waf-elk-dashboards_elasticsearch_1  -q)| grep running
   [2022-01-06T01:48:49,755][INFO ][logstash.agent           ] Pipelines running {:count=>2, :running_pipelines=>[:napdos, :napwaf], :non_running_pipelines=>[]}
   {"type":"log","@timestamp":"2022-01-06T01:49:06Z","tags":["info","http","server","Kibana"],"pid":327,"message":"http server running at http://0.0.0.0:5601"}
   {"type":"log","@timestamp":"2022-01-06T01:49:05Z","tags":["listening","info"],"pid":327,"message":"Server running at http://0.0.0.0:5601"}

   一定時間経過して状況が改善しない場合、再度docker-composeを実行してください
   docker-compose -f docker-compose.yaml down
   docker-compose -f docker-compose.yaml up -d
