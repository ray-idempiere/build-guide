# iDempiere Plug-in Project: ZK MVVM 開發指南

> **用途：** 供 AI Agent 在 iDempiere OSGI plug-in 中開發 CustomForm（ZK MVVM 模式）時參考。
>
> **來源：**
> - https://www.ninniku.tw/idempiere-customform-mvvm-tutorial/
> - https://www.ninniku.tw/idempiere-osgi-javascript-echarts-integration-guide/

---

## 1. 架構概覽

一個 CustomForm 由 4 個元件組成，缺一不可：

| 元件 | 角色 | 範例檔案 |
|------|------|---------|
| `IFormFactory` | OSGI 服務，負責實例化 Form | `RNDFormFactory.java` |
| `ADForm` (Controller) | 載入 ZUL、注入元件、橋接事件 | `DispensingForm.java` |
| POJO ViewModel | 業務邏輯、資料狀態管理 | `DispensingVM.java` |
| ZUL Template | 畫面佈局、MVVM 資料綁定 | `RND_DispensingForm.zul` |

```
IFormFactory ──creates──▶ ADForm (Controller)
                              │
                         initForm()
                              │
                    createComponents("~./zul/Form.zul")
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       POJO ViewModel    ZUL Template    iDempiere WebUI 元件
       (@Command)        (MVVM bind)    (WSearchEditor 等)
```

---

## 2. 專案目錄結構（必須遵守）

```
tw.topgiga.rnd/
├── META-INF/
│   └── MANIFEST.MF              ← Bundle-ClassPath 必須含 "."
├── OSGI-INF/
│   └── rnd_form_factory.xml     ← OSGI 服務宣告
├── src/
│   └── tw/topgiga/rnd/
│       ├── form/
│       │   └── DispensingForm.java      ← ADForm Controller
│       ├── viewmodel/
│       │   └── DispensingVM.java        ← POJO ViewModel
│       └── factories/
│           └── RNDFormFactory.java      ← IFormFactory
├── web/                                 ← ~./ 資源根目錄
│   ├── zul/
│   │   └── RND_DispensingForm.zul
│   ├── js/                              ← 自訂 JavaScript（如需要）
│   └── images/
└── build.properties                     ← bin.includes 必須含 web/
```

---

## 3. OSGI 資源路徑 `~./` 設定

### 3.1 規則

- ZK 的 `~./` 前綴 = 從 classpath 的 `web/` 目錄載入
- `~./zul/MyForm.zul` → classpath: `web/zul/MyForm.zul`

### 3.2 必要設定

**MANIFEST.MF — Bundle-ClassPath 必須含 `.`（bundle 根目錄）：**

```
Bundle-ClassPath: src/, .
```

**build.properties — `web/` 必須在打包範圍內：**

```properties
bin.includes = META-INF/,\
               .,\
               src/,\
               web/,\
               OSGI-INF/rnd_form_factory.xml
source.. = src/
src.includes = src/,\
               web/
output.. = bin/
```

### 3.3 ClassLoader 切換（關鍵）

**規則：** 在 `initForm()` 中載入 ZUL 之前，必須將 Thread Context ClassLoader 切換為 bundle 的 classloader，否則 ZK 無法解析 `~./` 路徑。

**原因：** iDempiere 預設的 Thread Context ClassLoader 是 Web Application classloader，看不到 bundle 內部的 `web/` 目錄。

**模式：**

```java
@Override
protected void initForm() {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    try {
        Thread.currentThread().setContextClassLoader(getClass().getClassLoader());

        // 在此區塊內執行所有 ~./ 資源載入
        Executions.createComponents("~./zul/MyForm.zul", this, args);
        Selectors.wireComponents(this, this, false);
        // ... 其他初始化 ...

    } catch (Exception e) {
        log.severe("Failed to init form: " + e.getMessage());
    } finally {
        // 必須還原，否則影響其他 bundle
        Thread.currentThread().setContextClassLoader(cl);
    }
}
```

---

## 4. Form Controller（ADForm）

**繼承：** `ADForm implements IFormController, ValueChangeListener`

**職責：**
1. 切換 ClassLoader + 載入 ZUL 模板
2. 建立 ViewModel 實體並透過 `args` 傳入 ZUL
3. 用 `@Wire` 綁定 ZUL 元件到 Java 欄位
4. 建立 iDempiere WebUI 元件（如 `WSearchEditor`）並 appendChild 到 ZUL 容器
5. 實作 `ValueChangeListener` 橋接 iDempiere 元件事件到 ViewModel

### 4.1 完整範例

```java
package tw.topgiga.rnd.form;

import java.util.HashMap;
import java.util.Map;
import java.util.logging.Logger;
import org.adempiere.webui.editor.WSearchEditor;
import org.adempiere.webui.event.ValueChangeEvent;
import org.adempiere.webui.event.ValueChangeListener;
import org.adempiere.webui.panel.ADForm;
import org.adempiere.webui.panel.IFormController;
import org.compiere.model.MColumn;
import org.compiere.model.MLookup;
import org.compiere.model.MLookupFactory;
import org.compiere.util.DisplayType;
import org.compiere.util.Env;
import org.zkoss.bind.Binder;
import org.zkoss.zk.ui.Component;
import org.zkoss.zk.ui.Executions;
import org.zkoss.zk.ui.select.Selectors;
import org.zkoss.zk.ui.select.annotation.Wire;
import tw.topgiga.rnd.viewmodel.DispensingVM;

public class DispensingForm extends ADForm
        implements IFormController, ValueChangeListener {

    private static final long serialVersionUID = 1L;
    private static final Logger log =
        Logger.getLogger(DispensingForm.class.getName());

    private WSearchEditor fProject;

    @Wire("#projectEditorContainer")
    private Component projectEditorContainer;

    @Wire("#dispensingVMContainer")
    private Component dispensingVMContainer;

    @Override
    protected void initForm() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        try {
            Thread.currentThread().setContextClassLoader(getClass().getClassLoader());

            Map<String, Object> args = new HashMap<>();
            args.put("vm", new DispensingVM());
            Executions.createComponents("~./zul/RND_DispensingForm.zul", this, args);

            Selectors.wireComponents(this, this, false);

            MLookup projectLookup = MLookupFactory.get(
                Env.getCtx(), 0, 0,
                MColumn.getColumn_ID("RND_Project", "RND_Project_ID"),
                DisplayType.Search);
            fProject = new WSearchEditor(
                "RND_Project_ID", false, false, true, projectLookup);
            fProject.setMandatory(true);
            fProject.addValueChangeListener(this);

            if (projectEditorContainer != null) {
                projectEditorContainer.appendChild(fProject.getComponent());
            }
        } catch (Exception e) {
            log.severe("Failed to init DispensingForm: " + e.getMessage());
        } finally {
            Thread.currentThread().setContextClassLoader(cl);
        }
    }

    @Override
    public void valueChange(ValueChangeEvent evt) {
        if (evt.getSource() == fProject) {
            Integer projectId = (Integer) evt.getNewValue();
            DispensingVM vm = getViewModel();
            if (vm != null) {
                vm.setSelectedProjectId(projectId);
            }
        }
    }

    private DispensingVM getViewModel() {
        if (dispensingVMContainer == null) return null;
        Binder binder = (Binder) dispensingVMContainer.getAttribute("binder");
        if (binder == null) return null;
        Object vmInstance = binder.getViewModel();
        if (vmInstance instanceof DispensingVM) {
            return (DispensingVM) vmInstance;
        }
        return null;
    }

    @Override
    public ADForm getForm() { return this; }
}
```

### 4.2 關鍵順序（不可調換）

```
1. ClassLoader 切換
2. Executions.createComponents(...)    ← 先建立 ZUL 元件
3. Selectors.wireComponents(...)       ← 再 wire（元件已存在）
4. 建立 iDempiere WebUI 元件
5. appendChild 到 ZUL 容器             ← @Wire 欄位此時已有值
6. finally: 還原 ClassLoader
```

---

## 5. POJO ViewModel

**規則：** ViewModel 是純 POJO，不繼承任何 ZK 類別。

### 5.1 核心 Annotation

| Annotation | 用途 | 注意事項 |
|-----------|------|---------|
| `@Init` | ViewModel 初始化方法 | 由 ZK Binder 在綁定時呼叫 |
| `@Command` | ZUL 中 `@command('methodName')` 觸發 | 必須是 `public void` |
| `@NotifyChange({"prop1","prop2"})` | `@Command` 方法返回後通知 UI 更新指定屬性 | **僅在 `@Command` 方法上有效** |
| `@DependsOn("otherProp")` | getter 依賴另一個屬性，自動連動更新 | 放在 getter 上 |
| `@BindingParam("name")` | 接收 ZUL `@command` 傳入的參數 | 參數名對應 ZUL 中的 key |

### 5.2 `@NotifyChange` vs `BindUtils.postNotifyChange` — 重要區別

| 場景 | 使用方式 |
|------|---------|
| `@Command` 方法內屬性變更 | 用 `@NotifyChange({"prop1","prop2"})` 標註方法 |
| 非 `@Command` 方法（如 Controller 直接呼叫） | 必須手動呼叫 `BindUtils.postNotifyChange(null, null, this, "prop")` |

**規則：** `@NotifyChange` 只在 `@Command` 標註的方法返回時才自動觸發。如果方法是由 Java 程式碼直接呼叫（非 ZUL `@command` 觸發），`@NotifyChange` 不生效，必須改用 `BindUtils.postNotifyChange`。

### 5.3 範例

```java
public class DispensingVM {

    private Integer selectedProjectId;
    private List<MFormula> projectFormulas = new ArrayList<>();
    private Set<MFormula> selectedFormulas = new HashSet<>();
    private BigDecimal scaleFactor = BigDecimal.ONE;

    @Init
    public void init() { }

    // ZUL @command 觸發 → 可用 @NotifyChange
    @Command
    @NotifyChange({"projectFormulas", "selectedFormulas", "batchTickets"})
    public void loadProjectData() {
        projectFormulas.clear();
        if (selectedProjectId == null) return;
        List<MFormula> formulas = new Query(
            Env.getCtx(), MFormula.Table_Name,
            "RND_Project_ID = ? AND IsActive='Y'", null)
            .setParameters(selectedProjectId)
            .setOrderBy("Name").list();
        if (formulas != null) projectFormulas.addAll(formulas);
    }

    @Command
    @NotifyChange({"selectedFormula", "formulaLines"})
    public void selectFormula(@BindingParam("formula") MFormula formula) {
        this.selectedFormula = formula;
        // ... load formula lines ...
    }

    @DependsOn("selectedFormulas")
    public String getSelectedCountText() {
        return "Selected: " + (selectedFormulas != null ? selectedFormulas.size() : 0);
    }

    // Controller 直接呼叫 → 必須用 BindUtils.postNotifyChange
    public void setSelectedProjectId(Integer projectId) {
        this.selectedProjectId = projectId;
        loadProjectData();
        BindUtils.postNotifyChange(null, null, this, "projectFormulas");
        BindUtils.postNotifyChange(null, null, this, "batchTickets");
    }

    // @bind 雙向綁定需要 getter + setter
    public BigDecimal getScaleFactor() { return scaleFactor; }

    @NotifyChange("scaleFactor")
    public void setScaleFactor(BigDecimal sf) { this.scaleFactor = sf; }
}
```

### 5.4 Getter/Setter 規則

- `@load`（單向 VM→UI）：只需 getter
- `@bind`（雙向 VM↔UI）：getter 和 setter **缺一不可**，否則拋 `PropertyNotWritableException`

---

## 6. ZUL 模板

### 6.1 ViewModel 初始化（必要設定）

```xml
<borderlayout hflex="1" vflex="1"
    id="dispensingVMContainer"
    apply="org.zkoss.bind.BindComposer"
    viewModel="@id('vm') @init(arg.vm)">
    ...
</borderlayout>
```

| 屬性 | 用途 |
|-----|------|
| `apply="org.zkoss.bind.BindComposer"` | 啟用 ZK MVVM 綁定 |
| `viewModel="@id('vm') @init(arg.vm)"` | 使用 Controller 傳入的 VM 實體（非 ZK 自行 new） |
| `id="dispensingVMContainer"` | 讓 Controller 能 `@Wire` 取得此元件以存取 Binder/ViewModel |

### 6.2 資料綁定語法速查

```xml
<!-- 單向綁定 VM → UI -->
<label value="@load(vm.selectedCountText)"/>

<!-- 雙向綁定 VM ↔ UI -->
<doublebox value="@bind(vm.scaleFactor)" format="##0.###"/>

<!-- 列表綁定 + 多選 -->
<listbox model="@load(vm.projectFormulas)"
         selectedItems="@bind(vm.selectedFormulas)"
         checkmark="true" multiple="true">
    <template name="model" var="f">
        <listitem onClick="@command('selectFormula', formula=f)">
            <listcell label="@load(f.name)"/>
        </listitem>
    </template>
</listbox>

<!-- Command 呼叫 -->
<button label="Generate" onClick="@command('generateBatchTickets')"/>

<!-- 條件 disabled -->
<button label="Compare"
        onClick="@command('compareFormulas')"
        disabled="@load(empty vm.selectedFormulas)"/>
```

### 6.3 iDempiere 元件容器

在 ZUL 中預留空 `<div>`，由 Controller 的 `@Wire` + `appendChild` 放入 iDempiere 元件：

```xml
<div id="projectEditorContainer" width="250px" style="min-height:25px;"/>
```

---

## 7. iDempiere WebUI 元件注入

### 7.1 WSearchEditor 建立模式

```java
// 1) MLookup
MLookup lookup = MLookupFactory.get(
    Env.getCtx(), 0, 0,
    MColumn.getColumn_ID("TableName", "ColumnName_ID"),
    DisplayType.Search);

// 2) WSearchEditor
WSearchEditor editor = new WSearchEditor("ColumnName_ID", false, false, true, lookup);
editor.setMandatory(true);
editor.addValueChangeListener(this);

// 3) 加入 ZUL 容器
container.appendChild(editor.getComponent());
```

### 7.2 透過 Binder 取得 ViewModel

```java
private MyViewModel getViewModel() {
    if (vmContainer == null) return null;
    Binder binder = (Binder) vmContainer.getAttribute("binder");
    if (binder == null) return null;
    Object vm = binder.getViewModel();
    return (vm instanceof MyViewModel) ? (MyViewModel) vm : null;
}
```

---

## 8. IFormFactory 與 OSGI 註冊

### 8.1 IFormFactory 實作

```java
public class RNDFormFactory implements IFormFactory {
    @Override
    public ADForm newFormInstance(String formName) {
        if (formName == null) return null;
        if (formName.equals(DispensingForm.class.getName()))
            return new DispensingForm();
        return null;
    }
}
```

### 8.2 OSGI Service Component XML

`OSGI-INF/rnd_form_factory.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<scr:component xmlns:scr="http://www.osgi.org/xmlns/scr/v1.1.0"
    name="tw.topgiga.rnd.factories.RNDFormFactory">
    <implementation class="tw.topgiga.rnd.factories.RNDFormFactory"/>
    <property name="service.ranking" type="Integer" value="10"/>
    <service>
        <provide interface="org.adempiere.webui.factory.IFormFactory"/>
    </service>
</scr:component>
```

### 8.3 MANIFEST.MF 必要行

```
Service-Component: OSGI-INF/rnd_form_factory.xml
```

### 8.4 AD_Form 設定

在 Application Dictionary 建立 Form 記錄：
- **Classname:** 完整類別名（如 `tw.topgiga.rnd.form.DispensingForm`）
- 建立 Menu 項目指向此 Form

### 8.5 完整 MANIFEST.MF 範例

```
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-Name: Rnd
Bundle-SymbolicName: tw.topgiga.rnd
Bundle-Version: 1.0.0.qualifier
Automatic-Module-Name: tw.topgiga.rnd
Bundle-ClassPath: src/, .
Bundle-RequiredExecutionEnvironment: JavaSE-17
Bundle-ActivationPolicy: lazy
Service-Component: OSGI-INF/rnd_form_factory.xml
Require-Bundle:
 org.adempiere.base;bundle-version="12.0.0",
 org.adempiere.ui.zk;bundle-version="12.0.0",
 zul;bundle-version="9.6.0",
 zk;bundle-version="9.6.0",
 zkbind;bundle-version="9.6.0",
 zcommon;bundle-version="9.6.0"
Import-Package:
 org.compiere.model,
 org.compiere.process,
 org.compiere.util,
 org.adempiere.base,
 org.adempiere.webui.editor,
 org.adempiere.webui.event,
 org.adempiere.webui.factory,
 org.adempiere.webui.panel,
 org.zkoss.bind,
 org.zkoss.bind.annotation,
 org.zkoss.zk.ui,
 org.zkoss.zk.ui.event,
 org.zkoss.zk.ui.select,
 org.zkoss.zk.ui.select.annotation,
 org.zkoss.zk.ui.util,
 org.zkoss.zul
Web-ContextPath: /rnd
```

---

## 9. JavaScript 整合（以 ECharts 為例）

### 9.1 禁止做法（全部會失敗）

| 做法 | 失敗原因 |
|------|---------|
| `<script>` 作為 `<borderlayout>` 子元素 | `<borderlayout>` 只接受 north/south/east/west/center |
| `<script>` 放在 `visible="false"` 容器內 | 執行時機不可預測，函數未定義 |
| `<?script src="~./js/xxx.js"?>` | `~./` 解析到主 webapp `/webui/`，非 bundle 的 `Web-ContextPath` |
| `<?script src="/rnd/js/xxx.js"?>` | OSGI bundle 的 `Web-ContextPath` 無法 serve 靜態 JS 檔 |
| `w:onCreate` 設定固定 DOM ID | ZK widget lifecycle 會覆蓋手動設定的 DOM ID，不可靠 |

> **唯一例外：** `<?script src="CDN URL"?>` 載入外部 CDN 可以成功，但 bundle 內部 JS 不行。

### 9.2 正確做法：`Clients.evalJavaScript()`

從 Java 端直接發送 JavaScript 到瀏覽器執行，繞過 ZK 路徑解析和元件生命週期。

#### 步驟一：initForm() 中動態載入 CDN + 定義函數

```java
private void loadChartScripts() {
    String js =
        "if(!window.echarts){" +
            "var s=document.createElement('script');" +
            "s.src='https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js';" +
            "s.onload=function(){" + getChartFunctionJs() + "};" +
            "document.head.appendChild(s);" +
        "}else if(!window.myChartFunction){" +
            getChartFunctionJs() +
        "}";
    Clients.evalJavaScript(js);
}
```

防禦邏輯：
- `if(!window.echarts)` → 首次載入：建立 `<script>` + onload 定義函數
- `s.onload` → 確保 CDN 載入完畢才定義函數
- `else if(!window.myChartFunction)` → 重複開啟 Form 時不重複載入 CDN

#### 步驟二：ZUL 用 CSS class（sclass）標記容器，不用 id

```xml
<div sclass="rnd-trend-chart" hflex="1" height="750px"
     visible="@load(not empty vm.testResults)"/>
```

**規則：** 不要用 `id` 或 `w:onCreate` 設定 DOM ID。ZK 管理 DOM ID，可能覆蓋。CSS class（`sclass`）ZK 不會動，用 `querySelector` 始終可靠。

#### 步驟三：ViewModel 中用 IIFE + setTimeout 重試

```java
@Command
public void refreshChart() {
    String jsonData = buildJsonData();
    Clients.evalJavaScript(
        "(function _try(){" +
            "var dom=document.querySelector('.rnd-trend-chart');" +
            "if(!dom||!window.echarts){setTimeout(_try,200);return;}" +
            "var chart=echarts.getInstanceByDom(dom)||echarts.init(dom);" +
            "chart.setOption(" + jsonData + ");" +
        "})()");
}
```

重試機制：
- `querySelector('.rnd-trend-chart')` → CSS class 定位
- `if(!dom||!window.echarts)` → 未就緒則 200ms 後重試
- `getInstanceByDom(dom)||echarts.init(dom)` → 有實例複用，沒有新建

### 9.3 Layout 規則

**固定高度 + vflex 衝突：**

```xml
<!-- ❌ 750px div 會把 vflex="1" listbox 擠到 0 高度 -->
<vlayout vflex="1">
    <listbox vflex="1"/>
    <div height="750px"/>
</vlayout>

<!-- ✅ 全部用固定 height + overflow-y:auto -->
<vlayout hflex="1" vflex="1" style="overflow-y:auto;">
    <listbox height="200px"/>
    <div sclass="rnd-trend-chart" height="750px"/>
</vlayout>
```

**動態高度（多子圖）：**

```javascript
var n = data.specs.length || 1;
dom.style.height = (n * 250) + 'px';
// 必須先 dispose 再 init，ECharts 初始化時記住容器尺寸
var chart = echarts.getInstanceByDom(dom);
if (chart) { chart.dispose(); }
chart = echarts.init(dom);
```

---

## 10. 常見錯誤速查表

| 錯誤訊息 / 症狀 | 原因 | 解法 |
|----------------|------|------|
| `Page not found: ~./zul/MyForm.zul` | Thread Context ClassLoader 非 bundle classloader | `initForm()` 中切換 ClassLoader（見 §3.3） |
| CSS `url('~./images/bg.png')` 無效 | CSS `url()` 由瀏覽器解析，不經 ZK 的 `~./` | 改用 Base64 Data URI：`url('data:image/png;base64,...')` |
| `@NotifyChange` 不觸發 UI 更新 | 方法非 `@Command` 標註，`@NotifyChange` 不生效 | 改用 `BindUtils.postNotifyChange(null, null, this, "prop")` |
| `@Wire` 欄位為 null | `wireComponents` 在 `createComponents` 之前呼叫 | 確保順序：先 `createComponents` → 再 `wireComponents` |
| `PropertyNotWritableException` | `@bind` 雙向綁定缺少 setter | 補上 setter 方法 |
| `ClassCastException`（Yes-No 欄位） | `PO.get_Value()` 回傳 `Boolean` 而非 `String` | 先 `instanceof Boolean` 檢查再轉換 |
| `varchar(1)` 存不下 | 值超過一個字元 | DB 存縮寫（P/F），顯示用另一個 getter 返回全名 |
| ZK Parser 警告（重複 BindComposer） | `apply="BindComposer"` 與 `viewModel="@init(...)` 重複 | 移除多餘的 `apply="org.zkoss.bind.BindComposer"`（ZK 自動套用） |
| `Unsupported child for Borderlayout` | `<script>` 作為 `<borderlayout>` 子元素 | 不要在 ZUL 用 `<script>`，改用 `Clients.evalJavaScript()` |
| JS 函數 `is not defined` | ZUL `<script>` / `<?script?>` 載入失敗或時機錯誤 | 改用 `Clients.evalJavaScript()` 動態載入（見 §9.2） |
| MANIFEST.MF 需要 `Export-Package`？ | — | **不需要。** `~./` 透過 ClassLoader 存取內部資源，不需 export |
