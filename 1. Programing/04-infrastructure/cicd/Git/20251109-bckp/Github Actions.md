# 介紹
---
Github Action 是一個 **automate developer workflow** 的平台，而 CI/CD 只是其中一個 workflow。

### workflow
1. **Listen to github event**: e.g PR created , Issue created, PR merged ...
2. **Trigger workflow**: 當事件發生，就會啟動一系列 github actions 做出相應的處理 e.g. 將 issue 指派給某個 contributer, testing等，這就是 workflow。

### CI/CD Pipeline with Github Acitons
* 流程：
	Commit Code -> Test -> Build -> Push -> Deploy
* 優勢：
	1. 對已經使用 github 的項目不用轉移到其他倉庫。
	2. 整合優良：可以結合很多其他技術
	3. 容易配置

# 使用
---
source: 
* [Events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
* [Built-in Actions](https://github.com/orgs/actions/repositories?type=all)
1. 建立 .github/workflows/myaction.yml
```yaml
name: Java CI with Gradle

# listen to events
on: 
	# 每當有人 push 到 mater branch 時啟動
	push: 
		branches: [ master ]
jobs:
	# name of jobs 可以隨便設置
	build: 
		runs-on: ubuntu-latest
		strategy: 
			matrix:
				os: [ubuntu-latest, windows-latest, macOS-latest]
		
		steps:
			- uses: actions/checout@v2
			- name: Set up JDK 1.8
				uses: actions/setup-java@v1
				with: java-version: 1.8
			- name: Grant evecute permission for gradlew
				run: chmod +x gradlew
			- name: Bio;d wotj Gradle
				run: ./gradew build
				
	publish:
		# 因為默認每個 job 都是 parallel 執行在不同的 github server 
		# 但 publish 需要接續在 build 後面就要使用 need
		needs: build 
		
```
* uses: 執行 action
* run: 執行 command-line command
* needs: 表示需要的依賴
* runs-on: 表示要執行的平台 e.g. Ubuntu, Windows, macOS
* strategy matrix: 需要運行在以下三個系統
```ymal
jobs:
	build:
		runs-on: ${{matrix.os}}
		strategy: 
			matrix:
				os: [ubuntu-latest, windows-latest, macOS-latest]
```