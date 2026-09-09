# 00: Cコードから実行ファイルができるまで

## 今回の問い

3行のCコードは、どのような段階を通って実行可能なファイルになるのか。

## 環境

- CPUアーキテクチャ: Apple Silicon（arm64）
- OS: macOS 15.6.1（Darwin 24.6.0）
- コンパイラ: Apple clang 17.0.0
- デバッガ: LLDB 17.0.0
- 実行ファイル形式: Mach-O 64-bit arm64

ここで重要なのは、今後観察する命令がARM64用であることです。別のCPU、たとえばIntel／AMDのx86-64では、同じCコードでも異なる命令が生成されます。

## 元のCコード

```c
int main(void) {
    return 0;
}
```

このプログラムは、OSへ終了ステータス`0`を返します。一般に`0`は正常終了を表します。

## 予想

- コンパイラは`return 0;`を、戻り値を表す場所へ`0`を設定する命令に変換する
- 完成した実行ファイルには、`main`以外にも実行開始や終了に必要な情報が含まれる
- Cソース、アセンブリ、オブジェクトファイル、実行ファイルはそれぞれ異なる形式になる

## 4つの段階

### 1. 前処理

```sh
mkdir -p work/00-build-pipeline
clang -E experiments/00-build-pipeline/main.c \
  -o work/00-build-pipeline/main.i
```

`#include`やマクロを処理し、コンパイラが読むCコードを作ります。今回はヘッダーもマクロもないため、内容は元のコードとほぼ同じです。

### 2. コンパイル

```sh
clang -O0 -S experiments/00-build-pipeline/main.c \
  -o work/00-build-pipeline/main.s
```

CコードをARM64のアセンブリへ変換します。`-O0`は最適化を無効にし、最初の観察をしやすくする指定です。

### 3. アセンブル

```sh
clang -c work/00-build-pipeline/main.s \
  -o work/00-build-pipeline/main.o
```

アセンブリを機械語へ変換し、Mach-O形式のオブジェクトファイルを作ります。この時点では、まだ単独で実行できません。

### 4. リンク

```sh
clang work/00-build-pipeline/main.o \
  -o work/00-build-pipeline/main
```

オブジェクトファイルと、プログラムの開始・終了などに必要な仕組みを結びつけて実行ファイルを作ります。

```text
main.c ──前処理──> main.i ──コンパイル──> main.s
                                           │
                                      アセンブル
                                           ↓
                                        main.o
                                           │
                                         リンク
                                           ↓
                                         main
```

## 観察コマンド

ファイル形式を比較します。

```sh
file work/00-build-pipeline/main.i \
     work/00-build-pipeline/main.s \
     work/00-build-pipeline/main.o \
     work/00-build-pipeline/main
```

プログラムを実行し、終了ステータスを確認します。

```sh
work/00-build-pipeline/main
echo $?
```

`0`と表示されれば、Cの`return 0;`が最終的にOSへ届いたことを確認できます。

実行ファイルの`main`を逆アセンブルします。

```sh
otool -tvV work/00-build-pipeline/main
```

## 実測した結果

`file`で確認すると、途中生成物は次のように変化しました。

```text
main.i: c program text, ASCII text
main.s: assembler source text, ASCII text
main.o: Mach-O 64-bit object arm64
main:   Mach-O 64-bit executable arm64
```

`main`を実行した終了ステータスは`0`でした。

## アセンブリで見る`main`

この環境の`-O0`では、`main`の中心部分は次の形になりました。

```asm
sub sp, sp, #16
mov w0, #0
str wzr, [sp, #12]
add sp, sp, #16
ret
```

現時点では、各命令を暗記する必要はありません。まず次だけを押さえます。

- `sp`はスタックの現在位置を指すスタックポインタ
- `w0`は関数の整数の戻り値に使われるレジスタ
- `wzr`は読むと常に`0`になるARM64固有のゼロレジスタ
- `ret`は呼び出し元へ戻る命令

`mov w0, #0`によって、`main`の戻り値として`0`が設定されています。それ以外のスタック操作がなぜ存在するかは、関数呼び出しとスタックの実験で改めて扱います。

## この実験で分かったこと

- Cソースは、そのままCPUが読むわけではない
- コンパイラとアセンブラを経てARM64の機械語になる
- リンカがオブジェクトファイルを実行可能なMach-Oファイルにする
- CPUアーキテクチャによって生成される命令が変わる
- Cの`return 0;`は、ARM64の呼び出し規約に従って`w0`へ`0`を置く命令になる
- プログラムの戻り値は、最終的に終了ステータスとしてOSから観察できる

## まだ保留する疑問

- なぜ最適化なしでは、戻り値とは別にスタックへ`0`を書き込むのか
- `main`を誰が呼び出しているのか
- オブジェクトファイルと実行ファイルの中身は、具体的にどう違うのか
- CPUはどこから最初の命令を読み始めるのか

これらは後続の実験で一つずつ回収します。
