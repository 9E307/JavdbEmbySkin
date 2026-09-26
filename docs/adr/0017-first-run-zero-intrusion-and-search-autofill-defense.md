# 0017. 首次运行零侵入开关与 Chrome 搜索框账号防误填架构

## 状态
已接受 (Accepted)

## 上下文
在 JavdbEmbySkin 日常使用与新用户安装场景中，存在两项影响使用体验的关键架构缺陷：

1. **首次安装强行侵入接管界面（破坏用户控制权）**：
   * 原脚本第 199 行初始化开关逻辑为：
     ```javascript
     let enabled = (localStorage.getItem(STORAGE_KEY) !== '0');
     ```
   * 新用户首次在 Tampermonkey 中安装脚本并访问 JAVDB 时，`localStorage` 中的 `javdbEmbyEnabled` 键值为 `null`。
   * 条件 `null !== '0'` 计算为 `true`，导致新用户一打开网站就立刻被强行注入 Emby 样式并执行全页 DOM 重构。
   * 用户失去选择权，在未了解脚本功能前易产生混乱，且在网络波动时首屏资源尚未就绪便强行渲染。

2. **Chrome / Chromium 密码管理器误将账号回填到搜索框**：
   * 当用户在浏览器中保存了 JAVDB 站点的登录凭据（用户名/邮箱与密码）后，Chrome 密码管理器的自动填充启发式引擎（Autofill Heuristics）会主动探测页面上的 `<input>` 元素；
   * JAVDB 原版搜索框（以及 Emby 顶部克隆搜索框、弹窗搜索框 `#emby-search-modal`、女优搜索框 `#actor-search-inp` 等）使用的是普通的 `<input type="text">`；
   * Chrome 无法区分类似 `q` 或普通文本框与真实的登录表单，经常误将用户的登录账号/邮箱自动填入搜索框；
   * 用户在搜索栏中总能看到自己的账号名，不仅需要手动退格清空，还存在隐私外泄风险。

---

## 决策
秉持用户主权优先与全生命周期防御原则，重构**开关初态判定与统一搜索输入框防御流水线**：

1. **严格激活匹配，初次运行零侵入（Opt-In by Default）**：
   * 将全局启用开关判定改为严格匹配字符串 `'1'`：
     ```javascript
     const STORAGE_KEY = 'javdbEmbyEnabled';
     let enabled = (localStorage.getItem(STORAGE_KEY) === '1');
     ```
   * 新用户首次访问时，`enabled` 恒为 `false`，脚本绝不注入 Emby 样式、绝不挂载全屏面板、绝不破坏原生网页结构；
   * 仅在右下角注入悬浮按钮，并显示 4.5 秒温和引导气泡（`「点击开启 Emby 皮肤」`）；
   * 只有当用户主动左键点击开关后，状态才写入 `'1'` 并平滑过渡到 Emby 界面，以后访问自动常驻。

2. **搜索输入框多维反嗅探防御流水线（`sanitizeSearchInputs`）**：
   * 设立专门的统一搜索净化函数：
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
         '#actor-search-inp, #meta-correct-search, .dim-box input[type="text"], .search-input input'
       );

       inputs.forEach(function (inp) {
         if (!inp) return;
         // 1. 类型强制升级为 search，跳过密码管理器启发式嗅探
         if (inp.type !== 'search') {
           try { inp.type = 'search'; } catch (e) { inp.setAttribute('type', 'search'); }
         }
         // 2. 配置多重安全属性，压制各大密码管理插件
         inp.setAttribute('autocomplete', 'off');
         inp.setAttribute('autocorrect', 'off');
         inp.setAttribute('autocapitalize', 'none');
         inp.setAttribute('spellcheck', 'false');
         inp.setAttribute('data-lpignore', 'true');
         inp.setAttribute('data-form-type', 'other');

         // 3. 非搜索结果页且用户未手动输入时，主动冲刷掉误填的账号数据
         if (!isSearchPage || !urlQ) {
           if (inp.value && !inp.dataset.userEdited) {
             inp.value = '';
           }
         }

         // 4. 监听获得焦点与键盘输入状态，防范延时回填
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
   * **全生命周期挂载**：在 `init()` 执行即刻检查，并在 150ms、600ms、1500ms（覆盖 Chrome 异步回填阶段）、搜索弹窗打开时（`openSearchModal`）、克隆装配时（`wireSearchClone`）及 `observeMutations` 中全方位守护。

---

## 原因与权衡
* **为什么单纯设置 `autocomplete="off"` 无法完全阻止 Chrome 自动填充？**
  * 近代 Chrome 策略为了提升“用户登录便利度”，会特意忽略常规登录表单上的 `autocomplete="off"`。只有将输入框的语义类型升级为 `type="search"`，同时辅以 `data-lpignore`（屏蔽 LastPass/Bitwarden）、`data-form-type="other"` 并配合 JS 在获取焦点及异步就绪时的主动值比对清洗，才能 100% 根治顽固的密码管理器嗅探。

---

## 结果与影响
* **收益**：
  * 新用户首次安装脚本体验纯粹友好，拥有完备的知情权与开启主动权；
  * Chrome 浏览器中保存的 JAVDB 账号与密码绝不会被误填入首页、顶栏或弹窗搜索框；
  * 真实搜索结果页（`/search?q=abc`）的原生搜索词仍能正确回显，业务逻辑丝毫未受影响。
