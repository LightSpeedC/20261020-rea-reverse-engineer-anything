# REA と逆解析ツールの調査

AI エージェント向けの逆解析ツールキット REA（Reverse Engineer Anything）と、Ghidra・IDA Pro・Binary Ninja・Hopper など、REA が使う／並ぶツールの全体像

> 📅 作成: 2026-10-20 / 更新: 2026-10-20

[⌂](../)

1. [REA の概要](#1-rea-の概要)
2. [REA の仕組みと導入](#2-rea-の仕組みと導入)
3. [類似・関連ツール](#3-類似関連ツール)
4. [比較と使い分け](#4-比較と使い分け)
5. [注意点と出典](#5-注意点と出典)

## 1. REA の概要

- **正体**: AI コーディングエージェント（Claude Code・Codex・Cursor・Gemini CLI 等）に逆解析の道具を持たせる、オープンソースのツールキット。公式は GitHub の [morluto/rea](https://github.com/morluto/rea)（MIT ライセンス）。npm パッケージ名は `rea-agents`
- **役目**: 自分では解析しない「つなぎ役」。MCP サーバとして、Ghidra・Hopper・IDA などの既存の解析器をエージェントから呼べるようにする。CLI（`rea`）からも同じ機能を使える
- **出力**: 結論だけでなく、根拠（evidence）と未確定の点を一緒に返す設計
- **解析はローカル**。ただしエージェントが使うモデル提供元へは、解析結果が会話として渡る

### 対象

| 分類 | 対象 |
|---|---|
| ネイティブ | 実行ファイル全般、Go・Rust 製バイナリ、ファームウェア |
| マネージド・スクリプト | .NET アセンブリ、JavaScript・Electron アプリ、WASM、EVM バイトコード |
| モバイル・ゲーム | Android APK、ADB 接続の端末、Flutter、Unity IL2CPP、Godot PCK |
| その他 | Web サイト、SQLite DB、pcap、ロジックアナライザの記録、プロセスの挙動 |

## 2. REA の仕組みと導入

### つながる解析器

- **Ghidra・Hopper・IDA**: ネイティブの深い解析に使う。既にインストール済みのものを使う（Hopper は承認のうえで導入も可）。Windows の Ghidra は実験的扱い
- **JEB**: 呼び出し側で起動した JEB の MCP 経由
- ほかに Rizin・GNU objdump・Kuna（デコンパイラ）。TShark・pwntools・Binwalk 等は Linux 限定または利用者が用意

### 導入

```shell
npx rea-agents setup          # 対応エージェントへ MCP と手順書を登録（変更を見て承認、既存設定は控えを取る）
npm install --global rea-agents   # CLI として rea を入れる。更新は rea update
```

- Node.js 22.19 以上（22.x）・24.11 以上（24.x）・26 以上が要る
- 大きなバイナリでは、起動の時間切れを環境変数で延ばす必要がある

（未検証: 本調査では導入・実行はしていない。上記は公式 README の記述）

## 3. 類似・関連ツール

| ツール | 種別 | 価格 | 特徴 | AI 連携 |
|---|---|---|---|---|
| Ghidra | 逆アセンブラ・デコンパイラ | 無料（OSS） | NSA 製。多アーキテクチャ・Java/DEX も扱う。12.x 系が現行（12.1.x の報告あり） | GhidraMCP（コミュニティ製。272 ツールの版も）、G-3PO 等 |
| IDA Pro | 逆アセンブラ・デコンパイラ | 年 $1,099〜$8,599 | 業界標準。Hex-Rays デコンパイラの成熟度が最も高い | IDA 用 MCP・AI プラグイン |
| Binary Ninja | 逆アセンブラ・デコンパイラ | 無料版あり、有償 $199〜 | 中間言語（BNIL）と API が使いやすい。価格と機能の釣り合いが良い | Sidekick（公式 AI） |
| Hopper | 逆アセンブラ・デコンパイラ | $99〜$129（買い切り＋1 年更新） | macOS・Linux 向け。Apple 系（Mach-O・Objective-C・Swift）に強く手軽。最新は v6 系の報告 | REA から利用可 |
| JEB | デコンパイラ | 有償 | Android（DEX）・WASM・EVM などに強い | MCP |
| radare2 / Rizin（Cutter） | 逆解析フレームワーク | 無料（OSS） | CLI 主体で自動化向き。Cutter が GUI | r2ai 等 |
| x64dbg・WinDbg | デバッガ | 無料 | Windows の動的解析 | — |
| Frida | 動的計装 | 無料（OSS） | 実行中のアプリに JS を差し込み、関数をフック | — |
| dnSpy / ILSpy・jadx | マネージドのデコンパイラ | 無料 | .NET（dnSpy・ILSpy）、Android（jadx）をほぼ元のソースに戻せる | — |
| angr | 記号実行 | 無料（OSS） | Python から経路探索・制約解決 | — |

価格・版は検索結果からの転記で、各社の公式ページで改めて確かめていない。

## 4. 比較と使い分け

- **REA は上の表と競合しない**。解析器の上に載る「エージェント用の窓口」で、Ghidra 等が下に要る。比べる相手は GhidraMCP・IDA 用 MCP などの単体 MCP
- **単体 MCP との違い**: 1 つの導入で複数の解析器と、APK・Electron・.NET・pcap など解析器の外にある対象までまとめて扱える。根拠と未確定点を返す形式が揃う
- **費用を掛けずに始めるなら** Ghidra ＋ REA。Windows では Ghidra が実験的扱いなので、WSL・Linux 側で動かす選択肢もある
- **精度を最優先するなら** IDA Pro。**Mac で手軽に**なら Hopper。**スクリプトで自動化**なら Binary Ninja・Rizin
- **.NET・Java・Android** は専用デコンパイラ（ILSpy・jadx）のほうが元のソースに近く戻る。REA もこの種の対象を扱う

## 5. 注意点と出典

> [!IMPORTANT]
> <strong>スター数は当てにしない。</strong>GitHub のページでは約 34 万、紹介記事では 12.9 万・23.4 万と食い違い、短期間で極端に伸びている。人気の指標として使わない

> [!WARNING]
> <strong>偽サイト・ミラーに注意。</strong>reverseengineeranything.org・reverseengineeringanything.com・SourceForge のミラー等は公式ではない。導入は GitHub と npm の `rea-agents` から行う

- **法令・ライセンス**: 解析してよいのは、権利を持つか許可を得た対象だけ。ソフトウェアの利用規約が逆解析を禁じていることがある。IDA・Hopper・JEB などの商用ライセンスは利用者が用意する
- **情報の流出**: 解析そのものはローカルでも、逆アセンブル結果や文字列はモデル提供元へ送られる。機密のバイナリでは提供元のデータ方針を確かめる
- **AI の結論を鵜呑みにしない**: 根拠付きで返る設計だが、解釈の誤りは残る

### 出典

- [morluto/rea（GitHub・公式）](https://github.com/morluto/rea)
- [Cyber Security News の紹介記事](https://cybersecuritynews.com/reverse-engineer-anything-tool/)
- [DataCamp のチュートリアル](https://www.datacamp.com/tutorial/reverse-engineer-anything-rea-tutorial)
- [Hopper 公式](https://www.hopperapp.com)
- [bethington/ghidra-mcp のリリース](https://github.com/bethington/ghidra-mcp/releases)
- [IDA Pro の代替ツール比較（The CTO Club）](https://thectoclub.com/tools/best-ida-pro-alternatives/)
- [2026 年の逆解析ツール解説（youngju.dev）](https://youngju.dev/blog/culture/2026-05-16-reverse-engineering-tools-2026-ghidra-ida-pro-binary-ninja-radare2-frida-x64dbg-angr-deep-dive)

[⌂](../)
