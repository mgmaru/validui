# ValidUI — Draft Concept

## 1. 概要

本プロジェクトは、フロントエンドUIにおける「壊れたUI」を、
完成後のレビュー・Lint・Visual Test・Storybookなどで発見するのではなく、
**コードを書いている時点で、そもそも表現できないようにする**ことを目指す。

現在のフロントエンド開発では、UIの堅牢性を保つために以下のような複数の仕組みを組み合わせることが多い。

- TypeScript
- ESLint
- Component Library
- Design Token
- Storybook
- Visual Regression Test
- Design Guideline
- Accessibility Test
- 人間によるコードレビュー / デザインレビュー

これらは有効だが、導入・保守・学習・運用コストが高い。

そこで本プロジェクトでは、

> **UIを「自由に描画するもの」ではなく、「制約付き構造として記述するもの」と捉える。**

この思想を実装するプロジェクトを **ValidUI** と呼ぶ。

そして、プログラミング言語の型システムのように、
UIにも「表現可能な状態」と「表現不可能な状態」を定義する。

---

# 2. 問題意識

## 2.1 現在のUI実装は自由度が高すぎる

一般的なReactでは、以下のようなコードを簡単に記述できる。

```tsx
<button
  style={{
    padding: "13px",
    background: "#ff00ff",
    borderRadius: "17px",
  }}
>
  Save
</button>
```

このコードは技術的には正しい。

しかし、プロジェクトのデザインルールとして、

- spacingは4 / 8 / 12 / 16 / 24 / 32のみ
- Button色はprimary / secondary / dangerのみ
- border-radiusはsm / md / lgのみ

という制約があった場合、このUIは「設計上は不正」である。

つまり現在のUI実装は、極端に表現すると以下の状態に近い。

```text
UI = any
```

---

## 2.2 壊れたUIを後から発見している

現在の開発では、不正なUIを防ぐために複数の検査レイヤーを導入する。

```text
実装
 ↓
TypeScript
 ↓
Lint
 ↓
Storybook
 ↓
Visual Test
 ↓
Accessibility Test
 ↓
Code Review
 ↓
Design Review
```

しかし、この方法には問題がある。

### 問題

- ツールが多い
- 設定ファイルが増える
- CIが複雑になる
- 新規メンバーの学習コストが高い
- Design Ruleと実装が乖離する
- 最終的に人間のレビューに依存する部分が多い

---

# 3. 中心となる設計思想

## 3.1 Make Invalid UI Unrepresentable

本プロジェクトの中心思想は以下である。

> **壊れたUIを検出するのではなく、壊れたUIをそもそも表現できなくする。**

これは型設計で使われる、

> Make illegal states unrepresentable

という考え方をUIへ適用したものである。

---

## 3.2 UIにも「型」がある

UIの型は単一ではない。

本プロジェクトでは、UIの型を少なくとも以下の4層として扱う。

| 層 | 内容 | 例 |
|---|---|---|
| Value Type | UIで使用可能な値 | spacing, color, radius |
| Component Type | コンポーネントのAPI | Button, Input, Card |
| Composition Type | コンポーネントの組み合わせ | Form, Dialog, Page |
| State Type | UI状態と状態遷移 | idle, loading, error |

---

# 4. Value Type

## 4.1 任意値を禁止する

例えばSpacingを以下のように定義する。

```ts
type Spacing =
  | "xs"
  | "sm"
  | "md"
  | "lg";
```

使用側は、

```tsx
<Stack spacing="md">
```

と書く。

以下は書けない。

```tsx
<Stack spacing="13px">
```

これにより、

```text
任意値
 ↓
許可された意味的値
```

へ変換する。

---

## 4.2 Semantic Tokenを優先する

単なる物理値ではなく、意味を持った値を利用する。

悪い例:

```tsx
<Text color="blue-500">
```

よりも、

```tsx
<Text tone="primary">
```

を推奨する。

内部では、

```text
primary
 ↓
Design Token
 ↓
Theme
 ↓
CSS
```

へ解決される。

---

# 5. Component Type

Buttonなどのコンポーネントは、自由なCSS属性を直接受け取らない。

```tsx
<Button
  intent="primary"
  size="md"
>
  Save
</Button>
```

例えば、

```tsx
<Button
  padding="13px"
  background="#ff00ff"
/>
```

というAPIそのものを存在させない。

---

# 6. Composition Type

本プロジェクトの重要な差別化ポイントの一つ。

既存のUIライブラリは、

> コンポーネント単体を正しく作る

ことには強い。

一方、本プロジェクトでは、

> **コンポーネントをどのように組み合わせてよいか**

まで制約する。

---

## 6.1 Form

許可:

```tsx
<Form>
  <Field />
  <Field />
  <Submit />
</Form>
```

不許可:

```tsx
<Form>
  <Modal />
</Form>
```

概念的には、

```ts
type FormChildren =
  | Field
  | FieldGroup
  | Submit;
```

のような構造を持つ。

---

## 6.2 Dialog

UI文法として、

```text
Dialog
├─ Title?
├─ Body
└─ Actions?
```

と定義する。

以下のようなUIは不正とする。

```tsx
<Dialog>
  <Dialog.Actions />
  <Dialog.Actions />
</Dialog>
```

---

# 7. UI Grammar

UIをプログラミング言語と同じように、

```text
Grammar
Type
Semantics
```

を持つものとして捉える。

例えばPage:

```text
Page
├─ Header?
├─ Main
│  ├─ Section*
│  └─ PrimaryAction <= 1
└─ Footer?
```

この場合、

```tsx
<Page>
  <Button intent="primary" />
  <Button intent="primary" />
</Page>
```

は設計上不正である。

---

# 8. State Type

UIの状態についても、不正な組み合わせを排除する。

悪い例:

```tsx
<Input
  loading
  error
  success
/>
```

この設計では、

```text
loading = true
error = true
success = true
```

という矛盾状態を作ることができる。

代わりに、

```ts
type InputState =
  | "idle"
  | "loading"
  | "error"
  | "success";
```

とする。

```tsx
<Input state="loading" />
```

これにより、

```text
loading + error + success
```

という状態そのものを表現できなくする。

---

# 9. 状態遷移

将来的には単一状態だけでなく、状態遷移も制約対象とする。

```text
Idle
 ↓
Loading
 ↓
Success
   or
Error
```

例えば、

```text
Success → Loading
```

は許可するが、

```text
Disabled → Success
```

など意味的に不正な遷移は拒否する。

UIをState Machineとして扱う。

---

# 10. 不正UIの例

## 10.1 Value違反

```css
margin: 13px;
```

Design Token外の値。

---

## 10.2 Component違反

```tsx
<Button variant="rainbow" />
```

存在しないvariant。

---

## 10.3 Composition違反

```tsx
<Dialog>
  <Dialog />
</Dialog>
```

Nested Dialog禁止ルールに違反。

---

## 10.4 State違反

```tsx
<Button
  loading
  success
/>
```

意味的に矛盾。

---

## 10.5 Page Rule違反

```tsx
<Page>
  <PrimaryButton />
  <PrimaryButton />
  <PrimaryButton />
</Page>
```

Primary Actionは最大1つというルールに違反。

---

## 10.6 Accessibility違反

```tsx
<div onClick={submit}>
  Submit
</div>
```

Buttonとして振る舞う要素をdivで作る。

フレームワーク側では、Interactive ElementとしてButton等を利用させる。

---

# 11. エラー体験

コンパイル時またはエディタ上で、次のようなエラーを目指す。

```text
UI1023: Page may contain only one primary action.

12 | <Button intent="primary" />
13 | <Button intent="primary" />
     ^^^^^^^^^^^^^^^^^^^^^^^^^

Suggestion:
Change intent to "secondary".
```

目的は単にエラーを出すことではなく、

> **なぜそのUIが不正なのかを開発者に説明すること**

である。

---

# 12. Core Rules と Configurable Rules

制約を強くしすぎると、UIの表現力が失われる。

そのため、

```text
Framework Core Rules
+
Project Configurable Rules
```

という二層構造を採用する。

---

## 12.1 Core Rules

フレームワーク側で保証する。

例:

- Interactive ElementのSemantic HTML
- 不正Stateの排除
- Token外値の禁止
- 基本的なComposition制約

---

## 12.2 Configurable Rules

プロジェクト側で定義する。

```ts
defineUI({
  spacing: ["xs", "sm", "md", "lg"],

  page: {
    maxPrimaryActions: 1,
  },

  modal: {
    allowNestedModal: false,
  },

  button: {
    intents: [
      "primary",
      "secondary",
      "danger",
    ],
  },
});
```

---

# 13. Framework Architecture

概念構成:

```text
                 UI Schema
                    │
        ┌───────────┼───────────┐
        │           │           │
      Token     Component     Rules
        │           │           │
        └───────────┼───────────┘
                    │
                  Compiler
                    │
             Type / Validation
                    │
                 Runtime
                    │
                   DOM
```

---

# 14. Source of Truth

可能な限りGit上のコードをSource of Truthとする。

```text
Repository
├─ ui.config.ts
├─ tokens.ts
├─ components/
├─ pages/
└─ rules/
```

これにより、

```bash
git diff
```

で、

- Design Token変更
- Component変更
- UI Rule変更
- Page Structure変更

を追跡できる。

---

# 15. 既存ツールとの違い

## React / Next.js

役割:

```text
UIを自由に表現する
```

本フレームワーク:

```text
UIを制約付きで表現する
```

---

## Component Library

例:

- MUI
- Chakra UI
- shadcn/ui
- Radix UI

主な責務:

```text
Component correctness
```

本プロジェクト:

```text
Component correctness
       ↓
Composition correctness
       ↓
Page correctness
       ↓
Interaction correctness
```

まで拡張する。

---

# 16. 目指さないもの

初期段階では以下を目指さない。

- Figma代替
- CSS Framework全置換
- React代替
- Browser Runtime開発
- 完全なDesign Tool

まずはReact上で動作する、

> **UI Correctness Layer**

として検証する。

---

# 17. MVP

最初からWeb Framework全体を作らない。

React上に乗る小規模ライブラリとして開始する。

仮称:

```text
ValidUI
```

MVPでは以下の3つに限定する。

## 1. Design Tokenの強制

```tsx
<Stack spacing="md">
```

---

## 2. Component Composition制約

```tsx
<Form>
  <Field />
  <Submit />
</Form>
```

---

## 3. State矛盾の禁止

```tsx
<Button state="loading">
```

---

# 18. MVPコードイメージ

```tsx
<Page>
  <Stack spacing="lg">

    <Heading level={1}>
      Account
    </Heading>

    <Form>

      <Field
        label="Email"
      >
        <Input />
      </Field>

      <Submit>
        Save
      </Submit>

    </Form>

  </Stack>
</Page>
```

UIを書く開発者は、

```text
CSSを書く
```

というより、

```text
意味のあるUI構造を書く
```

ことになる。

---


# 19. 用途別コンポーネント制約

## 19.1 「Button一般」ではなく「用途ごとのButton」に型を持たせる

単にButton全体で `loading` を禁止・許可するのではなく、
**Buttonの用途によって許される状態を変える**。

例えば:

```text
Button
├─ SubmitButton
│   └─ idle / loading / success / error
│
├─ DeleteButton
│   └─ idle / loading / confirmation
│
└─ NavigationButton
    └─ idle / disabled
```

この場合、

```tsx
<SubmitButton state="loading">
  保存
</SubmitButton>
```

は許可される。

一方で、

```tsx
<NavigationButton state="loading">
  戻る
</NavigationButton>
```

は設計上不正とする。

理由は、
Navigationという役割とLoadingという状態の意味が噛み合わないためである。

---

## 19.2 役割を型へ持ち込む

以下のようなAPIでは、

```tsx
<Button variant="primary" />
```

見た目は分かるが、
このButtonが何のためのButtonなのかは分からない。

代わりに、

```tsx
<SubmitButton />
<NavigationButton />
<DeleteButton />
```

のように、
**UIの役割そのものをComponent Typeへ取り込む**。

これにより、

```text
Visual Type
+
Semantic Role
+
Allowed State
```

を一つのコンポーネント定義として扱える。

---

## 19.3 TypeScriptだけでも一部は実現可能

例えば、

```ts
type ButtonProps = {
  loading?: boolean;
  disabled?: boolean;
};

type NavigationButtonProps =
  Omit<ButtonProps, "loading"> & {
    loading?: never;
  };
```

とすれば、

```tsx
<NavigationButton loading />
```

をTypeScriptの型エラーにできる。

つまり、

> 「特定コンポーネントだけ特定Propsを禁止する」

ことは既存のTypeScriptでも可能である。

ただし、
本プロジェクトではこれを個別実装に頼るのではなく、
**Schemaから一貫して生成・保証する**ことを目指す。

---

# 20. UI Schema を Source of Truth にする

## 20.1 Component定義から複数成果物を生成する

本プロジェクトでは、
Storybook・TypeScript型・ドキュメントを個別に管理するのではなく、
一つのUI Schemaから生成する構成を検討する。

例:

```ts
const NavigationButton = defineButton({
  role: "navigation",

  states: [
    "idle",
    "disabled",
  ],
});
```

SubmitButton:

```ts
const SubmitButton = defineButton({
  role: "submit",

  states: [
    "idle",
    "loading",
    "success",
    "error",
  ],
});
```

この定義から、

```text
                 UI Schema
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
  TypeScript     Component   Storybook
     Types         Runtime      Stories
        │                         │
        └───────────┬─────────────┘
                    ↓
                  Docs
```

を生成する。

---

## 20.2 UI Schemaによる状態制約

例えば、

```ts
defineButton({
  role: "navigation",
  states: [
    "idle",
    "disabled",
  ],
});
```

から、

```ts
type NavigationButtonState =
  | "idle"
  | "disabled";
```

を生成する。

したがって、

```tsx
<NavigationButton state="loading" />
```

を書いた場合、

```text
UI2011

"loading" is not a valid state for NavigationButton.

Allowed states:
  idle
  disabled
```

のようなエラーを出す。

---

## 20.3 Git差分をUI仕様変更として扱う

例えばNavigationButtonでLoadingを新しく許可する場合、

```diff
states: [
  "idle",
  "disabled",
+ "loading",
]
```

というSchema変更だけで、

- TypeScript型
- Runtime Component
- Storybook Story
- Documentation
- Test Case

へ変更を反映できる。

つまり、

```text
UI仕様変更
   ↓
Schema変更
   ↓
Git diff
   ↓
生成物へ反映
```

という構造を目指す。

これにより、
Design Rule・Type・Story・Docsの乖離を減らす。

---

# 21. Storybookの位置づけ

## 21.1 Storybookは「型」ではない

Storybookは、
設計上不正なUIをアプリコード上で禁止する仕組みではない。

Storybookの主な役割は、

> **許されたUI状態を可視化・確認・検証すること**

である。

したがって責務は、

```text
Framework / TypeScript
    ↓
不正状態を作れない

Storybook
    ↓
許された状態を確認する
```

と分ける。

---

## 21.2 StorybookをSource of Truthにしない

例えばNavigationButtonに、

```text
Default
Disabled
```

というStoryだけを作ったとしても、

アプリケーション側で

```tsx
<NavigationButton loading />
```

を書けない保証にはならない。

Storyが存在しないことと、
状態が禁止されていることは別問題である。

そのため、

```text
Storybook
    ↓
UI Rule
```

ではなく、

```text
UI Schema
    ↓
Type / Component / Storybook / Docs
```

という方向を採用する。

---

## 21.3 Storybookの役割

Storybookには以下を担当させる。

### 状態カタログ

```text
NavigationButton
├─ Idle
└─ Disabled
```

### Interaction Test

例えば、

```text
NavigationButton
→ click
→ navigation
```

のような操作を検証する。

### Accessibility Test

レンダリング結果に対して、

- Semantic HTML
- ARIA
- Contrast
- Keyboard操作

などを確認する。

---

# 22. Schema-driven Story Generation

UI SchemaからStorybookのStoryを自動生成することを検討する。

例えば、

```ts
states: [
  "idle",
  "disabled",
]
```

という定義から、

```text
NavigationButton
├── Idle
└── Disabled
```

を自動生成する。

SubmitButtonなら、

```text
SubmitButton
├── Idle
├── Loading
├── Success
└── Error
```

となる。

この方式では、

```text
Component Specification
        ↓
Story Catalog
```

を自動同期できる。

---

# 23. 「意味」をUI型へ含める

今回の議論から、
本プロジェクトにおけるUI TypeはProps型だけでは不足している。

以下を一つの型概念として扱う。

```text
UI Type
├─ Value
├─ Component
├─ Semantic Role
├─ Composition
└─ State
```

特に、

```text
Semantic Role
    ↓
Allowed State
```

という関係が重要である。

例:

```text
Submit
→ loading可能

Navigation
→ loading禁止

Delete
→ confirmation可能

Display-only
→ interaction禁止
```

つまり、

> **同じ見た目のコンポーネントでも、意味が違えば型も違う**

と考える。

---

# 24. 更新後のアーキテクチャ

```text
                      UI Schema
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
      Role             State             Token
       │                 │                 │
       └──────────┬──────┴──────┬─────────┘
                  │             │
              Type Generator  Rule Engine
                  │             │
                  └──────┬──────┘
                         │
                    Component API
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
    Application       Storybook          Docs
       │                 │
       ↓                 ↓
  Compile Error    Visual / Interaction /
                   Accessibility Test
```

---

# 25. この方式で減らしたいコスト

従来:

```text
Component APIを修正
+
Typeを修正
+
Storyを修正
+
Docsを修正
+
Testを修正
```

Schema-driven:

```text
UI Schemaを修正
        ↓
必要な成果物を生成
```

目的はStorybookやTestをなくすことではない。

> **複数ツールに同じUIルールを重複記述するコストを減らすこと**

が狙いである。

---


# 26. 段階的な技術検証

以下の順番で検証する。

```text
TypeScript
 ↓
ESLint
 ↓
AST解析
 ↓
Compiler Plugin
 ↓
独自DSL
```

最初から独自DSLを作らない。

まず、

> TypeScriptの型システムだけでどこまでUI制約を表現できるか

を検証する。

---

# 27. 重要な技術課題

## 20.1 React Childrenの静的解析

Composition制約をどこまでTypeScriptだけで保証できるか。

---

## 20.2 Page全体ルール

例えば、

```text
Primary Button <= 1
```

のようなルールはProps型だけでは表現が難しい。

AST解析やCompiler Pluginが必要になる可能性がある。

---

## 20.3 Dynamic UI

以下のような実装:

```tsx
{items.map(item => (
  <Button />
))}
```

では、静的解析だけでは個数を完全に判断できない場合がある。

---

## 20.4 Escape Hatch

強い制約は表現力を下げる。

そのため、

```tsx
<Unsafe />
```

などのEscape Hatchを提供する可能性がある。

ただし、

```text
Unsafeを利用した箇所は明示的に分かる
```

ことが重要。

---

# 28. 設計原則

## Principle 1

**Invalid UI should be hard or impossible to represent.**

---

## Principle 2

**Semantic API over visual API.**

悪い:

```tsx
<Text color="#666">
```

良い:

```tsx
<Text tone="secondary">
```

---

## Principle 3

**Structure is part of the type.**

UIの型はPropsだけではない。

Component Compositionも型として扱う。

---

## Principle 4

**State must be explicit.**

Boolean Propsの乱立を避ける。

---

## Principle 5

**Design rules should live close to code.**

ルールを外部ドキュメントだけに置かない。

---

## Principle 6

**Git should describe the UI system.**

Token・Rule・Component・StructureをGitで追跡可能にする。

---

## Principle 7

**Prefer prevention over detection.**

```text
Detect broken UI
```

よりも、

```text
Prevent broken UI
```

を優先する。

---

# 29. プロジェクトの価値仮説

現在:

```text
Design Rule
+
Design Token
+
TypeScript
+
Lint
+
Storybook
+
Visual Test
+
Review
```

将来:

```text
Strongly Typed UI Framework
```

へ可能な範囲で統合する。

狙いはツールを完全に不要にすることではない。

目的は、

> **UIの堅牢性を保つために必要な「人間の注意」と「後工程の検査」を減らすこと**

である。

---

# 30. プロジェクト名

本プロジェクトの仮称は、

> **ValidUI**

とする。

名前には、

- Valid = 設計上有効・正しい状態
- UI = User Interface

という意味を持たせる。

本プロジェクトの中心思想である、

> **Invalid UIを後から検出するのではなく、ValidなUIしか表現できないようにする**

という考え方を直接表す名称とする。

設計思想・技術概念としては引き続き、

- Strongly Typed UI
- UI Correctness
- UI Grammar
- Schema-driven UI

といった言葉を用いる。

---

# 31. 一文で表すと

> **UIを自由に描画するのではなく、型・文法・意味・状態を持つ制約付き構造として記述し、設計上不正なUIをコード上で表現できなくするためのフロントエンド基盤。**

---

# 32. 次の検討事項

次の議論では、以下を検討する。

1. 何を「不正UI」と定義するか
2. TypeScriptだけで保証できる範囲
3. Semantic Roleをどう型へ取り込むか
4. RoleごとのAllowed Stateをどう定義するか
5. AST解析が必要なルール
6. Runtime Validationが必要なルール
7. Reactとの統合方法
8. Component API設計
9. UI SchemaのDSL
10. SchemaからStorybook Storyをどう生成するか
11. SchemaからDocs / Testをどう生成するか
12. Escape Hatchの設計
13. エラーメッセージ設計
14. MVPの具体的な実装範囲
