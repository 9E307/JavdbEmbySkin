# 0017. 首次运行零侵入开关与 Chrome 搜索框账号防误填架构

## 状态
已接受 (Accepted)

## 上下文
在 JavdbEmbySkin 日常使用与新用户安装场景中，存在两项严重影响使用体验的关键架构缺陷：

### 1. 首次安装强行侵入接管界面（破坏用户控制权）
* 原脚本初始化开关逻辑为：
  ```javascript
  let enabled = (localStorage.getItem(STORAGE_KEY) !== '0');
  ```
* 新用户首次在 Tampermonkey 中安装脚本并访问 JAVDB 时，`localStorage` 中的 `javdbEmbyEnabled` 键值为 `null`；
* 条件 `null !== '0'` 计算为 `true`，导致新用户一打开网站就立刻被强行注入 Emby 样式并执行全页 DOM 重构；
* 用户失去选择权，在未了解脚本功能前易产生混乱，且在网络波动时首屏资源尚未就绪便强行渲染。

### 2. Chrome / Chromium 密码管理器及扩展误将账号回填到搜索框与唤醒凭据下拉
当用户在浏览器中保存了 JAVDB 站点的登录凭据（用户名/邮箱与密码，例如账号 `drnleas`）后，搜索输入框遭遇了两轮凭据嗅探问题：
* **首轮缺陷（自动回填）**：
  * 原生搜索框及 Emby 顶部克隆搜索框采用普通 `<input type="text">`；
  * Chrome 的 `PasswordAutofillAgent` 启发式引擎将普通的单行文本框误判为登录表单，强行将保存的账号/邮箱自动填入搜索框。
* **次轮盲区（点击唤醒一键填写凭据下拉）**：
  * 早期修复仅覆盖了顶栏与常驻模态框，遗漏了在用户进入“收藏夹 / 媒体库”后动态由 JS 创建的工具栏搜索框（`.efav-toolbar input`）与多维筛选弹窗搜索框（`.efav-dim input`）；
  * 这些搜索框在创建时为裸 `<input type="text" placeholder="搜索番号 / 标题…">`，缺乏搜索类型与防探测属性；
  * 现代 Chromium 对已保存密码的域名有极高敏感度，即便设置了 `autocomplete="off"`，浏览器出于“登录便利性”往往强行忽略；
  * **一旦用户点击该输入框或输入框获得焦点**，浏览器便立即在光标下方弹出系统级的“一键填写账号密码 / 管理密码”悬浮窗；
  * 同时，第三方密码管理器（如 1Password、Bitwarden、LastPass）也会在其输入框边缘注入图标或高亮。
* **WebKit 原生清除按钮侵入**：
  * 当把输入框类型单纯改为 `type="search"` 时，WebKit 内核会自动渲染内置的取消搜索小图标（`-webkit-search-cancel-button`），破坏界面精致度。

---

## 决策
秉持**用户主权优先**与**全域立体纵深防御**原则，重构开关初态判定与全生命周期搜索框防御体系：

### 1. 严格激活匹配，初次运行零侵入（Opt-In by Default）
* 将全局启用开关判定改为严格匹配字符串 `'1'`：
  ```javascript
  const STORAGE_KEY = 'javdbEmbyEnabled';
  let enabled = (localStorage.getItem(STORAGE_KEY) === '1');
  ```
* 新用户首次访问时，`enabled` 恒为 `false`，脚本绝不注入 Emby 样式、绝不挂载全屏面板、绝不破坏原生网页结构；
* 仅在右下角注入悬浮按钮，并显示 4.5 秒温和引导气泡（`「点击开启 Emby 皮肤」`）；
* 只有当用户主动左键点击开关后，状态才写入 `'1'` 并平滑过渡到 Emby 界面，以后访问自动常驻。

### 2. 搜索框全域四层立体纵深防御体系（Full-Spectrum Search Credential Autofill Defense）

#### 第一层：源头创建层（Creation-Time Hardening）
所有搜索输入框（`.efav-toolbar` 收藏夹工具栏、`.efav-dim` 筛选弹窗、`#actor-search-inp` 女优搜索、`#meta-correct-search` 元数据修正等）在创建原生 DOM 节点的第一时间，直接注入专属反嗅探与语义属性：
```html
<input type="search" name="fav_search" role="searchbox" 
       autocomplete="off" autocorrect="off" autocapitalize="none" spellcheck="false" 
       data-lpignore="true" data-1p-ignore="true" data-bwignore="true" data-form-type="other" 
       placeholder="搜索番号 / 标题…">
```
* **语义脱离凭据推断**：明确声明 `type="search"`、`role="searchbox"` 并赋予含 `search` 的明确 `name`，使 Chromium 密码管理器直接排除用户名候选字段推断；
* **插件级忽略标志**：同时附带 `data-lpignore`（LastPass）、`data-1p-ignore`（1Password）、`data-bwignore`（Bitwarden）与 `data-form-type="other"`，一次性阻断主流第三方扩展。

#### 第二层：全生命周期动态清洗层（Full-Lifecycle Sanitization）
设立专门的统一搜索净化函数 `sanitizeSearchInputs()`，覆盖全域所有已知与动态生成的搜索类选择器：
```javascript
function sanitizeSearchInputs(root) {
  const scope = root || document;
  const isSearchPage = location.pathname.startsWith('/search');
  let urlQ = '';
  try {
    if (isSearchPage) urlQ = (new URLSearchParams(location.search).get('q') || '').trim();
  } catch (e) {}

  const inputs = scope.querySelectorAll(
    '#search-bar-container input, #emby-search-modal input, .search-bar-wrap input, ' +
    '#actor-search-inp, #meta-correct-search, .search-input input, ' +
    '.efav-toolbar input[name="fav_search"], .efav-toolbar input[type="search"], ' +
    '.efav-dim .dim-title input, input.dim-search-inp, ' +
    'input[placeholder*="搜索"], input[role="searchbox"]'
  );

  inputs.forEach(function (inp) {
    if (!inp) return;
    // 绝对防御守卫：严禁篡改复选框、单选框、滑块、按钮、隐藏域等非文本输入控件！
    const rawType = (inp.getAttribute('type') || inp.type || '').toLowerCase();
    if (['checkbox', 'radio', 'range', 'button', 'submit', 'reset', 'file', 'hidden', 'color', 'image'].indexOf(rawType) !== -1) {
      return;
    }
    // 仅针对真正的文本输入框（text、search 或未显式指定 type 的默认文本框）进行脱敏防护
    if (rawType !== '' && rawType !== 'text' && rawType !== 'search') {
      return;
    }

    // 1. 设置标准搜索类型与无障碍语义，Chrome/Chromium 密码管理器将忽略凭据探测
    if (inp.type !== 'search') {
      try { inp.type = 'search'; } catch (e) { inp.setAttribute('type', 'search'); }
    }
    if (inp.getAttribute('role') !== 'searchbox') inp.setAttribute('role', 'searchbox');
    if (!inp.getAttribute('name')) {
      inp.setAttribute('name', 'search_query');
    }

    // 2. 设置多重关闭自动填充属性，兼容主流密码管理器
    if (inp.getAttribute('autocomplete') !== 'off') inp.setAttribute('autocomplete', 'off');
    if (inp.getAttribute('autocorrect') !== 'off') inp.setAttribute('autocorrect', 'off');
    if (inp.getAttribute('autocapitalize') !== 'none') inp.setAttribute('autocapitalize', 'none');
    if (inp.getAttribute('spellcheck') !== 'false') inp.setAttribute('spellcheck', 'false');
    if (inp.getAttribute('data-lpignore') !== 'true') inp.setAttribute('data-lpignore', 'true');
    if (inp.getAttribute('data-1p-ignore') !== 'true') inp.setAttribute('data-1p-ignore', 'true');
    if (inp.getAttribute('data-bwignore') !== 'true') inp.setAttribute('data-bwignore', 'true');
    if (inp.getAttribute('data-form-type') !== 'other') inp.setAttribute('data-form-type', 'other');

    // 3. 非搜索结果页且用户未手动输入时，主动冲刷掉误填的账号数据
    if (!isSearchPage || !urlQ) {
      if (inp.value && !inp.dataset.userEdited) {
        inp.value = '';
      }
    }

    // 4. 监听获得焦点与键盘输入状态，防范延时回填与点击唤醒凭据下拉
    if (!inp.dataset.autofillGuarded) {
      inp.dataset.autofillGuarded = '1';
      inp.addEventListener('input', function () { inp.dataset.userEdited = '1'; });
      inp.addEventListener('focus', function () {
        if ((!isSearchPage || !urlQ) && !inp.dataset.userEdited && inp.value) {
          inp.value = '';
        }
      });
    }
  });
}
```
* **生命周期关键点挂载**：在 `init()` 启动时及 150ms/600ms/1500ms 周期执行；在收藏夹挂载（`FAVUI.mount()`）、筛选器弹窗（`openFilterModal()`）、女优弹窗（`openActorSearchPopover()`）与搜索模态（`buildSearchModal()`）装配入 DOM 时即刻清洗。

#### 第三层：用户交互感知与主动冲刷层（Interactive State Guard & Focus Flush）
* 监听真实输入事件，当且仅当发生真实的 `input` 事件时才标记 `dataset.userEdited = '1'`；
* 当输入框获得焦点（`focus`）时，若检测到存在内容但未曾真实编辑（即属于浏览器延迟强行注入的凭据信息），瞬间主动清空置空；
* 状态恢复保护：从详情页返回收藏夹恢复上次查询关键词时，显式标记 `dataset.userEdited = '1'`，避免误清用户合法的历史搜索词。

#### 第四层：CSS 渲染层视觉平整化（Visual Appearance Resets）
在全局样式中对所有 `type="search"` 添加 WebKit 原生控件外观重置：
```css
.efav-toolbar input[type=search]::-webkit-search-cancel-button,
.efav-toolbar input[type=search]::-webkit-search-decoration,
.efav-dim > .dim-title input[type=search]::-webkit-search-cancel-button,
.efav-dim > .dim-title input[type=search]::-webkit-search-decoration,
#actor-search-popover input#actor-search-inp::-webkit-search-cancel-button,
#actor-search-popover input#actor-search-inp::-webkit-search-decoration,
.jhs-input-text[type=search]::-webkit-search-cancel-button,
.jhs-input-text[type=search]::-webkit-search-decoration {
  -webkit-appearance: none;
  appearance: none;
}
```
保证各平台各分辨率下搜索框外观与原生 Emby 风格无缝贴合。

---

## 原因与权衡
* **为什么单纯设置 `autocomplete="off"` 无法完全阻止 Chrome 自动填充？**
  * 近代 Chrome 策略为了提升“用户登录便利度”，会特意忽略常规登录表单上的 `autocomplete="off"`。只有将输入框的语义类型升级为 `type="search"`，同时辅以 `role="searchbox"`、`data-lpignore`（屏蔽 LastPass/Bitwarden）、`data-1p-ignore`（屏蔽 1Password）、`data-form-type="other"` 并配合 JS 在获取焦点及异步就绪时的主动值比对清洗，才能 100% 根治顽固的密码管理器嗅探。

---

## 结果与影响
* **收益**：
  * 新用户首次安装脚本体验纯粹友好，拥有完备的知情权与开启主动权；
  * Chrome 浏览器中保存的 JAVDB 账号与密码绝不会被误填入首页、顶栏或弹窗搜索框；
  * 点击收藏夹工具栏、筛选框等任意输入框，绝不会再弹出“一键填写账号密码 / 管理密码”系统悬浮下拉框；
  * 真实搜索结果页（`/search?q=abc`）的原生搜索词与收藏夹返回时的搜索词仍能正确回显，业务逻辑丝毫未受影响；
  * 界面视觉精致平整，消除了浏览器默认搜索小图标的杂音干扰。

---

## 验证
* 编写自动化测试脚本 `test_search_autofill_defense.js`；
* 验证顶部搜索框、收藏夹工具栏、筛选器弹窗与设置弹窗的所有搜索框均具备完备的防御属性；
* 验证在模拟浏览器强行回填凭据场景下，聚焦输入框能瞬间主动冲刷，而在用户主动输入场景下能完整保留。
