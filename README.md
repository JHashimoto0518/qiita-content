# qiita-content

This is a repository for managing Qiita articles.

## 利用方法のメモ

[Qiitaの記事をGitHubリポジトリで管理する方法 #GitHub - Qiita](https://qiita.com/Qiita/items/32c79014509987541130)

1. 作業ディレクトリの初期化

    ```bash
    npx qiita init 
    ```

1. push する

    ```bash
    git push
    ```

main への push で、ワークフローがトリガーされる

1. リモートリポジトリ main ブランチの`public/`に既存コンテンツが作成される
2. 作業ディレクトリの`public/`に pull される

push の前に、`npx qiita preview`をしないこと。既存コンテンツがダウンロードされるため、リモートと作業ディレクトリで差分が生じ、push に失敗する。
