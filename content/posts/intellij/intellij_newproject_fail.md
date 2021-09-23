---
title: "IntelliJ IDEAでNew Projectできない"
date: 2021-09-23T14:53:03+09:00
draft: false
tags: ['IntelliJ IDEA', 'Kotlin']
---

## 直し方
IntelliJ IDEAのKotlinプラグインをアンインストールして、IntelliJ IDEAを再起動する。

> As a workaround, open "File | Settings | Plugins", uninstall the Kotlin plugin and restart. It will downgrade to the bundled version 1.5.10.

https://youtrack.jetbrains.com/issue/KTIJ-19251#focus=Comments-27-5193849.0-0

## 症状

- IntelliJ IDEAで、"File | New | Project..."を選択してもポップアップが出てこないため、新規プロジェクトを作成できない。
- IntelliJ IDEAで、起動時のダイアログからNew Projectを押してもポップアップが出てこないため、新規プロジェクトを作成できない。

![起動時のダイアログ](../idea_newproject_bug_20210923.png)