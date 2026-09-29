// ============================================================
// AI 消息 → 世界书 整合脚本（v20 - 增加导入预设）
// ============================================================

const CONFIG = {
    entry_name_prefix: 'AI消息-',
    substitute_macros: true,
    min_message_length: 30,
    entry_enabled: true,
    entry_strategy_type: 'selective',
};

let targetWorldbook = '';
let excludeYamlBlock = false;
let writeCardMode = false;
let recognizeTriggerKeys = false;
let removeThinking = false;

const DEFAULT_REPLACE_RULES = [
    { from: '萝莉', to: '萝.莉', smart: false },
    { from: '岁', to: '.岁', smart: true },
];
const STORAGE_KEY = 'wb_replace_rules_v1';

function getReplaceRules() {
    try {
        const raw = localStorage.getItem(STORAGE_KEY);
        if (raw) { const p = JSON.parse(raw); if (Array.isArray(p)) return p; }
    } catch (e) {}
    return JSON.parse(JSON.stringify(DEFAULT_REPLACE_RULES));
}
function setReplaceRules(rules) {
    try { localStorage.setItem(STORAGE_KEY, JSON.stringify(rules)); } catch (e) {}
}
function sanitizeText(text, rules) {
    if (!rules) rules = getReplaceRules();
    let result = text;
    for (const rule of rules) {
        if (!rule.from) continue;
        if (rule.smart) {
            let output = ''; let i = 0;
            const from = rule.from, to = rule.to;
            while (i < result.length) {
                if (result.substring(i, i + from.length) === from) {
                    if (output.length > 0 && output[output.length - 1] === '.') output += from;
                    else output += to;
                    i += from.length;
                } else { output += result[i]; i++; }
            }
            result = output;
        } else {
            result = result.split(rule.from).join(rule.to);
        }
    }
    return result;
}

// ============================================================
// 🆕 获取 SillyTavern context
// ============================================================
function getSTContext() {
    try {
        if (typeof SillyTavern !== 'undefined' && SillyTavern.getContext) return SillyTavern.getContext();
        if (window.parent && window.parent.SillyTavern && window.parent.SillyTavern.getContext) return window.parent.SillyTavern.getContext();
    } catch (e) {}
    return null;
}

async function getCurrentWorldbookName() {
    if (targetWorldbook && targetWorldbook.trim() !== '') return targetWorldbook.trim();
    try {
        const name = await TavernHelper.getCurrentCharPrimaryLorebook();
        if (name) return name;
    } catch (e) {}
    return null;
}

function stripThinkingChain(text) {
    if (!removeThinking || !text) return text;
    let r = text;
    r = r.replace(/\[metacognition\][\s\S]*?<\/(?:thinking|metacognition)>/gi, '');
    r = r.replace(/<thinking>[\s\S]*?<\/thinking>/gi, '');
    r = r.replace(/<metacognition>[\s\S]*?<\/metacognition>/gi, '');
    r = r.replace(/<\/?content>/gi, '');
    r = r.replace(/\n{3,}/g, '\n\n').trim();
    return r;
}
function filterContent(content) {
    if (!excludeYamlBlock) return content;
    return content.replace(/```yaml\s*[\s\S]*?```/gi, '').trim();
}
function applyMacros(content) {
    if (CONFIG.substitute_macros && typeof substitudeMacros === 'function') {
        try { return substitudeMacros(content); } catch (e) {}
    }
    return content;
}

function extractMarkdownEntries(text, baseId) {
    const regex = /```markdown\s*([\s\S]*?)```/gi;
    let match, lastIndex = 0, count = 0;
    const entries = [];
    while ((match = regex.exec(text)) !== null) {
        let markdownContent = match[1].trim();
        if (!markdownContent) { lastIndex = regex.lastIndex; continue; }
        markdownContent = filterContent(markdownContent);
        if (markdownContent.length < CONFIG.min_message_length) { lastIndex = regex.lastIndex; continue; }
        const metaText = text.slice(lastIndex, match.index);
        let keys = [];
        if (recognizeTriggerKeys) {
            const keyMatch = metaText.match(/[-*]\s*触发关键词[:：]\s*([^\n\r]+)/);
            if (keyMatch) {
                const ks = keyMatch[1].split(/[,，、|]/).map(k => k.trim()).filter(k => k);
                keys = keys.concat(ks);
            }
            const nickMatch = markdownContent.match(/nicknames\s*[:：]\s*\[([^\]]+)\]/i);
            if (nickMatch) {
                const nicks = nickMatch[1].split(/[,，]/).map(k => k.trim()).filter(k => k);
                keys = keys.concat(nicks);
            }
            keys = [...new Set(keys)];
        }
        let entryName = `${CONFIG.entry_name_prefix}${baseId}_${count + 1}`;
        if (recognizeTriggerKeys) {
            const nameMatch = metaText.match(/\*\*世界书条目[:：]\s*([^\*]+)\*\*/) || metaText.match(/世界书条目[:：]\s*([^\n\r]+)/);
            if (nameMatch) entryName = nameMatch[1].trim();
        }
        let finalContent = applyMacros(markdownContent);
        entries.push({
            name: entryName,
            content: finalContent,
            enabled: CONFIG.entry_enabled,
            strategy: {
                type: (recognizeTriggerKeys && keys.length > 0) ? 'selective' : 'constant',
                keys: keys,
                logic: 'or'
            }
        });
        lastIndex = regex.lastIndex;
        count++;
    }
    return entries;
}

function extractMarkdownEntriesForce(text, baseId) {
    const regex = /```markdown\s*([\s\S]*?)```/gi;
    let match, lastIndex = 0, count = 0;
    const entries = [];
    while ((match = regex.exec(text)) !== null) {
        let markdownContent = match[1].trim();
        if (!markdownContent) { lastIndex = regex.lastIndex; continue; }
        markdownContent = filterContent(markdownContent);
        if (markdownContent.length < CONFIG.min_message_length) { lastIndex = regex.lastIndex; continue; }
        const metaText = text.slice(lastIndex, match.index);

        let keys = [];
        const keyMatch = metaText.match(/[-*]\s*触发关键词[:：]\s*([^\n\r]+)/);
        if (keyMatch) {
            const ks = keyMatch[1].split(/[,，、|]/).map(k => k.trim()).filter(k => k);
            keys = keys.concat(ks);
        }
        const nickMatch = markdownContent.match(/nicknames\s*[:：]\s*\[([^\]]+)\]/i);
        if (nickMatch) {
            const nicks = nickMatch[1].split(/[,，]/).map(k => k.trim()).filter(k => k);
            keys = keys.concat(nicks);
        }
        keys = [...new Set(keys)];

        let entryName = `${CONFIG.entry_name_prefix}${baseId}_${count + 1}`;
        const nameMatch = metaText.match(/\*\*世界书条目[:：]\s*([^\*]+)\*\*/) || metaText.match(/世界书条目[:：]\s*([^\n\r]+)/);
        if (nameMatch) entryName = nameMatch[1].trim();

        let finalContent = applyMacros(markdownContent);
        entries.push({
            name: entryName,
            content: finalContent,
            enabled: CONFIG.entry_enabled,
            strategy: {
                type: keys.length > 0 ? 'selective' : 'constant',
                keys: keys,
                logic: 'or'
            }
        });
        lastIndex = regex.lastIndex;
        count++;
    }
    return entries;
}

function buildEntriesFromMessage(msg, msgId) {
    let raw = stripThinkingChain(msg.message);
    if (!writeCardMode) {
        let content = filterContent(raw);
        if (content.length < CONFIG.min_message_length) return [];
        content = applyMacros(content);
        return [{
            name: `${CONFIG.entry_name_prefix}${msgId}`,
            content: content,
            enabled: CONFIG.entry_enabled,
            strategy: { type: CONFIG.entry_strategy_type, keys: [] }
        }];
    }
    const subEntries = extractMarkdownEntries(raw, msgId);
    if (subEntries.length > 0) return subEntries;
    let content = filterContent(raw);
    if (content.length < CONFIG.min_message_length) return [];
    content = applyMacros(content);
    return [{
        name: `${CONFIG.entry_name_prefix}${msgId}`,
        content: content,
        enabled: CONFIG.entry_enabled,
        strategy: { type: CONFIG.entry_strategy_type, keys: [] }
    }];
}

function buildEntriesForExport(msg, msgId) {
    const raw = stripThinkingChain(msg.message);
    return extractMarkdownEntriesForce(raw, msgId);
}

// ============================================================
// 🆕 导入预设
// ============================================================
async function importPreset() {
    const doc = window.parent ? window.parent.document : document;

    const fileInput = doc.createElement('input');
    fileInput.type = 'file';
    fileInput.accept = '.json,application/json';
    fileInput.style.cssText = 'position:fixed;left:-9999px;top:0;';
    doc.body.appendChild(fileInput);

    fileInput.onchange = async () => {
        const file = fileInput.files && fileInput.files[0];
        if (fileInput.parentNode) fileInput.parentNode.removeChild(fileInput);
        if (!file) return;

        try {
            const text = await file.text();
            let presetData;
            try {
                presetData = JSON.parse(text);
            } catch (err) {
                toastr.error('JSON 解析失败：' + err.message);
                return;
            }

            const presetName = (presetData.name && String(presetData.name).trim())
                || file.name.replace(/\.json$/i, '');

            await applyPresetToSillyTavern(presetData, presetName);
        } catch (err) {
            console.error('导入预设失败:', err);
            toastr.error('导入失败：' + err.message);
        }
    };

    fileInput.click();
}

async function applyPresetToSillyTavern(presetData, presetName) {
    const ctx = getSTContext();
    if (!ctx) {
        toastr.error('无法获取 SillyTavern 环境');
        return;
    }

    // 方式1：优先尝试原生 importPreset API
    try {
        if (typeof ctx.importPreset === 'function') {
            await ctx.importPreset(presetData, presetName);
            toastr.success(`已导入预设「${presetName}」`);
            setTimeout(() => {
                if (confirm('预设已导入。是否刷新页面以生效？')) location.reload();
            }, 800);
            return;
        }
    } catch (e) {
        console.warn('ctx.importPreset 调用失败，尝试直接写入 extensionSettings:', e);
    }

    // 方式2：直接写入 extensionSettings
    try {
        const settings = ctx.extensionSettings;
        if (!settings) {
            toastr.error('无法访问 SillyTavern 设置');
            return;
        }

        if (!settings.presets || typeof settings.presets !== 'object') {
            settings.presets = {};
        }

        // 保证 presetData 里有必要的字段
        if (!presetData.name) presetData.name = presetName;

        settings.presets[presetName] = presetData;
        settings.preset = presetName;

        // 保存
        if (typeof ctx.saveSettingsDebounced === 'function') {
            ctx.saveSettingsDebounced();
        } else if (typeof window.saveSettingsDebounced === 'function') {
            window.saveSettingsDebounced();
        } else if (typeof window.parent.saveSettingsDebounced === 'function') {
            window.parent.saveSettingsDebounced();
        }

        // 尝试触发预设变更事件
        try {
            if (ctx.eventSource && ctx.event_types && ctx.event_types.PRESET_CHANGED) {
                ctx.eventSource.emit(ctx.event_types.PRESET_CHANGED);
            }
        } catch (e) {}

        toastr.success(`已导入预设「${presetName}」`);

        // 询问是否刷新
        setTimeout(() => {
            if (confirm(`预设「${presetName}」已导入。是否刷新页面让预设列表更新？`)) {
                location.reload();
            }
        }, 500);
    } catch (e) {
        console.error(e);
        toastr.error('导入失败：' + e.message);
    }
}

// ============================================================
// 存入最新 / 复制HTML / 选择世界书 / 新建世界书 / 开关
// ============================================================
async function saveLatestReply() {
    const lastId = getLastMessageId();
    if (lastId < 0) { toastr.warning('当前没有消息'); return; }
    const msg = getChatMessages(lastId)[0];
    if (!msg || msg.role !== 'assistant' || msg.is_hidden) { toastr.warning('最后一条不是有效的AI回复'); return; }
    if (!msg.message || msg.message.trim().length < CONFIG.min_message_length) { toastr.warning(`消息少于 ${CONFIG.min_message_length} 字，已忽略`); return; }
    const wbName = await getCurrentWorldbookName();
    if (!wbName) { toastr.warning('未选择目标世界书'); return; }
    const entries = buildEntriesFromMessage(msg, lastId);
    if (!entries.length) { toastr.warning('没有可写入的内容'); return; }
    try {
        await createWorldbookEntries(wbName, entries);
        toastr.success(`成功写入 ${entries.length} 条到「${wbName}」`);
    } catch (e) { toastr.error('写入失败：' + e.message); }
}

async function copyLatestHTML() {
    const lastId = getLastMessageId();
    if (lastId < 0) { toastr.warning('当前没有消息'); return; }
    const msg = getChatMessages(lastId)[0];
    if (!msg || msg.role !== 'assistant' || msg.is_hidden) { toastr.warning('最后一条不是有效的AI回复'); return; }
    const raw = stripThinkingChain(msg.message);
    const regex = /```html\s*([\s\S]*?)```/gi;
    const blocks = [];
    let match;
    while ((match = regex.exec(raw)) !== null) {
        const content = match[1].trim();
        if (content) blocks.push(content);
    }
    if (!blocks.length) { toastr.warning('未在当前AI回复中找到 ```html``` 内容'); return; }
    const text = blocks.join('\n\n');
    showCopyDialog(text, blocks.length);
}

async function copyToClipboard(text) {
    try { if (typeof TavernHelper !== 'undefined' && TavernHelper.copyText) { await TavernHelper.copyText(text); return true; } } catch (e) {}
    try { if (typeof copyText === 'function') { copyText(text); return true; } } catch (e) {}
    try { if (navigator.clipboard && navigator.clipboard.writeText) { await navigator.clipboard.writeText(text); return true; } } catch (e) {}
    try {
        const ta = document.createElement('textarea');
        ta.value = text;
        ta.style.position = 'fixed'; ta.style.left = '-9999px'; ta.style.top = '0';
        document.body.appendChild(ta); ta.focus(); ta.select();
        const ok = document.execCommand('copy');
        document.body.removeChild(ta);
        return ok;
    } catch (e) { console.error('复制失败:', e); return false; }
}

function showCopyDialog(originalText, blockCount) {
    const doc = window.parent ? window.parent.document : document;
    const oldDlg = doc.getElementById('wb-copy-dialog');
    if (oldDlg) oldDlg.remove();
    let sanitizeOn = true;
    let rules = getReplaceRules();
    let editRules = null;

    const overlay = doc.createElement('div');
    overlay.id = 'wb-copy-dialog';
    overlay.style.cssText = `position: fixed !important; top: 0; left: 0; width: 100vw; height: 100vh;
        background: rgba(0,0,0,0.9); z-index: 2147483647 !important;
        display: flex; align-items: center; justify-content: center; padding: 10px; box-sizing: border-box;`;

    const box = doc.createElement('div');
    box.style.cssText = `background: #1e1e1e; border-radius: 12px; padding: 15px;
        width: 100%; max-width: 620px; height: 90vh; display: flex; flex-direction: column; gap: 8px;
        color: #fff; box-shadow: 0 4px 20px rgba(0,0,0,0.9); border: 1px solid #555; box-sizing: border-box;`;

    const mainView = doc.createElement('div');
    mainView.style.cssText = 'display:flex; flex-direction:column; gap:8px; flex:1; min-height:0;';
    const mainTitle = doc.createElement('div');
    mainTitle.style.cssText = 'text-align:center; font-size:14px; font-weight:bold; color:#ffcc66;';
    mainView.appendChild(mainTitle);
    const sanitizeToggle = doc.createElement('button');
    sanitizeToggle.style.cssText = 'padding:10px; border:none; border-radius:8px; color:#fff; font-size:14px; cursor:pointer; font-weight:bold;';
    mainView.appendChild(sanitizeToggle);
    const ta = doc.createElement('textarea');
    ta.readOnly = false;
    ta.style.cssText = `flex: 1; width: 100%; padding: 10px; box-sizing: border-box;
        background: #0d0d0d; color: #d4d4d4; border: 1px solid #444; border-radius: 8px;
        font-family: Consolas, monospace; font-size: 12px; resize: none; outline: none;
        line-height: 1.5; -webkit-user-select: text; user-select: text;
        white-space: pre; overflow: auto; min-height: 0;`;
    mainView.appendChild(ta);
    const mainBtnRow = doc.createElement('div');
    mainBtnRow.style.cssText = 'display:flex; gap:8px; flex-wrap:wrap;';

    const editBtn = doc.createElement('button');
    editBtn.innerText = '⚙️ 编辑敏感词';
    editBtn.style.cssText = 'flex:1; min-width:100px; padding:12px; border:none; border-radius:8px; background:#6a4c93; color:#fff; font-size:14px; cursor:pointer; font-weight:bold;';
    editBtn.onclick = () => switchView('edit');
    mainBtnRow.appendChild(editBtn);

    const copyBtn = doc.createElement('button');
    copyBtn.innerText = '📋 复制';
    copyBtn.style.cssText = 'flex:1; min-width:80px; padding:12px; border:none; border-radius:8px; background:#2a6f97; color:#fff; font-size:15px; cursor:pointer; font-weight:bold;';
    copyBtn.onclick = async () => {
        const toCopy = sanitizeOn ? sanitizeText(originalText, rules) : originalText;
        const ok = await copyToClipboard(toCopy);
        if (ok) { toastr.success(`已复制 ${toCopy.length} 字符`); overlay.remove(); }
        else { ta.focus(); ta.setSelectionRange(0, ta.value.length); toastr.warning('自动复制失败，已全选，请长按选择"复制"'); }
    };
    mainBtnRow.appendChild(copyBtn);

    const backBtn = doc.createElement('button');
    backBtn.innerText = '↩️ 返回';
    backBtn.style.cssText = 'flex:1; min-width:80px; padding:12px; border:none; border-radius:8px; background:#8b0000; color:#fff; font-size:15px; cursor:pointer; font-weight:bold;';
    backBtn.onclick = () => { overlay.remove(); setTimeout(() => openWorldbookMenu('main'), 50); };
    mainBtnRow.appendChild(backBtn);
    mainView.appendChild(mainBtnRow);

    const editView = doc.createElement('div');
    editView.style.cssText = 'display:none; flex-direction:column; gap:8px; flex:1; min-height:0;';
    const editTitle = doc.createElement('div');
    editTitle.innerText = '⚙️ 编辑敏感词替换规则';
    editTitle.style.cssText = 'text-align:center; font-size:15px; font-weight:bold; color:#ffcc66;';
    editView.appendChild(editTitle);
    const editHint = doc.createElement('div');
    editHint.innerText = '每行一条：原文 → 替换为。🛡 智能模式（前一字为句点时跳过）';
    editHint.style.cssText = 'text-align:center; font-size:11px; color:#999;';
    editView.appendChild(editHint);
    const rulesList = doc.createElement('div');
    rulesList.style.cssText = 'flex:1; overflow-y:auto; padding:5px; background:#151515; border-radius:8px; border:1px solid #333; display:flex; flex-direction:column; gap:8px; min-height:0;';
    editView.appendChild(rulesList);

    const editBtnRow = doc.createElement('div');
    editBtnRow.style.cssText = 'display:flex; gap:6px; flex-wrap:wrap;';
    const addBtn = doc.createElement('button');
    addBtn.innerText = '➕ 添加规则';
    addBtn.style.cssText = 'flex:1; min-width:100px; padding:10px; border:none; border-radius:8px; background:#3a3a3a; color:#fff; font-size:13px; cursor:pointer; font-weight:bold;';
    addBtn.onclick = () => { editRules.push({ from: '', to: '', smart: false }); renderEditRules(); };
    editBtnRow.appendChild(addBtn);
    const resetBtn = doc.createElement('button');
    resetBtn.innerText = '🔄 恢复默认';
    resetBtn.style.cssText = 'flex:1; min-width:100px; padding:10px; border:none; border-radius:8px; background:#6a4c93; color:#fff; font-size:13px; cursor:pointer; font-weight:bold;';
    resetBtn.onclick = () => {
        if (!confirm('确定恢复默认规则？')) return;
        editRules = JSON.parse(JSON.stringify(DEFAULT_REPLACE_RULES));
        renderEditRules();
    };
    editBtnRow.appendChild(resetBtn);
    editView.appendChild(editBtnRow);

    const editBtnRow2 = doc.createElement('div');
    editBtnRow2.style.cssText = 'display:flex; gap:8px;';
    const saveBtn = doc.createElement('button');
    saveBtn.innerText = '💾 保存';
    saveBtn.style.cssText = 'flex:1; padding:12px; border:none; border-radius:8px; background:#2a6f97; color:#fff; font-size:15px; cursor:pointer; font-weight:bold;';
    saveBtn.onclick = () => {
        rules = editRules.filter(r => r.from && r.from.length > 0);
        setReplaceRules(rules);
        toastr.success(`已保存 ${rules.length} 条规则`);
        switchView('main');
    };
    editBtnRow2.appendChild(saveBtn);
    const cancelEditBtn = doc.createElement('button');
    cancelEditBtn.innerText = '↩️ 取消';
    cancelEditBtn.style.cssText = 'flex:1; padding:12px; border:none; border-radius:8px; background:#8b0000; color:#fff; font-size:15px; cursor:pointer; font-weight:bold;';
    cancelEditBtn.onclick = () => switchView('main');
    editBtnRow2.appendChild(cancelEditBtn);
    editView.appendChild(editBtnRow2);

    function refreshMainUI() {
        const currentText = sanitizeOn ? sanitizeText(originalText, rules) : originalText;
        ta.value = currentText;
        mainTitle.innerText = `📋 HTML 预览（${blockCount} 个块，共 ${currentText.length} 字符）｜ 敏感词替换：${sanitizeOn ? '开启' : '关闭'}`;
        sanitizeToggle.innerText = sanitizeOn ? '🔒 敏感词替换：开启（点击关闭）' : '🔓 敏感词替换：关闭（点击开启）';
        sanitizeToggle.style.background = sanitizeOn ? '#2a6f97' : '#444';
    }
    sanitizeToggle.onclick = () => { sanitizeOn = !sanitizeOn; refreshMainUI(); };

    function renderEditRules() {
        rulesList.innerHTML = '';
        if (editRules.length === 0) {
            const empty = doc.createElement('div');
            empty.innerText = '（无规则）';
            empty.style.cssText = 'text-align:center; color:#666; padding:15px; font-size:13px;';
            rulesList.appendChild(empty); return;
        }
        editRules.forEach((rule, idx) => {
            const row = doc.createElement('div');
            row.style.cssText = 'display:flex; gap:6px; align-items:center;';
            const fromInput = doc.createElement('input');
            fromInput.type = 'text'; fromInput.value = rule.from || ''; fromInput.placeholder = '原文';
            fromInput.style.cssText = 'flex:1; min-width:0; padding:8px; background:#0d0d0d; color:#fff; border:1px solid #444; border-radius:6px; font-size:13px;';
            fromInput.oninput = () => { editRules[idx].from = fromInput.value; };
            row.appendChild(fromInput);
            const arrow = doc.createElement('span');
            arrow.innerText = '→'; arrow.style.cssText = 'color:#888; flex-shrink:0;';
            row.appendChild(arrow);
            const toInput = doc.createElement('input');
            toInput.type = 'text'; toInput.value = rule.to || ''; toInput.placeholder = '替换为';
            toInput.style.cssText = 'flex:1; min-width:0; padding:8px; background:#0d0d0d; color:#fff; border:1px solid #444; border-radius:6px; font-size:13px;';
            toInput.oninput = () => { editRules[idx].to = toInput.value; };
            row.appendChild(toInput);
            const smartBtn = doc.createElement('button');
            smartBtn.innerText = rule.smart ? '🛡' : '·';
            smartBtn.style.cssText = `padding:6px 8px; border:none; border-radius:6px; color:#fff; font-size:13px; cursor:pointer; flex-shrink:0; background:${rule.smart ? '#2a6f97' : '#444'};`;
            smartBtn.onclick = () => {
                editRules[idx].smart = !editRules[idx].smart;
                smartBtn.innerText = editRules[idx].smart ? '🛡' : '·';
                smartBtn.style.background = editRules[idx].smart ? '#2a6f97' : '#444';
            };
            row.appendChild(smartBtn);
            const delBtn = doc.createElement('button');
            delBtn.innerText = '✕';
            delBtn.style.cssText = 'padding:6px 10px; border:none; border-radius:6px; background:#8b0000; color:#fff; font-size:13px; cursor:pointer; flex-shrink:0;';
            delBtn.onclick = () => { editRules.splice(idx, 1); renderEditRules(); };
            row.appendChild(delBtn);
            rulesList.appendChild(row);
        });
    }

    function switchView(v) {
        if (v === 'edit') {
            editRules = JSON.parse(JSON.stringify(rules));
            if (editRules.length === 0) editRules.push({ from: '', to: '', smart: false });
            renderEditRules();
            mainView.style.display = 'none';
            editView.style.display = 'flex';
        } else {
            refreshMainUI();
            editView.style.display = 'none';
            mainView.style.display = 'flex';
        }
    }

    refreshMainUI();
    box.appendChild(mainView);
    box.appendChild(editView);
    overlay.appendChild(overlay === box ? menu : box);
    doc.body.appendChild(overlay);
}

async function selectWorldbook() {
    let names = [];
    try { names = await TavernHelper.getWorldbookNames() || []; } catch (e) {}
    if (!names.length) { toastr.warning('没有找到任何世界书，请先新建。'); return; }
    let msg = "请选择要写入的世界书（输入序号）：\n";
    names.forEach((n, i) => msg += `${i + 1}. ${n}\n`);
    const idx = prompt(msg);
    if (idx && !isNaN(idx)) {
        const num = parseInt(idx);
        if (num > 0 && num <= names.length) {
            targetWorldbook = names[num - 1];
            toastr.success(`已选择：${targetWorldbook}`);
        } else { toastr.warning('无效的序号'); }
    }
}

async function createNewWorldbook() {
    const name = prompt("请输入新世界书的名称：");
    if (!name || name.trim() === '') return;
    const cleanName = name.trim();
    try {
        const names = await TavernHelper.getWorldbookNames() || [];
        if (names.includes(cleanName)) { targetWorldbook = cleanName; toastr.info(`已存在，已自动选中。`); return; }
        if (TavernHelper.createWorldbook) {
            await TavernHelper.createWorldbook(cleanName);
            targetWorldbook = cleanName;
            toastr.success(`已新建并选中：${cleanName}`);
        } else { toastr.error('当前版本不支持自动新建。'); }
    } catch (e) { toastr.error('新建失败：' + e.message); }
}

function toggleYamlFilter() {
    excludeYamlBlock = !excludeYamlBlock;
    toastr.success(`过滤 YAML 块已${excludeYamlBlock ? '开启 ✅' : '关闭 ❌'}`);
}
function toggleWriteCardMode() {
    writeCardMode = !writeCardMode;
    toastr.success(`写卡助手定制已${writeCardMode ? '开启 ✅' : '关闭 ❌'}`);
}
function toggleTriggerKeys() {
    recognizeTriggerKeys = !recognizeTriggerKeys;
    toastr.success(`触发关键词识别已${recognizeTriggerKeys ? '开启 ✅' : '关闭 ❌'}`);
}
function toggleRemoveThinking() {
    removeThinking = !removeThinking;
    toastr.success(`去思维链已${removeThinking ? '开启 ✅' : '关闭 ❌'}`);
}

function parseCharacterFields(text) {
    const result = { name: '', nicknames: [] };
    const nameMatch = text.match(/^\s*name\s*[:：]\s*(.+?)$/m);
    if (nameMatch) result.name = nameMatch[1].trim();
    if (!result.name) {
        const titleMatch = text.match(/^#\s+(.+?)$/m);
        if (titleMatch) result.name = titleMatch[1].replace(/[（(].+?[）)]/g, '').trim();
    }
    const nickMatch = text.match(/nicknames\s*[:：]\s*\[([^\]]+)\]/i);
    if (nickMatch) {
        result.nicknames = nickMatch[1].split(/[,，]/).map(k => k.trim()).filter(k => k);
    }
    return result;
}

function buildCharacterBookFromEntries(bookName, entries) {
    if (!entries || entries.length === 0) return null;
    const converted = entries.map((e, idx) => {
        const keys = Array.isArray(e.strategy?.keys) ? e.strategy.keys : (e.keys || []);
        const isSelective = e.strategy?.type === 'selective' || keys.length > 0;
        const isConstant = e.strategy?.type === 'constant';
        return {
            id: idx,
            keys: keys,
            secondary_keys: [],
            comment: e.name || `条目_${idx}`,
            name: e.name || `条目_${idx}`,
            content: e.content || '',
            constant: isConstant,
            selective: isSelective && !isConstant,
            insertion_order: 100 + idx,
            enabled: e.enabled !== false,
            position: 'before_char',
            use_regex: false,
            extensions: {}
        };
    });
    return {
        name: bookName || '对话世界书',
        description: '',
        scan_depth: 4,
        token_budget: 512,
        recursive_scanning: false,
        extensions: {},
        entries: converted
    };
}

function downloadFile(content, filename, mimeType) {
    try { if (typeof download === 'function') { download(content, filename, mimeType); return true; } } catch (e) {}
    try {
        const blob = new Blob([content], { type: mimeType });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url; a.download = filename;
        document.body.appendChild(a); a.click(); document.body.removeChild(a);
        setTimeout(() => URL.revokeObjectURL(url), 5000);
        return true;
    } catch (e) { console.error('blob 下载失败:', e); return false; }
}

function collectEntriesFromRange(startId, endId) {
    const worldbookEntries = [];
    const allNicknames = new Set();
    let guessedName = '';
    let skipped = 0;

    for (let i = startId; i <= endId; i++) {
        const msg = getChatMessages(i)[0];
        if (!msg || msg.role !== 'assistant' || msg.is_hidden) continue;
        if (!msg.message || msg.message.trim().length < CONFIG.min_message_length) continue;

        const subEntries = buildEntriesForExport(msg, i);
        if (subEntries.length === 0) { skipped++; continue; }
        worldbookEntries.push(...subEntries);

        for (const e of subEntries) {
            const parsed = parseCharacterFields(e.content || '');
            if (!guessedName && parsed.name) guessedName = parsed.name;
            parsed.nicknames.forEach(n => allNicknames.add(n));
        }
    }

    return { worldbookEntries, allNicknames, guessedName, skipped };
}

function collectHtmlBlocks(startId, endId) {
    const blocks = [];
    for (let i = startId; i <= endId; i++) {
        const msg = getChatMessages(i)[0];
        if (!msg || msg.role !== 'assistant' || msg.is_hidden) continue;
        if (!msg.message) continue;
        const raw = stripThinkingChain(msg.message);
        const regex = /```html\s*([\s\S]*?)```/gi;
        let match;
        while ((match = regex.exec(raw)) !== null) {
            const content = match[1].trim();
            if (content) blocks.push(content);
        }
    }
    return blocks;
}

async function exportAsCharacterCard() {
    const lastId = getLastMessageId();
    if (lastId < 0) { toastr.warning('当前没有消息'); return; }

    const scope = prompt(
        `请选择导出范围：\n` +
        `1. 全部对话（第 0 层到第 ${lastId} 层）\n` +
        `2. 最近 N 条 AI 回复\n` +
        `3. 指定楼层范围\n\n` +
        `输入数字：`,
        "1"
    );
    if (!scope) return;

    let startId = 0;
    let endId = lastId;

    if (scope === '2') {
        const countStr = prompt("要向前导出多少条AI回复：", "10");
        if (!countStr || isNaN(countStr)) return;
        const count = parseInt(countStr);
        let found = 0;
        for (let i = lastId; i >= 0; i--) {
            const msg = getChatMessages(i)[0];
            if (msg && msg.role === 'assistant' && !msg.is_hidden && msg.message.trim().length >= CONFIG.min_message_length) {
                found++;
                if (found >= count) { startId = i; break; }
            }
        }
    } else if (scope === '3') {
        const rangeStr = prompt("请输入楼层范围（格式如 5-15 或单层 10）：", `0-${lastId}`);
        if (!rangeStr) return;
        const parts = rangeStr.split('-');
        startId = parseInt(parts[0]);
        endId = parts.length === 2 ? parseInt(parts[1]) : startId;
        if (isNaN(startId) || isNaN(endId)) { toastr.warning("无效的范围"); return; }
    }

    const { worldbookEntries, guessedName, skipped } = collectEntriesFromRange(startId, endId);
    if (!worldbookEntries.length) { toastr.warning('指定范围内没有可导出的内容（未发现 markdown 块）'); return; }

    const selectiveEntries = [];
    const constantPieces = [];
    let withKeys = 0;
    let withoutKeys = 0;

    for (const e of worldbookEntries) {
        const keys = Array.isArray(e.strategy?.keys) ? e.strategy.keys : [];
        const isSelective = e.strategy?.type === 'selective' && keys.length > 0;
        if (isSelective && keys.length > 0) {
            withKeys++;
            selectiveEntries.push(e);
        } else {
            withoutKeys++;
            constantPieces.push(e.content || '');
        }
    }

    const systemPromptText = constantPieces.join('\n\n---\n\n');
    const characterBook = buildCharacterBookFromEntries(`${guessedName || '角色卡'}的世界书`, selectiveEntries);

    const defaultName = guessedName || `角色卡_${startId}-${endId}`;

    const cardName = prompt(
        `即将导出 1 张角色卡（V2）\n` +
        `范围：第 ${startId} - ${endId} 层\n` +
        `📦 markdown 块：${worldbookEntries.length} 条\n` +
        `📚 有触发词（进世界书）：${withKeys} 条\n` +
        `📝 无触发词（进主提示词）：${withoutKeys} 条\n` +
        `⏭️ 跳过（无 markdown 的对话）：${skipped} 条\n\n` +
        `请输入角色卡名称：`,
        defaultName
    );
    if (!cardName || cardName.trim() === '') { toastr.warning('已取消导出'); return; }
    const cleanName = cardName.trim();

    const card = {
        spec: "chara_card_v2",
        spec_version: "2.0",
        data: {
            name: cleanName,
            description: "",
            personality: "",
            scenario: "",
            first_mes: "",
            mes_example: "",
            creator_notes: "",
            system_prompt: systemPromptText,
            post_history_instructions: "",
            alternate_greetings: [],
            character_book: characterBook,
            tags: [],
            creator: "世界书助手",
            character_version: "1.0",
            extensions: {}
        }
    };

    const json = JSON.stringify(card, null, 2);
    const filename = `${cleanName}.json`;

    const ok = downloadFile(json, filename, 'application/json');
    if (ok) {
        toastr.success(`已导出 V2 卡「${cleanName}」（世界书 ${withKeys} + 主提示词 ${withoutKeys}）`);
    } else {
        toastr.error('导出失败，请检查浏览器下载权限');
    }
}

function generateUUID() {
    try {
        if (typeof crypto !== 'undefined' && crypto.randomUUID) return crypto.randomUUID();
    } catch (e) {}
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
        const r = Math.random() * 16 | 0;
        const v = c === 'x' ? r : (r & 0x3 | 0x8);
        return v.toString(16);
    });
}

async function exportAsCalendarCard() {
    const lastId = getLastMessageId();
    if (lastId < 0) { toastr.warning('当前没有消息'); return; }

    const scope = prompt(
        `🖼️ 日历角色卡导出\n\n` +
        `请选择导出范围：\n` +
        `1. 全部对话（第 0 层到第 ${lastId} 层）\n` +
        `2. 最近 N 条 AI 回复\n` +
        `3. 指定楼层范围\n\n` +
        `输入数字：`,
        "1"
    );
    if (!scope) return;

    let startId = 0;
    let endId = lastId;

    if (scope === '2') {
        const countStr = prompt("要向前导出多少条AI回复：", "10");
        if (!countStr || isNaN(countStr)) return;
        const count = parseInt(countStr);
        let found = 0;
        for (let i = lastId; i >= 0; i--) {
            const msg = getChatMessages(i)[0];
            if (msg && msg.role === 'assistant' && !msg.is_hidden && msg.message.trim().length >= CONFIG.min_message_length) {
                found++;
                if (found >= count) { startId = i; break; }
            }
        }
    } else if (scope === '3') {
        const rangeStr = prompt("请输入楼层范围（格式如 5-15 或单层 10）：", `0-${lastId}`);
        if (!rangeStr) return;
        const parts = rangeStr.split('-');
        startId = parseInt(parts[0]);
        endId = parts.length === 2 ? parseInt(parts[1]) : startId;
        if (isNaN(startId) || isNaN(endId)) { toastr.warning("无效的范围"); return; }
    }

    const { worldbookEntries, guessedName, skipped } = collectEntriesFromRange(startId, endId);
    const htmlBlocks = collectHtmlBlocks(startId, endId);
    const introPage = htmlBlocks.join('\n\n');

    if (!worldbookEntries.length && !introPage) {
        toastr.warning('指定范围内没有可导出的内容');
        return;
    }

    const lorebookItems = [];
    const themePieces = [];
    let withKeys = 0;
    let withoutKeys = 0;

    for (const e of worldbookEntries) {
        const keys = Array.isArray(e.strategy?.keys) ? e.strategy.keys : [];
        const isSelective = e.strategy?.type === 'selective' && keys.length > 0;

        if (isSelective && keys.length > 0) {
            withKeys++;
            lorebookItems.push({
                enabled: e.enabled !== false,
                triggerWords: keys,
                triggerWordsType: "OR",
                blackListWordsType: "OR",
                priority: 0,
                probability: 100,
                scanDepth: 0,
                triggerRange: ["system", "user"],
                impactRange: ["system", "headPrompt", "rearPrompt"],
                contentMethod: "single",
                content: e.content || ''
            });
        } else {
            withoutKeys++;
            themePieces.push(e.content || '');
        }
    }

    const headPrompt = themePieces.join('\n\n---\n\n');
    const defaultName = guessedName || `角色卡_${startId}-${endId}`;

    const cardName = prompt(
        `🖼️ 日历角色卡导出\n\n` +
        `范围：第 ${startId} - ${endId} 层\n` +
        `📦 markdown 块：${worldbookEntries.length} 条\n` +
        `📚 有触发词（进世界书）：${withKeys} 条\n` +
        `📝 无触发词（进前置词）：${withoutKeys} 条\n` +
        `🌐 HTML 网页块（进 introPage）：${htmlBlocks.length} 个\n` +
        `⏭️ 跳过（无 markdown 的对话）：${skipped} 条\n\n` +
        `请输入角色卡名称：`,
        defaultName
    );
    if (!cardName || cardName.trim() === '') { toastr.warning('已取消导出'); return; }
    const cleanName = cardName.trim();

    const now = new Date().toISOString();
    const calendarCard = {
        id: generateUUID(),
        authorId: "",
        title: cleanName,
        description: null,
        background: null,
        backgroundWidth: null,
        backgroundHeight: null,
        cover: null,
        coverWidth: null,
        coverHeight: null,
        introPage: introPage,
        changelog: null,
        createdAt: now,
        updatedAt: now,
        initPublicAt: null,
        prompt: {
            headPrompt: headPrompt,
            rearPrompt: "",
            systemPrompt: "",
            lorebook: lorebookItems
        },
        promptSizeBytes: null,
        renderTemplate: null,
        assistantRegexReplace: null,
        enableMemorySchema: null,
        memorySchema: null,
        injectStyles: null,
        injectStylesSizeBytes: null,
        options: null,
        language: "zh",
        containsNSFW: null,
        publicAccess: false,
        deletedAt: null,
        isAnonymous: null,
        enableSafeguard: null,
        defaultModelSettings: null,
        totalConversations: 0,
        totalPlayedUsers: 0,
        deeplyPlayed: 0,
        maxNonAuthorPlayCount: 0,
        fixedPromptLength: 0,
        lorebookLength: 0,
        cardsTags: [],
        cardsVoices: []
    };

    const json = JSON.stringify(calendarCard, null, 2);
    const filename = `${cleanName}.json`;

    const ok = downloadFile(json, filename, 'application/json');
    if (ok) {
        toastr.success(`已导出日历卡「${cleanName}」（世界书 ${withKeys} + 前置词 ${withoutKeys} + 网页 ${htmlBlocks.length}）`);
    } else {
        toastr.error('导出失败，请检查浏览器下载权限');
    }
}

async function batchImportRecent() {
    const countStr = prompt("请输入要向前导入多少条AI回复：", "10");
    if (!countStr || isNaN(countStr)) return;
    const count = parseInt(countStr);
    const endId = getLastMessageId();
    if (endId < 0) { toastr.warning("没有消息"); return; }
    let startId = 0, found = 0;
    for (let i = endId; i >= 0; i--) {
        const msg = getChatMessages(i)[0];
        if (msg && msg.role === 'assistant' && !msg.is_hidden && msg.message.trim().length >= CONFIG.min_message_length) {
            found++; if (found >= count) { startId = i; break; }
        }
    }
    await doBatchImport(startId, endId);
}

async function batchImportRange() {
    const rangeStr = prompt("请输入楼层范围（格式如 5-15 或单层 10）：", `0-${getLastMessageId()}`);
    if (!rangeStr) return;
    const parts = rangeStr.split('-');
    let startId = parseInt(parts[0]);
    let endId = parts.length === 2 ? parseInt(parts[1]) : startId;
    if (isNaN(startId) || isNaN(endId)) { toastr.warning("无效的范围"); return; }
    await doBatchImport(startId, endId);
}

async function doBatchImport(startId, endId) {
    const wbName = await getCurrentWorldbookName();
    if (!wbName) { toastr.warning('未选择目标世界书'); return; }
    const entries = [];
    let skipped = 0;
    for (let i = startId; i <= endId; i++) {
        const msg = getChatMessages(i)[0];
        if (!msg || msg.role !== 'assistant' || msg.is_hidden) { skipped++; continue; }
        if (!msg.message || msg.message.trim().length < CONFIG.min_message_length) { skipped++; continue; }
        const subEntries = buildEntriesFromMessage(msg, i);
        if (subEntries.length === 0) { skipped++; continue; }
        entries.push(...subEntries);
    }
    if (!entries.length) { toastr.warning(`指定范围内没有可导入的内容`); return; }
    try {
        await createWorldbookEntries(wbName, entries);
        toastr.success(`成功导入 ${entries.length} 条到「${wbName}」${skipped > 0 ? `（跳过 ${skipped} 条）` : ''}`);
    } catch (e) { toastr.error('批量导入失败：' + e.message); }
}

// ============================================================
// 悬浮菜单
// ============================================================
function openWorldbookMenu(page = 'main') {
    const doc = window.parent ? window.parent.document : document;
    const oldMenu = doc.getElementById('wb-helper-menu');
    if (oldMenu) oldMenu.remove();

    const overlay = doc.createElement('div');
    overlay.id = 'wb-helper-menu';
    overlay.style.cssText = `position: fixed !important; top: 0; left: 0; width: 100vw; height: 100vh;
        background: rgba(0,0,0,0.7); z-index: 2147483647 !important;
        display: flex; align-items: center; justify-content: center;`;

    const menu = doc.createElement('div');
    menu.style.cssText = `background: #1e1e1e; border-radius: 12px; padding: 20px;
        width: 85%; max-width: 340px; display: flex; flex-direction: column; gap: 8px;
        color: #fff; box-shadow: 0 4px 20px rgba(0,0,0,0.8); border: 1px solid #444;
        position: relative; z-index: 2147483647 !important; max-height: 90vh; overflow-y: auto;`;

    const header = doc.createElement('div');
    if (page === 'main') {
        header.innerHTML = `
            <div style="text-align:center; font-size:18px; font-weight:bold; margin-bottom:5px;">📚 世界书助手</div>
            <div style="text-align:center; font-size:12px; color:#aaa; margin-bottom:10px;">
                当前：${targetWorldbook || "（角色卡主世界书）"}
            </div>`;
    } else {
        header.innerHTML = `
            <div style="text-align:center; font-size:18px; font-weight:bold; margin-bottom:5px;">⚙️ 设置开关</div>
            <div style="text-align:center; font-size:12px; color:#aaa; margin-bottom:10px;">
                写卡：${writeCardMode ? '✅' : '❌'} ｜ 关键词：${recognizeTriggerKeys ? '✅' : '❌'}<br>
                YAML过滤：${excludeYamlBlock ? '✅' : '❌'} ｜ 去思维链：${removeThinking ? '✅' : '❌'}
            </div>`;
    }
    menu.appendChild(header);

    let actions;
    if (page === 'main') {
        actions = [
            { text: '📥 导入预设（SillyTavern）', fn: importPreset },
            { text: '📥 存入最新AI回复', fn: saveLatestReply },
            { text: '📋 复制最新HTML', fn: copyLatestHTML },
            { text: '📚 选择目标世界书', fn: selectWorldbook },
            { text: '➕ 新建世界书', fn: createNewWorldbook },
            { text: '📦 批量导入最近N条', fn: batchImportRecent },
            { text: '📦 批量导入指定范围', fn: batchImportRange },
            { text: '🎴 导出为角色卡（V2）', fn: exportAsCharacterCard },
            { text: '🖼️ 日历角色卡导出', fn: exportAsCalendarCard },
            { text: '⚙️ 设置开关 ▸', fn: () => openWorldbookMenu('settings'), keepOpen: true, rawOpen: true }
        ];
    } else {
        actions = [
            { text: `⚙️ YAML 过滤：${excludeYamlBlock ? '开启' : '关闭'}`, fn: toggleYamlFilter, keepOpen: true },
            { text: `✍️ 写卡助手定制：${writeCardMode ? '开启' : '关闭'}`, fn: toggleWriteCardMode, keepOpen: true },
            { text: `🔑 触发关键词识别：${recognizeTriggerKeys ? '开启' : '关闭'}`, fn: toggleTriggerKeys, keepOpen: true },
            { text: `🧠 去思维链：${removeThinking ? '开启' : '关闭'}`, fn: toggleRemoveThinking, keepOpen: true },
            { text: '↩️ 返回主菜单', fn: () => openWorldbookMenu('main'), keepOpen: true, rawOpen: true }
        ];
    }

    actions.forEach(action => {
        const btn = doc.createElement('button');
        btn.innerText = action.text;
        btn.style.cssText = `padding: 12px; border-radius: 8px; border: none;
            background: #333; color: #fff; font-size: 15px; cursor: pointer;
            text-align: left; padding-left: 20px;`;
        btn.onmouseover = () => btn.style.background = '#444';
        btn.onmouseout = () => btn.style.background = '#333';
        btn.onclick = () => {
            overlay.remove();
            if (action.rawOpen) {
                setTimeout(() => action.fn(), 50);
            } else if (action.keepOpen) {
                setTimeout(() => { action.fn(); setTimeout(() => openWorldbookMenu(page), 50); }, 0);
            } else { action.fn(); }
        };
        menu.appendChild(btn);
    });

    const cancelBtn = doc.createElement('button');
    cancelBtn.innerText = '❌ 取消';
    cancelBtn.style.cssText = `padding: 10px; border-radius: 8px; border: none;
        background: #8b0000; color: #fff; font-size: 14px; cursor: pointer; margin-top: 5px;`;
    cancelBtn.onclick = () => overlay.remove();
    menu.appendChild(cancelBtn);

    overlay.appendChild(menu);
    overlay.onclick = (e) => { if (e.target === overlay) overlay.remove(); };
    doc.body.appendChild(overlay);
}

// ============================================================
// 事件绑定
// ============================================================
eventOn(getButtonEvent('📥 世界书助手'), async () => {
    openWorldbookMenu('main');
});

// 🆕 同时也绑定独立按钮（如果你在编辑器里加了"📥 导入预设"按钮）
try {
    eventOn(getButtonEvent('📥 导入预设'), async () => {
        await importPreset();
    });
} catch (e) {}

window.saveLatestReply = saveLatestReply;
window.selectWorldbook = selectWorldbook;
window.createNewWorldbook = createNewWorldbook;
window.batchImportRecent = batchImportRecent;
window.batchImportRange = batchImportRange;
window.toggleYamlFilter = toggleYamlFilter;
window.toggleWriteCardMode = toggleWriteCardMode;
window.toggleTriggerKeys = toggleTriggerKeys;
window.toggleRemoveThinking = toggleRemoveThinking;
window.exportAsCharacterCard = exportAsCharacterCard;
window.exportAsCalendarCard = exportAsCalendarCard;
window.copyLatestHTML = copyLatestHTML;
window.importPreset = importPreset;
window.openWorldbookMenu = openWorldbookMenu;