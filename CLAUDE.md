# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

ユーザー個人の「秘書」として働く Claude Code の Skill（と今後の Agent）を置く場所。アプリのコードはなく、ビルド・lint・テストのコマンドもない。成果物は Markdown の手順書だけ。

## 構成

- Skill は `.claude/skills/<skill-name>/SKILL.md` に置く。ディレクトリ名とフロントマターの `name` は同じにする。
- フロントマターの `description` は日本語で「〜するときに使う。」の形にする。Claude はこの一文を見て Skill を呼ぶか決めるので、何をするときに使うかを具体的に書く。
- 本文は日本語。短い文で、番号付きの手順と確認リストで書く。

新しい Skill を作る・直すときは `skill-creator` スキルを使う。
