
# Aivis

💠 **Aivis:** **AI** **V**oice **I**mitation **S**ystem

<img width="100%" alt="image" src="https://github.com/tsukumijima/Aivis/assets/39271166/c5b2a9cd-74ec-4b4b-8981-4ed81d0f4345">

## Overview

**Aivis は、高音質で感情豊かな音声を生成できる [Bert-VITS2](https://github.com/fishaudio/Bert-VITS2) 用のデータセットの作成・学習・推論を、オールインワンで行えるツールです。**

通常、専用に作成された音声コーパス以外の音源から学習用データセットを作成するには、膨大な手間と時間がかかります。  
Aivis では、**音声コーパスからデータセットを作成するための工程を AI で可能な限り自動化し、さらに最終的な人力でのアノテーション作業を簡単に行える Web UI を通して、データセット作成の手間と時間を大幅に削減します。**  
さらに Bert-VITS2 でのモデルの学習や推論 (Web UI の起動) も、簡単なコマンドひとつで実行できます。

https://github.com/tsukumijima/Aivis/assets/39271166/2234e7c3-6d08-4d26-ae63-c451ef61923a

大元の音源の量・質・話し方にもよりますが、上のサンプル音声の通り、音声コーパスを使い学習させたモデルと比べても遜色ないクオリティの音声を生成できます。  
Bert-VITS2 の事前学習モデル自体の性能が高いようで、私の環境ではわずか3分弱のデータセットを学習したモデルでも (イントネーションはともかく) かなり近い声質 & 明瞭な発音の音声を生成できています。

> [!NOTE]  
> Aivis では、実用途に合わせて細部を調整した [フォーク版の Bert-VITS2](https://github.com/tsukumijima/Bert-VITS2) を利用しています。  
> 今のところ学習/推論アルゴリズムは変更していません。設定ファイルのデフォルト値や学習時に必要なモデルの自動ダウンロード以外は、オリジナルの Bert-VITS2 と同等です。

## Installation

Linux (Ubuntu 20.04 LTS) x64 でのみ検証しています。  
Windows 上での動作は想定していません。Windows では WSL2 を使ってください (動作未検証) 。

当然ですが、Aivis の実行には NVIDIA GPU が必要です。  
Geforce GTX1080 (VRAM 8GB) での動作を確認しています。VRAM は最低でも 8GB は必要なはずです (VRAM 12GB のグラボが欲しい…) 。

### Non-Docker

Docker を使わない場合、事前に Python 3.11・Poetry・FFmpeg がインストールされている必要があります。

```bash
# サブモジュールが含まれているため --recurse を付ける
git clone --recurse https://github.com/tsukumijima/Aivis.git

# 依存関係のインストール
cd Aivis
poetry env use 3.11
poetry install --no-root

# ヘルプを表示
./Aivis.sh --help
```

### Docker

Docker を使う場合、事前に Docker がインストールされている必要があります。  
Docker を使わない場合と比べてあまり検証できていないため、うまく動かないことがあるかもしれません。


```bash
# サブモジュールが含まれているため --recurse を付ける
git clone --recurse https://github.com/tsukumijima/Aivis.git

# 依存関係のインストール
cd Aivis
./Aivis-Docker.sh build

# ヘルプを表示
./Aivis-Docker.sh --help
```

## Dataset Directory Structure

Aivis のデータセットディレクトリは、4段階に分けて構成されています。

- **01-Sources:** データセットにする音声をそのまま入れるディレクトリ
  - データセットの素材にする音声ファイルをそのまま入れてください。
    - 基本どの音声フォーマットでも大丈夫です。`create-segments` での下処理にて、自動的に wav に変換されます。
    - 背景 BGM の除去などの下処理を行う必要はありません。`create-segments` での下処理にて、自動的に BGM や雑音の除去が行われます。
    - 数十分〜数時間ある音声ファイルの場合は `create-segments` での書き起こしの精度が悪くなることがあるため、事前に10分前後に分割することをおすすめします。
  - `create-segments` サブコマンドを実行すると、BGM や雑音の除去・書き起こし・一文ごとのセグメントへの分割・セグメント化した音声の音量/フォーマット調整が、すべて自動的に行われます。
- **02-PreparedSources:** `create-segments` サブコマンドで下処理が行われた音声ファイルと、その書き起こしテキストが入るディレクトリ
  - `create-segments` サブコマンドを実行すると、`01-Sources/` にある音声ファイルの BGM や雑音が除去され、このディレクトリに書き起こしテキストとともに保存されます。
  - `create-segments` の実行時、このディレクトリに当該音声の下処理済みの音声ファイルや書き起こしテキストが存在する場合は、そのファイルが再利用されます。
  - 下処理済みの音声ファイル名は `02-PreparedSources/(01-Sourceでのファイル名).wav` となります。
  - 書き起こしテキストのファイル名は `02-PreparedSources/(01-Sourceでのファイル名).json` となります。
    - 書き起こしの精度がよくない (Whisper の音声認識ガチャで外れを引いた) 場合は、書き起こしテキストの JSON ファイルを削除してから `create-segments` を実行すると、再度書き起こし処理が行われます。
- **03-Segments:** `create-segments` サブコマンドでセグメント化された音声ファイルが入るディレクトリ
  - `create-segments` サブコマンドを実行すると、`02-PreparedSources/` 以下にある音声ファイルが書き起こし文や無音区間などをもとに一文ごとにセグメント化され、このディレクトリに保存されます。
  - セグメントデータのファイル名は `03-Segments/(01-Sourceでのファイル名)/(4桁の連番)_(書き起こし文).wav` となります。
    - 基本発生しませんが、万が一書き起こし文がファイル名の最大長を超える場合は、ファイル名が切り詰められ、代わりにフルの書き起こし文が `03-Segments/(01-Sourceでのファイル名)/(4桁の連番)_(書き起こし文).txt` に保存されます。
  - なんらかの理由でもう一度セグメント化を行いたい場合は、`03-Segments/(01-Sourceでのファイル名)/` を削除してから `create-segments` を実行すると、再度セグメント化が行われます。
- **04-Datasets:** `create-datasets` サブコマンドで手動で作成されたデータセットが入るディレクトリ
  - `create-datasets` サブコマンドを実行すると Gradio の Web UI が起動し、`03-Segments/` 以下にある一文ごとにセグメント化された音声と書き起こし文をもとにアノテーションを行い、手動でデータセットを作成できます。
  - `03-Segments/` までの処理は AI 技術を使い完全に自動化されています。
    - 調整を重ねそれなりに高い精度で自動生成できるようになった一方で、他の人と声が被っていたり発音がはっきりしないなど、データセットにするにはふさわしくない音声が含まれていることもあります。
    - また、書き起こし文が微妙に誤っていたり、句読点がなかったりすることもあります。
    - さらに元の音声に複数の話者の声が含まれている場合、必要な話者の音声だけを抽出する必要もあります。
  - `create-datasets` サブコマンドで起動する Web UI は、どうしても最後は人力で行う必要があるアノテーション作業を、簡単に手早く行えるようにするためのものです。
    - 話者の選別 (データセットから除外することも可能)・音声の再生・音声のトリミング (切り出し)・書き起こし文の修正を一つの画面で行えます。
    - 確定ボタンを押すと、そのセグメントが指定された話者のデータセットに追加されます (データセットからの除外が指定された場合はスキップされる) 。
    - `create-datasets` サブコマンドによって、`03-Segments/` 以下のセグメント化された音声ファイルが変更されることはありません。
  - データセットは音声ファイルが `04-Datasets/(話者名)/audio/wavs/(連番).wav` に、書き起こし文が `04-Datasets/(話者名)/filelists/speaker.list` にそれぞれ保存されます。
    - このディレクトリ構造は Bert-VITS2 のデータセット構造に概ね準拠したものですが、config.json など一部のファイルやディレクトリは存在しません。
    - `train` サブコマンドを実行すると、指定された話者のデータセットディレクトリが Bert-VITS2 側にコピーされ、別途 config.json など必要なファイルもコピーされた上で Bert-VITS2 の学習処理が開始されます。
    - Bert-VITS2 の学習処理によって、`04-Datasets/` 以下のデータセットが変更されることはありません。

## Usage

概ね前述した通りですが、念のためにここでも説明します。  
ここでは、学習するモデルの名前を「MySpeaker1」「MySpeaker2」とします。

### 1. データセットの準備

`01-Sources/` 以下に、データセットにする音声ファイルをそのまま入れます。

基本どの音声フォーマットでも大丈夫です。`create-segments` での下処理にて、自動的に wav に変換されます。  
また、背景 BGM の除去などの下処理を行う必要はありません。`create-segments` での下処理にて、自動的に BGM や雑音の除去が行われます。

なお、数十分〜数時間ある音声ファイルの場合は `create-segments` での書き起こしの精度が悪くなることがあるため、事前に10分前後に分割することをおすすめします。

### 2. データセットの下処理とセグメント化

```bash
# Non-Docker
./Aivis.sh create-segments

# Docker
./Aivis-Docker.sh create-segments
```

実行すると、音源抽出 AI の [Demucs (htdemucs_ft)](https://github.com/facebookresearch/demucs) により `01-Sources/` 以下にある音声ファイルの BGM や雑音が除去され、書き起こしテキストとともに `02-PreparedSources/` 以下に保存されます。  
書き起こしテキストは音声認識 AI の [faster-whisper (large-v3)](https://github.com/SYSTRAN/faster-whisper) によって生成され、[stable-ts](https://github.com/jianfch/stable-ts) によってアライメントされます。  
すでに `02-PreparedSources/` 以下に当該音声の下処理済みの音声ファイルや書き起こしテキストが存在する場合は、そのファイルが再利用されます。

上記の処理が完了すると、`02-PreparedSources/` 以下にある音声ファイルが書き起こし文や無音区間などをもとに一文ごとにセグメント化され、`03-Segments/` 以下に保存されます。  
すでに `03-Segments/` 以下に当該音声のセグメント化された音声ファイルが存在する場合は、当該音声のセグメント化はスキップされます。

### 3. データセットの作成 (アノテーション)

<img width="100%" alt="image" src="https://github.com/tsukumijima/Aivis/assets/39271166/a8e0412d-4bee-4496-a723-70114df9cb48"><br><br>

```bash
# Non-Docker
./Aivis.sh create-datasets '*' 'MySpeaker1,MySpeaker2'

# Docker
./Aivis-Docker.sh create-datasets '*' 'MySpeaker1,MySpeaker2'
```

Aivis でのデータセット作成工程は、大半が `create-segments` サブコマンドで自動化されています。  
しかし、話者やセグメント時代の選別・書き起こし文の修正・うまく切り出せていない音声のトリミングなどのアノテーション作業は、どうしても人力で行う必要があります。

`create-datasets` サブコマンドを実行すると、Gradio の Web UI が起動します。  
この Web UI から `03-Segments/` 以下の一文ごとにセグメント化された音声と書き起こし文をもとにアノテーションを行い、人力でアノテーションを行った最終的なデータセットを作成できます。

`create-datasets` サブコマンドの第一引数には、`03-Segments/` 以下に自動生成されている、セグメント化された音声ファイルのディレクトリ名を指定します。通常は `01-Sources/` 以下の音声ファイル名と同じです。  
内部では Glob が使われているため、ワイルドカード (`*`) を活用し、複数のディレクトリのアノテーション作業を一括で実行できます。

`create-datasets` サブコマンドの第二引数には、データセットを作成する話者の名前をカンマ区切りで指定します。  
ここで指定した話者のデータセットが、`04-Datasets/` 以下に作成されます。  
Web UI 上では、セグメント化された音声ファイルごとにどの話者に割り当てるか、あるいはデータセットから除外するかを選択できます。

Web UI 上で確定ボタンを押すと、次のセグメントのアノテーション作業に移ります。  
実装上、一度確定したセグメントのアノテーションをやり直すことはできません。間違いがないかよく確認してください。  
作成中のデータセットの進捗ログは、`create-datasets` サブコマンドの標準出力に表示されます。

> [!NOTE]  
> すでにデータセットが途中まで作成されている状態で再度 `create-datasets` サブコマンドを実行すると、途中まで作成されているデータセットの次の連番から、データセット作成が再開されます。  
> 最初からアノテーション作業をやり直したい場合は、`04-Datasets/` 以下の話者ごとのデータセットディレクトリを削除してから、再度 `create-datasets` サブコマンドを実行してください。

```bash
# Non-Docker
./Aivis.sh check-dataset 'MySpeaker1'

# Docker
./Aivis-Docker.sh check-dataset 'MySpeaker1'
```

`check-dataset` サブコマンドを実行すると、指定された話者のデータセットディレクトリにある音声ファイルと書き起こし文、音声ファイルの総時間を確認できます。  

`create-datasets` サブコマンドの第一引数には、データセットを確認したい話者の名前 (`04-Datasets/` 以下のディレクトリ名と一致する) を指定します。ワイルドカードは使えないため注意してください。

### 4. 学習の実行

<img width="100%" alt="image" src="https://github.com/tsukumijima/Aivis/assets/39271166/6d67a57b-d53e-465c-a454-d94d981278ad"><br><br>

-----

```bash
# Non-Docker
./Aivis.sh train 'MySpeaker1' --epochs 50 --batch-size 4

# Docker
./Aivis-Docker.sh train 'MySpeaker1' --epochs 50 --batch-size 4
```

`train` サブコマンドを実行すると、指定された話者のデータセットディレクトリが Bert-VITS2 側にコピーされ、別途 config.json など必要なファイルもコピーされた上で Bert-VITS2 の学習処理が開始されます。

NVIDIA GPU のスペックにもよりますが、学習の完了には相応の時間がかかります。  
Geforce GTX1080 (バッチサイズ: 4) で 3000 エポック以上学習させたときは1時間15分程度かかりました。

一般的に、2000 ステップ 〜 4000 ステップで十分似た声質になるようです。最大でも 7000 ステップ程度学習させれば十分でしょう。

> [!NOTE]  
> ステップ数は、`(データセットの総数 ÷ バッチサイズ) × エポック数` で求められます。データセットの総数が少ない場合は、エポック数を増やして 3000 ステップ程度になるように調整すると良いでしょう。

> [!NOTE]  
> VRAM 不足で実行途中に CUDA Out Of Memory エラーが出る場合は、`--batch-size` で学習時のバッチサイズを小さくしてください。  
> Geforce GTX1080 ではバッチサイズ 3 〜 4 でギリギリな印象です。

学習中は、標準出力に学習の進捗ログが表示されます。  
学習したモデルは `Bert-VITS2/Data/(話者名)/models/` 以下に保存されます。  
モデルは 1000 ステップごとに異なるファイル名で保存されます。  
もしモデルディレクトリに `G_7000.pth` が存在する場合は、7000 ステップで学習したモデルです。

### 5. 学習済みモデルの推論

<img width="100%" alt="image" src="https://github.com/tsukumijima/Aivis/assets/39271166/b2ec1a91-defd-4ba5-aba7-2e41118bc73e"><br><br>

```bash
# Non-Docker
./Aivis.sh infer 'MySpeaker1' --model-step 5000

# Docker
./Aivis-Docker.sh infer 'MySpeaker1' --model-step 5000
```

`infer` サブコマンドを実行すると、引数で指定された話者の学習済みモデルを使って音声を生成できる、推論用 Web UI を起動できます。  
この Web UI はオリジナルの Bert-VITS2 の Web UI を日本語化したもので、コマンド実行時に引数に合わせて `Bert-VITS2/config.yml` が自動的に書き換えられます。

Web UI にはいくつかボタンがありますが、基本は喋らせたい音声を入力して、「スライス生成」をクリックするだけです。  
通常の「音声を生成」ボタンもありますが、スライス生成、つまり改行ごとに音声を生成する方が、より自然な音声を生成できることが多いです。

さらに「文ごとの分割」にチェックを入れると、一文ごとに分割して音声を生成します。より感情豊かな音声を生成できることがある一方、前の文との抑揚のトーンの差が激しくなることがあります。

`--model-step` はオプションで、指定しなかった場合は一番最後に保存されたステップ数のモデルが使われます。  
もし一番最後に保存されたステップ数のモデルでの性能が芳しくない場合は、過学習気味かもしれません。より前の低いステップ数のモデルの方が、発声やイントネーションが安定した音声を生成できることもあります。

## License

[MIT License](License.txt)
