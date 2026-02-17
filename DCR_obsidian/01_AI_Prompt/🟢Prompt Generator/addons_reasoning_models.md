# addons_reasoning_models

> Version: v1.0  
> Last updated: 2026-02-17  
> Changelog: 初版。Anthropic/OpenAI/Google公式ドキュメントおよび実務報告を一次ソースとして作成。  
> 適用先: Prompt Editorial Lab v6.0 の「モデル固有ガイダンス」節および「対象モデル選定」を補強する addons。  
> 注記: モデル別の**プロンプトテクニック**は addons_prompt_engineering.md に格納済み。本ファイルは「モデル選定ロジック」と「推論モード設計」に特化する。

---

## このファイルの役割（addons_prompt_engineering.md との分担）

| 観点 | addons_prompt_engineering.md | **本ファイル（addons_reasoning_models.md）** |
| --- | --- | --- |
| 焦点 | テクニック × 適用条件 | モデル選定 × 推論モード設計 |
| 主な参照タイミング | Execute（再設計）節 | Think（理解）節・モデル選定時 |
| 重複回避 | プロンプト構造・指示スタイル | タスク適合・コスト・Latency・推論モード制御 |

---

## 1. 推論モデルとは何か

### 定義と設計思想

| 分類 | 代表モデル | 設計思想 | 向いているタスク |
| --- | --- | --- | --- |
| **推論モデル（Reasoning）** | Claude 4（Extended Thinking）、o3/o4-mini、Gemini 3（Deep Think） | 回答前に内部で多段階思考。複雑な問題を自分で分解する | 数学・コーディング・法律・戦略立案・曖昧な大量情報の整理 |
| **標準モデル（Standard）** | GPT-4o、Claude Haiku 4.5、Gemini Flash | 直接生成。高速・低コスト | 文章生成・要約・FAQ・定型タスク・会話 |

**重要：** 推論モデルは「賢いシニア社員」に高レベルのゴールを渡すイメージ。標準モデルは「優秀なジュニア社員」に明示的な手順書を渡すイメージ。（Fact：OpenAI公式ガイドより）

---

## 2. モデル選定フロー（タスク × 要件マトリクス）

### ステップ1：タスクタイプ判定

```text
タスクは「多段階推論が必要か」を問う

YES → ステップ2へ
NO  → 標準モデルを選ぶ（コスト・Latency優先）
```

**多段階推論が必要なタスクの例（Fact）：**

- 複雑な数学・統計・証明
- 大規模コードベースの設計・デバッグ
- 法的文書の解釈・リスク分析
- 複数の矛盾する情報源の統合
- 長期的な戦略立案・意思決定
- Prompt Engineering の再設計・監査

**標準モデルで十分なタスクの例（Fact）：**

- テキスト要約・翻訳・言い換え
- FAQ・定型応答
- データ抽出・フォーマット変換
- 一般的なコード補完（複雑でない場合）
- マーケティング文章の生成

### ステップ2：推論モデル選定

| 条件 | 推奨モデル | 理由 |
| --- | --- | --- |
| 長文脈（100k〜）＋コーディング | Claude 4 Sonnet / Opus（Extended Thinking ON） | SWE-Bench Verified で最高クラス（77.2%）。長期自律運用に強い（Fact） |
| 純粋な論理・数学・構造化推論 | o3 / o4-mini | 構造的な step-by-step 推論が最も明示的（Fact） |
| マルチモーダル＋超長文脈（1M token） | Gemini 3 Pro（Deep Think） | 動画・音声・画像を含む文書全体の分析に強い（Fact） |
| コスト重視 ＋ 推論が必要 | o4-mini（reasoning_effort: medium） または Gemini 3 Flash | 推論能力を維持しつつコスト削減（推定） |
| モデル不明・汎用 | まず標準モデルで試し、品質不足なら推論モードへ | 必要以上の推論はコスト増大とLatency悪化を招く |

---

## 3. プロバイダー別 推論モード実装の違い

各プロバイダーが推論機能をどう実装しているかは設計が異なる。プロンプト設計に影響するため把握が必要。

| プロバイダー | 推論モードの制御方法 | デフォルト挙動 | 透明性 |
| --- | --- | --- | --- |
| **Anthropic（Claude）** | Extended Thinking を明示的に ON/OFF。`budget_tokens` でトークン上限を設定。Adaptive Thinking（自動）も選択可 | OFF（通常モード）が既定 | 要約版の思考過程を提示（Fact） |
| **OpenAI（o3/o4-mini）** | `reasoning_effort: low/medium/high` で制御。内部推論は不可視 | medium が既定 | 推論トークンは隠蔽（Fact） |
| **Google（Gemini 3）** | 推論がモデルに内包。`thinking_level: LOW/HIGH` で制御可。手動選択不要 | 自動（タスク複雑度に応じてモデルが判断） | 思考過程は基本非公開（Fact） |

---

## 4. Claude Extended Thinking 設計ガイド

Prompt Editorial Lab v6.0 の主軸モデルが Claude のため、詳細設計を追加する。

### いつ Extended Thinking を使うか（Fact：Anthropic公式）

**ON にすべきタスク：**

- 数学・コーディング・複雑な分析
- 複数条件が競合する意思決定
- 曖昧な大量情報から構造を抽出するタスク
- Prompt の監査・再設計（v6.0 の用途に合致）

**ON にしなくていいタスク：**

- 単純な質問応答・翻訳
- 形式変換・データ抽出
- 一般的な文章生成
- Latency が重要なリアルタイム用途

### budget_tokens の設定方針（Fact：Anthropic公式）

| フェーズ | 推奨 budget_tokens | 備考 |
| --- | --- | --- |
| 初期テスト | 1,024（最小） | 有益かどうかを確認してから増やす |
| 通常利用 | 4,000〜16,000 | 多くの複雑タスクに十分 |
| 高難度タスク | 16,000〜32,000 | 32k 超はバッチ処理を推奨 |
| 超高難度タスク | 32,000〜 | ネットワークタイムアウトリスク。Streaming 必須（21,333 token 超） |

**逓減法則：** budget を増やすと精度は上がるが、ある閾値を超えると改善幅が小さくなる。コスト意識を持ってチューニングする。

### プロンプト設計上の注意（Fact：Anthropic公式）

- Extended Thinking を ON にするとき、**CoT 指示（"step by step" 等）を削除する**。モデルが自分で推論するため、指示が干渉する。
- Prefill は Extended Thinking 有効時に使用不可。
- Temperature・top_k の変更は不可（top_p は 0.95〜1.0 の範囲で可）。
- 思考ブロックを多ターン会話で引き継ぐ場合、**完全・未改変のブロックを渡す**こと。

---

## 5. タスク × 推論モード × プロバイダー 選定表（統合版）

| タスクカテゴリ | 推論モード必要度 | 推奨プロバイダー | 補足 |
| --- | --- | --- | --- |
| 複雑なコーディング・設計 | 高 | Claude 4 Sonnet（Thinking ON） | SWE-Bench で最高スコア（Fact） |
| 数学・証明・STEM | 高 | o3 または Gemini 3 Pro（Deep Think） | 構造的推論が明示的（Fact） |
| 長文書分析（100k〜） | 高 | Gemini 3 Pro または Claude 4 Opus | 大規模文脈の処理（Fact） |
| 法的・規制文書の解釈 | 高 | o3 または Claude 4 Opus | 高精度の論理追跡が必要（推定） |
| Prompt 再設計・監査 | 中〜高 | Claude 4 Sonnet（Thinking ON） | v6.0 の用途に最適合（推定） |
| 文章生成・翻訳・要約 | 低〜不要 | Claude Haiku / Gemini Flash | コスト・速度優先（Fact） |
| マルチモーダル分析（動画等） | 中〜高 | Gemini 3 Pro | 動画・音声の理解に特化（Fact） |
| リアルタイム・低遅延 | 推論モード不向き | Gemini Flash / GPT-4o mini | Latency 最優先（Fact） |

---

## 6. コスト・Latency トレードオフ（Unknown 含む）

### 定性的ガイドライン（Fact）

- 推論モデルは標準モデルより **Latency が高い**（思考トークン分の処理時間が追加される）
- Extended Thinking 32k tokens 超のリクエストは **応答時間が5分超になる可能性がある**（バッチ処理推奨）
- budget_tokens を大きく設定するほど **コストと時間が増大**し、逓減的な精度改善になる

### Unknown（要確認）

- 各モデルの具体的なトークン単価は変動が激しいため、本ファイルには記載しない。  
  確認方法：Anthropic `anthropic.com/pricing`、OpenAI `platform.openai.com/pricing`、Google `cloud.google.com/vertex-ai/pricing` を都度参照。
- 推論モデルの「実際のコスト対効果」はタスク依存で、ベンチマーク数値とは乖離することがある。本番環境では **A/Bテストまたは Eval** で検証することを推奨。

---

## 7. 事故パターンと対策（推論モデル固有）

| パターン | 原因 | 対策 |
| --- | --- | --- |
| 推論モデルに詳細手順を渡しすぎた | 「CoT を指示すれば精度が上がる」という誤解 | ゴールのみ渡す。CoT 指示を削除する |
| budget_tokens が少なすぎた | デフォルト 1024 のまま使用 | タスク複雑度に応じて漸増。モデルのトークン使用量を観察する |
| Latency に不満が出た | 推論モード不要なタスクに適用した | タスク選定フローに従って適用タスクを絞る |
| 思考ブロックを改変して渡した | 多ターン会話でブロックを編集した | 必ず完全・未改変のブロックをそのまま渡す |
| 推論モデルに CoT ガイドを残した | 旧プロンプトからの引き継ぎ | Extended Thinking ON 時は CoT 指示を削除する |

---

## 8. v6.0 との接続マップ

| このファイルの節 | v6.0 の接続箇所 |
| --- | --- |
| 2. モデル選定フロー | Think（理解）節の「暗黙の前提・制約」確認時 |
| 3. プロバイダー別実装の違い | モデル固有ガイダンス節の設計基盤 |
| 4. Extended Thinking 設計ガイド | Execute（再設計）節で Claude を対象モデルとした場合 |
| 5. 統合選定表 | 対象モデル選定（v6.0 入力欄「対象モデル」）の補助 |
| 6. コスト・Latency | Change Budget / Stop Verdict の「コスト衝突」判定 |
| 7. 事故パターン | Critique（検証）節の失敗パターン補強 |

---

## 参照元（一次ソース）

- **Fact：** Anthropic 公式ドキュメント Extended Thinking `docs.anthropic.com/en/docs/build-with-claude/extended-thinking`（2026-02取得）
- **Fact：** OpenAI 公式ガイド `platform.openai.com/docs/guides/reasoning-best-practices`（2026-02取得）
- **Fact：** Google Vertex AI Gemini 3 prompting guide（2026-02取得）
- **推定：** モデル比較記事（vellum.ai・magai.co・labellerr.com）複数の実務報告を総合

---

## 改訂ルール

- **モデルファミリーが更新された場合**、該当モデル行のみを差し替える
- **コスト・Latency 定量情報は格納しない**（陳腐化が最も速い情報のため、参照先リンクのみ保持）
- Unknown は必ず隔離し、確認方法を添える
- Change Budget：新規行 最大3行/改訂

---

*このファイルは Prompt Editorial Lab v6.0 の addons として機能します。正本 canonical の優先順位・衝突処理に従ってください。*
*モデル別プロンプトテクニックは addons_prompt_engineering.md を参照してください。*
