# 滴度模块按方法切换独立输入区 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 计算方法(TCID50/TU)切换时,输入区、表格与结果整体切换,两方法各自独立输入、互不干扰、互不丢值。

**Architecture:** 单文件 HTML 应用。保留现有 `titer_method` 下拉;顶部输入字段按方法用 `hidden-input` 显隐;两张 12 排表格常驻渲染(`titer_table_tcid50` / `titer_table_tu`)仅显隐,行输入 ID 沿用现有(`titer_row_N_pos` / `titer_row_N_count`),因此切换与刷新都不丢值;`updateTiter` 按方法分支计算,TCID50 沿用现有 Reed-Muench 逻辑并新增累计病变率列。

**Tech Stack:** 原生 HTML/CSS/JS,无依赖。

**Spec:** `docs/superpowers/specs/2026-11-16-titer-modes-design.md`

---

### Task 1: HTML — 输入字段包装 + 双表格容器

**Files:** Modify `F:\Projects\calculator\index.html`(titer-sub-titer 块)

- [ ] **Step 1: 替换输入与表格容器区**

旧内容(第 622-631 行,input-grid 与提示/表格区)替换为:

```html
            <div class="input-grid">
                <div class="input-field"><label>计算方法</label><select id="titer_method"><option value="tcid50" selected>TCID50（Reed–Muench）</option><option value="tu">TU（病变细胞计数）</option></select></div>
                <div class="input-field" id="titer_field_firstvol"><label>首孔病毒体积（μL）</label><input type="number" id="titer_first_vol" value="10"></div>
                <div class="input-field"><label>每排复孔数</label><input type="number" id="titer_repeats" value="3" min="1" max="8"></div>
                <div class="input-field" id="titer_field_inocvol"><label>每孔接种体积（μL）</label><input type="number" id="titer_inoc_vol" value="100" min="0.1" step="0.1"></div>
            </div>
            <div style="margin: 16px 0;">
                <p id="titer_table_hint" style="font-weight:700; font-size:0.9rem; margin-bottom:8px;">12 排梯度稀释。TCID50 填阳性孔数，TU 填每排病变细胞总数</p>
                <div class="table-wrapper" id="titer_table_tcid50_wrap"><div id="titer_table_tcid50"></div></div>
                <div class="table-wrapper hidden-input" id="titer_table_tu_wrap"><div id="titer_table_tu"></div></div>
            </div>
```

- [ ] **Step 2: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(titer): 输入字段按方法包装,双表格容器"
```

---

### Task 2: JS — renderTiterTables 双表渲染

**Files:** Modify `F:\Projects\calculator\index.html`(函数 `renderTiterTable`)

- [ ] **Step 1: 用 renderTiterTables 替换 renderTiterTable**

将整个 `renderTiterTable` 函数(当前第 1451-1470 行)替换为:

```js
    function renderTiterTables() {
        // TCID50 表:排 | 稀释倍数 | 阳性孔数(必填) | 病变率 | 累计病变率
        let html = `<table class="lab-table"><thead><tr><th>排</th><th>稀释倍数</th><th>阳性孔数（必填）</th><th>病变率</th><th>累计病变率</th></tr></thead><tbody>`;
        for (let r = 1; r <= TITER_ROWS; r++) {
            html += `<tr>
                <td>${r}</td>
                <td>10<sup>${1-r}</sup></td>
                <td><input type="number" id="titer_row_${r}_pos" min="0" max="8" style="width:70px;"></td>
                <td id="titer_row_${r}_rate">—</td>
                <td id="titer_row_${r}_cum">—</td>
            </tr>`;
        }
        html += `</tbody></table>`;
        document.getElementById('titer_table_tcid50').innerHTML = html;

        // TU 表:排 | 稀释倍数 | 病变细胞总数 | 平均每孔病变细胞数 | 该排 TU/mL
        html = `<table class="lab-table"><thead><tr><th>排</th><th>稀释倍数</th><th>病变细胞总数</th><th>平均每孔病变细胞数</th><th>该排 TU/mL</th></tr></thead><tbody>`;
        for (let r = 1; r <= TITER_ROWS; r++) {
            html += `<tr>
                <td>${r}</td>
                <td>10<sup>${1-r}</sup></td>
                <td><input type="number" id="titer_row_${r}_count" min="0" style="width:70px;" placeholder="0"></td>
                <td id="titer_row_${r}_avg">—</td>
                <td id="titer_row_${r}_titer">—</td>
            </tr>`;
        }
        html += `</tbody></table>`;
        document.getElementById('titer_table_tu').innerHTML = html;

        for (let r = 1; r <= TITER_ROWS; r++) {
            document.getElementById(`titer_row_${r}_pos`).addEventListener('input', updateTiter);
            document.getElementById(`titer_row_${r}_count`).addEventListener('input', updateTiter);
        }
        updateTiter();
    }
```

- [ ] **Step 2: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(titer): 双表格渲染,TCID50 表加病变率与累计病变率列"
```

---

### Task 3: JS — updateTiter 按方法显隐与分支计算

**Files:** Modify `F:\Projects\calculator\index.html`(函数 `updateTiter`)

- [ ] **Step 1: 整体替换 updateTiter**

将整个 `updateTiter` 函数(当前第 1472-1570 行)替换为:

```js
    function updateTiter() {
        const method = document.getElementById('titer_method')?.value || 'tcid50';
        const repeats = Math.min(8, Math.max(1, parseInt(document.getElementById('titer_repeats').value) || 1));

        // 方法切换:显隐输入字段与表格
        document.getElementById('titer_field_firstvol').classList.toggle('hidden-input', method !== 'tu');
        document.getElementById('titer_field_inocvol').classList.toggle('hidden-input', method !== 'tcid50');
        document.getElementById('titer_table_tcid50_wrap').classList.toggle('hidden-input', method !== 'tcid50');
        document.getElementById('titer_table_tu_wrap').classList.toggle('hidden-input', method !== 'tu');
        document.getElementById('titer_table_hint').textContent = method === 'tu'
            ? '12 排梯度稀释。每排填写该排所有复孔的病变细胞总数,按 TU 公式换算每排 TU/mL'
            : '12 排梯度稀释。每排填写阳性孔数(必填),按 Reed–Muench 法计算 TCID50';

        let html = '';
        const outputs = {};
        if (method === 'tu') {
            const firstVol = parseFloat(document.getElementById('titer_first_vol').value) || 0;
            let anyCount = false;
            const counts = [];
            for (let r = 1; r <= TITER_ROWS; r++) {
                const el = document.getElementById(`titer_row_${r}_count`);
                let count = Math.max(0, parseInt(el?.value) || 0);
                if (el && el.value !== '' && parseInt(el.value) !== count) el.value = count;
                counts.push(count);
                if (count > 0) anyCount = true;
                const avg = repeats > 0 ? count / repeats : 0;
                const volUl = firstVol / Math.pow(10, r - 1);
                const tu = volUl > 0 ? avg / volUl * 1000 : 0;
                document.getElementById(`titer_row_${r}_avg`).textContent = repeats > 0 ? avg.toFixed(1) : '—';
                document.getElementById(`titer_row_${r}_titer`).innerHTML = volUl > 0 && count > 0 ? toSciSup(tu) : '—';
            }
            html = anyCount
                ? `<div class="result-badge"><label>TU 计算结果</label><div class="val" style="color:var(--primary-dark);">各排 TU/mL 已列于上表</div></div><div class="formula-line">TU/mL =（该排病变细胞总数 ÷ 复孔数）÷ 该排病毒等效体积（μL）× 1000。每一排代表一个稀释度,请结合实验室规定的可计数范围选用对应排。</div>`
                : `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TU</label><div class="val" style="color:#d97706; font-size:1rem;">请填写至少一排病变细胞数</div></div>`;
            outputs['TU'] = anyCount ? '各排结果见表格' : '暂无有效数据';
        } else {
            const inocVol = parseFloat(document.getElementById('titer_inoc_vol').value) || 0;
            const posCounts = [];
            let missingPos = false, anyPos = false;
            for (let r = 1; r <= TITER_ROWS; r++) {
                const el = document.getElementById(`titer_row_${r}_pos`);
                let pos;
                if (el && el.value !== '') {
                    pos = Math.min(repeats, Math.max(0, parseInt(el.value) || 0));
                    if (parseInt(el.value) !== pos) el.value = pos;
                } else { pos = null; missingPos = true; }
                posCounts.push(pos);
                if (pos > 0) anyPos = true;
                document.getElementById(`titer_row_${r}_rate`).textContent = repeats > 0 && pos !== null ? Math.round(pos / repeats * 100) + '%' : '—';
            }
            // Reed-Muench 累计(含本排),供累计病变率列与端点计算
            const effective = posCounts.map(p => p === null ? 0 : p);
            const cumPos = new Array(TITER_ROWS + 1).fill(0);
            let acc = 0;
            for (let r = TITER_ROWS; r >= 1; r--) { acc += effective[r - 1]; cumPos[r] = acc; }
            const cumNeg = new Array(TITER_ROWS + 1).fill(0);
            acc = 0;
            for (let r = 1; r <= TITER_ROWS; r++) { acc += repeats - effective[r - 1]; cumNeg[r] = acc; }
            for (let r = 1; r <= TITER_ROWS; r++) {
                const total = cumPos[r] + cumNeg[r];
                document.getElementById(`titer_row_${r}_cum`).textContent = total > 0 ? Math.round(cumPos[r] / total * 100) + '%' : '—';
            }
            if (missingPos) {
                html = `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TCID50</label><div class="val" style="color:#d97706; font-size:1rem;">请填写每排阳性孔数后再计算 TCID50</div></div>`;
                outputs['TCID50'] = '缺少阳性孔数';
            } else if (!anyPos) {
                html = `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TCID50</label><div class="val" style="color:#d97706; font-size:1rem;">未检出病变,建议提高病毒浓度或减少稀释</div></div>`;
                outputs['TCID50'] = '未检出病变';
            } else if (inocVol <= 0) {
                html = `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TCID50</label><div class="val" style="color:#d97706; font-size:1rem;">每孔接种体积需大于 0</div></div>`;
                outputs['TCID50'] = '接种体积无效';
            } else {
                let above = 0, below = 0;
                for (let r = TITER_ROWS; r >= 1; r--) {
                    const total = cumPos[r] + cumNeg[r];
                    if (total > 0 && cumPos[r] / total * 100 >= 50) { above = r; below = r + 1; break; }
                }
                if (above === TITER_ROWS && below > TITER_ROWS) {
                    html = `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TCID50</label><div class="val" style="color:#d97706; font-size:1rem;">终点超出最稀排,建议增加稀释梯度</div></div>`;
                    outputs['TCID50'] = '超出量程';
                } else if (above === 0 || below > TITER_ROWS) {
                    html = `<div class="result-badge" style="background:#fef3c7; border-color:#fcd34d;"><label>TCID50</label><div class="val" style="color:#d97706; font-size:1rem;">当前数据无法找到 50% 终点</div></div>`;
                    outputs['TCID50'] = '无法确定终点';
                } else {
                    const pAboveDen = cumPos[above] + cumNeg[above];
                    const pBelowDen = cumPos[below] + cumNeg[below];
                    const pAbove = pAboveDen > 0 ? cumPos[above] / pAboveDen * 100 : 0;
                    const pBelow = pBelowDen > 0 ? cumPos[below] / pBelowDen * 100 : 0;
                    const denominator = pAbove - pBelow;
                    if (denominator === 0 || !Number.isFinite(denominator)) {
                        html = `<div class="result-badge"><label>TCID50</label><div class="val">当前数据无法插值</div></div>`;
                        outputs['TCID50'] = '无法插值';
                    } else {
                        const pd = (pAbove - 50) / denominator;
                        const endpoint = (1 - above) - pd;
                        const log10PerMl = -endpoint + Math.log10(1000 / inocVol);
                        const linear = Math.pow(10, log10PerMl);
                        html = `<div class="result-badge"><label>log₁₀(TCID50/mL)</label><div class="val" style="color:var(--primary-dark);">${log10PerMl.toFixed(2)}</div></div><div class="result-badge"><label>TCID50 (个/mL)</label><div class="val" style="color:var(--primary-dark);">${toSciSup(linear)}</div></div><div class="result-badge"><label>终点稀释度</label><div class="val">10<sup>${endpoint.toFixed(2)}</sup> 稀释</div></div><div class="formula-line">Reed–Muench 法。使用每排阳性孔数按 Reed–Muench 法计算。</div>`;
                        outputs['TCID50'] = `log10=${log10PerMl.toFixed(2)} (${toSciText(linear)}/mL)`;
                    }
                }
            }
        }
        document.getElementById('titerResult').innerHTML = html;
        debounceRecord('titer', () => {
            const fields = { titer_method: method, titer_first_vol: document.getElementById('titer_first_vol').value, titer_repeats: document.getElementById('titer_repeats').value, titer_inoc_vol: document.getElementById('titer_inoc_vol').value };
            for (let r = 1; r <= TITER_ROWS; r++) { fields[`titer_row_${r}_count`] = document.getElementById(`titer_row_${r}_count`).value; fields[`titer_row_${r}_pos`] = document.getElementById(`titer_row_${r}_pos`).value; }
            recordHistory('titer', { '计算方法': method === 'tu' ? 'TU' : 'TCID50', '首孔体积': `${document.getElementById('titer_first_vol').value} μL`, '复孔数': repeats, '接种体积': `${document.getElementById('titer_inoc_vol').value} μL` }, outputs, fields);
        });
    }
```

- [ ] **Step 2: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "feat(titer): 方法切换显隐输入与表格,TU/TCID50 分支计算"
```

---

### Task 4: 调用点改名 + 使用说明

**Files:** Modify `F:\Projects\calculator\index.html`(5 处 `renderTiterTable` 调用 + guide 第 7 条)

- [ ] **Step 1: 改名 5 处调用**

将以下 5 处 `renderTiterTable()` 全部替换为 `renderTiterTables()`:
- `if (realKey === 'titer') renderTiterTable();`(restoreHistory 内)
- `if (sub === 'titer') renderTiterTable();`(子页签切换)
- `if (mod === 'titer') { renderTiterTable(); updateMOI(); }`(navigateTo)
- `renderTiterTable();`(init 首次渲染)

- [ ] **Step 2: 更新使用说明第 7 条**

第 707-710 行替换为:

```html
                <li><strong>输入：</strong>选择计算方法：TCID50（Reed–Muench）填每排阳性孔数、每排复孔数、每孔接种体积；TU（病变细胞计数）填首孔病毒体积、每排复孔数、每排病变细胞总数。均为 12 排十倍梯度（10⁰–10⁻¹¹）。MOI 部分输入细胞数、目标 MOI、病毒滴度。</li>
                <li><strong>计算：</strong>TU 模式：平均每孔病变细胞数 = 病变细胞总数 ÷ 复孔数；该排 TU/mL = 平均每孔 ÷ 该排病毒等效体积（μL）× 1000。<br>
                TCID50（Reed-Muench）：累计阳性自最稀排向浓端累计、累计阴性自最浓排向稀端累计（均含本排），取累计病变率 ≥50% 的最稀排为上排；PD = (上排% − 50) / (上排% − 下排%)；终点指数 = 上排指数 − PD；TCID50/mL = 10^(−终点指数) × 1000 / 接种体积。<br>
                所需病毒体积 (μL) = (细胞数 × MOI / 滴度) × 1000。</li>
```

- [ ] **Step 3: Commit**

```bash
cd /f/Projects/calculator
git add index.html
git commit -m "fix(titer): 调用点改名,使用说明同步两种方法"
```

---

### Task 5: Playwright 验证 + 文档更新 + 推送

- [ ] **Step 1: 编写并运行验证脚本** `C:\Users\19216\AppData\Local\Temp\verify_modes.py`

```python
# -*- coding: utf-8 -*-
import pathlib, sys
sys.stdout.reconfigure(encoding="utf-8")
from playwright.sync_api import sync_playwright

URL = pathlib.Path(r"F:\Projects\calculator\index.html").as_uri()
errors = []
def check(name, cond, detail=""):
    print(("PASS" if cond else "FAIL"), "|", name, ("| " + str(detail) if detail else ""))
    if not cond: errors.append(name)

with sync_playwright() as p:
    b = p.chromium.launch(headless=True)
    pg = b.new_page()
    cerr = []
    pg.on("console", lambda m: cerr.append(m.text) if m.type == "error" else None)
    pg.goto(URL); pg.wait_for_load_state("load")
    pg.click('.home-card[onclick="navigateTo(\'titer\')"]')
    pg.wait_for_selector("#titer_table_tcid50 table")

    # 1. 默认 TCID50:首孔隐藏、接种显示、TCID50 表显示、TU 表隐藏
    check("默认 TCID50:首孔字段隐藏", pg.locator("#titer_field_firstvol").evaluate("el => el.classList.contains('hidden-input')"))
    check("默认 TCID50:接种字段显示", not pg.locator("#titer_field_inocvol").evaluate("el => el.classList.contains('hidden-input')"))
    check("TCID50 表显示", pg.locator("#titer_table_tcid50_wrap").is_visible())
    check("TU 表隐藏", not pg.locator("#titer_table_tu_wrap").is_visible())

    # 2. TCID50 手算例
    pg.fill("#titer_repeats", "8"); pg.fill("#titer_inoc_vol", "100")
    pos = [8,8,8,8,8,8,6,2,0,0,0,0]
    for r, v in enumerate(pos, 1): pg.fill(f"#titer_row_{r}_pos", str(v))
    pg.wait_for_timeout(100)
    badges = pg.eval_on_selector_all("#titerResult .result-badge", "els => els.map(e => e.innerText)")
    check("log10 = 7.50", any("7.50" in x for x in badges), str(badges))
    check("TCID50 3.16e7", any("3.16e7" in x.replace(" \u00d7 10", "e") for x in badges))
    r7rate = pg.locator("#titer_row_7_rate").inner_text(); r7cum = pg.locator("#titer_row_7_cum").inner_text()
    check("第7排病变率 75%", r7rate == "75%", r7rate)
    check("第7排累计病变率 80%", r7cum == "80%", r7cum)

    # 3. 边界:缺阳性孔数
    pg.fill("#titer_row_3_pos", "")
    pg.wait_for_timeout(100)
    body = pg.locator("#titerResult").inner_text()
    check("缺阳性孔数提示", "\u9633\u6027\u5b54\u6570" in body, body[:50])
    pg.fill("#titer_row_3_pos", "8")

    # 4. 切 TU:字段/表格对调,值互不干扰
    pg.select_option("#titer_method", "tu")
    pg.wait_for_timeout(100)
    check("TU:首孔字段显示", not pg.locator("#titer_field_firstvol").evaluate("el => el.classList.contains('hidden-input')"))
    check("TU:接种字段隐藏", pg.locator("#titer_field_inocvol").evaluate("el => el.classList.contains('hidden-input')"))
    check("TU 表显示", pg.locator("#titer_table_tu_wrap").is_visible())
    check("TCID50 表隐藏", not pg.locator("#titer_table_tcid50_wrap").is_visible())
    pg.fill("#titer_repeats", "3"); pg.fill("#titer_first_vol", "10")
    pg.fill("#titer_row_5_count", "21")
    pg.wait_for_timeout(100)
    avg5 = pg.locator("#titer_row_5_avg").inner_text(); t5 = pg.locator("#titer_row_5_titer").inner_text().replace(" \u00d7 10", "e")
    check("TU 第5排平均 7.0", avg5 == "7.0", avg5)
    check("TU 第5排 7.00e6", t5 == "7.00e6", t5)

    # 5. 切回 TCID50:阳性孔数保留、结果仍 7.50
    pg.select_option("#titer_method", "tcid50")
    pg.wait_for_timeout(100)
    badges2 = pg.eval_on_selector_all("#titerResult .result-badge", "els => els.map(e => e.innerText)")
    check("切回后 log10 仍 7.50", any("7.50" in x for x in badges2), str(badges2))
    check("切回后 pos 保留", pg.input_value("#titer_row_7_pos") == "6", pg.input_value("#titer_row_7_pos"))

    # 6. 历史:TU 模式记录并恢复
    pg.select_option("#titer_method", "tu")
    pg.wait_for_timeout(3800)
    hist = pg.evaluate("JSON.parse(localStorage.getItem('labsuite_history') || '[]')")
    newest = hist[0] if hist else {}
    check("历史含计算方法 TU", str(newest.get("inputs", {}).get("\u8ba1\u7b97\u65b9\u6cd5", "")) == "TU", str(newest.get("inputs")))
    # 恢复到默认态再点击历史第一条(恢复弹窗用 onclick,直接调用 restoreHistory)
    pg.evaluate("restoreHistory('titer', JSON.parse(localStorage.getItem('labsuite_history'))[0].fields)")
    pg.wait_for_timeout(400)
    check("恢复后 TU 模式", pg.input_value("#titer_method") == "tu", pg.input_value("#titer_method"))
    check("恢复后第5排 21", pg.input_value("#titer_row_5_count") == "21", pg.input_value("#titer_row_5_count"))

    # 7. 刷新后保留
    pg.reload(); pg.wait_for_load_state("load")
    pg.click('.home-card[onclick="navigateTo(\'titer\')"]')
    pg.wait_for_selector("#titer_table_tcid50 table")
    check("刷新后方法仍 TU", pg.input_value("#titer_method") == "tu", pg.input_value("#titer_method"))
    check("刷新后 TU 值保留", pg.input_value("#titer_row_5_count") == "21", pg.input_value("#titer_row_5_count"))
    check("刷新后 pos 值保留", pg.input_value("#titer_row_7_pos") == "6", pg.input_value("#titer_row_7_pos"))

    # 8. 术语 + MOI 回归 + 控制台
    body_all = pg.locator("body").inner_text()
    check("无 空斑/荧光 字样", ("\u7a7a\u6591" not in body_all) and ("\u8367\u5149" not in body_all))
    pg.click('button[data-sub="moi"]')
    pg.wait_for_selector("#moiResult .result-badge")
    check("MOI 子页正常", pg.locator("#moiResult .result-badge").count() >= 1)
    check("无控制台报错", len(cerr) == 0, str(cerr[:5]))
    b.close()

print("RESULT:", "ALL PASS" if not errors else f"{len(errors)} FAILURES: {errors}")
```

Run: `cd /c/Users/19216/AppData/Local/Temp && python verify_modes.py`
Expected: `RESULT: ALL PASS`(若失败,修复后重跑,不得跳过)。

- [ ] **Step 2: 更新项目文档(工作规则)**

- `PROGRESS.md`:"已完成"增加"滴度模块按方法切换独立输入区"条目;最近提交更新;当前状态改为"P0 已完成,待回归"。
- `PROJECT.md`:"当前关键约定"更新滴度条目为"TCID50 与 TU 为两个独立输入区,方法下拉切换";"当前主要限制"更新为已完成说明或移除。
- `TODO.md`:P0 全部勾销(改为 `- [x]` 或删除),P1 保留待回归。
- 三个文件一并 `git add` 入库。

- [ ] **Step 3: 推送与线上确认**

```bash
cd /f/Projects/calculator
git add -A
git commit -m "docs: 更新 PROGRESS/PROJECT/TODO,滴度独立输入区完成"
git push origin main
```

等 1-2 分钟后:
- `curl -sS -o /dev/null -w "%{http_code}" https://duyang9159-code.github.io/calculator/d359b035f4dab73de0763f9a679d1960.txt` → 200
- 访问线上页面确认滴度模块两种方法切换正常
