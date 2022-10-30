---
title: "[Next.js(Typescript)] プロジェクト作成とLinter,Formatterの設定"
emoji: "👷‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, nextjs, prettier]
published: false
---

Next.jsのプロジェクトを作成し、Linter,Formatterの設定をVSCodeで行う際の手順です。

## プロジェクトの作成

### 作成

`yarn create next-app {app-name}`

` cd {app-name}`

`yarn dev` 

正常に動けば（= ローカル環境で初期ページが表示されれば）OK

### Typescriptを導入

`touch tsconfig.json`

`yarn add -D typescript @types/react @types/node`

pages/\_app.js → pages/\_app.tsx へ変更
pages/index.js → pages/index.tsx へ変更

`yarn dev`

正常に動けば（= ローカル環境で初期ページが表示されれば）OK

### Linter,Formatterの設定

prettierのインストール

`yarn add -D prettier eslint-config-prettier`

.eslintrc.jsonのextendに”prettier”を追加しprettierのルールを最優先とする
[prettier/eslint-config-prettier: Turns off all rules that are unnecessary or might conflict with Prettier.](https://github.com/prettier/eslint-config-prettier)

```json
{
 "extends": ["next", "prettier"]
}
```

package.jsonにフォーマットのルールを追加する（prettierの箇所。ルールの詳細は[こちら](https://prettier.io/docs/en/options.html)。）

```json
{
  "name": "grit-nft-app",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "prettier": {
    "trailingComma": "all",
    "tabWidth": 2,
    "semi": false,
    "singleQuote": true,
    "jsxSingleQuote": true,
    "printWidth": 100
  },
  
  ...
```

settings.jsonに設定を追加し、保存時にフォーマットが適用されるようにする

```json
{
 "editor.defaultFormatter": "esbenp.prettier-vscode",
 "editor.formatOnSave": true,
 "editor.codeActionsOnSave": {
  "source.fixAll": true
 }
}
```

以上でTypescriptを用いて保存時に自動フォーマットされるプロジェクトが作成できます。

## 参考

* [Basic Features: ESLint | Next.js](https://nextjs.org/docs/basic-features/eslint)

* 本記事で使用したバージョン情報

```
"next": "13.0.0",
"react": "18.2.0",
"react-dom": "18.2.0"
"@types/node": "^18.11.7",
"@types/react": "^18.0.24",
"eslint": "8.26.0",
"eslint-config-next": "13.0.0",
"eslint-config-prettier": "^8.5.0",
"prettier": "^2.7.1",
"typescript": "^4.8.4"
```

