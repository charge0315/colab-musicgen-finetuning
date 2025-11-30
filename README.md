# MusicGen-Large LoRA ファインチューニング

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/charge0315/colab-musicgen-finetuning/blob/main/musicgen-lora-finetune.ipynb)

Meta（Facebook）のMusicGen-Largeモデルを独自データセットでLoRAファインチューニングするためのGoogle Colabノートブックです。A100 GPU 1台での学習を想定し、WandBによる詳細なロギング、自動チェックポイント、学習再開機能を備えています。

---

## 🎯 特徴

- **LoRA（Low-Rank Adaptation）** による効率的なファインチューニング
- **WandB統合** でリアルタイムのメトリクス追跡
- **自動チェックポイント** による学習の安全性
- **Google Drive統合** で全てのモデルを自動保存
- **段階的な処理** でトークナイズ速度を事前計測
- **少数サンプルテスト** で本番前に動作確認
- **完全な学習再開機能** で中断からの復帰が可能

---

## 📋 事前準備

### 1. Google Colab環境

- **GPU**: A100 (80GB) を強く推奨（V100でも動作可能だが要調整）
- **ランタイムタイプ**: GPU を選択

### 2. シークレットの設定

Colabの「🔑シークレット」メニューから以下を設定：

- `WANDB_API_KEY`: [WandB](https://wandb.ai/settings)のAPIキー
- `HF_TOKEN`: [Hugging Face](https://huggingface.co/settings/tokens)のトークン

### 3. データセットの準備

Google Driveに以下のいずれかの形式でデータを配置：


#### ZIPファイル

```text
/content/drive/MyDrive/Archive_wav
├── archive_batch_xxxx.zip  # ZIPファイル（複数可）
└── metadata.jsonl       # メタデータ（オプション）
ZIPファイルは001～139まで存在し、１つのZIPに約1000ファイルが含まれます。
```

**metadata.jsonl の形式** :

```json
{"path": "audio/track1.wav", "duration": 30.0, "sample_rate": 32000, "description": "説明文"}
{"path": "audio/track2.wav", "duration": 30.0, "sample_rate": 32000, "description": "説明文"}
```

---

## 🚀 使い方

### ステップ1: ノートブックを開く

上部の「Open in Colab」バッジをクリックしてノートブックを開きます。

### ステップ2: シークレットを設定

Colabの左サイドバーから「🔑」アイコンをクリックし、`WANDB_API_KEY`と`HF_TOKEN`を追加します。

### ステップ3: セルを順番に実行

1. **依存関係のインストール** - 必要なライブラリをインストール
2. **Google Driveマウント** - データセットにアクセス
3. **認証設定** - WandBとHugging Faceにログイン
4. **テスト実行（推奨）** - 5サンプルで動作確認
5. **トークナイズ速度計測** - 処理時間を推定
6. **全データのトークナイズ** - EnCodecでエンコード
7. **モデル準備** - LoRA適用とデータローダ作成
8. **学習実行** - メイン学習ループ
9. **楽曲生成** - 学習済みモデルで生成

---

## 順次処理戦略 (Sequential Processing Strategy)

Google Colabのディスク容量制限（特に無料枠や標準インスタンスの場合）を回避するため、以下のループ処理を採用して効率的にデータセットを処理します。

1. **抽出 (Extract)**:
    ZIPバッチ（例: `archive_batch_xxxx.zip`）を1つずつ `DATA_DIR` に展開します。

2. **トークン化 (Tokenize)**:
    展開された `DATA_DIR` 内のすべてのWAVファイルを EnCodec モデルで処理し、結果のテンソルを `TOKEN_DIR` に保存します。

3. **クリーンアップ (Cleanup)**:
    処理が完了したWAVファイルを即座に削除し、ディスク領域を解放します。

このサイクルを全てのバッチに対して繰り返すことで、一時的なディスク使用量を最小限に抑えつつ、大規模なデータセット全体のトークン化を完了させることができます。

---

## 💾 保存構造

学習中、全てのモデルとチェックポイントがGoogle Driveに自動保存されます：

```text
/content/drive/MyDrive/MusicGen_Checkpoints/
│
├── full_models/                    # 完全なモデル（モデル + LoRA）
│   ├── best_model.pt              # 最良の損失を記録したモデル
│   ├── final_model.pt             # 学習完了時の最終モデル
│   └── checkpoint_epoch_*.pt      # 各エポック終了時のスナップショット
│
├── lora_weights/                   # LoRA重みのみ（軽量版）
│   ├── best_lora.pt               # ベストモデルのLoRA重み
│   ├── final_lora.pt              # 最終モデルのLoRA重み
│   ├── latest_lora.pt             # 最新のLoRA重み
│   └── lora_epoch_*.pt            # 各エポックのLoRA重み
│
└── latest.pt                       # 最新チェックポイント（再開用）
```

### 保存されるファイル

| ファイル | 説明 | サイズ目安 |
|---------|------|-----------|
| `best_model.pt` | 最良損失のモデル | 数GB |
| `final_model.pt` | 最終モデル | 数GB |
| `*_lora.pt` | LoRA重みのみ | 数十MB |
| `checkpoint_step_*.pt` | 500ステップごとの中間保存 | 数GB |

**注意**: LoRA重みのみのファイル（`*_lora.pt`）は容量が小さく、ダウンロードや共有に便利です。

---

## 📊 WandBでのモニタリング

学習中、以下のメトリクスがWandBで自動的にトラッキングされます：

### トークナイズ段階

- 処理速度（ファイル/秒）
- 推定残り時間
- 成功/失敗数

### 学習段階

- 訓練損失（ステップごと・エポックごと）
- 学習率
- エポック処理時間
- ベストモデルの更新履歴

### アーティファクト

- ベストモデルのLoRA重み
- 最終モデルのLoRA重み

WandBダッシュボード: https://wandb.ai/

---

## 🔄 学習の再開

ノートブックは自動的にチェックポイントから再開します：

1. `latest.pt` が存在する場合、自動的にロードされます
2. エポック、ステップ、オプティマイザの状態が復元されます
3. ベストモデルの情報も引き継がれます

手動で特定のチェックポイントから再開する場合は、学習セルで以下を変更：

```python
latest_ckpt = '/content/drive/MyDrive/MusicGen_Checkpoints/full_models/checkpoint_epoch_3.pt'
```

---

## ⚙️ 推奨ハイパーパラメータ

### 初期設定（A100 80GB）

```python
lora_r = 8                        # LoRAランク（16まで増やせる）
lora_alpha = 32                   # LoRAアルファ
learning_rate = 1e-4              # 学習率
batch_size = 2                    # バッチサイズ
gradient_accumulation_steps = 8   # 勾配累積（実効バッチ=16）
num_epochs = 5                    # エポック数
```

### GPU別の推奨設定

| GPU | batch_size | grad_accum | 実効バッチ |
|-----|-----------|-----------|----------|
| A100 (80GB) | 4 | 4 | 16 |
| V100 (32GB) | 2 | 8 | 16 |
| T4 (16GB) | 1 | 16 | 16 |

---

## 🎵 楽曲生成

学習後、最終セルで楽曲を生成できます：

```python
descriptions = [
    "A dynamic heavy metal song with fast drums and guitar solo",
    "Relaxing jazz piano with soft background ambience",
    "Upbeat electronic dance music with strong bass",
]
```

生成された楽曲は以下に保存されます：

- ローカル: `/content/generated_music/`
- Google Drive: `/content/drive/MyDrive/MusicGen_Generated/`

---

## 🛠️ トラブルシューティング

### Out of Memory (OOM) エラー

- `batch_size` を 1 に減らす
- `gradient_accumulation_steps` を増やす
- GPU メモリをクリア: `torch.cuda.empty_cache()`

### チェックポイントが見つからない

- Google Driveが正しくマウントされているか確認
- パスが正しいか確認: `/content/drive/MyDrive/MusicGen_Checkpoints/`

### トークナイズが遅い

- 最初の100ファイルで推定時間を確認
- 必要に応じてサンプル数を減らしてテスト

### WandBログインエラー

- シークレットに`WANDB_API_KEY`が設定されているか確認
- APIキーが有効か確認: https://wandb.ai/settings

---

## 📝 ベストプラクティス

1. **テストを必ず実行** - 本番前に5-10サンプルで動作確認
2. **小規模から開始** - 最初は500-1000サンプルで試す
3. **定期的にバックアップ** - 重要なチェックポイントをローカルに保存
4. **WandBで監視** - 損失が正常に減少しているか確認
5. **LoRA重みを活用** - 軽量な`*_lora.pt`ファイルを共有用に使用

---

## 📚 参考資料

- [MusicGen公式リポジトリ](https://github.com/facebookresearch/audiocraft)
- [LoRA論文](https://arxiv.org/abs/2106.09685)
- [WandBドキュメント](https://docs.wandb.ai/)
- [Google Colab FAQ](https://research.google.com/colaboratory/faq.html)

---

## 🤝 貢献

Issues や Pull Requests を歓迎します！

---

## 📄 ライセンス

このプロジェクトはMITライセンスの下で公開されています。