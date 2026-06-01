# pixiv-deleted

```javascript
// ==UserScript==
// @name Pixiv 已删除或不公开
// @name:zh-CN Pixiv 已删除或不公开
// @name:zh-TW Pixiv 已刪除或非公開
// @name:ja Pixiv 削除済みもしくは非公開
// @name:en Pixiv Deleted or Private
// @namespace https://greasyfork.org/users/1271023
// @version 1.41
// @author 朧月猫 (修改 by Grok)
// @description 获取 Pixiv 已删除或不公开的插画 ID，并加入复制、Danbooru、AIBooru 与 Google 快捷搜索
// @description:zh-TW 獲取 Pixiv 已刪除或非公開的插畫 ID，並加入複製、Danbooru、AIBooru 與 Google 快捷搜索
// @match https://www.pixiv.net/users/*
// @match https://www.pixiv.net/*/users/*
// @icon https://www.pixiv.net/favicon.ico
// @license MIT
// @grant none
// ==/UserScript==


(function() {
    'use strict';

    const siteLang = document.documentElement.lang.toLowerCase();
    const i18n = {
        'zh-cn': { hint: '点击复制 ID', copied: '✅ 已复制', search: '搜索' },
        'zh-tw': { hint: '點擊複製 ID', copied: '✅ 已複製', search: '搜尋' },
        'ja': { hint: 'IDをコピー', copied: '✅ コピー完了', search: '検索' },
        'en': { hint: 'Click to copy ID', copied: '✅ Copied', search: 'Search' },
    };
    const msg = i18n[siteLang] || (siteLang.includes('zh') ? i18n['zh-cn'] : i18n.en);

    const globalTip = document.createElement('div');
    globalTip.style.cssText = `position:fixed;z-index:999999;background:rgba(0,0,0,0.85);color:#fff;padding:5px 10px;border-radius:4px;font-size:12px;pointer-events:none;visibility:hidden;opacity:0;transition:opacity 0.1s;white-space:nowrap;`;
    document.body.appendChild(globalTip);

    const style = document.createElement('style');
    style.textContent = `
        .px-id-wrapper {
            display: inline-flex;
            flex-direction: column;
            align-items: flex-start;
            vertical-align: middle;
        }
        .px-id-text {
            font-weight: bold;
            cursor: pointer;
            color: inherit;
            transition: color 0.1s;
        }
        .px-id-text:hover {
            color: #ff4081;
        }
        .px-icon-row {
            margin-top: 4px;
            display: flex;
            gap: 10px;
        }
        .px-emoji-link {
            text-decoration: none;
            font-size: 16px;
            transition: transform 0.15s;
        }
        .px-emoji-link:hover { 
            transform: scale(1.4); 
        }
    `;
    document.head.appendChild(style);

    function showTip(target, text) {
        const rect = target.getBoundingClientRect();
        globalTip.textContent = text;
        globalTip.style.visibility = 'visible';
        globalTip.style.opacity = '1';
        globalTip.style.left = `${rect.left}px`;
        globalTip.style.top = `${rect.top - globalTip.offsetHeight - 8}px`;
    }

    function extractArtworkId(url) {
        if (!url) return null;
        const m = url.match(/artworks\/(\d+)/);
        return m ? m[1] : null;
    }

    function replaceSingle(el) {
        if (!el || el.dataset.pixivIdReplaced) return;
        const urlAttr = el.getAttribute('to') || el.getAttribute('href') || '';
        const id = extractArtworkId(urlAttr);
        const text = el.textContent.trim();

        if (id && /^[-—]{3,}$/.test(text)) {
            el.dataset.pixivIdReplaced = '1';
            
            const wrapper = document.createElement('span');
            wrapper.className = 'px-id-wrapper';

            // ID 文字
            const idSpan = document.createElement('span');
            idSpan.textContent = id;
            idSpan.className = 'px-id-text';
            idSpan.onmouseenter = () => showTip(idSpan, msg.hint);
            idSpan.onmouseleave = () => { globalTip.style.visibility = 'hidden'; };
            idSpan.onclick = (e) => {
                e.preventDefault(); e.stopPropagation();
                navigator.clipboard.writeText(id).then(() => showTip(idSpan, msg.copied));
            };
            wrapper.appendChild(idSpan);

            // 图标行（在下方）
            const iconRow = document.createElement('span');
            iconRow.className = 'px-icon-row';

            const links = [
                { emoji: '📖', title: `Danbooru ${msg.search}`, url: `https://danbooru.donmai.us/posts?tags=pixiv%3A${id}` },
                { emoji: '🤖', title: `AIBooru ${msg.search}`, url: `https://aibooru.online/posts?tags=pixiv%3A${id}` },
                { emoji: '🌐', title: `Google ${msg.search}`, url: `https://www.google.com/search?q=pixiv.net/artworks/${id}` }   // ← 恢复成和你原来完全一样
            ];

            links.forEach(link => {
                const a = document.createElement('a');
                a.href = link.url;
                a.textContent = link.emoji;
                a.target = '_blank';
                a.className = 'px-emoji-link';
                a.onmouseenter = () => showTip(a, link.title);
                a.onmouseleave = () => { globalTip.style.visibility = 'hidden'; };
                iconRow.appendChild(a);
            });

            wrapper.appendChild(iconRow);
            el.textContent = '';
            el.appendChild(wrapper);
        }
    }

    function scan() {
        const selectors = '[to*="/artworks/"],[href*="/artworks/"]';
        document.querySelectorAll(selectors).forEach(replaceSingle);
    }

    const observer = new MutationObserver(scan);
    observer.observe(document.body, { childList: true, subtree: true });
    scan();
})();
