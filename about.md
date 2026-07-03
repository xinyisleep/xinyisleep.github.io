---
layout: post
title: "About"
author: "XinYiSleep"
permalink: /About/
---

<style>
.ascii-banner {
    font-family: monospace;
    font-size: 10px;
    line-height: 1.2;
    text-align: center;
    color: #c00;
    margin: 30px 0;
}
.terminal-box {
    background: #1a1a2e !important;
    color: #00ff41 !important;
    font-family: monospace !important;
    padding: 20px !important;
    border-radius: 6px !important;
    margin: 20px 0 !important;
}
.terminal-box .prompt { color: #00ff41 !important; }
.terminal-box .cmd { color: #fff !important; }
.terminal-box span { display: block; min-height: 1.4em; color: #00ff41 !important; }
#cursor { 
    animation: blink 1s step-end infinite; 
    color: #00ff41 !important; 
    display: block !important; 
    margin-top: 4px !important; 
}
@keyframes blink {
    50% { opacity: 0; }
}
.tagline { color: #888; font-family: monospace; }
.divider { border: none; border-top: 1px dashed #ccc; margin: 30px 0; }
table { width: 100%; }
</style>

<div class="ascii-banner">
<pre>
██╗  ██╗██╗███╗   ██╗██╗   ██╗██╗
╚██╗██╔╝██║████╗  ██║╚██╗ ██╔╝██║
 ╚███╔╝ ██║██╔██╗ ██║ ╚████╔╝ ██║
 ██╔██╗ ██║██║╚██╗██║  ╚██╔╝  ╚═╝
██╔╝ ██╗██║██║ ╚████║   ██║   ██╗
╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝   ╚═╝   ╚═╝
</pre>
</div>

<div class="terminal-box" id="terminal">
    <span id="line1"></span>
    <span id="line2"></span>
    <span id="line3"></span>
    <span id="line4"></span>
    <span id="line5"></span>
    <span id="cursor">$ █</span>
</div>

<script>
(function() {
    var lines = [
        {text: '$ whoami', delay: 600},
        {text: 'root@xinyisleep', delay: 400},
        {text: '$ cat /etc/passwd', delay: 600},
        {text: 'xinyisleep:x:1000:1000:Security Researcher:/home/xinyisleep:/bin/bash', delay: 500}
    ];
    var idx = 0, charIdx = 0;
    var el = document.getElementById('line1');
    var els = ['line1','line2','line3','line4','line5'];
    var cursor = document.getElementById('cursor');
    
    function type() {
        if (idx >= lines.length) { cursor.innerHTML = '$ █'; return; }
        var line = lines[idx];
        if (charIdx < line.text.length) {
            el.innerHTML += line.text.charAt(charIdx);
            charIdx++;
            setTimeout(type, 80 + Math.random() * 60);
        } else {
            el = document.getElementById(els[idx + 1]);
            idx++;
            charIdx = 0;
            if (idx < lines.length) setTimeout(type, lines[idx].delay);
            else { cursor.innerHTML = '$ █'; }
        }
    }
    setTimeout(type, 500);
})();
</script>

---

## 🏴 原创漏洞

| 时间 | 漏洞名称 | 编号 |
|:---:|:---|:---:|
| 2026-07-01 | **XXL-JOB** XSS → RCE | — |
| 2024-07-15 | **ThinkPHP 3** 文件包含 | CVE-2025-50707 / CNVD-2024-39045 |
| 2024-04-22 | **ThinkPHP 5.1** 文件包含 | CVE-2025-50706 / CNVD-2024-29981 |

## ⚡ 1Day

| 时间 | 漏洞名称 | 状态 |
|:---:|:---|:---:|
| 2025-01-24 | **禅道** 二次注入 → RCE（Version 20.0.beta2） | ⚠️ 已有补丁 |

---

<div align="center" style="margin: 30px 0;">
    <span class="tagline">「 一个安全研究人员的日常 — 挖洞 · 写文 · 踩坑 · 复现 」</span>
    <br><br>
    可以给我发邮件 → <a href="mailto:">E-mail</a>
    <br><br>
    <code style="color: #999;">// 路虽远 行则将至</code>
</div>
