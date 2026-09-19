---
title: "Ternary Bonsai 2 27BをM1Pro・16GBで動かす"
emoji: "🌱"
type: "tech"
topics: ["llamacpp", "llm", "mac", "ローカルllm", "codex"]
published: true
---
こんにちは、はたけです。
今回はM1 Pro・メモリ16GBのMacBook Proで、Ternary Bonsai 2 27Bを動かしてみました。PrismML公式のllama.cpp forkをソースからビルドし、日本語の応答が返るところまで確認できました。

短い動作確認では **8.1 tokens/s**、その後の対話形式での自己紹介では **9.6 tokens/s** と表示されました。本格的なベンチマークではありませんが、手元のMacで27Bクラスのモデルを動かせたのはうれしいですね。

ただ、環境構築はなかなかめんどくさい。
なので、この記事ではPrismML公式のllama.cpp forkを使い、手元のMacで推論できるまでの手順をまとめます。

## 今回の環境と、使ったモデル

| 項目 | 内容 |
| --- | --- |
| 確認日 | 2026年9月19日 |
| Mac | MacBook Pro / Apple M1 Pro |
| CPU | 8コア |
| メモリ | 16GB |
| OS | macOS 26.6.2 |
| 実行エンジン | PrismML公式のllama.cpp fork |
| GPUバックエンド | Metal |
| モデル | Ternary-Bonsai-2-27B-PQ2_0.gguf |
| ファイルサイズ | 7,206,168,928 bytes（約7.21GB / 6.71GiB） |
| コンテキスト長 | 4,096トークン |
| 確認した入力 | テキストのみ |

「約5.9GB」と紹介されるBonsai 2ですが、[公式モデルカード](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf)では2種類のGGUFが配布されています。

| 形式 | モデルファイルのサイズ |
| --- | --- |
| PTQ1_0 | 約5.95GB |
| PQ2_0 | 約7.21GB |

今回は[公式実行ガイド](https://github.com/PrismML-Eng/Bonsai-demo)の既定の選択に合わせ、PQ2_0を使いました。以降の結果も、このファイルでの結果です。ファイルサイズと推論時のメモリ使用量は同じではなく、実行時にはキャッシュや計算用の領域も必要になります。

## Ollamaでは動かなかった

普段はOllamaでlocalLLMを動かしているので、今回もOllamaで起動したいと考えました。
次のコマンドで実行を試みました。

```bash
ollama run hf.co/prism-ml/Ternary-Bonsai-2-27B-gguf:Q2_0
```

次のエラーが出てしまい、実行できませんでした。

```text
Error: tensor "output.weight" size overflow
```

また、源 勝さんの[note記事「Ternary Bonsai 2 27B を Ollama で試してみた。 2bit で 27B が約 7GB ？と思ったら動かなかった」](https://note.com/minamotomasaru/n/n07649e8c3a7f)でも同様のエラーで動かなかったと報告されています。c

### Bonsai 2については、公式の対応状況を確認する

今回使うBonsai 2については、[公式実行ガイドの対応状況](https://github.com/PrismML-Eng/Bonsai-demo#upstream-status-for-bonsai-2)を確認しました。2026年9月19日の確認時点では、実行時に必要なHadamard変換が本家llama.cppにまだ取り込まれておらず、PrismMLのforkが必要とされています。通常のllama.cppではPQ2_0とPTQ1_0の読み込みが拒否されるという説明もあります。

うーん、めんどくさい。

ただそうは言っても、Ollamaで動かすにはGGUFが少し違うので無理っぽい。
そういうわけで不本意ながら、PrismML公式のllama.cpp forkをビルドする方法に切り替えました。以下は、その構成で日本語の推論まで確認できた手順です。

## 最初に整理した「fork」と「サンドボックス」

なかなかめんどくさい作業だったので、Codexに環境構築を任せることにしました。
また、Codexには「サンドボックス環境でllama.cpp forkを作り、Bonsai 2を動かしたい」と依頼しました。

実際に行ったのは、[PrismML公式fork](https://github.com/PrismML-Eng/llama.cpp)を手元にcloneし、ローカルの作業ブランチを作ることです。自分のGitHubアカウントに新しいforkを公開したわけではありません。推論エンジンのソースコードも変更していません。

また、環境は専用フォルダにまとめましたが、DockerやVMによる隔離ではありません。

Codexの制限付き実行環境からデバイスを調べた際は、MetalのGPU情報を正常に取得できませんでした。一方、実行許可を得て通常プロセスとして確認すると、次のように認識されました。

```text
Available devices:
  MTL0: Apple M1 Pro (12124 MiB, 12123 MiB free)
  BLAS: Accelerate (0 MiB, 0 MiB free)
```

そのため、最終的なGPU推論はMac上の通常プロセスとして実行しました。「専用フォルダに環境をまとめること」と「OSレベルで隔離すること」は別、というのが今回の整理です。

## 1. 公式forkを取得してビルドする

以下のコマンドは、今回実行した内容を記事用の作業ディレクトリに置き換えたものです。Macのターミナルで実行します。Git、Python 3、AppleのC/C++ビルド環境が必要です。

```bash
mkdir -p bonsai2/work bonsai2/outputs/models
cd bonsai2

git clone --depth 1 --branch prism \
  https://github.com/PrismML-Eng/llama.cpp.git outputs/llama.cpp

git -C outputs/llama.cpp switch -c bonsai2-sandbox
```

通常のllama.cppをそのまま使うのではなく、Bonsai 2に対応したPrismMLのforkを使います。対応するランタイムについては、[公式実行ガイド](https://github.com/PrismML-Eng/Bonsai-demo)を確認してください。

今回ビルドしたコミットは次のものです。上のcloneコマンドは実行時点の`prism`ブランチを取得するため、将来の実行では異なるコミットになります。

```text
9a9394a895b96003ca842a6041cb28ac49a108f7
```

CMakeが入っていなかったため、作業フォルダ内のPython仮想環境にインストールしました。今回使ったCMakeは4.4.3です。

```bash
python3 -m venv work/build-env
work/build-env/bin/pip install cmake==4.4.3

work/build-env/bin/cmake \
  -S outputs/llama.cpp \
  -B outputs/llama.cpp/build \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_METAL=ON \
  -DGGML_NATIVE=OFF \
  -DLLAMA_BUILD_TESTS=OFF \
  -DLLAMA_BUILD_EXAMPLES=OFF \
  -DLLAMA_BUILD_UI=OFF \
  -DLLAMA_OPENSSL=OFF

work/build-env/bin/cmake --build outputs/llama.cpp/build \
  --target llama-cli llama-server -j 6
```

今回はCLIでの推論を目的にしたため、Web UIは含めていません。モデル取得には別のダウンローダーを使う構成です。

ビルド後にGPUを確認します。

```bash
outputs/llama.cpp/build/bin/llama-cli --list-devices
```

## 2. モデルをダウンロードする

今回の環境にはHugging Face CLIの`hf`がすでにありました。`hf`がない場合は、先ほどの仮想環境に`huggingface_hub`をインストールし、環境を有効化してから続けます。

```bash
# hfが未インストールの場合
work/build-env/bin/pip install huggingface_hub
source work/build-env/bin/activate
```

以下は`bonsai2`ディレクトリから実行します。ダウンロード対象をファイル名で指定し、リポジトリ全体を取得しないようにしています。

```bash
export HF_HOME="$PWD/work/hf-cache"
export HF_XET_CACHE="$PWD/work/xet-cache"

hf download prism-ml/Ternary-Bonsai-2-27B-gguf \
  Ternary-Bonsai-2-27B-PQ2_0.gguf \
  --revision 6ed5e12bf84b7a63069882c91dd9e9218647d17b \
  --local-dir outputs/models
```

今回の取得環境は`huggingface_hub 0.35.0`、`hf_xet 1.1.10`でした。上の追加インストール手順では、その時点で利用可能なバージョンが入ります。

モデル取得はかなり時間がかかりました。途中で観測した実効速度は約2.4MiB/sでしたが、これは今回の通信環境での値です。

## 3. ダウンロードしたファイルを検証する

取得完了後、配布元のメタデータに記載されたSHA256と照合しました。

```bash
printf '%s\n' \
  '3907dc1658db1f78a9826bf8d5bcb8dc65db0d466388937af57f2294fae62ec1  outputs/models/Ternary-Bonsai-2-27B-PQ2_0.gguf' \
  | shasum -a 256 -c -
```

今回の結果は`OK`でした。このハッシュは、上で指定したモデルのrevisionとファイルに対応しています。

## 4. 日本語で推論してみる

モデルのダウンロードと検証が終わったら、いよいよ推論です。以下のコマンドは、ここまで作業していた`bonsai2`ディレクトリから実行します。モデルを取得し直したり、毎回ビルドしたりする必要はありません。

### まずは一問だけ試す

最初は、短い一問で動作確認しました。

```bash
outputs/llama.cpp/build/bin/llama-cli \
  --model outputs/models/Ternary-Bonsai-2-27B-PQ2_0.gguf \
  --offline \
  --ctx-size 4096 \
  --batch-size 256 \
  --ubatch-size 128 \
  --n-gpu-layers 99 \
  --flash-attn on \
  --jinja \
  --single-turn \
  --simple-io \
  --reasoning off \
  --temp 0 \
  --top-p 0.95 --top-k 20 --min-p 0 \
  --seed 42 \
  --n-predict 64 \
  --prompt '日本語で短く自己紹介してください。'
```

初回確認なので、コンテキストは4,096トークン、生成は最大64トークンに抑えています。このテストではreasoningをオフにしました。

実際に返ってきた応答は次のとおりです。

```text
こんにちは。私はAIアシスタントです。質問やタスクのサポートをします。何かお手伝いできることはありますか？

[ Prompt: 18.7 t/s | Generation: 8.1 t/s ]
```

ファイルの検証に加え、日本語の入力から自然な日本語の応答が返ることまで確認できました。

### 起動スクリプトを作って、対話する

毎回長いコマンドを入力せずに済むように、対話用の設定を`outputs/run-bonsai.sh`にまとめました。次のコマンドでスクリプトを作成できます。

```bash
cat > outputs/run-bonsai.sh <<'SH'
#!/bin/bash
set -euo pipefail
cd "$(dirname "$0")"
exec ./llama.cpp/build/bin/llama-cli \
  --model ./models/Ternary-Bonsai-2-27B-PQ2_0.gguf \
  --offline --ctx-size 4096 \
  --batch-size 256 --ubatch-size 128 \
  --n-gpu-layers 99 --flash-attn on --jinja \
  --reasoning-effort medium \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0 \
  "$@"
SH

chmod +x outputs/run-bonsai.sh
./outputs/run-bonsai.sh
```

次回からは、`bonsai2`ディレクトリで次の1行を実行するだけです。

```bash
./outputs/run-bonsai.sh
```

`Loading model...`の後にモデル情報と入力欄が表示されたら、質問を入力してEnterを押します。私の環境では、次のように表示されました。

```text
build      : b1-9a9394a
model      : ./models/Ternary-Bonsai-2-27B-PQ2_0.gguf
ftype      : PQ2_0 - 2.13 bpw (group 128)
modalities : text
```

ここで「自己紹介してください」と入力しました。回答の抜粋です。途中の思考表示と回答の一部は省略しています。

```text
> 自己紹介してください

こんにちは！私は **Qwen**（通義千問）です。

Alibabaグループの通義ラボにより開発された大規模言語モデルです。

（中略）

何かお手伝いできることはありますか？

[ Prompt: 14.2 t/s | Generation: 9.6 t/s ]
```

自己紹介ではQwenを名乗っていますが、起動時のモデル情報では、指定した`Ternary-Bonsai-2-27B-PQ2_0.gguf`が読み込まれていることを確認できます。

対話用の設定ではreasoningをオフにしていないため、今回のログでも`[Start thinking]`から`[End thinking]`までの思考表示が出た後に、日本語の回答が続きました。最初の一問テストとはプロンプトや設定が異なるので、8.1と9.6という数字だけで速度差の原因は判断できません。

思考表示なしで対話したい場合は、起動時に指定します。

```bash
./outputs/run-bonsai.sh --reasoning off
```

スクリプト経由で一問だけ試すこともできます。

```bash
./outputs/run-bonsai.sh \
  --single-turn --reasoning off --n-predict 128 \
  --prompt '日本語で自己紹介してください。'
```

対話を終了するには`/exit`または`Ctrl+C`、会話履歴を消してやり直すには`/clear`を使います。

## 今回は動作確認まで

今回は、M1 Pro・メモリ16GBのMacで、Ternary Bonsai 2 27Bを読み込み、日本語で対話できるところまで確認しました。PrismML公式のllama.cpp forkを使い、モデル取得後は起動スクリプト1本で試せる状態になりました。

表示された生成速度は、短い動作確認で8.1 tokens/s、対話での自己紹介で9.6 tokens/sでした。ただ、どちらも単発の実行結果です。これだけで「十分速い」「日本語に強い」「コーディングに使える」と判断するには、まだ材料が足りません。

次に試すなら、同じ条件で繰り返し測定した生成速度や応答開始までの時間、メモリ使用量を見てみたいです。日本語の要約やコード生成の品質、長い入力での挙動も気になります。今回はテキスト入力のみで、画像用のmmprojは取得していないため、画像理解も今後の検証対象です。

ひとまず今回は、「手元のMacで動かせた」ところまで。機会があれば、次回は条件をそろえた実測や本格的な検証に進み、その結果も記事として公開したいと思います。