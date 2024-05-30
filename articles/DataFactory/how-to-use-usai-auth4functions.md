---
title: Azure Functions へのシステム割り当てマネージド ID 認証の設定方法
date: 2024-05-30 09:00:00
tags:
  - Azure
  - Data Factory
  - Azure Functions
  - システム割り当てマネージドID
---

# 目次
- [目次](#目次)
- [概要](#概要)
- [検証環境](#検証環境)
- [手順](#手順)
  - [手順 1 ユーザー割り当てマネージド ID を設定](#手順-1-ユーザー割り当てマネージド-id-を設定)
  - [手順 2 Azure 関数の認証設定を行う](#手順-2-azure-関数の認証設定を行う)
    - [手順 2.1 (未作成の場合) アプリケーションの登録を行う](#手順-21-未作成の場合-アプリケーションの登録を行う)
    - [手順 2.2 認証にてアプリケーションを追加する](#手順-22-認証にてアプリケーションを追加する)
    - [手順 2.3 フェデレーション資格情報の設定](#手順-23-フェデレーション資格情報の設定)
    - [手順 2.4 クライアント シークレットの上書き](#手順-24-クライアント-シークレットの上書き)
  - [手順 3 Azure Data Factory のマネージド ID にロールを付与](#手順-3-azure-data-factory-のマネージド-id-にロールを付与)
  - [手順 4 Azure Data Factory の Azure 関数 アクティビティの設定](#手順-4-azure-data-factory-の-azure-関数-アクティビティの設定)



# 概要
この記事では、システム割り当てマネージド ID を使用して Azure Data Factory から Azure 関数を呼び出す方法について説明します。

# 検証環境
- Azure Data Factory V2 Data Flow
- Azure Functions 

# 手順


## 手順 1 ユーザー割り当てマネージド ID を設定
ユーザー割り当てマネージド ID を、関数アプリの [ID] > [ユーザー割り当て済み] より追加いたします。  
公式ドキュメントの [ユーザー割り当て ID を追加する](https://learn.microsoft.com/ja-jp/azure/app-service/overview-managed-identity?tabs=portal%2Chttp#add-a-user-assigned-identity) も併せてご確認ください。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-0.png)


Azure ポータルより、「エンタープライズ アプリケーション」と検索いただき、
ユーザー割り当てマネージド ID の「オブジェクト ID」 および 「アプリケーション ID」 を後述の設定にて用います。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-1.png)

## 手順 2 Azure 関数の認証設定を行う

### 手順 2.1 (未作成の場合) アプリケーションの登録を行う
Azure ポータルにて、[Microsoft Entra ID] と検索いただき、  
[+ 追加] > [アプリの登録] よりアプリの作成を行います。
名前を記載いただき、特別な理由がなければ規定値のまま進めます。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-2.png)
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-3.png)


作成後に表示される「アプリケーション (クライアント) ID」 「オブジェクト ID」 「ディレクトリ (テナント) ID」をメモします。  
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-4.png)


### 手順 2.2 認証にてアプリケーションを追加する
ご利用の Azure 関数の [認証] > [ID プロバイダーを追加] を開きます。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-5.png)

以下画像のように設定いたします。  
特筆すべき点を以下の表に記載しております。

|  項目  |  値  |
| ---- | ---- |
|  ID プロバイダー  |  Microsoft  |
|  アプリの登録  |  既存アプリの登録の詳細を提供します  |
|  アプリケーション (クライアント) ID  |  2.1 で作成したアプリのアプリケーション (クライアント) ID  |
|  Client application requiremen  |  「Allow requests from any application」もしくは 「Allow requests from specific client applications」 |
|  Allowed client applications (Allow requests from specific client applications を選択した場合) |  Azure Data Factory のマネージド ID のアプリケーション (クライアント) ID |
|  認証されていない要求  |  HTTP 401 認可されていない: API に推奨  |

![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-6.png)
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-7.png)


### 手順 2.3 フェデレーション資格情報の設定
手順 2.1 で作成したアプリケーションの設定を行います。  
Azure ポータルより、[アプリケーションの登録] から該当のアプリケーションを選択します。  
[管理] > [証明書とシークレット] > [フェデレーション資格情報] から、「+ 資格情報の追加」を押下します。
そして、Azure Functions に割り当てたユーザー割り当てマネージド ID を選択します。  

|  項目  |  値  |
| ---- | ---- |
|  フェデレーション資格情報のシナリオ  |  カスタマー マネージド キー  |
|  マネージド ID の選択  |  手順 1 で割り当てたユーザー割り当てマネージド ID  |

![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-8.png)
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-9.png)

PowerShell などから、az rest コマンドを用いて以下のように実行することも可能です。  

```
az rest --method POST --uri "https://graph.microsoft.com/beta/applications/<APP_REGISTRATION_OBJECT_ID>/federatedIdentityCredentials" --body "{'name': 'ManagedIdentityFederation', 'issuer': 'https://login.microsoftonline.com/<TENANT ID>/v2.0', 'subject': '<AZURE_FUNCTION_USER_ASSIGNED_MANAGED_IDENTITY_OBJECT_ID>', 'audiences': [ 'api://AzureADTokenExchange' ]}"
```

### 手順 2.4 クライアント シークレットの上書き
手順 2.2 で作成した ID プロバイダーにおける「クライアント シークレット設定の名前」を、ユーザー割り当てマネージド ID のクライアント ID に置き換えます。  

はじめに、該当の関数アプリより、[環境変数] > [+ 追加] より、以下の設定を行い、[適用] を押下します。  

|  項目  |  値  |
| ---- | ---- |
|  名前  |  OVERRIDE_USE_MI_FIC_ASSERTION_CLIENTID  |
|  値  |  手順 1 で割り当てたユーザー割り当てマネージド ID のクライアント ID |

![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-10.png)

その後、[認証] > [ID プロバイダー] より、該当の ID プロバイダーを編集を行います。  
以下画像の通り、「クライアント シークレット設定の名前」に作成した環境変数を設定します。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-11.png)


## 手順 3 Azure Data Factory のマネージド ID にロールを付与
関数アプリの [アクセス制御 (IAM)] より、ご利用の Azure Data Factory のマネージド ID に対して*「閲覧者」ロール* を付与します。　

## 手順 4 Azure Data Factory の Azure 関数 アクティビティの設定
Azure Data Factory Studio を開き、Azure 関数 アクティビティの設定を行います。  
Azure 関数アクティビティをドラッグアンドドロップした後、[設定] > [Azure 関数のリンク サービス] > [+ 新規] を選択します。
![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-12.png)  

  
接続先の関数アプリを選択いただき、[認証方法] を「システム割り当てマネージド ID」を選択し、[リソース ID] には手順 2.1 および手順 2.2 で登録したアプリケーションのクライアント ID を設定します。
詳細な設定項目につきましては、公式ドキュメントの [Azure Functions のリンクされたサービス](https://learn.microsoft.com/ja-jp/azure/data-factory/control-flow-azure-function-activity#azure-function-linked-service) もご覧ください。

![](.\how-to-use-usai-auth4functions\how-to-use-usai4functions-13.png)  

以上の設定で完了です。