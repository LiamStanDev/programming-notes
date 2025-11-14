### 基本介紹
Selenium 是一套廣泛用於**網頁應用自動化**的開源工具集。它支援多種瀏覽器，包括 Chrome、Firefox、Safari、Internet Explorer 和 Edge，同時也支持多種編程語言

source: https://selenium-python.readthedocs.io/index.html
### 使用方式
```python
from selenium import webdriver 
from selenium.webdriver.common.keys import Keys  # 提供鍵盤功能
from selenium.webdriver.common.by import By # 用於定位 element
from selenium.webdriver.chrome.options import Options

# 設置
chrome_options = Options()

# 初始化
driver = webdriver.Chrome(executable_path=("drive_path", options=chrome_options)

try:
    ## !-- 登入 -- ## 
    driver.get("https://www.facebook.com/login") # 登陸頁面網址
    username_box = driver.find_element(By.ID, "email")
    password_box = driver.find_element(By.ID, "pass")
    username_box.send_keys("YOUR_EMAIL")
    password_box.send_keys("YOUR_PASSWORD")

    # 模拟点击登录
    login_button = driver.find_element(By.NAME, "login")
    login_button.click()

    # 等待一段时间让页面加载完成
    time.sleep(5)

    # 导航到具体的帖文页面（替换成你想要获取点赞数的帖文URL）
    post_url = "https://www.facebook.com/vacweb1/posts/pfbid025n6Trn6BM6tJeTyXrdwtijvPRYMzDDDqRWuT7YQENzd8KqoGmU7NoCCXFW63bHrel"
    driver.get(post_url)

    time.sleep(5)

    # 找到点赞数元素，这里的选择器仅为示例，实际情况需自行调整
    # 注意：Facebook 的元素和类名经常变动，以下代码很可能需要根据实际情况修改
    likes = driver.find_element(By.XPATH, '//div[@aria-label="Like"]//span')
    print("点赞数：", likes.text)

finally:
    # 关闭浏览器
    driver.quit()
```