# skills

Claude Code と Codex に読ませる自作 skill を置くリポジトリ。

skill は「環境を取り替えても価値が変わらない知識」だけを置く単位とする。
どの skill をどの版で使うかという選択と、その配布は、このリポジトリを参照する側（dotfiles など）が持つ。
このリポジトリは参照元の存在を前提にせず、環境固有のパスや設定を内容に含めない。

## 収録している skill

| skill | 扱う領域 |
| --- | --- |
| `mywriting` | 日本語の文章規範。整形、段落と論証の構成、複数ファイル構成、推敲 |
| `mycoding` | コーディングの原則。How・What・Why・Why not の書き分け |
| `mygithub` | GitHub の PR・Issue・コードレビューの運用規範 |
| `myskill` | skill を作る・直す・レビューするときの方法論 |

## 構造

```
skills/
  <name>/
    SKILL.md
    references/*.md   # 責務ごとに分けた規範の本体（必要なものだけ）
```

`skills/` を一段挟むのは、README や LICENSE と skill が同じ階層に混ざらないようにするため。
[APM](https://github.com/microsoft/apm) の git 依存は `owner/repo/<subpath>` でサブディレクトリを指すので、この階層がそのまま依存パスになる。

## 参照の仕方

APM のマニフェストに、skill 単位で git 依存として書く。
版は git の SHA で固定する。

```yaml
dependencies:
  apm:
    - snamiki1212/skills/skills/mywriting#<sha>
    - snamiki1212/skills/skills/mycoding#<sha>
```

リリースタグは付けない。
複数の skill が独立に変わるため、リポジトリ全体に付けたタグは個々の skill の版を表さないからである。

参照側は、SHA を書き換えて再インストールすることで更新する。

## 変更の流れ

1. このリポジトリで skill を編集し、PR を経て `main` へマージする。
2. 参照側のマニフェストの SHA を書き換える。
3. 参照側で再インストールして配布する。

編集を往復している間は、参照側の該当行をローカル clone の絶対パスへ一時的に差し替えると速い。
内容が固まってから SHA 固定の形に戻す。

## 経緯

もとは dotfiles リポジトリの `agents/apm/skills/` に置いていた。
skill が版を持てず、private リポジトリの外から参照できない状態だったため、独立させた。
判断の記録は dotfiles 側の `adr/2026-09-16_skills-repository-extraction/` にある。
