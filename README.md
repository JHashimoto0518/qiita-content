# qiita-content
This is a repository for managing Qiita articles.

## 利用方法のメモ

[Qiitaの記事をGitHubリポジトリで管理する方法 #GitHub - Qiita](https://qiita.com/Qiita/items/32c79014509987541130)

1. 作業ディレクトリの初期化
   
    ```bash
    npx qiita init 
    ```

1. pushする
   
    ```bash
    git push
    ```

    mainへのpushで、ワークフローがトリガーされる
    
    1. リモートリポジトリmainブランチの`public/`に既存コンテンツが作成される
    1. 作業ディレクトリの`public/`にpullされる

    pushの前に、`npx qiita preview`をしないこと。既存コンテンツがダウンロードされるため、リモートと作業ディレクトリで差分が生じ、pushに失敗する。
