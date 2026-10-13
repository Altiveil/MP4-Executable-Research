# MP4-Executable-Research

**Investigating the convergence of MP4 media containers and native Windows executables within a single binary.**

[English](#english) · [日本語](#日本語)

---

## English

### Abstract

MP4-Executable-Research is a pre-implementation research project investigating whether an ISO Base Media File Format (ISOBMFF) container and a Windows Portable Executable (PE) image can meaningfully coexist within a single binary artifact, while supporting multiple logically independent execution regions selected by the file's launch name.

This document records the project's scope, motivation, structural hypotheses, constraints, and open problems. It does not present an implementation, a proof of concept, or any verified compatibility result.

---

### Motivation

Polyglot files — artifacts that are simultaneously interpretable as more than one format — have existed for decades in narrow, well-studied forms (e.g., GIF/JS, PDF/JS, ZIP/PE). Most practical polyglots exploit either:

- **Parser laxity** — a format that tolerates arbitrary trailing or leading bytes.
- **Overlapping magic numbers** — two formats whose fixed headers happen to align.

MP4 and PE are hostile to both strategies. Each imposes rigid constraints on early byte ranges, and neither is designed to tolerate the other's structural requirements. Whether a *useful* (not merely *technically parseable*) MP4/PE polyglot can exist is therefore a non-trivial research question rather than a known engineering trick.

The project additionally investigates a second, independent axis: whether a single artifact can carry **multiple logically distinct execution regions** without distributing project-specific DLLs or data files alongside it, and select among them purely from the launch filename.

---

### Research Objectives

- **Single-file architecture** — Store project-specific code and data within one binary artifact.
- **Filename-dependent execution** — Select an execution region based solely on the program's launch name.
- **Independent execution regions** — Maintain logically distinct code and data regions within the same file, without one region's layout constraints compromising another's.
- **MP4 compatibility** — Investigate whether the artifact can retain meaningful MP4 compatibility while satisfying Windows executable-format requirements. "Meaningful" is itself an open definition (see *Compatibility Criteria* below).
- **Zero third-party libraries** — Implement all project-specific functionality without third-party libraries, frameworks, or packages.
- **No custom external DLLs** — Avoid separately distributed project-specific dynamic-link libraries or runtime data files.

These objectives describe the intended research direction. None should be read as established functionality.

---

### Structural Background

The following is a condensed summary of the two formats' relevant constraints. It is provided for orientation and is not a substitute for the primary specifications.

#### ISO Base Media File Format (ISOBMFF / MP4)

- A file is a sequence of **boxes**, each prefixed by a 32-bit big-endian size and a 4-byte type code.
- Conventional files begin with an `ftyp` box identifying the brand and compatible brands.
- Parsers typically seek a `moov` box (metadata) and an `mdat` box (media data); their order and offsets are declared inside `moov` when not fragmented.
- Many parsers are tolerant of trailing bytes, unknown boxes, and reordering within documented limits, but the leading `ftyp` and the internal offsets in `moov` are load-bearing.

#### Portable Executable (PE)

- A file must begin with the DOS header, whose first two bytes are `MZ` (`0x4D 0x5A`), and whose field at offset `0x3C` (`e_lfanew`) points to the PE header.
- The PE header is followed by an optional header, section table, and sections mapped at alignment dictated by `FileAlignment` and `SectionAlignment`.
- The Windows loader enforces strict consistency between the headers and the mapped image; deviations typically cause load failure rather than graceful degradation.

#### Immediate Structural Tension

The two formats make incompatible demands on the first ~64 bytes of the file and on byte ranges referenced by their internal offset tables. This overlap is the project's central structural problem.

---

### Hypothetical Behavior

| Filename | Selected region |
|:--|:--|
| `game1.exe` | Execution Region A |
| `game2.exe` | Execution Region B |
| `game3.exe` | Execution Region C |

The artifacts may contain identical binary contents while selecting different execution regions according to their launch names. Whether this behavior can coexist with the required MP4 properties — and whether the selection mechanism itself can be implemented without external configuration — remains open.

---

### Compatibility Criteria (Open Definition)

"MP4 compatibility" and "native Windows execution" are not single-valued properties. Before any feasibility claim can be evaluated, the project must define measurable thresholds. Candidate axes include:

- **Which parser(s)** must accept the file (a specific reference parser, a specific player, a class of players)?
- **What must be preserved** (container parseability, seekability, decodability of at least one track, correct duration/bitrate metadata)?
- **What may degrade** (ignored trailing data, missing optional boxes, reordered boxes within spec)?
- **On which Windows versions and loader conditions** must native execution succeed?
- **Under which security postures** (signed/unsigned, SmartScreen on/off, AV present)?

Until these axes are fixed, no compatibility claim can be falsified or confirmed.

---

### Research Questions

1. **Coexistence** — Can MP4 container structures and PE structures coexist in a single binary while retaining useful compatibility with both formats, or does the overlap of their required byte ranges impose a hard impossibility?
2. **Structural constraints** — What constraints arise when media metadata, executable headers, code, and data must be laid out under both formats' offset and alignment rules simultaneously?
3. **Multiple execution regions** — Can multiple logically independent execution regions exist in one artifact without custom external DLLs, and without one region's layout requirements being sacrificed to another's?
4. **Launch-name identification** — How should a running process reliably determine the name under which it was launched, given that the same bytes must load under different names, and that path resolution, symlinks, hardlinks, and shell invocations can each alter what "launch name" means?
5. **Platform constraints** — What limitations arise from the Windows executable loader, image mapping, code signing, security software heuristics, and media-player compatibility assumptions?
6. **Taxonomy** — How should an MP4/PE polyglot be distinguished, in principle, from (a) an executable with embedded media, (b) a media file with a renamed extension, and (c) a file that merely fails to parse in both formats?
7. **Mutual exclusivity** — Which of the project's requirements are technically compatible, and which appear mutually exclusive even before implementation?

Each question is expected to spawn narrower sub-questions as the project moves from ideation to specification.

---

### Known or Anticipated Structural Tensions

The following tensions are already visible from the format specifications alone. They are recorded here as the project's most likely points of failure.

| # | Tension | Description |
|:--|:--|:--|
| T1 | **Leading magic** | PE requires `MZ` at offset 0; ISOBMFF expects a big-endian box size at offset 0 and `ftyp` at offset 4 |
| T2 | **Header offset field** | PE's `e_lfanew` at `0x3C` overlays a byte range that ISOBMFF parsers may treat as box data |
| T3 | **Alignment** | PE's `FileAlignment`/`SectionAlignment` and ISOBMFF's box-boundary conventions do not share a common alignment principle |
| T4 | **Offset tables** | `moov` chunk offsets point into `mdat`; if PE sections are interleaved with media data, those offsets must still resolve correctly |
| T5 | **Signing and trust** | Authenticode, SmartScreen, and AV heuristics have no defined behavior for polyglot artifacts and may treat them as suspicious by construction |
| T6 | **Parser tolerance** | ISOBMFF parser behavior on unexpected leading bytes, reordered boxes, or interleaved non-box data varies widely and is not fully specified |
| T7 | **Loader strictness** | The Windows loader validates header/section consistency; partial or nonstandard layouts generally fail to load rather than degrade gracefully |

T1 and T2 are the project's primary unresolved obstacles.

---

### Design Constraints

The proposed architecture follows these principles:

- One self-contained binary artifact.
- No third-party libraries, frameworks, or packages.
- No copied or vendored third-party library source code.
- No separately distributed project-specific DLLs or runtime data files.
- Multiple logically independent execution and data regions.
- Filename-dependent execution-region selection.
- Explicit verification of native Windows execution requirements.
- Clear separation between verified results, working assumptions, known limitations, and unresolved issues.

Windows APIs may be used when necessary, provided each use is documented. Development tools and compilers are permitted as build-time dependencies, but third-party libraries must not become implementation or runtime dependencies.

Changing a file extension from `.mp4` to `.exe` does not transform its internal structure into a valid Windows executable. Likewise, embedding video in an executable does not automatically produce a valid MP4/PE polyglot. The project treats both of these common misconceptions as explicit non-goals.

---

### Dependency Policy

**Third-party libraries are prohibited.**

This policy excludes:

- Third-party static and dynamic libraries.
- External frameworks and package-manager dependencies.
- Vendored or copied third-party library source code.
- Separately distributed project-specific DLLs or runtime data files.

All project-specific implementation logic must be developed within this repository.

Operating-system interfaces (e.g., Windows APIs) and development toolchains (compilers, linkers, assemblers) are treated separately from third-party library dependencies. This policy does not imply independence from Windows or its required system components; it constrains only the set of dependencies the project itself introduces.

---

### Scope and Safety

The project focuses on:

- File-format research.
- Binary-structure experimentation.
- Controlled execution tests using files and environments under the researcher's control.

The project does **not** aim to:

- Disguise executables as harmless media.
- Bypass security controls, signing, or detection mechanisms.
- Execute code without the user's knowledge.
- Provide, reproduce, or refine any specific malware technique.
- Reimplement or redistribute third-party libraries.

Any future implementation must document its behavior, dependencies, compatibility limitations, and relevant security considerations.

---

### Non-Goals

The following are explicitly out of scope:

- Delivering a production-ready polyglot file format.
- Providing a general-purpose tool for constructing MP4/PE artifacts.
- Any form of evasion, obfuscation, or anti-analysis capability.
- Supporting formats other than MP4-family containers and Windows PE.

---

### Terminology

- **MP4** — A media-container format based on the ISO Base Media File Format (ISOBMFF).
- **PE** — Portable Executable, the executable format used by Windows.
- **Polyglot file** — A single binary designed to be meaningfully interpreted as more than one file format.
- **Execution region** — A logically distinct area of code and associated data within the artifact.
- **Launch name** — The filename (including extension) under which the artifact is invoked.
- **Filename-dependent execution** — Selection of program behavior based on the launch name.
- **Structural tension** — A pair of format requirements that impose incompatible constraints on the same byte range or alignment rule.

---

### License

**Planned license:** Apache License 2.0 (`Apache-2.0`).

The license will be established by the repository's `LICENSE` file. Until that file is added, the intended license should not be treated as an already granted license.

---

### Disclaimer

This repository documents an experimental research concept. All proposed features remain subject to technical validation. No claim is made that MP4 compatibility, native Windows execution, and filename-dependent execution regions have been successfully combined in a single binary.

Contributions that clarify format constraints, identify architectural contradictions, or provide reproducible feasibility results are welcome.

---

## 日本語

### 概要

MP4-Executable-Research は、ISO Base Media File Format（ISOBMFF）コンテナと Windows Portable Executable（PE）イメージが、単一のバイナリ成果物の中で意味的に両立し得るか、さらにその成果物が起動時のファイル名に応じて複数の論理的に独立した実行領域を選択できるかを調査する、実装前段階の研究プロジェクトです。

本書は、プロジェクトの範囲、動機、構造上の仮説、制約、未解決の問題を記録します。
実装、概念実証、検証済みの互換性結果は一切提示しません。

---

### 動機

ポリグロットファイル、すなわち複数のファイル形式として同時に解釈可能な成果物は、数十年前から限定的な形で存在してきました（例：GIF/JS、PDF/JS、ZIP/PE）。
実用的なポリグロットの多くは、次のいずれかを利用しています。

- **パーサの寛容性** — 前後の任意バイトを許容する形式。
- **マジックナンバーの重複** — 固定ヘッダーがたまたま整合する二つの形式。

MP4 と PE は、どちらの戦略に対しても敵対的です。
両者はいずれも先頭バイト範囲に対して厳格な制約を課し、相手側の構造要件を許容するようには設計されていません。
したがって、「技術的に解析可能」なだけでなく「実用的に意味のある」MP4/PE ポリグロットが成立し得るかは、既知の工学的手法ではなく、未解決の研究課題です。

本プロジェクトはさらに、第二の独立した軸を調査します。
すなわち、単一の成果物が、プロジェクト固有の DLL やデータファイルを別途配布することなく、**複数の論理的に独立した実行領域**を保持し、起動ファイル名のみによってそれらを選択できるかという問題です。

---

### 研究目標

- **単一ファイル構造** — プロジェクト固有のコードとデータをひとつのバイナリ成果物に格納する。
- **ファイル名に応じた実行** — 起動時のファイル名のみに基づいて実行領域を選択する。
- **独立した実行領域** — 同一ファイル内に論理的に区別されたコード領域とデータ領域を設け、一方の領域のレイアウト制約が他方を損なわないようにする。
- **MP4 互換性** — Windows 実行ファイル形式の要件を満たしながら、MP4 としての実用的な互換性を維持できるか調査する。「実用的」の定義自体が未解決である（後述の*互換性基準*を参照）。
- **第三者製ライブラリ不使用** — ライブラリ、フレームワーク、パッケージに依存せず、プロジェクト固有の機能を実装する。
- **独自の外部 DLL 不使用** — プロジェクト固有の DLL およびランタイムデータファイルを別途配布する構成を避ける。

これらは研究の方向性を示すものであり、実装済みの機能を示すものではありません。

---

### 構造上の背景

以下は、両形式の関連する制約の要約です。
位置づけのための説明であり、一次仕様の代替ではありません。

#### ISO Base Media File Format（ISOBMFF / MP4）

- ファイルは**ボックス**の列であり、各ボックスは 32 ビットのビッグエンディアンサイズと 4 バイトの型コードで始まる。
- 慣例的なファイルは `ftyp` ボックスで始まり、ブランドおよび互換ブランドを識別する。
- パーサは通常 `moov` ボックス（メタデータ）と `mdat` ボックス（メディアデータ）を探す。非フラグメント形式では、これらの順序とオフセットは `moov` 内で宣言される。
- 多くのパーサは、末尾バイト、未知のボックス、仕様の範囲内での並び替えに対して寛容である。しかし、先頭の `ftyp` と `moov` 内の内部オフセットは構造上不可欠である。

#### Portable Executable（PE）

- ファイルは DOS ヘッダーで始まり、その先頭 2 バイトは `MZ`（`0x4D 0x5A`）、オフセット `0x3C` のフィールド（`e_lfanew`）が PE ヘッダーを指す。
- PE ヘッダーの後にはオプショナルヘッダー、セクションテーブル、セクションが続き、`FileAlignment` と `SectionAlignment` に従って配置される。
- Windows ローダーは、ヘッダーとマップ済みイメージの厳密な整合性を要求する。逸脱は通常、緩やかな劣化ではなくロード失敗を引き起こす。

#### 直ちに生じる構造上の緊張

両形式は、ファイルの先頭約 64 バイトおよび内部オフセットテーブルが参照するバイト範囲に対して、両立不能な要求を課します。
この重複が本プロジェクトの中心的構造問題です。

---

### 想定される動作

| ファイル名 | 選択される実行領域 |
|:--|:--|
| `game1.exe` | 実行領域 A |
| `game2.exe` | 実行領域 B |
| `game3.exe` | 実行領域 C |

成果物は、バイナリ内容が同一でありながら、起動ファイル名に応じて異なる実行領域を選択する構成を想定しています。
この動作が要求される MP4 の特性と両立し得るか、また選択機構そのものが外部設定なしに実装可能かは、未解決です。

---

### 互換性基準（未定義）

「MP4 互換性」および「Windows ネイティブ実行」は、単一の値を持つ性質ではありません。
実現可能性の主張を評価する前に、本プロジェクトは測定可能なしきい値を定義する必要があります。
候補となる軸は次のとおりです。

- **どのパーサ**がファイルを受け入れる必要があるか（特定の参照パーサ、特定のプレイヤー、プレイヤーのクラス）。
- **何を保持**すべきか（コンテナとしての解析可能性、シーク可能性、少なくとも一つのトラックのデコード可能性、正確な再生時間・ビットレートメタデータ）。
- **何の劣化**を許容するか（無視される末尾データ、欠落した任意ボックス、仕様内でのボックス並び替え）。
- **どの Windows バージョンとローダー条件下**でネイティブ実行が成功すべきか。
- **どのセキュリティ態勢下**で（署名あり/なし、SmartScreen 有効/無効、AV の存在）。

これらの軸が固定されるまで、いかなる互換性主張も反証も確認もできません。

---

### 研究課題

1. **共存** — MP4 コンテナ構造と PE 構造は、単一バイナリ内で双方の実用的な互換性を保ちながら共存できるか。それとも、要求されるバイト範囲の重複が本質的な不可能性を課すか。
2. **構造上の制約** — メディアメタデータ、実行ファイルヘッダー、コード、データを、両形式のオフセットおよびアラインメント規則の下で同時に配置する場合、どのような制約が生じるか。
3. **複数実行領域** — 独自の外部 DLL に依存せず、複数の論理的に独立した実行領域をひとつの成果物内に保持できるか。また、一方の領域のレイアウト要件を他方のために犠牲にすることなく実現できるか。
4. **起動名の識別** — 実行中のプロセスは、自身が起動された名前をどのようにして確実に識別すべきか。同一のバイト列が異なる名前でロードされる必要があり、パス解決、シンボリックリンク、ハードリンク、シェル経由の起動がそれぞれ「起動名」の意味を変え得る。
5. **プラットフォーム制約** — Windows の実行ファイルローダー、イメージマッピング、コード署名、セキュリティソフトのヒューリスティック、メディアプレイヤーの互換性前提には、どのような制約があるか。
6. **分類** — MP4/PE ポリグロットを、(a) 動画を内包する実行ファイル、(b) 拡張子だけを変更したメディアファイル、(c) 単にどちらの形式としても解析に失敗するファイル、から原理的にどう区別すべきか。
7. **相互排他性** — 本プロジェクトの要件のうち、技術的に両立可能なものと、実装以前に相互排他的と思われるものはどれか。

各課題は、プロジェクトが構想段階から仕様段階へ進むにつれて、より狭い副課題を生むことが想定されます。

---

### 既知または予想される構造上の緊張

以下の緊張は、形式仕様のみから既に見えています。
本プロジェクトが最も失敗しやすい点として記録します。

| # | 緊張 | 説明 |
|:--|:--|:--|
| T1 | **先頭マジック** | PE はオフセット 0 に `MZ` を要求し、ISOBMFF はオフセット 0 にビッグエンディアンのボックスサイズ、オフセット 4 に `ftyp` を期待する |
| T2 | **ヘッダーオフセットフィールド** | PE の `e_lfanew`（`0x3C`）が、ISOBMFF パーサがボックスデータとして扱い得るバイト範囲に重なる |
| T3 | **アラインメント** | PE の `FileAlignment` / `SectionAlignment` と、ISOBMFF のボックス境界の慣例は、共通のアラインメント原理を共有しない |
| T4 | **オフセットテーブル** | `moov` のチャンクオフセットは `mdat` 内を指す。PE セクションがメディアデータと交互に配置される場合、それらのオフセットは依然として正しく解決される必要がある |
| T5 | **署名と信頼** | Authenticode、SmartScreen、AV のヒューリスティックは、ポリグロット成果物に対して定義された挙動を持たず、構造上疑わしいものとして扱う可能性がある |
| T6 | **パーサの寛容性** | 予期しない先頭バイト、並び替えられたボックス、交互に配置された非ボックスデータに対する ISOBMFF パーサの挙動は、実装ごとに大きく異なり、十分に仕様化されていない |
| T7 | **ローダの厳格性** | Windows ローダーはヘッダーとセクションの整合性を検証する。部分的な、あるいは非標準のレイアウトは、緩やかな劣化ではなく一般にロード失敗を引き起こす |

T1 および T2 が、本プロジェクトの主要な未解決障害です。

---

### 設計上の制約

本プロジェクトでは、以下の原則を採用します。

- 単一の自己完結型バイナリ成果物。
- 第三者製ライブラリ、フレームワーク、パッケージの不使用。
- 第三者製ライブラリのソースコードの複製・同梱禁止。
- プロジェクト固有の DLL およびランタイムデータファイルの別途配布禁止。
- 複数の論理的に独立した実行領域とデータ領域。
- ファイル名に応じた実行領域の選択。
- Windows ネイティブ実行要件の明示的な検証。
- 検証結果、作業上の仮説、既知の制約、未解決の問題の明確な区別。

必要に応じて Windows API を利用する場合は、その利用箇所を記録します。
コンパイラやビルドツールはビルド時依存として認めますが、第三者製ライブラリを実装時または実行時の依存としてはなりません。

`.mp4` を `.exe` にリネームしても、内部構造が Windows の実行形式に変わるわけではありません。
同様に、実行ファイル内に動画を格納するだけで MP4/PE ポリグロットが成立するわけでもありません。
本プロジェクトは、これら二つの一般的な誤解を明示的な非目標として扱います。

---

### 依存関係ポリシー

**第三者製ライブラリは使用しません。**

本ポリシーは次を除外対象とします。

- 第三者製の静的・動的ライブラリ。
- 外部フレームワークおよびパッケージマネージャ依存。
- 複製・同梱された第三者製ライブラリのソースコード。
- 別途配布されるプロジェクト固有の DLL およびランタイムデータファイル。

プロジェクト固有の実装ロジックは、本リポジトリ内で開発します。

OS が提供するインターフェース（例：Windows API）および開発ツールチェーン（コンパイラ、リンカ、アセンブラ）は、第三者製ライブラリ依存とは区別します。
本ポリシーは、Windows や実行に必要なシステムコンポーネントへの依存まで排除するものではありません。
本プロジェクト自身が導入する依存の集合のみを制約します。

---

### 研究範囲と安全性

本プロジェクトが対象とするのは次のとおりです。

- ファイル形式の研究。
- バイナリ構造の実験。
- 研究者が管理するファイルおよび環境における、制御された実行検証。

本プロジェクトが目的としないのは次のとおりです。

- 無害なメディアを装った実行ファイルの偽装。
- セキュリティ制御、署名、検出機構の回避。
- 利用者に知らせないコード実行。
- 特定のマルウェア手法の提供、再現、改良。
- 第三者製ライブラリの再実装・再配布。

将来的に実装する場合は、実際の動作、依存関係、互換性の制約、関連するセキュリティ上の考慮事項を文書化します。

---

### 非目標

以下は明示的に対象外です。

- 製品品質のポリグロットファイル形式の提供。
- MP4/PE 成果物を構築するための汎用ツールの提供。
- 回避、難読化、解析妨害に関するあらゆる能力。
- MP4 系コンテナおよび Windows PE 以外の形式のサポート。

---

### 用語

- **MP4** — ISO Base Media File Format（ISOBMFF）を基礎とするメディアコンテナ形式。
- **PE** — Portable Executable。Windows の実行ファイルで使用される形式。
- **ポリグロットファイル** — 複数のファイル形式として意味のある形で解釈できるように設計された単一バイナリ。
- **実行領域** — 成果物内の、コードと関連データからなる論理的に区別された領域。
- **起動名** — 成果物が起動されたときのファイル名（拡張子を含む）。
- **ファイル名に応じた実行** — 起動名に基づいてプログラムの挙動を選択する仕組み。
- **構造上の緊張** — 同一のバイト範囲またはアラインメント規則に対して両立不能な制約を課す、二つの形式要件の組。

---

### ライセンス

**採用予定ライセンス：** Apache License 2.0（`Apache-2.0`）。

実際のライセンス条件は、リポジトリに追加する `LICENSE` ファイルによって定めます。
ファイルを追加するまでは、ライセンスがすでに付与されているものとして扱わないでください。

---

### 注意事項

本リポジトリは実験的な研究構想を記録するものです。
提案した機能はすべて技術的な検証を必要とします。
MP4 互換性、Windows ネイティブ実行、ファイル名に応じた実行領域の選択を、単一バイナリ内で実現できたと主張するものではありません。

ファイル形式上の制約を明確にする提案、設計上の矛盾の指摘、再現可能な検証結果の共有を歓迎します。
