# Bilibili-Audio-Matrix
B站声道控制器（油猴脚本）

```javascript

// ==UserScript==
// @name         B站声道控制器
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  B站声道管理系统
// @author       QAQ
// @match        https://www.bilibili.com/*
// @grant        GM_addStyle
// @run-at       document-end
// ==/UserScript==

(function() {
    'use strict';

    const MODES = {
        ORIGINAL: { id: 0, name: '原声', icon: '🔊', color: '#00a1d6' },
        SWAPPED:  { id: 1, name: '翻转', icon: '🔄', color: '#ff6050' }
    };
    const PLAYER_SELECTOR = '.bpx-player-video-wrap video, video#bilibili-player video';
    let currentMode = MODES.ORIGINAL;
    let audioNodes = new WeakMap();
    let intervalId = null;


    GM_addStyle(`
        .audio-ctrl-panel {
            position: absolute;
            right: 180px;
            bottom: 50px;
            z-index: 9999;
            background: rgba(16,18,20,0.95);
            border-radius: 12px;
            padding: 16px;
            width: 280px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.2);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255,255,255,0.1);
        }
        .mode-buttons {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 8px;
            margin-bottom: 16px;
        }
        .mode-btn {
            padding: 12px;
            background: rgba(255,255,255,0.1);
            border: none;
            border-radius: 8px;
            color: white;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
        }
        .mode-btn.active {
            background: var(--active-color);
            box-shadow: 0 4px 16px var(--active-glow);
        }
        .auto-control {
            border-top: 1px solid rgba(255,255,255,0.1);
            padding-top: 16px;
        }
        .time-input {
            width: 100%;
            padding: 10px;
            background: rgba(0,0,0,0.3);
            border: 1px solid rgba(255,255,255,0.1);
            border-radius: 6px;
            color: white;
            margin-bottom: 12px;
        }
        .toggle-btn {
            width: 100%;
            padding: 10px;
            background: linear-gradient(135deg, #00a1d6 0%, #008cba 100%);
            border: none;
            border-radius: 6px;
            color: white;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        .toggle-btn.active {
            background: linear-gradient(135deg, #ff6050 0%, #ff3b30 100%);
        }
    `);


    function init() {
        injectControlPanel();
        setupAudioSystem();
        setupPanelObserver();
    }


    function injectControlPanel() {
        const panel = document.createElement('div');
        panel.className = 'audio-ctrl-panel';
        panel.innerHTML = `
            <div class="mode-buttons">
                ${Object.values(MODES).map(mode => `
                    <button class="mode-btn"
                            data-mode="${mode.id}"
                            style="--active-color: ${mode.color};
                                   --active-glow: ${mode.color}40">
                        ${mode.icon} ${mode.name}
                    </button>
                `).join('')}
            </div>
            <div class="auto-control">
                <input type="number" class="time-input"
                       placeholder="自动切换间隔（秒）" min="1" value="5">
                <button class="toggle-btn">启动自动切换</button>
            </div>
        `;


        const observer = new MutationObserver((_, observer) => {
            const container = document.querySelector('.bpx-player-control-bottom-right');
            if (container) {
                container.prepend(panel);
                bindPanelEvents();
                observer.disconnect();
            }
        });
        observer.observe(document.body, { childList: true, subtree: true });
    }


    function bindPanelEvents() {
        const panel = document.querySelector('.audio-ctrl-panel');
        panel.querySelectorAll('.mode-btn').forEach(btn => {
            btn.addEventListener('click', () => {
                switchMode(MODES[btn.dataset.mode === '0' ? 'ORIGINAL' : 'SWAPPED']);
            });
        });
        panel.querySelector('.toggle-btn').addEventListener('click', toggleAutoSwitch);
        updateActiveButton();
    }


    function switchMode(mode) {
        if (intervalId) {
            clearInterval(intervalId);
            intervalId = null;
            updateToggleButton();
        }
        currentMode = mode;
        updateActiveButton();
        applyAudioSettings();
    }


    function toggleAutoSwitch() {
        const input = document.querySelector('.time-input');
        const seconds = parseFloat(input.value);

        if (isNaN(seconds) || seconds < 1) {
            input.style.borderColor = '#ff6050';
            setTimeout(() => input.style.borderColor = '', 1000);
            return;
        }

        const button = this;
        if (!intervalId) {
            intervalId = setInterval(() => {
                currentMode = currentMode === MODES.ORIGINAL ? MODES.SWAPPED : MODES.ORIGINAL;
                updateActiveButton();
                applyAudioSettings();
            }, seconds * 1000);
            button.classList.add('active');
        } else {
            clearInterval(intervalId);
            intervalId = null;
            button.classList.remove('active');
        }
    }


    function updateActiveButton() {
        document.querySelectorAll('.mode-btn').forEach(btn => {
            btn.classList.toggle('active', btn.dataset.mode == currentMode.id);
        });
    }

    function updateToggleButton() {
        const button = document.querySelector('.toggle-btn');
        button.textContent = intervalId ? '停止自动切换' : '启动自动切换';
    }


    function setupAudioSystem() {
        new MutationObserver(() => {
            document.querySelectorAll(PLAYER_SELECTOR).forEach(video => {
                if (!audioNodes.has(video)) initAudioContext(video);
            });
        }).observe(document.body, { subtree: true, childList: true });
    }

    function initAudioContext(video) {
        try {
            const ctx = new (window.AudioContext || window.webkitAudioContext)();
            const source = ctx.createMediaElementSource(video);
            const splitter = ctx.createChannelSplitter(2);
            const merger = ctx.createChannelMerger(2);

            source.connect(splitter);
            reconfigureAudio(splitter, merger);
            merger.connect(ctx.destination);

            audioNodes.set(video, { splitter, merger });


            video.addEventListener('play', () => ctx.resume());
            video.addEventListener('click', () => ctx.resume());
        } catch (error) {
            console.error('音频初始化失败:', error);
        }
    }


    function reconfigureAudio(splitter, merger) {
        splitter.disconnect();
        if (currentMode === MODES.ORIGINAL) {
            splitter.connect(merger, 0, 0);
            splitter.connect(merger, 1, 1);
        } else {
            splitter.connect(merger, 0, 1);
            splitter.connect(merger, 1, 0);
        }
    }


    function applyAudioSettings() {
        document.querySelectorAll(PLAYER_SELECTOR).forEach(video => {
            const nodes = audioNodes.get(video);
            if (nodes) reconfigureAudio(nodes.splitter, nodes.merger);
        });
    }


    function setupPanelObserver() {
        new MutationObserver(mutations => {
            mutations.forEach(mutation => {
                if (!document.querySelector('.audio-ctrl-panel')) {
                    injectControlPanel();
                }
            });
        }).observe(document.body, { childList: true, subtree: true });
    }


    window.addEventListener('load', init);
})();


```
