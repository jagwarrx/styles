
---
name: "Library/jagwarrx/Flashcards"
tags: meta/library
pageDecoration.prefix: "🃏 "
share.uri: "github:jagwarrx/styles/flashcards/flashcards.md"
share.hash: 0000000
share.mode: pull
---

# Floating Flashcards

A minimal flashcard extension for SilverBullet. Parses any page for Q&A pairs and presents them in a floating, draggable card with flip, shuffle, keyboard navigation, and drag-to-resize.

![Front](https://raw.githubusercontent.com/jagwarrx/styles/main/flashcards/img-front.png)

![Back](https://raw.githubusercontent.com/jagwarrx/styles/main/flashcards/img-back.png)

## Features

* **Flip:** Click card or ↑↓ arrow keys to flip between question and answer
* **Navigate:** ←→ arrow keys or ◀▶ buttons to move between cards
* **Shuffle:** ⟳ button to randomize deck order
* **Drag:** Grab the header to reposition anywhere on screen (remembers position)
* **Resize:** Drag the bottom edge to adjust card height (remembers height)
* **Markdown:** Supports **bold**, *italic*, and ==highlighted== text in answers
* **Paragraphs:** Empty lines in answers are preserved as paragraph spacing
* **Theming:** Built-in support for both Dark and Light modes

## How to Use

Run the `Flashcards: Start` command on any page containing Q&A data.

### Card Format

Questions start with `**Q: ...**` (bold) or plain `Q: ...` at the beginning of a line. Everything below until the next question or a `---` horizontal rule becomes the answer.

---

### Controls

| Action | Control |
|---|---|
| Flip card | Click card / ↑ / ↓ |
| Next card | ▶ button / → |
| Previous card | ◀ button / ← |
| Shuffle deck | ⟳ button |
| Move widget | Drag header |
| Resize height | Drag bottom edge |
| Close | ✕ button |

## Implementation

### Space Style

```space-style
body.sb-dragging-active {
    user-select: none !important;
    -webkit-user-select: none !important;
}

#sb-flashcard-root {
    position: fixed;
    width: 390px;
    z-index: 101;
    font-family: -apple-system, "SF Pro Display", "Helvetica Neue", system-ui, sans-serif !important;
    font-size: 13px !important;
    -webkit-font-smoothing: antialiased !important;
    user-select: none;
    touch-action: none;
}

#sb-flashcard-root * {
    font-family: -apple-system, "SF Pro Display", system-ui, sans-serif !important;
}

html[data-theme="dark"] #sb-flashcard-root {
    --fc-bg: #2b2b2d;
    --fc-border: rgba(255,255,255,0.2);
    --fc-text: #e5e5e5;
    --fc-dim: #777;
    --fc-accent: #3B82F6;
    --fc-hover: rgba(255,255,255,0.1);
    --fc-back-bg: #333;
}

html[data-theme="light"] #sb-flashcard-root {
    --fc-bg: #fafafa;
    --fc-border: rgba(0,0,0,0.18);
    --fc-text: #1a1a1a;
    --fc-dim: #999;
    --fc-accent: #2862CF;
    --fc-hover: rgba(0,0,0,0.06);
    --fc-back-bg: #f0f0f0;
}

.fc-card {
    background: var(--fc-bg);
    color: var(--fc-text);
    border-radius: 14px;
    border: 2px solid var(--fc-border);
    overflow: hidden;
    display: flex;
    flex-direction: column;
}

.fc-header {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 6px 10px;
    cursor: grab;
    position: relative;
    gap: 6px;
}

.fc-header-label { font-size: 13px; font-weight: 700; color: var(--fc-text); }

.fc-close {
    position: absolute;
    right: 10px;
    background: transparent;
    color: var(--fc-dim);
    border: none;
    border-radius: 4px;
    padding: 0 3px;
    cursor: pointer;
    font-size: 0.8em;
    line-height: 1;
    opacity: 0.4;
}
.fc-close:hover { opacity: 1; background: oklch(0.65 0.2 30); color: white; }

.fc-nav-sm {
    background: transparent;
    color: var(--fc-dim);
    border: none;
    border-radius: 4px;
    padding: 1px 4px;
    cursor: pointer;
    font-size: 10px;
    font-weight: 700;
    opacity: 0.6;
}
.fc-nav-sm:hover { opacity: 1; color: var(--fc-text); }
.fc-nav-sm:disabled { opacity: 0.2; cursor: default; }

.fc-body {
    height: var(--fc-height, 140px);
    padding: 0;
    text-align: center;
    cursor: pointer;
    font-size: 13px;
    line-height: 1.5;
    transition: background 0.2s ease;
    border-top: 1px solid var(--fc-border);
    border-bottom: 1px solid var(--fc-border);
    overflow-y: auto;
    overflow-x: hidden;
}

.fc-body:hover { background: var(--fc-hover); }
.fc-body.is-back { background: var(--fc-back-bg); text-align: left; }
.fc-body.is-centered { display: flex; align-items: center; justify-content: center; }

#fc-text p { margin: 0 0 10px 0; }
#fc-text p:last-child { margin-bottom: 0; }

#sb-flashcard-root mark {
    background: rgba(255, 213, 79, 0.3);
    padding: 1px 3px;
    border-radius: 2px;
}

.fc-footer {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 5px 10px;
    gap: 8px;
}

.fc-counter { font-size: 11px; color: var(--fc-dim); font-weight: 600; font-variant-numeric: tabular-nums; }

.fc-shuffle {
    background: transparent;
    border: none;
    color: var(--fc-dim);
    cursor: pointer;
    font-size: 13px;
    padding: 1px 4px;
    border-radius: 4px;
    opacity: 0.5;
}
.fc-shuffle:hover { opacity: 1; color: var(--fc-accent); }
```

### Space Lua

```space-lua
-- priority: -1

function toggleFlashcards()
    local existing = js.window.document.getElementById("sb-flashcard-root")
    if existing then
        existing.remove()
        return
    end

    local txt = tostring(editor.getText())
    local cards_json_items = {}
    local lines = {}
    for line in string.gmatch(txt .. "\n", "(.-)\n") do
        table.insert(lines, line)
    end

    local i = 1
    local total = 0
    for _ in ipairs(lines) do total = total + 1 end

    while i <= total do
        local line = lines[i]
        local q = string.match(line, "%*%*Q:%s*(.-)%*%*")
        if not q then
            q = string.match(line, "Q:%s*(.+)")
        end

        if q then
            local answer_parts = {}
            local j = i + 1
            while j <= total do
                local next_line = lines[j]
                if string.match(next_line, "%*%*Q:") or string.match(next_line, "^Q:") then
                    break
                end
                if string.match(next_line, "^%-%-%-") then
                    break
                end
                if string.len(next_line) > 0 or #answer_parts > 0 then
                    table.insert(answer_parts, next_line)
                end
                j = j + 1
            end

            local cleaned_parts = {}
            local prev_empty = false
            for _, part in ipairs(answer_parts) do
                if string.len(part) == 0 then
                    if not prev_empty then
                        table.insert(cleaned_parts, "</p><p>")
                    end
                    prev_empty = true
                else
                    prev_empty = false
                    table.insert(cleaned_parts, part)
                end
            end
            local a = "<p>" .. table.concat(cleaned_parts, " ") .. "</p>"
            a = string.gsub(a, "%s+$", "")

            if string.len(q) > 0 and string.len(a) > 0 then
                q = string.gsub(q, "%*%*", "")
                q = string.gsub(q, "==", "")
                q = string.gsub(q, '"', '\\"')

                a = string.gsub(a, "%*%*(.-)%*%*", "<b>%1</b>")
                a = string.gsub(a, "%*(.-)%*", "<i>%1</i>")
                a = string.gsub(a, "==(.-)==", "<mark>%1</mark>")
                a = string.gsub(a, '"', '\\"')

                table.insert(cards_json_items, '{"q":"' .. q .. '","a":"' .. a .. '"}')
            end
            i = j
        else
            i = i + 1
        end
    end

    local count = 0
    for _ in ipairs(cards_json_items) do count = count + 1 end

    if count == 0 then
        editor.flashNotification("No flashcard blocks found.", "error")
        return
    end

    local cards_json = "[" .. table.concat(cards_json_items, ",") .. "]"

    local saved_top = clientStore.get("fc_pos_top") or "80px"
    local saved_left = clientStore.get("fc_pos_left") or "auto"
    local initial_right = "20px"
    if saved_left ~= "auto" then
        initial_right = "auto"
    end
    local saved_height = clientStore.get("fc_height") or "140"

    local container = js.window.document.createElement("div")
    container.id = "sb-flashcard-root"
    container.style.cssText = "position:fixed;z-index:101;width:390px;--fc-height:" .. saved_height .. "px;"
        .. "top:" .. saved_top .. ";left:" .. saved_left .. ";right:" .. initial_right .. ";"

    container.innerHTML = '<div class="fc-card">'
        .. '<div class="fc-header" id="fc-handle">'
        .. '<span class="fc-header-label">Flashcards</span>'
        .. '<button class="fc-close" id="fc-close-btn">✕</button>'
        .. '</div>'
        .. '<div class="fc-body" id="fc-body">'
        .. '<div id="fc-text"></div>'
        .. '</div>'
        .. '<div class="fc-footer">'
        .. '<button class="fc-nav-sm" id="fc-prev">◀</button>'
        .. '<span class="fc-counter" id="fc-header-label">1 / 1</span>'
        .. '<button class="fc-nav-sm" id="fc-next">▶</button>'
        .. '<button class="fc-shuffle" id="fc-shuffle" title="Shuffle all">⟳</button>'
        .. '</div>'
        .. '<div id="fc-resize" style="height:6px;cursor:ns-resize;display:flex;justify-content:center;align-items:center;opacity:0.3;"><div style="width:40px;height:2px;background:var(--fc-dim,#999);border-radius:1px;"></div></div>'
        .. '</div>'

    js.window.document.body.appendChild(container)

    local scriptEl = js.window.document.createElement("script")
    scriptEl.textContent = '(function(){'
        .. 'var cards=' .. cards_json .. ';'
        .. 'var root=document.getElementById("sb-flashcard-root");'
        .. 'var body=document.getElementById("fc-body");'
        .. 'var txt=document.getElementById("fc-text");'
        .. 'var label=document.getElementById("fc-header-label");'
        .. 'var prevBtn=document.getElementById("fc-prev");'
        .. 'var nextBtn=document.getElementById("fc-next");'
        .. 'var SNAP=15,TOP_OFFSET=60,idx=0,flipped=false;'
        .. 'function render(){'
        .. 'var c=cards[idx];flipped=false;'
        .. 'body.innerHTML="<div id=fc-text style=\\"padding:8px 20px\\">"+c.q+"</div>";'
        .. 'txt=document.getElementById("fc-text");'
        .. 'body.classList.remove("is-back");'
        .. 'body.classList.add("is-centered");'
        .. 'label.textContent=(idx+1)+" / "+cards.length;'
        .. 'prevBtn.disabled=idx===0;'
        .. 'nextBtn.disabled=idx===cards.length-1;'
        .. '}'
        .. 'body.onclick=function(){'
        .. 'flipped=!flipped;var c=cards[idx];'
        .. 'body.innerHTML="<div id=fc-text style=\\"padding:8px 20px\\">"+(flipped?c.a:c.q)+"</div>";'
        .. 'txt=document.getElementById("fc-text");'
        .. 'if(flipped){body.classList.add("is-back");body.classList.remove("is-centered")}'
        .. 'else{body.classList.remove("is-back");body.classList.add("is-centered")}'
        .. '};'
        .. 'prevBtn.onclick=function(e){e.stopPropagation();if(idx>0){idx--;render()}};'
        .. 'nextBtn.onclick=function(e){e.stopPropagation();if(idx<cards.length-1){idx++;render()}};'
        .. 'document.getElementById("fc-shuffle").onclick=function(e){e.stopPropagation();'
        .. 'for(var i=cards.length-1;i>0;i--){'
        .. 'var j=Math.floor(Math.random()*(i+1));'
        .. 'var tmp=cards[i];cards[i]=cards[j];cards[j]=tmp;'
        .. '}'
        .. 'idx=0;render();'
        .. '};'
        .. 'var curH=parseInt("' .. saved_height .. '");'
        .. 'document.getElementById("fc-resize").onpointerdown=function(e){'
        .. 'e.preventDefault();e.stopPropagation();'
        .. 'var startY=e.clientY,startH=body.offsetHeight;'
        .. 'function onmove(m){'
        .. 'var newH=Math.max(80,Math.min(500,startH+(m.clientY-startY)));'
        .. 'root.style.setProperty("--fc-height",newH+"px");'
        .. 'curH=newH;'
        .. '}'
        .. 'function onup(){'
        .. 'window.removeEventListener("pointermove",onmove);'
        .. 'window.__fc_height=curH;'
        .. '}'
        .. 'window.addEventListener("pointermove",onmove);'
        .. 'window.addEventListener("pointerup",onup,{once:true});'
        .. '};'
        .. 'document.addEventListener("keydown",function(e){'
        .. 'if(!root.parentNode)return;'
        .. 'if(e.key==="ArrowLeft"&&idx>0){idx--;render()}'
        .. 'if(e.key==="ArrowRight"&&idx<cards.length-1){idx++;render()}'
        .. 'if(e.key==="ArrowUp"||e.key==="ArrowDown"){e.preventDefault();'
        .. 'flipped=!flipped;var c=cards[idx];'
        .. 'body.innerHTML="<div id=fc-text style=\\"padding:8px 20px\\">"+(flipped?c.a:c.q)+"</div>";'
        .. 'txt=document.getElementById("fc-text");'
        .. 'if(flipped){body.classList.add("is-back");body.classList.remove("is-centered")}'
        .. 'else{body.classList.remove("is-back");body.classList.add("is-centered")}'
        .. '}'
        .. '});'
        .. 'document.getElementById("fc-close-btn").onclick=function(e){e.stopPropagation();root.remove()};'
        .. 'function clamp(){'
        .. 'var rect=root.getBoundingClientRect();'
        .. 'var l=rect.left,t=rect.top;'
        .. 'if(l<0)l=0;if(t<TOP_OFFSET)t=TOP_OFFSET;'
        .. 'if(l+rect.width>window.innerWidth)l=window.innerWidth-rect.width;'
        .. 'if(t+rect.height>window.innerHeight)t=window.innerHeight-rect.height;'
        .. 'root.style.left=l+"px";root.style.top=t+"px";root.style.right="auto";'
        .. '}'
        .. 'document.getElementById("fc-handle").onpointerdown=function(e){'
        .. 'if(e.target.closest("button"))return;'
        .. 'document.body.classList.add("sb-dragging-active");'
        .. 'var sX=e.clientX-root.offsetLeft,sY=e.clientY-root.offsetTop;'
        .. 'function move(m){'
        .. 'var nL=m.clientX-sX,nT=m.clientY-sY;'
        .. 'var w=root.offsetWidth,h=root.offsetHeight;'
        .. 'if(nL<SNAP)nL=0;if(nT<TOP_OFFSET+SNAP)nT=TOP_OFFSET;'
        .. 'if(nL+w>window.innerWidth-SNAP)nL=window.innerWidth-w;'
        .. 'if(nT+h>window.innerHeight-SNAP)nT=window.innerHeight-h;'
        .. 'root.style.left=Math.max(0,Math.min(nL,window.innerWidth-w))+"px";'
        .. 'root.style.top=Math.max(TOP_OFFSET,Math.min(nT,window.innerHeight-h))+"px";'
        .. 'root.style.right="auto";'
        .. '}'
        .. 'function up(){'
        .. 'document.body.classList.remove("sb-dragging-active");'
        .. 'window.removeEventListener("pointermove",move);'
        .. 'window.__fc_pos={top:root.style.top,left:root.style.left};'
        .. '}'
        .. 'window.addEventListener("pointermove",move);'
        .. 'window.addEventListener("pointerup",up,{once:true});'
        .. '};'
        .. 'window.addEventListener("resize",clamp);'
        .. 'render();setTimeout(clamp,50);'
        .. '})();'

    container.appendChild(scriptEl)
end

js.window.addEventListener("beforeunload", function()
    local pos = js.window.__fc_pos
    if pos then
        clientStore.set("fc_pos_top", tostring(pos.top))
        clientStore.set("fc_pos_left", tostring(pos.left))
    end
    local h = js.window.__fc_height
    if h then
        clientStore.set("fc_height", tostring(h))
    end
end)

command.define {
    name = "Flashcards: Start",
    run = function() toggleFlashcards() end
}
```


