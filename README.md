---
name: portmanager
description: TCP/UDP ポート + Windows ファイルロックを確認・開放する統合 CLI。「portmanager」「ポート確認」「ファイルロック」「lock-check」「port kill」「ポート占有」で発動。WSL + PowerShell 連携で Windows のプロセス・ファイルロックも操作。
---

# portmanager — ポート + ファイルロック 管理 CLI

bash + ss（netstat）+ PowerShell で **TCP ポート状況と Windows ファイルロックを確認・開放**する。

下位ツールとして `portmap` `nodeman` `serverman` を持ち、`manman` から統合呼び出しされる。

## インストール

```bash
chmod +x <このスキル>/portmanager
ln -sf <このディレクトリ>/portmanager ~/bin/portmanager
```

## CLI 一覧（ポート）

| コマンド | 説明 |
|----------|------|
| `portmanager check [port]` | ポート状況確認（全ポート or 特定ポート） |
| `portmanager find-free [start] [end]` | 空いているポートを検索 |
| `portmanager kill-port <port>` | ポート占有プロセスを kill |

## CLI 一覧（ファイルロック — Windows/WSL）

| コマンド | 説明 |
|----------|------|
| `portmanager lock-check <path>` | ファイル/ディレクトリのロック状況を確認 |
| `portmanager lock-list` | github-mcp-server / opencode / node などの常時ロック疑いプロセス一覧 |
| `portmanager lock-kill <pid>` | PID 指定でロックプロセスを停止 |
| `portmanager lock-kill <path>` | パス含むプロセスを検索・停止 → rm |

## lock / ロック調査の流れ（標準手順）

### 1. 失敗確認
```bash
rm -rf /mnt/c/Users/0501JP/hokusai-manga
# => Permission denied / 別のプロセスで使用
```

### 2. ロック特定（最高精度：resmon.exe）
Windows で直接：
```
resmon.exe → CPU タブ → Associated Handles → 検索: hokusai-manga
```

### 3. 簡易 CLI 版（WSL 内）
```bash
# おすすめプロセスのコマンドラインを確認
portmanager lock-list

# 対象ディレクトリを参照しているプロセスを探す
portmanager lock-check /mnt/c/Users/0501JP/hokusai-manga

# PID 指定で停止（github-mcp-server などが該当しやすい）
portmanager lock-kill 14888
portmanager lock-kill 21328
portmanager lock-kill 32356

# あるいはパス指定で自動検索・停止 → 削除
portmanager lock-kill /mnt/c/Users/0501JP/hokusai-manga
```

### 4. 再削除
```bash
rm -rf /mnt/c/Users/0501JP/hokusai-manga
```

## 典型的な犯人（1028 / hokusai-manga 移設時の経験）

| プロセス | 症状 | 対処 |
|----------|------|------|
| `remote-server.mjs` (node.exe) | ポート 8787 LISTEN、カレントディレクトリロック | `taskkill /PID <pid>` or `nodeman kill all` |
| `github-mcp-server` (×3) | .git ディレクトリを監視中、削除ブロック | `portmanager lock-kill <PID>` |
| `opencode` | スキルディレクトリをホットリロード監視 | `portmanager lock-check` で確認 |
| `explorer.exe` | フォルダを操作中 | エクスプローラーを閉じる |

## 下位マネージャ構成

```
portmanager/          ← ポート + ロックの低層 API
├── portmanager       ← ポートcheck/find-free/kill + lock-check/list/kill
├── portmap           ← ポート→プロセス→skillマッピング表
├── nodeman           ← node.exe 専門 ps/kill/tree
├── serverman         ← skillサーバーの start/stop/open/qr/health
└── SKILL.md          ← このファイル

manman/               ← 最上位統合ダッシュボード
└── manman            ← ls/ps/ports/health/clean/win-check
```

## 設計方針

- **依存**: bash, ss（または netstat）, powershell.exe（Windows lock 用）
- **安全**: kill は明示的な指定のみ。`-9` 相当は `lock-kill` のみ。
- **CLI-first**: 全て bash + PowerShell one-liner
- **WSL 対応**: `wslpath` で自動パス変換

## 境界

- UDP ポートは確認のみ（kill 不可）
- ポートフォワーディング操作はしない
- root 権限が必要ない操作のみ
- **Windows のファイルロックは完全な自動化は難しく、resmon.exe（GUI）との併用が確実**
