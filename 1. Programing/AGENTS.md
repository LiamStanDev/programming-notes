# ⚙️ 知識庫代理指南

本文件定義此個人技術筆記庫的內容結構、風格規範與操作指南。

## 📚 核心概覽

涵蓋軟體工程全貌的技術筆記庫,內容包含:
* **CS 基礎**: 演算法、OS、網路、設計原則
* **程式語言**: Rust, C++, C#, Java, Python, JavaScript, Golang
* **軟體開發**: Frontend/Backend、系統設計、架構模式
* **基礎設施**: Cloud, Containers, DB, Messaging, CI/CD, Monitoring
* **Linux 系統**: 發行版、管理與工具
* **AI/ML**: LLM, Prompting, AI Infrastructure
* **職涯**: 面試準備材料

> 本知識庫為純文件庫,無 Build/Test 指令。

## 🗂️ 目錄結構

```
01-fundamentals/    - CS 基礎 (演算法, OS, 網路, 設計原則)
02-languages/       - 程式語言 (rust, cpp, csharp, java, python, js, go)
03-web-development/ - 網頁開發 (Frontend, Backend, Deployment)
04-infrastructure/  - 基礎設施 (Cloud, Containers, DB, Messaging, CI/CD)
05-linux/           - Linux 系統管理與工具
06-ai-ml/           - AI/ML 概念 (LLM, Prompting, AI Infra)
07-system-design/   - 系統設計與架構模式
08-interview/       - 面試準備
99-misc/            - 雜項工具、配置與資源
```

## 📝 內容規範
	
### 基本格式
* **檔案格式**: Markdown (`.md`), UTF-8 編碼
* **語言**: 中文(繁體) + 英文術語，禁止使用 emoji
* **換行**: Unix-style (LF)
* **命名**: 描述性名稱 + 數字前綴 (如 `01-基礎概念.md`)

### 術語標註
首次出現的中文技術名詞須標註英文:
* `分散式系統 (Distributed Systems)`
* `領域驅動設計 (Domain-Driven Design, DDD)`

### 圖表
* 使用 [Mermaid](https://mermaid.js.org/) 繪製流程圖、概念圖
* Mermaid 中字串內容必須使用雙引號 `""` 包裹

### 參考資料
每篇筆記結尾必須加入:

```markdown
---

## 參考資料 (References)

1. [線上資源](URL)
2. 《書籍名稱》 (作者, 年份)
```

## 🛠️ 操作原則

1. **內容品質**: 簡潔、正確、完整 (Concise, Correct, Complete)
2. **結構遵循**: 新筆記放置於對應目錄
3. **內容獨立**: 每篇筆記自洽,嚴禁「下篇將討論...」等轉場敘述
4. **參考標註**: 每篇筆記必須標註來源
5. **去重整合** - 消除冗餘,避免重複
6. **質量優先** - 移除過時/低價值內容,確保每篇筆記的實用性
7. **深入說明** - 重要或困難概念必須有詳細文字說明
