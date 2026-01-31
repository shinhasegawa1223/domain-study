---
description: ドキュメント追加時にREADME.mdの目次を更新する
---

# ドキュメント追加時のルール

新しいドキュメント（study*.md）やExcalidrawファイルを追加した場合、必ず以下を実行する。

## 手順

1. **README.mdの目次を更新**
   - `doc/` に新しいmdファイルを追加した場合 → ドキュメントテーブルに行を追加
   - `other/` に新しいexcalidrawファイルを追加した場合 → 図解テーブルに行を追加

2. **フォーマット**
   ```markdown
   | No. | [タイトル](doc/study*.md) | 内容の簡単な説明 |
   ```

3. **連番を確認**
   - study1, study2, study3... の順番を確認

// turbo-all
