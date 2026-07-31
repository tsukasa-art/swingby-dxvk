# swingby-dxvk

[English](README.md)

**swingby-dxvk**は、Orreryの非公開ランチャー本体**Melammu**で利用する、
[DXVK](https://github.com/doitsujin/dxvk) 2.4.1をbaseとした改変forkです。
DirectXをVulkan経由でMetalへ変換するDXVKに、MoltenVK / Apple Silicon向けの
互換性修正を加えています。

**これはDXVK本家そのものではなく改変版です。** zlib/libpngライセンス
（[LICENSE](LICENSE)参照）が定める「改変版はそれと分かるよう明記する」
という条件に従い、この文書と[README.md](README.md)冒頭の notice がその明記です。

swingby-wineのようにupstreamを継続追従するforkとは異なり、このrepoは
特定のpatch一式を再現可能な状態でgit管理下に置くために作りました。以前は
patch済みsourceが非git作業ツリーにしか存在せず、単一障害点になっていました。

## このpatchで直していること

MoltenVK / Apple Siliconの互換性修正6件です。詳細な技術説明・機構・
検証手順は[SWINGBY_DXVK_PATCH.md](SWINGBY_DXVK_PATCH.md)（英語）にまとめています。

| 領域 | 内容 |
|---|---|
| null-stream頂点binding（主修正） | Fixed-function頂点シェーダがnull-stream(binding 16)へ出す属性を、UP系draw経路がbindし忘れる問題。MoltenVK上で頂点fetch全体が壊れ黒画面になる |
| device feature gating | `geometryShader`/`shaderCullDistance`/`maintenance4`/`robustBufferAccess2`/`nullDescriptor`をMoltenVKの実サポートに合わせてgate。強制有効化で`vkCreateDevice`が失敗していた |
| depth-clip → depth-clamp | MoltenVKが`VK_EXT_depth_clip_enable`未対応のため、既存のdepthClamp fallback経路を確実に通す |
| SM1 samplerの簡略化 | Shader Model 1のpixel shaderで、Metalのresource-index制約に抵触する冗長なsampler variant宣言を削減 |
| device作成時の診断ログ | 有効化したが未サポートのfeatureを名前付きで検出するログを追加（挙動は変えない） |
| mingw-w64 14.x header衝突対応 | build時のみの問題。DXVKのdummy構造体定義が現行mingw-w64の実定義と衝突するのを回避 |

うち複数（feature gating、depth-clamp、mingw header対応）はタイトル固有でなく、
一般的なMoltenVK/Apple Silicon互換性の問題である可能性が高いです。他のVulkan
driverでの検証はしておらず、upstream DXVKへのPRは検討事項として残しています。

## Orreryにおける位置づけ

| 対象 | 役割 |
|---|---|
| **Melammu** | Orreryで実際に開発・利用する非公開のmacOSランチャー本体 |
| **[swingby-wine](https://github.com/tsukasa-art/swingby-wine)** | Melammuが利用する公開Wine source fork |
| **swingby-dxvk** | Melammuが利用する公開DXVK source fork（このrepo） |
| **[melammu-vn](https://github.com/tsukasa-art/melammu-vn)** | SwiftUI UIと汎用判定を切り出したsource-only公開参照実装 |

## Build

```sh
meson setup --cross-file build-win32.txt --buildtype release --strip \
    --bindir x32 --libdir x32 -Dbuild_id=false <builddir>
cd <builddir> && ninja install
```

`x32/d3d9.dll`が生成されます。2026-07-31、この手順での再ビルドと配備済み
artifactのセクション構成がbyte-for-byte一致すること、差分27byteが全て
link timestamp系フィールドであることを実測確認しています（詳細=
[SWINGBY_DXVK_PATCH.md](SWINGBY_DXVK_PATCH.md)）。

## 関連リンク

- [Orrery — プロジェクト概要](https://tsukasa-art.com/projects/orrery/)
- [swingby-wine](https://github.com/tsukasa-art/swingby-wine) — Melammuが利用する公開Wine fork
- [melammu-vn — source-only公開参照実装](https://github.com/tsukasa-art/melammu-vn)

## License

DXVKとこのforkはzlib/libpngライセンスに従います。詳細は
[LICENSE](LICENSE)を参照してください。
