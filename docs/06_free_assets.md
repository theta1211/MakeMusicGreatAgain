# 06. 無料素材(音色)の調査と方針

方針: **無料素材のみを使う**。商用利用・再配布(アプリへの同梱)が認められているものに限る。

> ライセンスは 2026 年 11 月時点の Web 調査に基づく。素材ごとに配布元の LICENSE / readme を確認し、[素材台帳](#5-素材台帳の運用)に記録してから使うこと。

## 1. 使ってよいライセンス

| ライセンス | 可否 | 条件 |
|---|---|---|
| CC0 / パブリックドメイン | ◎ | なし(クレジットは任意だが記載する) |
| MIT | ◎ | 著作権表示とライセンス文をアプリ内に表示 |
| CC-BY(3.0 / 4.0) | ○ | 作者名・ライセンス名をアプリ内に表示 |
| 独自の「商用可」ライセンス | △ | 条文を個別に確認。出所が不明確なものは避ける |
| CC-BY-NC(非営利のみ)、CC-BY-ND、CC Sampling Plus | × | アプリ同梱・商用に使えない / 改変できない |
| ライセンス不明・「フリー」とだけ書かれたもの | × | 使わない |

## 2. 楽器別の候補

### 2.1 GM 音源一式(最初の土台。128 音色をまとめて確保)
| 素材 | ライセンス | サイズ目安 | 評価 |
|---|---|---|---|
| **MuseScore_General** | MIT(一部 CC0 / パブリックドメイン) | 数十〜200MB 程度 | **◎ 第一候補**。FluidR3 系を改良したもの |
| FluidR3_GM | MIT | 約 140MB | ○ 予備 |
| GeneralUser GS | 独自ライセンス(商用利用可と明記) | 約 30MB | △ 軽くて音も良いが、作者自身が一部サンプルの出所を確認しきれていないと述べている。採用するなら作者に確認 |

→ MuseScore_General から**必要な音色だけを抜き出して軽量化**し、アプリに同梱する。

### 2.2 ピアノ
| 素材 | ライセンス | 評価 |
|---|---|---|
| Salamander Grand Piano | CC-BY 3.0(作者 Alexander Holm) | ○ 定番で高音質。クレジット表示が必要。ベロシティ層を減らして軽量化 |
| Splendid Grand Piano | パブリックドメイン(SFZ Instruments の一覧による) | ○ |
| VCSL のキーボード類 | CC0 | ○ |

### 2.3 ギター
| 素材 | ライセンス | 評価 |
|---|---|---|
| sfzinstruments(GitHub)のアコースティックギター | CC0 とされる(作者のフォーラム投稿による。リポジトリで要確認) | ○ 要確認 |
| MuseScore_General のギター音色 | MIT | ○ 確実に使える土台 |
| Karoryfer のエレキギター(Emily 等) | 要確認(Karoryfer の新しい無料素材は CC0 が多い) | △ 要確認 |
| **物理モデル(DaisySP StringVoice / Karplus-Strong)** | 自前コード | ◎ 素材不要。ディストーションと組み合わせてエレキ風に |

### 2.4 ベース
| 素材 | ライセンス | 評価 |
|---|---|---|
| Karoryfer Pastabass、Ergo など | CC0 | ◎ |
| シンセベース | 自前シンセ | ◎ 素材不要 |

### 2.5 ドラム
| 素材 | ライセンス | 評価 |
|---|---|---|
| **電子ドラム(808 / 909 風)** | 自前合成(DaisySP) | ◎ 素材不要・サイズ 0 |
| SM Drums | パブリックドメイン | ◎ 生ドラム。元は 2.2GB → ワンショット数音に絞って数 MB に |
| Virtuosity Drums(Versilian) | CC0 | ◎ ジャズ系キット |
| Karoryfer のドラム(Big Rusty Drums、Swirly Drums 等) | CC0 | ◎ ロック / インディー系 |

### 2.6 ストリングス・管楽器・その他
| 素材 | ライセンス | 評価 |
|---|---|---|
| VSCO 2 Community Edition | CC0(「Vanilla」版。派生版は他素材を含むため除外) | ◎ 弦・管・打楽器 |
| VCSL(Versilian Community Sample Library) | CC0 | ◎ 多目的 |
| Karoryfer Weresax、Bigcat Cello 等 | CC0 | ○ |

### 2.7 効果音・ワンショット
| 素材 | ライセンス | 評価 |
|---|---|---|
| Freesound(CC0 で絞り込み) | CC0 のものだけ | ○ 1 音ずつライセンスを記録 |

## 3. 避けるもの(例)
- Sonatina Symphonic Orchestra(CC Sampling Plus 1.0 → 商用同梱に不向き)
- CC-BY-NC 系のギター / ピアノ素材(無料配布でも非営利限定のもの)
- 「Freepats」の GM セット(個々の音色でライセンスが異なり、CC0 でないものを含む)

## 4. サイズ対策(端末を広くカバーするため)

| 対策 | 効果 |
|---|---|
| 合成で作れる音(シンセ、電子ドラム、シンセベース、弦モデル)は素材を使わない | 同梱サイズ 0 |
| サンプルは 16bit・モノラル(ピアノ等を除く)・ベロシティ層 2〜3・ループ化 | 元素材の 1/10〜1/50 に |
| SF2 に変換して 1 楽器 5〜20MB を目安にする | |
| 同梱は「基本パック」(合計 60MB 以下)のみ。追加音色は **Play Asset Delivery(オンデマンド)** で後から無料ダウンロード | 初回インストールを軽く、低容量端末にも入る |
| 低メモリ端末では、使用中のトラックの音色だけを読み込む | メモリ 300MB 以下を維持 |

### 同梱する「基本パック」案
| 分類 | 内容 | 方式 |
|---|---|---|
| シンセ | リード、パッド、ベース、プラック等 プリセット 30 | 合成 |
| ドラム | 808、909、エレクトロ(合成)/ アコースティック 1 キット(SM Drums から) | 合成 + サンプル |
| ギター | アコギ、クリーンエレキ(サンプル)、ディストーションギター(サンプル + エフェクト)、弦モデル | サンプル + 合成 |
| ピアノ / キーボード | ピアノ(軽量版)、エレピ、オルガン | サンプル |
| ベース | エレキベース、シンセベース | サンプル + 合成 |
| その他 | ストリングス、ブラス | サンプル(VSCO 2 CE / MuseScore_General) |

## 5. 素材台帳の運用

リポジトリに `assets-licenses/ledger.csv` を置き、素材 1 つごとに記録する。

| 列 | 例 |
|---|---|
| 素材 ID | `piano_salamander_lite` |
| 元素材名 / URL | Salamander Grand Piano v3 / (配布元 URL) |
| 作者 | Alexander Holm |
| ライセンス | CC-BY-3.0 |
| 取得日 | 2026-11-13 |
| 加工内容 | 48kHz 16bit に変換、ベロシティ 3 層に削減、SF2 化 |
| 表示義務 | あり(クレジット画面) |
| 確認者 | (氏名) |

- アプリの「設定 > クレジットとライセンス」に、台帳から自動生成した一覧を表示する。
- ライセンスファイルの写しを `assets-licenses/<素材 ID>/` に保存しておく。

## 参考資料(2026-11 調査)
- SFZ Instruments(ライセンス付きの無料素材一覧): https://sfzinstruments.github.io/
- Salamander Grand Piano: https://sfzinstruments.github.io/pianos/salamander
- VSCO 2 CE: https://versilian-studios.com/vsco-community/ / https://github.com/sgossner/VSCO-2-CE
- VCSL: https://github.com/sgossner/VCSL
- Virtuosity Drums: https://versilian-studios.com/virtuosity-drums/
- SM Drums: https://sfzinstruments.github.io/drums/sm_drums
- MuseScore_General のライセンス: https://src.fedoraproject.org/rpms/mscore/blob/f36/f/MuseScore_General_License.md
- GeneralUser GS のライセンス: https://scancode-licensedb.aboutcode.org/generaluser-gs-2.0.html
- CC0 の生楽器素材についての議論: https://www.kvraudio.com/forum/viewtopic.php?p=9179000
