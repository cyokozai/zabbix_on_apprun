# Zabbix Web/Server を AppRun に建てて宅内の VM の死活監視を行う

## アーキテクチャ

```mermaid
flowchart TD

    subgraph SakuraCloud["さくらのクラウド"]
        subgraph AppRun["AppRun (サーバレス環境)"]
            ZabbixServer["Zabbix Server (Dockerコンテナ)"]
            ZabbixWeb["Zabbix Web (Frontend)"]
        end

        subgraph EnhancedDB["エンハンスドDB (MariaDB)"]
            ZabbixDB["DB: zabbix"]
        end
    end

    subgraph OnPrem["オンプレミス環境"]
        Agent1["Zabbix Agent 1"]
        Agent2["Zabbix Agent 2"]
        AgentN["..."]
    end

    %% Connections
    Agent1 -->|監視データ送信| ZabbixServer
    Agent2 -->|監視データ送信| ZabbixServer
    AgentN -->|監視データ送信| ZabbixServer

    ZabbixServer -->|"SQL接続: 3306/TCP+SSL"| ZabbixDB
    ZabbixWeb -->|"API/SQL利用"| ZabbixDB
    ZabbixWeb -->|"UIアクセス: HTTPS"| User["管理者"]
```

## 技術スタック

| 分類           | 技術・サービス                 | 用途・役割                                  | 備考                  |
| ----------- | ---------------------- | ------------------------------------- | ------------------ |
| **クラウド基盤**   | さくらのクラウド AppRun         | サーバレス環境で Zabbix Server / Web を実行       | コンテナベースで運用可能        |
| **DB**       | エンハンスドデータベース (MariaDB)  | Zabbix のデータ永続化                         | 作成済み DB: `zabbix`   |
| **監視サーバ**    | Zabbix Server (Docker)  | オンプレミスの死活監視・メトリクス収集                    | 公式イメージ使用            |
| **監視フロント**   | Zabbix Web (Docker)     | 管理者向け GUI / ダッシュボード                    | Nginx + PHP + DB接続  |
| **監視対象**     | Zabbix Agent            | 各オンプレミスサーバの死活監視・メトリクス送信                | エージェントをインストールする必要あり |
| **通信**       | SSL / TCP               | Agent ↔ Zabbix Server 間、Server ↔ DB 接続 | セキュア接続を前提           |
| **DBクライアント** | MariaDB クライアント / Docker | 接続確認・スキーマ投入                            | ローカル macOS からの接続に使用 |
| **バックアップ**   | オプション: Object Storage   | DBや設定のバックアップ保管                         | PoCでは未導入でも可         |

## セットアップ

- [Download](https://www.zabbix.com/jp/download)

1. Become root user

    Start new shell session with root privileges.

    ```shell
    sudo -s
    ```

2. Install Zabbix repository

    ``` shell
    wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb
    dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb
    apt update
    ```

3. Install Zabbix agent 2

    Install zabbix-agent2 package.

    ```shell
    apt install zabbix-agent2
    ```

4. Install Zabbix agent 2 plugins

    You may want to install Zabbix agent 2 plugins.

    ```shell
    apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
    ```

5. Start Zabbix agent 2 process

    Start Zabbix agent 2 process and make it start at system boot.

    ```shell
    systemctl restart zabbix-agent2
    systemctl enable zabbix-agent2
    ```
