# Service Broker リファレンス アプリを開く

##  概要
すべての [IBM Cloud カタログでのサービスのオンボーディング](https://cloud.ibm.com/docs/sell?topic=sell-overview) では、[オープン サービス ブローカー](https://github) に従うブローカー アプリを構築する必要があります。 .com/openservicebrokerapi/servicebroker/blob/v2.12/spec.md) (OSB) 仕様。このプロジェクトは、クイック セットアップのリファレンス ブローカー実装を提供します。

[パートナー センター](https://cloud.ibm.com/partner-center/sell) を通じてオンボーディングする場合、ブローカー アプリの URL を入力する必要があります。この Readme は、サービスに接続するリファレンス ブローカー アプリを構成およびデプロイする方法に関するステップバイステップ ガイドを提供し、オンボーディング プロセスの初期段階でエンドツーエンド フローをテストするのに役立ちます。これはすべて、次のタスクを実行する CLI メイクファイル自動化の助けを借りて行われます。

 - Maven を使用して Java ベースの OSB ブローカー コードを構築します  
 - OSB ブローカー アプリの Docker コンテナー イメージを構築します。
 - IBM Container Registry (ICR) 名前空間を作成し (存在する場合)、イメージを ICR 名前空間にアップロードします。
 - IBM Code Engineにアプリをデプロイします

> このブローカーは運用環境での使用が推奨されないことに注意してください。本番ブローカーを構築する際の参考として使用してください。

## オンボーディングおよびデプロイメント IBM Cloud アカウントとは何ですか
以下のセクションでは、オンボーディングおよびデプロイメント IBM Cloud アカウントに関する多くの言及が出てきます。このセクションでは、この 2 つについてわかりやすく説明します。

#### *_Onboarding IBM Cloud Account_*
は、サービスがオンボーディングされている IBM Cloud アカウントです。パートナー センターを使用する場合は、クラウド アカウントの選択になります。

> IBM 社員への注意: IBM 社員の場合、**test**.cloud.ibm.com 経由でオンボーディングする可能性が高く、デプロイメント IBM Cloud アカウントは **cloud.ibm.com** である必要があります。

#### _Deployment IBM Cloud Account_ 
コード エンジンを使用してブローカー アプリをデプロイするクラウド アカウントです。Deployment Cloud アカウントは、アカウントがcloud.ibm.com (IBM Cloud Production) 上にある限り、Onboarding Cloud アカウントと同じにすることができます。

> IBM 担当者への注意: デプロイメント クラウド アカウントは **cloud.ibm.com** のアカウントである必要があります**。test.cloud.ibm.com やその他の IBM クラウドのアカウントではありません**。これは、以降のセクションでデプロイされる IBM コンテナー レジストリー、名前空間、および IBM コード エンジン アプリが、cloud.ibm.com 上に存在する必要があることを意味します。

## 前提条件


1. #### [Docker](https://docs.docker.com/engine/install/) をコンピューター上でローカルにセットアップする
2. #### IBM クラウド アクセス
    以下の権限が必要です
    1. サービスがオンボードされている IBM Cloud アカウントに招待される必要があります
    2. デプロイメント IBM Cloud アカウントには次のアクセス権が必要です。
       
        ａ．IBM Container Registry へのライターおよびエディターのアクセス権

        b. IBM Cloud Code Engineへの編集者アクセス
        
        ユーザーを検索し、[こちら](https://cloud.ibm.com/iam/users) で必要なアクセス権があることを確認します。
    3. [グローバル カタログ](https://globalcatalog.cloud.ibm.com) のサービスに追加する必要があります (UI の [可視性] タブ経由)。




## IBM Cloud API キーを作成する


  プロジェクトでは 3 つの IBM Cloud API キーが必要です。2 つについてはこのセクションで説明し、3 つ目については次のセクションで説明します。

  - **ONBOARDING_IAM_API_KEY**

    パートナー センター経由で SaaS サービスがオンボーディングされているクラウド アカウントで作成された API キーです。API キーは、[グローバル カタログ](https://globalcatalog.cloud.ibm.com) API にアクセスするために使用されます。
    
  - **DEPLOYMENT_IAM_API_KEY**

    ブローカーがデプロイされるクラウド・アカウントで作成された IBM クラウド API キーです。API キー (したがって API キーを所有するユーザー) には、デプロイメント IBM Cloud アカウントで次のアクセス権が必要です。
    
      ａ．IBM Container Registry へのライターおよびエディターのアクセス権

      b. IBM Cloud Code Engineへの編集者アクセス
    
    ユーザーを見つけて、[こちら](https://cloud.ibm.com/iam/users) で必要なアクセス権があることを確認してください。

    [この](https://cloud.ibm.com/docs/account?topic=account-userapikey&interface=ui#create_user_key) ドキュメントに従って API キーを作成できます。作成したら、API キーの値をローカル マシン上の安全な場所に保存します。  

  - **METERING_API_KEY**

    Reference Broker は、サービスの BSS にテスト測定データを送信できるダッシュボードを提供します。これを可能にするには、計測サービス ID と API キーを設定する必要があります。  
    [パートナー センター](https://cloud.ibm.com/partner-center/sell) 経由でオンボーディングしている場合は、
    
    - [マイ製品] -> [ダッシュボード] に移動します
    - [サービス ID の作成] ボタンをクリックします。
    - 同じページの「API キーの作成」リンクを使用して、サービス ID の API キーを作成します。
    
    デプロイ手順中に METERING_API_KEY として使用されるため、API キーをメモしておきます。   
    
    

## ブローカーの構築

 ### 1. リポジトリのクローンを作成し、ディレクトリに cd します。

    git clone https://github.com/IBM-Cloud/onboarding-osb.git
    cd onboarding-osb

### 2. [`deploy/build.config.properties`](deploy/build.config.properties) 構成を完了します。
以下の手順に従ってプロパティ ファイルを完成させ、環境変数としてエクスポートします。

   

  - #### ONBOARDING_ENV
    - IBM Cloud Onboarding が test.cloud.ibm.com (IBMer によって使用される) にある場合は「stage」に設定され、IBM Cloud Onboarding がcloud.ibm.com にある場合は「prod」に設定されます。
  - #### GC_OBJECT_ID
    
    - パートナー センターでは、この値は [ブローカー] タブにあります。「ブローカー」タブの「ガイドを開く」ボタンをクリックすると、追加のヘルプが表示されます。
    
  - #### BROKER_ICR_NAMESPACE_URL
    - これは、OSB コンテナー イメージがアップロードされる [Deployment Cloud Account](#deployment-ibm-cloud-account) 内の IBM Container Registry 名前空間です。[こちら](https://cloud.ibm.com/registry/namespaces) で利用可能な名前空間の 1 つを選択できます。
    
      既存の名前空間を使用したくない場合は、一意の名前を指定するだけで、自動化によって名前空間が作成されます。手動で作成する場合は、[これらの](https://cloud.ibm.com/docs/Registry?topic=Registry-registry_setup_cli_namespace) の手順に従って、名前を指定してください。
    - 例: 「us.icr.io/あなたの名前空間」
  - #### ICR_IMAGE
    - [Deployment Cloud Account](#deployment-ibm-cloud-account) の ICR 名前空間にプッシュされるブローカー コンテナー イメージの名前です。
    
      例えば。`broker-img`
  - #### ICR_NAMESPACE_REGION
    - は、名前空間が作成される (または作成される) リージョンです。リージョンは [ここ](https://cloud.ibm.com/registry/start) で見つけることができます。

      例えば。ダラスは「us-south」、フランクフルトは「eu-central」
  - #### ICR_RESOURCE_GROUP
    - ICR 名前空間が存在する (または作成される) リソース グループです。リソース グループ名は [ここ](https://cloud.ibm.com/registry/namespaces) で見つけることができます。
      例えば。「Default」
    


### 3. 以下を使用して環境内の変数を設定します。


    export $(cat deploy/build.config.properties)


​     
### 4. ブローカーを構築します。

ターミナルで以下のコマンドを実行すると、Maven でブローカーが構築され、コンテナー イメージとしてパッケージ化され、指定した名前空間の IBM Container Registry にアップロードされます。


    DEPLOYMENT_IAM_API_KEY=replace-with-your-DEPLOYMENT_IAM_API_KEY ONBOARDING_IAM_API_KEY=replace-with-your-ONBOARDING_IAM_API_KEY make build

> 注: プロンプトが表示されたら、ローカル コンピュータのパスワードを入力します。

*おめでとうございます!* ブローカー アプリケーションのイメージが完成しました。  
IBM Cloud にログインし、[ICR 名前空間](https://cloud.ibm.com/registry/namespaces) の下でイメージを見つけます。

次のステップは、作成したばかりのブローカー アプリケーション イメージをデプロイすることです。当社が提供する CLI ツールは、[IBM Cloud Code Engine](https://www.ibm.com/cloud/code-engine) へのデプロイメントをサポートしています。選択したプラットフォームに基づいて、次の適切なセクションに進みます

## IBM Cloud Code Engine へのブローカーのデプロイ

### 1. [`deploy/ce/ce.config.properties`](deploy/ce/ce.config.properties) 構成を完了します。
以下の手順に従ってプロパティ ファイルを完成させ、環境変数としてエクスポートします。    
  - #### APP_NAME
    - は、コード エンジン上のブローカーに付けるアプリケーション名です。競合が発生しないように、名前に一意の識別子を使用してみてください。  
  - #### ONBOARDING_ENV
    - IBM Cloud Onboarding が test.cloud.ibm.com (IBMer によって使用される) にある場合は「stage」に設定され、IBM Cloud Onboarding がcloud.ibm.com にある場合は「prod」に設定されます。
  - #### BROKER_USERNAME
    - ブローカーに設定するユーザー名です
  - #### BROKER_PASSWORD
    - ブローカーに設定したいパスワードです
    > 注: ここで提供される BROKER_USERNAME および BROKER_PASSWORD の値も、ブローカーを公開するときにパートナー センターで構成する必要があります。
  - #### BROKER_ICR_NAMESPACE_URL
    - ビルドステップ中に「deploy/build.config.properties」で提供されるのと同じ名前空間 URL を使用します。
    - 例: 「us.icr.io/あなたの名前空間」
  - #### ICR_IMAGE
    - ビルドステップ中に「deploy/build.config.properties」で指定されたのと同じイメージ名を使用します。
  - #### CE_プロジェクト
    - [利用可能なプロジェクトのリスト](https://cloud.ibm.com/codeengine/projects) からプロジェクトを選択します。自分で [新しいものを作成](https://cloud.ibm.com/docs/codeengine?topic=codeengine-manage-project#create-a-project) することも、名前を指定すれば私たちが作成します。 。
  - #### CE_REGION
    - IBM Code Engine デプロイメントのリージョンを設定します。これは、アプリの作成に使用する [利用可能なプロジェクトのリスト](https://cloud.ibm.com/codeengine/projects) の「Location」列の値です。
  - #### CE_RESOURCE_GROUP
    - IBM Code Engine のターゲットとなるリソース グループを選択します。これは、アプリの作成に使用する [利用可能なプロジェクトのリスト](https://cloud.ibm.com/codeengine/projects) の「リソース グループ」列の値です。
  - #### CE_REGISTRY_SECRET_NAME
    - プロジェクトから既存のレジストリー アクセスを選択するか、[作成](https://cloud.ibm.com/docs/codeengine?topic=codeengine-add-registry#add-registry-access-ce)
  - #### PC_URL [オプション]
    - ダッシュボードのパートナー センターの URL。[これ](https://cloud.ibm.com/partner-center/sell) を使用してアプリケーションを確認し、URL をコピーします。


​    
### 2. 環境内の変数をエクスポートします。

    エクスポート $(catdeploy/ce/ce.config.properties)

### 3. IBM コードエンジンへのデプロイ

ターミナルで以下のコマンドを実行すると、アプリが IBM Code Engine にデプロイされます。必要な API キーをまだ作成していない場合は、[IBM Cloud API キー](#ibm-cloud-api-keys) のセクションを参照してください。

    DEPLOYMENT_IAM_API_KEY=あなたのDEPLOYMENT_IAM_API_KEYに置き換えます METERING_API_KEY=あなたのMETERING_API_KEYに置き換えます makedeploy-ce





## 1 つのコマンドでビルドとデプロイ

CLI ツールは、ブローカー アプリの構築とデプロイの両方を行う単一のコマンドも提供します。   

 コード エンジンの場合:

    DEPLOYMENT_IAM_API_KEY=replace-with-your-DEPLOYMENT_IAM_API_KEY ONBOARDING_IAM_API_KEY=replace-with-your-ONBOARDING_IAM_API_KEY METERING_API_KEY=your-replace-with-your-METERING_API_KEY make build-deploy-ce

  



## ブローカーのテスト
- [postman_collection.json](postman_collection.json) をダウンロードし、postmanにインポートして API をテストします
- サービスに一致するように postman コレクションの planID と ServiceID を変更します。






## UI の変更に関するガイド
[こちらを参照](client/README.md)