# iDempiere ZK MVVM Plug-in Skill

## 目標

本文件為 Agent 提供 iDempiere OSGI plug-in 中開發 CustomForm（ZK MVVM 模式）的專用指引。當需要處理 `plugin-zk-mvvm-guide` 相關問題時，請優先參考這份文件。

## 核心概念

- CustomForm 由 4 個元件構成：`IFormFactory`、`ADForm` Controller、POJO ViewModel、ZUL Template。
- `~./` 資源路徑必須從 bundle 的 `web/` 目錄載入。
- `initForm()` 中必須切換 Thread Context ClassLoader 為 bundle classloader，並在最後還原。
- ZK MVVM 綁定最好透過 `@command`、`@NotifyChange`、`@DependsOn` 等 annotation 處理狀態變更。
- `@NotifyChange` 只對 ZUL `@command` 觸發的方法有效；Controller 直接呼叫時，須使用 `BindUtils.postNotifyChange(...)`。

## 專案結構

必須遵守以下目錄與檔案佈局：

- `META-INF/MANIFEST.MF`：`Bundle-ClassPath` 必須含 `.`
- `OSGI-INF/rnd_form_factory.xml`：OSGI service component 宣告
- `src/.../form/DispensingForm.java`：ADForm Controller
- `src/.../viewmodel/DispensingVM.java`：POJO ViewModel
- `src/.../factories/RNDFormFactory.java`：IFormFactory
- `web/zul/RND_DispensingForm.zul`
- `web/js/`：自訂 JavaScript（可選）
- `web/images/`
- `build.properties`：`web/` 必須在打包範圍內

## ADForm Controller 關鍵流程

1. 切換 ClassLoader
2. `Executions.createComponents("~./zul/...")`
3. `Selectors.wireComponents(this, this, false)`
4. 建立 iDempiere WebUI 元件
5. 將元件 `appendChild` 至 ZUL 容器
6. finally 還原 ClassLoader

### ADForm 職責

- 先用 bundle classloader 載入 ZUL 模板
- 用 `args` 傳入 ViewModel 實例
- 用 `@Wire` 綁定 ZUL 元件
- 建立並注入 iDempiere WebUI 元件（如 `WSearchEditor`）
- 實作 `ValueChangeListener` 或其他事件橋接

## ViewModel 配置規則

- ViewModel 必須是純 POJO，不繼承 ZK 類別
- `@Init`：初始化方法
- `@Command`：ZUL command 觸發的方法，應為 `public void`
- `@NotifyChange({"prop"})`：通知 UI 更新，只在 `@Command` 方法返回後自動觸發
- `@DependsOn("otherProp")`：getter 依賴屬性更新
- `@BindingParam("name")`：接收 ZUL 傳入參數
- `@bind` 雙向綁定必須有 getter 和 setter

## ZUL MVVM 寫法要點

- 在容器上使用 `apply="org.zkoss.bind.BindComposer"`
- `viewModel="@id('vm') @init(arg.vm)"` 使用 Controller 傳入 ViewModel
- 容器 `id`（如 `dispensingVMContainer`）需讓 Controller `@Wire` 取得
- 單向綁定：`@load(vm.prop)`
- 雙向綁定：`@bind(vm.prop)`
- `@command('method')` 觸發 ViewModel 方法
- iDempiere 元件放在空容器，Controller 透過 `appendChild` 注入

## iDempiere WebUI 元件與 JavaScript

- 在 ZUL 中不要直接放 `script` 或 `<?script?>` 來載入 bundle JS
- bundle 內 JS 不可用 `~./` 或 `Web-ContextPath` 存取
- 建議用 `Clients.evalJavaScript()` 動態載入 CDN 或執行 JS
- 用 `sclass` 取代 `id` 來定位容器，避免 ZK 自行覆蓋 DOM ID
- `refreshChart()` 等方法應使用重試機制，直到容器與 ECharts 函式可用

## OSGI 註冊與 Manifest

- `RNDFormFactory` 實作 `IFormFactory` 並回傳對應 `ADForm`
- `OSGI-INF/rnd_form_factory.xml` 宣告 `org.adempiere.webui.factory.IFormFactory`
- `MANIFEST.MF` 必須包含：
  - `Bundle-ClassPath: src/, .`
  - `Service-Component: OSGI-INF/rnd_form_factory.xml`
- `build.properties` 必須將 `web/` 包含進 `bin.includes`
- AD Form 設定中，Application Dictionary 的 Classname 要填完整類別名

## 常見錯誤與修正

- `Page not found: ~./zul/MyForm.zul`：未切換 ClassLoader
- `@Wire` 欄位為 null：`wireComponents` 早於 `createComponents`
- `@NotifyChange` 不觸發：手動呼叫 `BindUtils.postNotifyChange(...)`
- `PropertyNotWritableException`：`@bind` 雙向綁定缺 setter
- `Unsupported child for Borderlayout`：不要在 `<borderlayout>` 內放 `<script>`
- CSS `url('~./images/...')` 無效：改用 Base64 Data URI

## 使用建議

- 當需要開發 iDempiere CustomForm 時，優先用此 skill 檢查:
  - ClassLoader 切換流程
  - ZUL 與 ViewModel 綁定設定
  - bundle 資源與 `web/` 路徑配置
  - OSGi 註冊與 `MANIFEST.MF` 設定
- 若遇到 JavaScript 或 ECharts 整合問題，先檢查是否使用 `Clients.evalJavaScript()` 以及 `sclass` 容器定位
