---
theme: seriph
background: 'https://source.unsplash.com/1600x900/?texture,patterns'
class: 'text-center'
highlighter: shiki
lineNumbers: true
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# 技术攻坚-专题图性能突破

<div>
  <h3 class="font-bold">突破现有专题图性能限制</h3>
  <p class="text-xs opacity-60">
    <span>1. 单一专题图查看时性能提升</span>
    <span class="pl-12px">2. 多专题图轮播时性能提升</span>
  </p>
</div>

<style>
  h1 {
    font-weight: bold;
  }
</style>

---

## 专题图功能痛点

### 后端

- 🤷‍♂️ 后端生成专题图很困难，前端生成图片，保存到后端更靠谱。

### 前端

- ♾ **重复渲染** 图层实例化后调用 setTileUrl 方法，重复渲染严重
- 🐢 **速度** 查看专题图详情时速度缓慢
- 🚫 **轮播功能不可用** 地图形式轮播查看专题图，速度太慢，不可用

## 解决思路

### 后端

- 🖼 **Headless Browser** 定期定时为自动生成的专题图生成图片

### 前端

- 🔑 **内部优化、内部调用** 图层 updateState 生命周期内进行数据比对，内部调用 setTileUrl 方法
- 🖼 专题图轮播及查看专题图详情时，使用**图片**查看

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

<style>
h2 {
  margin:0;
  font-size:24px !important;
  line-height: 1 !important;
  margin-bottom: 10px;
  font-weight: bold;
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
h3 {
  margin-top:16px;
  margin-bottom:12px;
  font-size:16px !important;
  line-height: 1 !important;
}

ul li {
  font-size: 16px !important;
}
</style>

---

# Selenium

Selenium 是一个涵盖一系列工具和库的项目，这些工具和库支持和支持 Web 浏览器的自动化

官网地址：https://www.selenium.dev/

Selenium 的核心是 WebDriver，这是一个编写指令集的接口

---

# Selenium 原理

> **直接通信**：WebDriver 通过驱动程序将命令传递给浏览器，并通过相同的路由接收返回的信息
> 驱动是特定于浏览器的，例如谷歌的 Chrome/Chromium 的 ChromeDriver，Mozilla 的火狐的 GeckoDriver 等

<img src="/images/s-1.jpg" class="block mt-12px"/>

---

# Selenium 原理

> **远程通信**：与浏览器的通信也可以是通过 Selenium Server 或 RemoteWebDriver 进行的远程通信。RemoteWebDriver 与驱动程序和浏览器在同一系统上运行。

<img src="/images/s-2.jpg" class="block mt-12px"/>

---

# 实践

- 安装 Selenium

```xml
<dependency>
  <groupId>org.seleniumhq.selenium</groupId>
  <artifactId>selenium-java</artifactId>
  <version>3.141.59</version>
</dependency>
```

<br />

- 安装 Browser Drivers

<img src="/images/s-3.jpg" />

1. Selenium 4 适配 Chrome v75 以上。
2. 驱动和浏览器大版本一致！如 96.0.4664.45，都要是 96 版本

---

# 实践

```java
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;

public class HelloSelenium {
    public static void main(String[] args) {
        driver = new ChromeDriver();

        driver.get("https://google.com");

        driver.getTitle(); // => "Google"
        driver.quit();
    }
}

```

---

# 定期保存图片

```java
public class SaveImage{
  public static void main(String[] args) {
    //1.参数
    ChromeOptions options = new ChromeOptions();
    options.addArguments(“--headless”)
    WebDriver driver = new ChromeDriver(options);
    //2.页面
    driver.get("https:menggu_url");
    WebElement button= driver.findElement(By.Id("#save"));
    //3.点击按钮
    button.click(); // => "save image"
    driver.quit();
  }
}
```

---

# Chrome 参数

- \-\-headless **无头模式**
- \-\-no-sandbox **禁用沙箱**
- \-\-enable-logging **日志**
- \-\-user-agent="xxx" **使用自定义代理**
- \-\-disable-gpu **禁止 gpu 加速**

---

# Headless 浏览器代码

```java {all|2-8|10-22|12-18}
public void saveImage() {
  List<ThematicMap> thematicMaps = thematicMapRepository.findByImageIdIsNull();
  String loginUrl = publishHost + LOGIN_PATH;
  WebDriver driver = getWebDriver();
  driver.get(loginUrl);
  LocalStorage local = ((WebStorage) driver).getLocalStorage();
  local.setItem(KEY, TOKEN);
  driver.navigate().refresh();

  thematicMaps.forEach(map -> {
    try {
      String path = THEMATIC_MAP_DETAIL_PATH.replace("{id}", String.valueOf(map.getId()));
      String thematicDetailUrl = publishHost + path;
      driver.get(thematicDetailUrl);
      ThreadUtil.sleep(2 * 60 * 1000);   // 等待五分钟
      WebElement element = driver.findElement(By.id("save"));
      element.click();
      ThreadUtil.sleep(6 * 1000);
    } catch (Exception e) {
      // ...
    }
  });
  driver.quit();
}
```

---

# 减少重复渲染代码

updateState 生命周期内比对 props、oldProps 及 state

<br>

```ts {all|2-3|5-8|all}
updateState({ oldProps, props, context }: UpdateStateInfo<any>): void {
  const { url: oldUrl, regionCode: oldRegionCode } = oldProps;
  const { url, map, regionCode, visible } = props;

  if ((url !== oldUrl || regionCode !== oldRegionCode || !isEqual(this.state.map, map)) && visible) {
    // set map
    // set tile url
  }
}

```

---

# 保存专题图代码

```ts {all|1|2-7|11-15|all}
html2canvas(document.getElementById('canvas')).then((canvas) => {
  const url = canvas.toDataURL();
  const blob = dataUrlToFile(url, 'image/jpeg');
  const formData = new FormData();
  const file = new File([blob], new Date().getTime() + '.jpg');

  formData.append('file', file);

  uploadMutation.mutate(formData, {
    onSuccess(res) {
      const { id } = res;
      const submitData = {
        imageId: id,
        // rest of data
      };

      // 创建或更新专题图
    },
  });
});
```

---

# 查看专题图详情示例

<img  src="/images/eg-1.gif" />

---

# 专题图轮播示例

<img  src="/images/eg-2.gif" />

---

# 成果总结

- 👏🏽 使纹理图层与外部其余数据隔离，避免了**不必要的渲染**。
- 🌟 手动或自动生成的专题图，或手动或定期为其生成，专题图详情中包含了**图片字段**，
  查看专题图详情及专题图轮播时，可以使用图片来替代原先方案，极大提高了专题图详情查看速度，并且使得轮播功能可用。

# 其他可优化点

- ❓ 针对除特殊区域外的一般行政区，请求 meta 数据时，是否可以返回相同的 url，来提高浏览器的缓存能力，进一步提高渲染速度。

<style>
  h1 {
    margin-top: 40px;
  }
</style>
