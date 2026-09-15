Google Books API と Gemini AI を組み合わせた書籍検索アプリです。

概要
キーワード（著者名・タイトル）で書籍を検索できます
AIチャット（Book Advisor）に好みや気分を伝えると、おすすめの本を提案してくれます
気になった本はお気に入り登録でき、AIのおすすめ本もまとめて管理できます
主な機能
🔍 書籍検索：著者名・タイトルでGoogle Booksから検索
🤖 AIおすすめ機能：Gemini AIと会話しながら本を提案してもらえる
⭐ お気に入り：通常検索・AIおすすめの両方の本をお気に入り登録可能（ブラウザのlocalStorageに保存）
📖 書籍詳細モーダル：表紙・概要・出版情報などをまとめて確認
技術スタック
Vue 3（<script setup> を使用したSFC構成）
Vite
Google Books API
Gemini API
セットアップ
1. 依存パッケージのインストール
bash
npm install
2. 環境変数の設定

プロジェクトルートに .env ファイルを作成し、以下を設定してください。

env
VITE_API_KEY=あなたのGoogle Books APIキー
VITE_GEMINI_API_KEY=あなたのGemini APIキー

⚠️ .env はGitにコミットしないでください（.gitignore で除外済みです）。

3. 開発サーバーの起動
bash
npm run dev

http://localhost:5173/ にアクセスして動作を確認できます。

ビルド
bash
npm run build
ディレクトリ構成（抜粋）
src/
  App.vue          # メインコンポーネント（検索・お気に入り・AIチャット）
  main.js
  style.css
public/
  favicon.svg
注意事項
Google Books APIは、Google Cloud Console上で該当プロジェクトのAPIが有効化されている必要があります。
APIキーの利用量には日次クォータの上限があります。
