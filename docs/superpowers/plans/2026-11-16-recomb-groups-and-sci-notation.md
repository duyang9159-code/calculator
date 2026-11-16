# 同源重组多组计算 + 悬液总细胞量科学计数法 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让多片段同源重组模块支持最多 5 个实验组并行计算,并将悬液总细胞量显示改为自适应科学计数法。

**Architecture:** 单文件 HTML 应用(F:\Projects\calculator\index.html),所有逻辑为内联 JS。参照现有转染模块的"组选择器 + 保存-重建渲染 + 转置表格"模式改造重组模块;细胞计数模块复用现有 `toSciSup` 函数并新增纯文本版 `toSciText`。无构建系统、无测试框架,验证方式为浏览器手动核对 + 可选 Playwright(webapp-testing skill)。

**Tech Stack:** 原生 HTML/CSS/JS,无依赖。

**Spec:** `docs/superpowers/specs/2026-11-16-recomb-groups-and-sci-notation-design.md`

---

### Task 1: 重组模块 HTML 改造(组数选择器 + 移除顶部载体输入)

**Files:**
- Modify: `F:\Projects\calculator\index.html:458-471`

- [ ] **Step 1: 替换 input-grid 内容**

将第 458-471 行的 `<div class="input-grid">...</div>`(当前含 4 个 input-field:hr_vector_bp、hr_vector_conc、hr_vector_mass、fragment_count)整体替换为:

```html
        <div class="input-grid">
            <div class="input-field"><label>插入片段数量</label>
                <select id="fragment_count">
                    <option value="1">1 个片段</option>
                    <option value="2">2 个片段</option>
                    <option value="3" selected>3 个片段</option>
                    <option value="4">4 个片段</option>
                    <option value="5">5 个片段</option>
                </select>
            </div>
            <div class="input-field"><label>实验组数</label>
                <select id="hr_group_count">
                    <option value="1" selected>1 组</option>
                    <option value="2">2 组</option>
                    <option value="3">3 组</option>
                    <option value="4">4 组</option>
                    <option value="5">5 组</option>
                </select>
            </div>
        </div>
```

- [ ] **Step 2: 浏览器打开确认无 JS 报错**

打开 `F:\Projects\calculator\index.html`,点进"多片段重组"模块。预期:此时该模块只剩两个下拉框,`fragments_container` 与结果区为空或旧内容;控制台可能报错(hr_vector_* 已不存在,Task 3 修复前属预期),**不要**在此步骤处理。

- [ ] **Step 3: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(recomb): 添加实验组数选择器,移除顶部载体输入框"
```

---

### Task 2: 渲染逻辑改为"组 × 片段"并保留已填值

**Files:**
- Modify: `F:\Projects\calculator\index.html:1100-1134`(替换 `currentFragmentCount` 声明与 `renderFragments` 整个函数)

- [ ] **Step 1: 替换第 1100-1134 行代码**

原代码(从 `// --- 2. 多片段重组 ---` 到 `renderFragments` 函数结束的 `}`)整体替换为:

```js
    // --- 2. 多片段重组 ---
    let currentFragmentCount = 3;
    let currentHrGroupCount = 1;

    const FRAG_DEFAULTS = { bp: [2000,1500,800,600,400], conc: [500,400,300,250,200], ratio: 2 };
    const HR_VECTOR_DEFAULTS = { bp: 12000, conc: 1000, mass: 100 };

    function renderFragments() {
        let count = parseInt(document.getElementById('fragment_count').value) || 3;
        let gCount = parseInt(document.getElementById('hr_group_count').value) || 1;
        // Save current values before re-rendering
        let saved = {};
        for (let g = 1; g <= 5; g++) {
            saved[g] = { vector: {}, frags: [] };
            ['bp','conc','mass'].forEach(s => {
                const el = document.getElementById(`hrg${g}_vector_${s}`);
                if (el) saved[g].vector[s] = el.value;
            });
            for (let i = 1; i <= 5; i++) {
                const row = {};
                ['bp','conc','ratio'].forEach(s => {
                    const el = document.getElementById(`hrg${g}_frag${i}_${s}`);
                    if (el) row[s] = el.value;
                });
                saved[g].frags.push(row);
            }
        }
        currentFragmentCount = count;
        currentHrGroupCount = gCount;
        const container = document.getElementById('fragments_container');
        container.innerHTML = '';
        for (let g = 1; g <= gCount; g++) {
            const groupDiv = document.createElement('div');
            groupDiv.className = 'card-item';
            const vec = saved[g].vector;
            let html = `<div class="card-item-title">实验组 ${g}</div>
                <div style="display:grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap:10px;">
                    <div class="input-field"><label>载体长度（bp）</label><input type="number" id="hrg${g}_vector_bp" value="${vec.bp || HR_VECTOR_DEFAULTS.bp}"></div>
                    <div class="input-field"><label>载体浓度（ng/μL）</label><input type="number" id="hrg${g}_vector_conc" value="${vec.conc || HR_VECTOR_DEFAULTS.conc}"></div>
                    <div class="input-field"><label>载体加入质量（ng）</label><input type="number" id="hrg${g}_vector_mass" value="${vec.mass || HR_VECTOR_DEFAULTS.mass}"></div>
                </div>
                <div style="display:grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap:10px; margin-top:10px;">`;
            for (let i = 1; i <= count; i++) {
                const row = saved[g].frags[i-1] || {};
                html += `
                    <div class="input-field"><label>片段 ${i} 长度（bp）</label><input type="number" id="hrg${g}_frag${i}_bp" value="${row.bp || FRAG_DEFAULTS.bp[i-1] || 500}"></div>
                    <div class="input-field"><label>片段 ${i} 浓度（ng/μL）</label><input type="number" id="hrg${g}_frag${i}_conc" value="${row.conc || FRAG_DEFAULTS.conc[i-1] || 100}"></div>
                    <div class="input-field"><label>片段 ${i} 摩尔比 (片段:载体)</label><input type="number" id="hrg${g}_frag${i}_ratio" value="${row.ratio || FRAG_DEFAULTS.ratio}" step="0.5"></div>
                `;
            }
            html += `</div>`;
            groupDiv.innerHTML = html;
            container.appendChild(groupDiv);
        }
        for (let g = 1; g <= gCount; g++) {
            ['bp','conc','mass'].forEach(s => document.getElementById(`hrg${g}_vector_${s}`).addEventListener('input', updateMultiFragment));
            for (let i = 1; i <= count; i++) {
                ['bp','conc','ratio'].forEach(s => document.getElementById(`hrg${g}_frag${i}_${s}`).addEventListener('input', updateMultiFragment));
            }
        }
        updateMultiFragment();
    }
```

- [ ] **Step 2: 浏览器验证渲染**

打开页面进入"多片段重组":预期默认显示 1 张"实验组 1"卡片,内含载体 3 个输入 + 片段 1-3 共 9 个输入,默认值分别为 12000/1000/100 与 2000/500/2、1500/400/2、800/300/2。结果区暂由 Task 3 接管(当前仍显示旧逻辑报错属预期)。

- [ ] **Step 3: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(recomb): 组×片段渲染,切换时保留已填值"
```

---

### Task 3: 逐组计算 + 转置结果表格

**Files:**
- Modify: `F:\Projects\calculator\index.html:1136-1163`(替换 `updateMultiFragment` 整个函数)

- [ ] **Step 1: 替换第 1136-1163 行代码**

将 `updateMultiFragment` 函数整体替换为:

```js
    function updateMultiFragment() {
        let html = `<div class="table-wrapper"><table class="lab-table"><thead><tr><th>组分</th>`;
        for (let g = 1; g <= currentHrGroupCount; g++) html += `<th>组 ${g}</th>`;
        html += `</tr></thead><tbody>`;
        html += `<tr><td style="font-weight:600;">载体体积</td>`;
        for (let g = 1; g <= currentHrGroupCount; g++) {
            let vecConc = parseFloat(document.getElementById(`hrg${g}_vector_conc`)?.value) || 1;
            let vecMass = parseFloat(document.getElementById(`hrg${g}_vector_mass`)?.value) || 0;
            html += `<td>${(vecMass / vecConc).toFixed(2)} μL</td>`;
        }
        html += `</tr>`;
        for (let i = 1; i <= currentFragmentCount; i++) {
            html += `<tr><td style="font-weight:600;">片段 ${i}</td>`;
            for (let g = 1; g <= currentHrGroupCount; g++) {
                let vecBp = parseFloat(document.getElementById(`hrg${g}_vector_bp`)?.value) || 1;
                let vecMass = parseFloat(document.getElementById(`hrg${g}_vector_mass`)?.value) || 0;
                let bp = parseFloat(document.getElementById(`hrg${g}_frag${i}_bp`)?.value) || 1;
                let conc = parseFloat(document.getElementById(`hrg${g}_frag${i}_conc`)?.value) || 1;
                let ratio = parseFloat(document.getElementById(`hrg${g}_frag${i}_ratio`)?.value) || 1;
                let reqMass = (vecMass / vecBp) * ratio * bp;
                let reqVol = reqMass / conc;
                html += `<td>${reqVol.toFixed(2)} μL (${reqMass.toFixed(1)} ng)</td>`;
            }
            html += `</tr>`;
        }
        html += `</tbody></table></div>`;
        document.getElementById('multiFragmentResult').innerHTML = html;
        debounceRecord("recomb", () => { recordHistory('recomb', { "片段数": currentFragmentCount, "组数": currentHrGroupCount }, buildHrOutputs(), {
            fragment_count: document.getElementById('fragment_count').value,
            hr_group_count: document.getElementById('hr_group_count').value
        }); });
    }

    function buildHrOutputs() {
        let outputs = {};
        for (let g = 1; g <= currentHrGroupCount; g++) {
            let vecConc = parseFloat(document.getElementById(`hrg${g}_vector_conc`)?.value) || 1;
            let vecMass = parseFloat(document.getElementById(`hrg${g}_vector_mass`)?.value) || 0;
            let vecBp = parseFloat(document.getElementById(`hrg${g}_vector_bp`)?.value) || 1;
            let parts = [`载体 ${(vecMass / vecConc).toFixed(2)} μL`];
            for (let i = 1; i <= currentFragmentCount; i++) {
                let bp = parseFloat(document.getElementById(`hrg${g}_frag${i}_bp`)?.value) || 1;
                let conc = parseFloat(document.getElementById(`hrg${g}_frag${i}_conc`)?.value) || 1;
                let ratio = parseFloat(document.getElementById(`hrg${g}_frag${i}_ratio`)?.value) || 1;
                let reqMass = (vecMass / vecBp) * ratio * bp;
                parts.push(`片段${i} ${(reqMass / conc).toFixed(2)} μL`);
            }
            outputs[`组${g}`] = parts.join(' | ');
        }
        return outputs;
    }
```

- [ ] **Step 2: 浏览器验证表格数值**

默认(1 组 3 片段,载体 12000/1000/100)预期表格:

| 组分 | 组 1 |
|---|---|
| 载体体积 | 0.10 μL |
| 片段 1 | 0.07 μL (33.3 ng) |
| 片段 2 | 0.06 μL (25.0 ng) |
| 片段 3 | 0.04 μL (13.3 ng) |

手算核对:片段1 = (100/12000)×2×2000 = 33.33 ng,÷500 = 0.067 μL → 0.07;片段2 = (100/12000)×2×1500 = 25.0 ng,÷400 = 0.0625 → 0.06;片段3 = (100/12000)×2×800 = 13.33 ng,÷300 = 0.0444 → 0.04。

- [ ] **Step 3: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(recomb): 逐组计算并输出转置结果表格"
```

---

### Task 4: 清理 init 绑定 + 组数选择器监听

**Files:**
- Modify: `F:\Projects\calculator\index.html:1462-1463`

- [ ] **Step 1: 替换第 1462-1463 行**

原代码:

```js
        bindInputs(['hr_vector_bp','hr_vector_conc','hr_vector_mass'], updateMultiFragment);
        document.getElementById('fragment_count').addEventListener('change', renderFragments);
```

替换为:

```js
        document.getElementById('fragment_count').addEventListener('change', renderFragments);
        document.getElementById('hr_group_count').addEventListener('change', renderFragments);
```

- [ ] **Step 2: 浏览器验证切换**

组数 1→3:出现 3 张卡片、表格变 4 列(组分 + 组1-3);组 2 载体质量改为 200 后,组 2 列载体体积变 0.20 μL、片段1 变 0.13 μL (66.7 ng)。切回 1 组再切回 3 组:组 2 的 200 仍保留。片段数 3→5:每组新增片段 4、5(600/250/2、400/200/2);切回 3 后片段 4 的值仍保留。

- [ ] **Step 3: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "fix(recomb): 更新 init 绑定,移除已删除的 hr_vector 输入监听"
```

---

### Task 5: 悬液总细胞量自适应科学计数法

**Files:**
- Modify: `F:\Projects\calculator\index.html:1387-1392`(toSciSup 后新增 toSciText)
- Modify: `F:\Projects\calculator\index.html:1052`(显示)
- Modify: `F:\Projects\calculator\index.html:1058`(历史文本)

- [ ] **Step 1: 新增 toSciText 函数**

在 `toSciSup` 函数(第 1387-1392 行)之后新增:

```js
    function toSciText(num) {
        if (num === 0) return '0';
        const exp = Math.floor(Math.log10(Math.abs(num)));
        const mant = num / Math.pow(10, exp);
        return `${mant.toFixed(2)}x10^${exp}`;
    }
```

- [ ] **Step 2: 修改悬液总细胞量显示**

第 1052 行原文:

```js
            <div class="result-badge"><label>悬液总细胞量</label><div class="val">${totalCells.toFixed(2)} <span style="font-size:0.8rem">×10⁶ 个</span></div></div>
```

替换为:

```js
            <div class="result-badge"><label>悬液总细胞量</label><div class="val">${toSciSup(totalCells * 1e6)} <span style="font-size:0.8rem">个</span></div></div>
```

- [ ] **Step 3: 修改历史记录文本**

第 1058 行原文:

```js
        let outputs = { "浓度": `${currentConc.toFixed(3)}x10^6 个/mL`, "总量": `${totalCells.toFixed(2)}x10^6 个` };
```

替换为:

```js
        let outputs = { "浓度": `${currentConc.toFixed(3)}x10^6 个/mL`, "总量": `${toSciText(totalCells * 1e6)} 个` };
```

- [ ] **Step 4: 浏览器验证三种场景**

细胞计数模块(浓度调整模式),核对"悬液总细胞量"徽章与 3 秒后历史记录文本:

| 4大格总数 | 稀释倍数 | 悬液体积(mL) | 预期显示 | 预期历史"总量" |
|---|---|---|---|---|
| 20 | 1 | 5 | 2.50 × 10⁵ 个 | 2.50x10^5 个 |
| 4 | 1 | 0.05 | 5.00 × 10² 个 | 5.00x10^2 个 |
| 20 | 1 | 0 | 0 个 | 0 个 |

(浓度 = (总数/4)×10⁴×稀释/10⁶;总细胞量 = 体积×浓度×10⁶。0.25×10⁶ = 250000 → 2.50×10⁵;0.05×0.01×10⁶ = 500 → 5.00×10²。)

- [ ] **Step 5: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(cell): 悬液总细胞量改为自适应科学计数法"
```

---

### Task 6: 回归检查 + 推送

**Files:**
- 无改动(仅验证)

- [ ] **Step 1: 回归检查其他模块**

浏览器逐项确认:转染模块组数切换正常;细胞计数三种模式(浓度调整/细胞获取/已知浓度)计算值与此前一致;铺板、溶液配制、PCR、滴度/MOI 模块各点一遍无控制台报错;首页历史记录能正常恢复重组模块(组数与片段数被恢复)。

- [ ] **Step 2: 可选 — Playwright 自动化核对(webapp-testing skill)**

若需脚本化验证,可用 webapp-testing skill 启动本地服务并断言 Task 3/5 的表格与徽章文本。

- [ ] **Step 3: 推送**

```bash
cd /f/Projects/calculator
git push origin main
```

- [ ] **Step 4: 确认线上生效**

等 1-2 分钟后访问 `https://duyang9159-code.github.io/calculator/`,点开多片段重组与细胞计数模块,确认线上行为与本地一致;同时确认 `https://duyang9159-code.github.io/calculator/d359b035f4dab73de0763f9a679d1960.txt` 仍正常返回(不影响微信申诉)。
