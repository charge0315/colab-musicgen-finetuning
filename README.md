# MusicGen Fine-Tuning on Google Colab

このリポジトリは、Google Colab上でMeta社の**MusicGen-Large**モデルをファインチューニングするためのノートブックを提供します。
独自のデータセット（音声ファイル + テキスト記述）を使用して、特定のジャンルやスタイルの音楽を生成できるモデルを作成することを目的としています。

## 特徴

*   **MusicGen-Large対応**: 高品質な生成が可能なLargeモデルの学習設定済み。
*   **Google Drive連携**: データセットの読み込みと学習済みモデルの保存をGoogle Driveで行います。
*   **データセット処理**: JSONL形式のメタデータと音声ファイルを自動的に処理し、学習用・検証用に分割します。
*   **生成デモ機能**: 学習前のベースモデルと、学習後のファインチューニングモデルの両方で生成テストが可能です。

## 前提条件

*   **Googleアカウント**: Google ColabおよびGoogle Driveを使用するために必要です。
*   **Google Colab Pro/Pro+ (推奨)**: MusicGen-Largeの学習には**A100 GPU**の使用が強く推奨されます。V100でも動作する可能性がありますが、メモリ不足に注意が必要です。
*   **データセット**: 以下の形式でGoogle Driveに保存されている必要があります。

## データセットの構成

Google Driveの `My Drive` 直下に `MusicGen_Dataset` フォルダを作成し、以下の構造で配置してください。

```text
My Drive/
└── MusicGen_Dataset/
    ├── audio/                  # 音声ファイル (.wav) を格納するフォルダ
    │   ├── track1_segment01.wav
    │   ├── track1_segment02.wav
    │   └── ...
    └── metadata.jsonl          # メタデータファイル
```

### metadata.jsonl の形式

各行が1つの音声データに対応するJSONオブジェクトです。

```json
{"path": "audio/track1_segment01.wav", "duration": 30.0, "sample_rate": 32000, "amplitude": 0.15, "description": "A classic rock guitar riff with distortion, upbeat tempo"}
{"path": "audio/track1_segment02.wav", "duration": 30.0, "sample_rate": 32000, "amplitude": 0.14, "description": "Electric guitar solo, fast picking, minor key"}
```

*   `path`: `metadata.jsonl` からの相対パス。
*   `description`: 楽曲のテキスト記述（プロンプト）。

## 使い方

1.  このリポジトリの `MusicGen_FineTuning.ipynb` をGoogle Colabで開きます（GitHubから直接開くか、ダウンロードしてアップロードしてください）。
2.  Colabのメニューから「ランタイム」→「ランタイムのタイプを変更」を選択し、ハードウェアアクセラレータを **GPU** (A100推奨) に設定します。
3.  ノートブックのセルを上から順に実行します。
    *   環境構築
    *   Google Driveのマウント（認証が必要です）
    *   データセットの準備
    *   学習設定の作成
    *   学習の実行 (`dora run`)
    *   生成・推論

## 生成されるモデルについて

*   学習中のチェックポイントは Google Drive の `My Drive/MusicGen_Checkpoints/` に保存されます。
*   学習完了後、推論セクションで保存されたモデルをロードして楽曲生成を行えます。

## ライセンス

このプロジェクトのコードはMITライセンスの下で公開されています。
MusicGenモデル自体のライセンスについては、[Metaの公式リポジトリ](https://github.com/facebookresearch/audiocraft)をご確認ください。
