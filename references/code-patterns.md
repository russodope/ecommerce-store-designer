# 代码模板库（经过验证的实现）

> 生成 HTML 编辑器时必须使用本文件的核心代码。内部按 **1440px 参考宽度**设计、预览 360px（缩放 0.25）；**导出时缩放到用户指定的出图宽度** `BRAND.exportWidth`（见 SKILL.md 第二部分）。
> 本文件分两层，**务必分清**：

## ⭐ 核心架构：固定机器 vs 板块现做（必读）

| 层 | 内容 | 规则 |
|---|---|---|
| **【固定机器·照抄勿改】** | `BRAND` / `el` / `makeUploadable` / `imgZone` / `mkdb` / 字体 FontFace 加载(`getBlocks`/`ensureFonts`) / `renderMod` / `waitForStableHeight` / `compressCanvas` / `expMod`/`expZip`/`expLong` / 排序 / `toast` / `safeFileName` / CDN 与字体 host | **逐字照抄**。这是导出可靠性的地基，每个细节都是踩坑后的修正，不要简化或省略。 |
| **【板块现做·按概念组合】** | 第 5 节的**组件 kit**：选哪些、什么顺序、各自参数(cfg)，由品牌**概念**决定（见 SKILL.md 第一/二部分） | 鼓励差异化，但**每个板块必须遵守第 6 节"导出安全清单"**。 |

**差异化来自"组合 × 参数化"，不是"每次重新发明 DOM"。** kit 是乐高积木——有限、验证过、导出安全；不同品牌靠不同的组合搭出不同的店。**严禁把某个组件或某套组合当成固定模板套给所有品牌。**

每个板块由一个 **cfg 对象**驱动渲染（`renderSection(cfg)` → 返回一个 `class="db"` 的 DOM）。cfg 是**单一数据源**：文案、已上传图片、样式/布局都在 cfg 里（这样 Phase A 的逐板块配置面板改 cfg 即可重渲染，不丢用户编辑）。

---

## 1. BRAND 配置对象

```js
const BRAND = {
  name: '品牌名', nameEn: 'BRAND',
  // 画布（固定值，不要改）
  canvasWidth: 1440, previewWidth: 360, scale: 0.25,
  // 色彩（按品牌概念/素材调校后的值；可参考 industry-*.md 的配色配方做种子）
  bgLight: '#F8F6F3', bgMid: '#EEEAE4', bgDark: '#1C1814',
  textPrimary: '#1C1814', textLight: '#F8F6F3', accent: '#C5A55A',
  // 字体（设计字体优先 + 同类系统字体兜底；host 用 .cn 镜像，国内可达，见第 9 节）
  fontCN: '"Noto Serif SC","Songti SC","SimSun",serif',
  fontEN: '"Cormorant Garamond",serif',
  googleFontsUrl: 'https://fonts.googleapis.cn/css2?family=Noto+Serif+SC:wght@200;300;400&family=Cormorant+Garamond:ital,wght@0,400;1,400&display=swap',
  webFontCheck: '300 32px "Noto Serif SC"',   // 导出前 iDoc.fonts.check 校验的主 web 字体
  exportWidth: 1440,   // ⭐用户指定的【出图宽度】：内部按 1440 设计，导出时缩放到这个宽度（默认 1440=不缩放）
};
const B = BRAND;
```

---

## 2. 固定机器：基础

```js
function el(t,c,x,ce){const e=document.createElement(t); if(c)e.style.cssText=c; if(x!=null)e.textContent=x; if(ce)e.setAttribute('contenteditable','true'); return e;}
function safeFileName(s){return (s||'m').replace(/[^\w一-龥\-]/g,'_').slice(0,40);}
function mkdb(h,bg){const d=el('div','height:'+h+'px;position:relative;overflow:hidden;background:'+(bg||B.bgLight)+';'); d.className='db'; return d;}

// 图片上传区（zone 必须 position:relative，makeUploadable 的硬前提）
// 可编辑性是第一类要求：编辑入口必须【始终可见】。空图→虚线"点击上传"；已填图→保留"⤓ 换图"角标 + hover 遮罩提示。全部带 class=edit-cue，renderMod 导出时隐藏。
function makeUploadable(zone){
  const ph=el('div','position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:6px;border:3px dashed rgba(0,0,0,.18);pointer-events:none;z-index:5');
  ph.className='upload-placeholder edit-cue';
  ph.appendChild(el('div','font:13px sans-serif;color:rgba(0,0,0,.4)','＋ 点击上传图片'));
  zone.appendChild(ph); zone._ph=ph;
  const badge=el('div','position:absolute;right:10px;top:10px;z-index:7;background:rgba(0,0,0,.55);color:#fff;font:12px sans-serif;padding:4px 10px;border-radius:20px;pointer-events:none;display:none');
  badge.className='edit-cue'; badge.textContent='⤓ 换图'; zone.appendChild(badge); zone._badge=badge;     // 已填图也持久可见
  const hov=el('div','position:absolute;inset:0;z-index:6;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,.35);color:#fff;font:15px sans-serif;opacity:0;transition:opacity .15s;pointer-events:none');
  hov.className='edit-cue'; hov.textContent='点击替换图片'; zone.appendChild(hov);
  const lb=el('label','position:absolute;inset:0;cursor:pointer;z-index:10');
  const ip=document.createElement('input'); ip.type='file'; ip.accept='image/*'; ip.style.cssText='width:0;height:0;opacity:0;position:absolute';
  ip.addEventListener('change',e=>{ if(!e.target.files[0])return; const r=new FileReader();
    r.onload=ev=>{ zone.style.backgroundImage='url('+JSON.stringify(ev.target.result)+')'; zone.style.backgroundSize='cover'; zone.style.backgroundPosition='center'; ph.style.display='none'; badge.style.display='block'; ip.value=''; };
    r.readAsDataURL(e.target.files[0]); });
  lb.onmouseenter=()=>{ if(zone.style.backgroundImage&&zone.style.backgroundImage!=='none') hov.style.opacity='1'; };
  lb.onmouseleave=()=>{ hov.style.opacity='0'; };
  lb.appendChild(ip); zone.appendChild(lb);
}
// 图片区：css 不含 position 时自动补 relative（hero 用绝对铺满，分栏/网格用 flex 子项）
function imgZone(css){ let base=css||'position:absolute;inset:0;'; if(!/position\s*:/.test(base)) base='position:relative;'+base; if(base&&!base.trim().endsWith(';')) base+=';';  // 补尾分号，否则与 background-color 粘连导致最后一个属性(常是 height)失效→图片区0高→不显示
  const z=el('div',base+'background-color:'+B.bgMid+';background-size:cover;background-position:center;'); makeUploadable(z); return z; }
function prefill(z,dataUrl){ if(dataUrl){ z.style.backgroundImage='url('+JSON.stringify(dataUrl)+')'; z.style.backgroundSize='cover'; z.style.backgroundPosition='center'; if(z._ph)z._ph.style.display='none'; if(z._badge)z._badge.style.display='block'; } }

function toast(m,d){let t=document.getElementById('gt'); if(!t){t=el('div','position:fixed;bottom:30px;left:50%;transform:translateX(-50%);background:rgba(0,0,0,.8);color:#fff;padding:10px 22px;border-radius:8px;z-index:99999;font-size:13px');t.id='gt';document.body.appendChild(t);} t.textContent=m;t.style.opacity='1';clearTimeout(t._t);t._t=setTimeout(()=>t.style.opacity='0',d||2500);}
```

### mod-wrapper 高度塌陷修法（transform:scale 不影响布局空间，必须手动设高）
```css
.mod-wrapper{position:relative;width:360px;overflow:hidden}
.mod-wrapper.sel{outline:2px solid #3b6fe0;outline-offset:-2px}   /* 选中态：新模块插它下方 */
.mod-wrapper .db{width:1440px;transform-origin:top left;transform:scale(0.25);display:block}
/* 可见编辑入口（编辑器 chrome，导出时 renderMod 会隐藏 .edit-cue 并清掉 outline）*/
.mod-actions{position:absolute;right:6px;top:3px;z-index:50}
.mod-actions button{background:rgba(0,0,0,.62);color:#fff;border:none;border-radius:5px;padding:3px 9px;font-size:11px;cursor:pointer}
.db [contenteditable]:hover{outline:1.5px dashed #4a90d9;outline-offset:2px;cursor:text}   /* 文字可编辑提示 */
.db [contenteditable]:focus{outline:2px solid #4a90d9;background:rgba(74,144,217,.08)}
```
> 编辑提示放在**左侧面板顶部**（见下方"编辑器外壳"的 `.shints`）；`mod-actions` 放 `⚙ ↑ ↓ ✕`。**三类编辑入口（文字/图片/版式）必须始终可见**，否则用户以为是改不了的成品。
```js
function updateWH(wrap,db){const h=db.scrollHeight||db.offsetHeight; if(h>0)wrap.style.height=Math.ceil(h*0.25)+'px';
  requestAnimationFrame(()=>{const h2=db.scrollHeight; if(h2>0)wrap.style.height=Math.ceil(h2*0.25)+'px';});}
```

### 编辑器外壳：左侧"添加模块"面板 + 干净画布（**不要手机外框**）

> 手机黑外框 + 限高会让画布"有黑边、显示不全"。**改为：左侧深色"添加模块"面板（点模块即添加）+ 干净白画布（整页可滚动、完整显示）**。
```css
#side{position:fixed;left:0;top:0;width:236px;height:100vh;background:#1c1c1c;color:#cfcfcf;overflow:auto}
.stitle{padding:16px 16px 6px;font-size:15px;font-weight:600;color:#fff} .shints{padding:0 16px 14px;color:#8a8a8a;font-size:11px;line-height:1.95;border-bottom:1px solid #333}
.sgrp{padding:14px 16px 6px;color:#777;font-size:11px;letter-spacing:2px}
.palette .row{display:flex;align-items:center;gap:10px;padding:9px 8px;border-radius:7px;cursor:pointer} .palette .row:hover{background:#2b2b2b} .palette .nm{flex:1;color:#e2e2e2;font-size:13px} .palette .sz{color:#6f6f6f;font-size:11px}
.exps button{display:block;width:100%;margin-bottom:8px;padding:11px;border:none;border-radius:8px;cursor:pointer} .exps .primary{background:#3b6fe0;color:#fff} .exps .ghost{background:#2b2b2b;color:#ddd}
#main{margin-left:236px;padding:34px 24px;display:flex;justify-content:center;min-height:100vh}
#preview{width:360px;background:#fff;border:1px solid #dcdcdc;border-radius:8px;box-shadow:0 8px 36px rgba(0,0,0,.12);overflow:hidden;align-self:flex-start} /* 干净画布：无黑框、不限高=完整显示 */
```
```html
<div id="side">
  <div class="stitle">品牌名 <small>编辑器</small></div>
  <div class="shints">✏️ 文字可点击编辑<br>🖼️ 图片区点击/悬停换图<br>⚙ 模块右上调版式<br>🖱️ 从下方拖模块到画布指定位置（或点击加末尾）<br>导出 1440px PNG</div>
  <div class="sgrp">添加模块</div>
  <div class="palette" id="palette"><!-- 每组件一行：图标+名称+×高度，data-i=索引 --></div>
  <div class="exps"><button class="primary" onclick="expZip()">⬇ 打包下载 ZIP</button><button class="ghost" onclick="expLong()">⬇ 导出合并长图</button></div>
  <div class="sfoot">导出尺寸：<出图宽度>px 宽 / 格式 PNG / 上传到你店铺的对应模块</div>
</div>
<div id="main"><div id="preview"></div></div>
```
```js
// 画布初始铺好"按 concept 推荐的差异化组合"；左侧 palette 点击=【插在当前选中模块下方】(没选中才追加末尾)
let _seq=0,_sel=null;
function selectWrap(w){document.querySelectorAll('.mod-wrapper.sel').forEach(x=>x.classList.remove('sel'));if(w){w.classList.add('sel');_sel=w;}}
function addModuleToCanvas(m, after){ const vp=document.getElementById('preview'); const db=KIT[m.type](m);
  const wrap=el('div'); wrap.className='mod-wrapper'; wrap.id='mod'+(_seq++); wrap.dataset.name=(m.name||m.type);
  const lab=el('div'); lab.className='mod-label'; lab.appendChild(el('span',null,m.name||m.type));
  lab.title='点击选中（新加模块会插在它下方）'; lab.style.cursor='pointer'; lab.onclick=()=>selectWrap(wrap);
  const act=el('div'); act.className='mod-actions'; const panel=makeConfigPanel(wrap,db);
  const bCfg=el('button',null,'⚙'); bCfg.onclick=()=>panel.style.display=panel.style.display==='none'?'block':'none';
  const bUp=el('button',null,'↑'); bUp.onclick=()=>{if(wrap.previousElementSibling)vp.insertBefore(wrap,wrap.previousElementSibling);};
  const bDn=el('button',null,'↓'); bDn.onclick=()=>{if(wrap.nextElementSibling)vp.insertBefore(wrap.nextElementSibling,wrap);};
  const bDel=el('button',null,'✕'); bDel.onclick=()=>{if(confirm('删除此模块？')){if(_sel===wrap)_sel=null;wrap.remove();}};
  act.append(bCfg,bUp,bDn,bDel);
  const sz=el('div'); sz.className='mod-size'; wrap.append(lab,act,panel,db,sz);
  if(after&&after.parentNode===vp) vp.insertBefore(wrap, after.nextSibling); else vp.appendChild(wrap);   // 指定位置插入
  updateWH(wrap,db);
  setTimeout(()=>{ if(db.scrollHeight>db.clientHeight+1)db.style.height=db.scrollHeight+'px'; updateWH(wrap,db); const H=db.scrollHeight||db.offsetHeight; const W=B.exportWidth||1440; sz.textContent=W+'×'+Math.round(H*W/1440)+'px';},350); return wrap; }
MODULES.forEach(m=>addModuleToCanvas(m));   // 初始铺好概念组合
// 左侧 palette → 拖到画布指定位置插入（dragover 显示落点蓝线）；点击=加到末尾
(function(){ const vp=document.getElementById('preview');
  const line=el('div','height:4px;background:#3b6fe0;border-radius:2px;margin:2px 0;display:none'); line.id='dropline';
  const refBelow=y=>{for(const w of vp.querySelectorAll('.mod-wrapper')){const r=w.getBoundingClientRect();if(y<r.top+r.height/2)return w;}return null;};
  document.querySelectorAll('#palette .row').forEach(r=>{ r.draggable=true; r.style.cursor='grab';
    r.addEventListener('dragstart',e=>e.dataTransfer.setData('text/plain',r.dataset.i));
    r.onclick=()=>addModuleToCanvas(MODULES[+r.dataset.i]).scrollIntoView({behavior:'smooth',block:'center'}); });
  vp.addEventListener('dragover',e=>{e.preventDefault();vp.insertBefore(line,refBelow(e.clientY));line.style.display='block';});
  vp.addEventListener('dragleave',e=>{if(!vp.contains(e.relatedTarget))line.style.display='none';});
  vp.addEventListener('drop',e=>{e.preventDefault();const idx=e.dataTransfer.getData('text/plain');line.style.display='none';if(line.parentNode)line.remove();if(idx==='')return;const ref=refBelow(e.clientY);const w=addModuleToCanvas(MODULES[+idx]);if(ref)vp.insertBefore(w,ref);w.scrollIntoView({behavior:'smooth',block:'center'});});
})();
```

### 模块排序 / 删除
```js
function moveModule(id,dir){const s=document.getElementById(id),p=s.parentNode;
  if(dir==='up'&&s.previousElementSibling)p.insertBefore(s,s.previousElementSibling);
  else if(dir==='down'&&s.nextElementSibling)p.insertBefore(s.nextElementSibling,s); renumber();}
function deleteModule(id){const s=document.getElementById(id); if(s&&confirm('删除此模块？')){s.remove();renumber();}}
function renumber(){document.querySelectorAll('.mod-wrapper').forEach((m,i)=>{const l=m.querySelector('.mod-label span'); if(l)l.textContent=(i+1)+'. '+m.dataset.name;});}
```

---

## 3. 固定机器：字体加载 + 渲染导出（关键，已实测）

> **为什么用 FontFace 而不是只靠 `<link>`**：实测 html2canvas 在部分浏览器**不会**把 `<link>` 注入的跨域 web 字体烤进导出图（中文会变回兜底字体且无报错）。把字体用 **FontFace 注册进 iframe 的 iDoc** 才能稳定烤进。CJK 是 unicode-range 分片，只加载文案命中的分片，很轻。

```js
let _blk=null,_buf={};
async function getBlocks(){ if(_blk)return _blk; const css=await (await fetch(B.googleFontsUrl)).text();
  _blk=css.split('@font-face').slice(1).map(b=>({
    family:(b.match(/font-family:\s*'([^']+)'/)||[])[1], weight:(b.match(/font-weight:\s*([^;]+);/)||[])[1]||'400',
    style:(b.match(/font-style:\s*([^;]+);/)||[])[1]||'normal', range:(b.match(/unicode-range:\s*([^;]+);/)||[])[1]||'',
    src:(b.match(/url\((https:\/\/[^)]+\.woff2)\)/)||[])[1]
  })).filter(b=>b.src&&b.range&&b.family); return _blk; }
function covers(ur,cp){return ur.split(',').some(p=>{p=p.trim().replace(/^U\+/i,'');
  if(p.includes('-')){const[a,b]=p.split('-').map(h=>parseInt(h,16));return cp>=a&&cp<=b;}
  if(p.includes('?')){const a=parseInt(p.replace(/\?/g,'0'),16),b=parseInt(p.replace(/\?/g,'f'),16);return cp>=a&&cp<=b;}
  return cp===parseInt(p,16);});}
// 把文案需要的字体分片用 iframe 自己的 FontFace 构造器注册进 iDoc（跨 realm，必须用 iWin.FontFace）
async function ensureFonts(iDoc,iWin,text){ const blocks=await getBlocks();
  const chars=[...new Set((text||'').replace(/\s/g,'').split(''))]; if(!chars.length)return;
  for(const b of blocks){ if(chars.some(c=>covers(b.range,c.codePointAt(0)))){ try{
    if(!_buf[b.src])_buf[b.src]=await (await fetch(b.src)).arrayBuffer();
    const ff=new iWin.FontFace(b.family,_buf[b.src],{weight:b.weight,style:b.style,unicodeRange:b.range});
    await ff.load(); iDoc.fonts.add(ff);
  }catch(e){} } } }

async function waitForStableHeight(elm,to){to=to||1500;return new Promise(r=>{let l=0,c=0,s=Date.now();
  const ck=()=>{const h=elm.scrollHeight; if(h>0&&h===l){c++;if(c>=3){r(h);return;}}else{c=0;l=h;}
    if(Date.now()-s>to){r(h);return;} setTimeout(ck,80);}; setTimeout(ck,100);});}
function compressCanvas(cv,mx){mx=mx||2*1024*1024; const png=cv.toDataURL('image/png'); if(png.length*0.75<=mx)return{url:png,ext:'png'};
  for(const q of[0.97,0.94,0.9,0.85,0.8,0.72]){const j=cv.toDataURL('image/jpeg',q); if(j.length*0.75<=mx)return{url:j,ext:'jpg'};}
  return{url:cv.toDataURL('image/jpeg',0.72),ext:'jpg'};}

// renderMod：iframe 隔离 + FontFace 烤字 + 高度稳定检测 + 1440 截图
async function renderMod(id){
  const shell=document.getElementById(id); const block=shell.querySelector('.db'); if(!block)return null;
  const actions=shell.querySelector('.mod-actions'); if(actions)actions.style.visibility='hidden';
  // 提取同源样式（跨域 sheet 的 cssRules 会抛错，try 兜住）
  const allStyles=Array.from(document.styleSheets).map(ss=>{try{return Array.from(ss.cssRules).map(r=>r.cssText).join('\n');}catch(e){return '';}}).join('\n');
  const ifr=el('iframe','position:fixed;top:-9999px;left:-9999px;width:1440px;height:1px;border:none;z-index:-1'); document.body.appendChild(ifr);
  const iDoc=ifr.contentDocument,iWin=ifr.contentWindow;
  iDoc.open();
  iDoc.write('<!DOCTYPE html><html><head><link href="'+B.googleFontsUrl+'" rel="stylesheet"><style>html,body{margin:0;padding:0;width:1440px;overflow:visible}'+allStyles+
    ' .db{transform:none!important;width:1440px!important}.mod-actions{display:none!important}.upload-placeholder{display:none!important}.cfg-panel{display:none!important}.edit-cue{display:none!important}[contenteditable]{outline:none!important}</style></head><body>'+block.outerHTML+'</body></html>');
  iDoc.close();
  await ensureFonts(iDoc,iWin,block.textContent);            // FontFace 注册进 iDoc（保证烤字）
  try{ await iDoc.fonts.ready; }catch(e){ await new Promise(r=>setTimeout(r,400)); }
  const cjk=((block.textContent||'').match(/[一-龥]/g)||[]).join('');
  try{ if(cjk) await iDoc.fonts.load(B.webFontCheck,cjk); }catch(e){}   // 显式加载主字体（消除竞态/分片误判）
  if(cjk && iDoc.fonts && !iDoc.fonts.check(B.webFontCheck,cjk)){ console.warn('字体未就位'); toast('⚠ 字体可能未加载完成，导出字体可能不准'); }
  const inner=iDoc.querySelector('.db'); await waitForStableHeight(inner);
  const realH=inner.scrollHeight; ifr.style.height=realH+'px'; await new Promise(r=>setTimeout(r,80));
  let cv=null;
  try{ cv=await html2canvas(inner,{scale:1,useCORS:true,allowTaint:true,backgroundColor:null,logging:false,width:1440,height:realH,windowWidth:1440,windowHeight:realH}); }
  catch(e){ console.error('renderMod',e); }
  document.body.removeChild(ifr); if(actions)actions.style.visibility='';
  // ⭐导出归一化：强制把成图宽度拉到 exportWidth。用 cv.width!==exportWidth 判断（不是 !==1440）——
  // 某些环境(浏览器缩放改 devicePixelRatio 等)下 html2canvas 返回的宽度可能 ≠1440，必须兜底纠正；高度按 canvas 自身比例算。
  if(cv && B.exportWidth && cv.width!==B.exportWidth){ const oc=el('canvas'); oc.width=B.exportWidth; oc.height=Math.round(cv.height*B.exportWidth/cv.width); oc.getContext('2d').drawImage(cv,0,0,oc.width,oc.height); cv=oc; }
  return cv;
}

async function expMod(id){ const lab=document.getElementById(id).querySelector('.mod-label').textContent; toast('⏳ 渲染中…');
  const cv=await renderMod(id); if(!cv){toast('渲染失败');return;} const r=compressCanvas(cv);
  const a=el('a'); a.download=safeFileName(B.name)+'_'+safeFileName(lab)+'_1440px.'+r.ext; a.href=r.url; a.click(); toast('✓ 已导出 '+cv.width+'×'+cv.height); }
async function expZip(){ const mods=[...document.querySelectorAll('.mod-wrapper')]; if(!mods.length){toast('还没有模块');return;}
  const zip=new JSZip(); for(let i=0;i<mods.length;i++){ toast('⏳ 渲染 '+(i+1)+'/'+mods.length+' …',60000); const lab=(mods[i].querySelector('.mod-label').textContent||'').replace(/^\d+\.\s*/,''); const cv=await renderMod(mods[i].id); if(!cv)continue; const r=compressCanvas(cv);
    zip.file(String(i+1).padStart(2,'0')+'_'+safeFileName(lab)+'.'+r.ext, r.url.split(',')[1], {base64:true}); await new Promise(s=>setTimeout(s,150)); }
  toast('📦 打包中…',60000); const blob=await zip.generateAsync({type:'blob'}); const a=el('a'); a.download=safeFileName(B.name)+'_店铺模块.zip'; a.href=URL.createObjectURL(blob); a.click(); toast('✓ ZIP 完成，共 '+mods.length+' 张'); }
async function expLong(){ const mods=[...document.querySelectorAll('.mod-wrapper')]; if(!mods.length){toast('还没有模块');return;}
  const cvs=[]; for(let i=0;i<mods.length;i++){ toast('⏳ 渲染 '+(i+1)+'/'+mods.length+' …',60000); const c=await renderMod(mods[i].id); if(c)cvs.push(c);} toast('🧵 拼接长图…',60000); const H=cvs.reduce((s,c)=>s+c.height,0);
  const mg=el('canvas'); mg.width=1440; mg.height=H; const x=mg.getContext('2d'); let y=0; for(const c of cvs){x.drawImage(c,0,y);y+=c.height;}
  const r=compressCanvas(mg,10*1024*1024); const a=el('a'); a.download=safeFileName(B.name)+'_首页长图.'+r.ext; a.href=r.url; a.click(); toast('✓ 长图 1440×'+H); }
```

---

## 4. cfg / renderSection 约定

每个组件是 `fn(cfg)`：读 cfg → 返回 `class="db"` 的 DOM。**渲染所需的一切都在 cfg 里**（单一数据源），便于增删/排序/Phase A 配置面板重渲染而不丢编辑。

```js
// cfg 通用字段（按组件取用）
// { type, name,                      // 组件名 / 显示名
//   h, bg,                           // 高度(200–2000)、背景色
//   img / items[].img,              // 图片 dataURL（用户上传后写回这里，单一数据源）
//   titleEN/titleCN/sub/desc/no/kicker/label/big/quoteEN/quoteCN, // 文案（用户编辑后写回）
//   side:'left'|'right', cols, ratio, size, weight, align, dark } // 版式参数
```
> ⚠️ 导出契约（renderMod 提取的是**全局 styleSheets + `.db` 的 outerHTML**，与 cfg 无关）：组件根节点必须 `class="db"`；样式只走**全局 `<style>` 或 inline**，**禁止 `insertRule`/构造式 stylesheet**（否则 renderMod 提取不到，导出错乱）。

---

## 5. 组件 kit（按概念组合，勿整套照搬）

> 这是一套**导出安全的积木**。不同品牌**选不同的组件 + 不同顺序 + 不同 cfg** 搭出不同的店。要鼓励：① 用对方没有的"专属组件"制造结构差异（hero 用哪种、要不要 specBlock/swatchStrip/editorialAsym…）；② 至少让 2 个板块的结构按品牌概念定制。**已实测：同品类两品牌用不同组合，独立盲评判为"不同设计系统"。**

```js
function heroFull(o){const db=mkdb(o.h,o.bg||B.bgDark);const z=imgZone();prefill(z,o.img);db.appendChild(z);
  db.appendChild(el('div','position:absolute;inset:0;background:linear-gradient(180deg,rgba(0,0,0,.12),transparent 40%,transparent 50%,rgba(0,0,0,.62));z-index:2'));
  const t=el('div','position:absolute;left:100px;right:100px;bottom:130px;z-index:5;color:'+B.textLight);
  t.appendChild(el('div','font-family:'+B.fontEN+';font-size:128px;font-weight:500;line-height:1',o.titleEN,1));
  t.appendChild(el('div','font-family:'+B.fontCN+';font-size:46px;font-weight:300;margin-top:20px;letter-spacing:6px',o.titleCN,1));
  if(o.sub)t.appendChild(el('div','font-family:'+B.fontEN+';font-size:24px;letter-spacing:5px;margin-top:24px;opacity:.85',o.sub,1));
  db.appendChild(t);return db;}

function heroMinimal(o){const db=mkdb(o.h,o.bg||B.bgLight);
  const w=el('div','position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:120px 0');
  w.appendChild(el('div','font-family:'+B.fontEN+';font-size:30px;letter-spacing:16px;color:'+B.textPrimary+';opacity:.7;text-transform:uppercase',o.kicker||'',1));
  const z=imgZone('width:54%;height:60%;margin-top:46px');prefill(z,o.img);w.appendChild(z);
  w.appendChild(el('div','font-family:'+B.fontEN+';font-style:italic;font-size:84px;color:'+B.textPrimary+';margin-top:46px',o.titleEN,1));
  w.appendChild(el('div','font-family:'+B.fontCN+';font-size:30px;letter-spacing:8px;color:'+B.textPrimary+';margin-top:12px;opacity:.7',o.titleCN,1));
  db.appendChild(w);return db;}

function heroSplit(o){const db=mkdb(o.h,o.bg||B.bgLight);db.style.display='flex';
  const L=el('div','flex:0 0 46%;display:flex;flex-direction:column;justify-content:center;padding:0 90px;color:'+B.textPrimary);
  L.appendChild(el('div','font-family:'+B.fontEN+';font-size:26px;letter-spacing:8px;opacity:.6',o.kicker||'',1));
  L.appendChild(el('div','font-family:'+B.fontEN+';font-size:92px;font-weight:500;line-height:1.05;margin-top:22px',o.titleEN,1));
  L.appendChild(el('div','font-family:'+B.fontCN+';font-size:34px;letter-spacing:6px;margin-top:18px;opacity:.8',o.titleCN,1));
  const z=imgZone('flex:1;align-self:stretch');prefill(z,o.img);db.append(L,z);return db;}

function featureSplit(o){const db=mkdb(o.h,o.bg);db.style.display='flex';db.style.flexDirection=(o.side==='right'?'row-reverse':'row');
  const tc=o.dark?B.textLight:B.textPrimary;const z=imgZone('flex:0 0 '+(o.ratio||58)+'%;align-self:stretch');prefill(z,o.img);db.appendChild(z);
  const p=el('div','flex:1;display:flex;flex-direction:column;justify-content:center;padding:0 80px;color:'+tc);
  if(o.no)p.appendChild(el('div','font-family:'+B.fontEN+';font-size:120px;font-weight:300;line-height:1;opacity:.45',o.no));
  p.appendChild(el('div','font-family:'+B.fontEN+';font-size:64px;margin-top:8px;'+(o.dark?'font-style:italic;':''),o.nameEN,1));
  p.appendChild(el('div','font-family:'+B.fontCN+';font-size:40px;font-weight:300;margin-top:14px;letter-spacing:6px',o.nameCN,1));
  if(o.desc)p.appendChild(el('div','font-family:'+B.fontCN+';font-size:26px;line-height:2.1;margin-top:30px;opacity:.85',o.desc,1));
  db.appendChild(p);return db;}

function editorialAsym(o){const db=mkdb(o.h,o.bg||B.bgLight);const side=o.side||'left';
  const z=imgZone('position:absolute;top:80px;'+(side==='left'?'left:0;':'right:0;')+'width:56%;height:'+(o.h-160)+'px');prefill(z,o.img);db.appendChild(z);
  const t=el('div','position:absolute;'+(side==='left'?'left:60%;right:60px;':'right:60%;left:60px;')+'top:50%;transform:translateY(-50%);color:'+B.textPrimary);  // 图56%+文字另占~40%，不重叠
  if(o.no)t.appendChild(el('div','font-family:'+B.fontEN+';font-size:128px;font-weight:300;line-height:1;opacity:.5',o.no));
  t.appendChild(el('div','font-family:'+B.fontEN+';font-style:italic;font-size:60px;margin-top:6px',o.nameEN,1));
  t.appendChild(el('div','font-family:'+B.fontCN+';font-size:32px;letter-spacing:6px;margin-top:14px',o.nameCN,1));
  if(o.desc)t.appendChild(el('div','font-family:'+B.fontCN+';font-size:23px;line-height:2.1;margin-top:22px;opacity:.8',o.desc,1));
  db.appendChild(t);return db;}

function grid(o){const pad=70,gap=24,cols=o.cols;const cw=Math.floor((1440-pad*2-gap*(cols-1))/cols);const db=mkdb(Math.max(o.h||0,(o.title?110:60)+Math.round(cw*1.2)+106),o.bg);  // 按内容算高度，避免裁切/白边
  if(o.title)db.appendChild(el('div','position:absolute;left:'+pad+'px;top:40px;font-family:'+B.fontEN+';font-size:30px;letter-spacing:6px;color:'+B.textPrimary,o.title,1));
  const w=el('div','position:absolute;left:'+pad+'px;right:'+pad+'px;top:'+(o.title?110:60)+'px;display:flex;gap:'+gap+'px');
  o.items.forEach(it=>{const c=el('div','width:'+cw+'px');const z=imgZone('width:'+cw+'px;height:'+Math.round(cw*1.2)+'px');prefill(z,it.img);c.appendChild(z);c.appendChild(el('div','font-family:'+B.fontEN+';font-size:24px;margin-top:14px;letter-spacing:2px;color:'+B.textPrimary,it.label,1));w.appendChild(c);});
  db.appendChild(w);return db;}

function gridMasonry(o){const pad=70,gap=24;const cw=Math.floor((1440-pad*2-gap)/2);const _ih=it=>(it.tall?Math.round(cw*1.32):Math.round(cw*0.92))+46;const _ch=(a,mt)=>mt+a.reduce((s,it)=>s+_ih(it),0)+Math.max(0,a.length-1)*gap;const db=mkdb(Math.max(o.h||0,80+Math.max(_ch(o.items.filter((_,i)=>i%2===0),0),_ch(o.items.filter((_,i)=>i%2===1),80))+70),o.bg);  // 按内容算高度，避免裁切/白边
  const w=el('div','position:absolute;left:'+pad+'px;right:'+pad+'px;top:80px;display:flex;gap:'+gap+'px');
  [0,1].forEach(ci=>{const col=el('div','width:'+cw+'px;display:flex;flex-direction:column;gap:'+gap+'px;'+(ci===1?'margin-top:80px':''));
    o.items.filter((_,i)=>i%2===ci).forEach(it=>{const z=imgZone('width:'+cw+'px;height:'+(it.tall?Math.round(cw*1.32):Math.round(cw*0.92))+'px');prefill(z,it.img);const c=el('div');c.appendChild(z);c.appendChild(el('div','font-family:'+B.fontEN+';font-size:22px;margin-top:12px;color:'+B.textPrimary,it.label,1));col.appendChild(c);});
    w.appendChild(col);});
  db.appendChild(w);return db;}

function statement(o){const db=mkdb(o.h,o.bg||B.bgDark);db.style.display='flex';db.style.flexDirection='column';db.style.alignItems='center';db.style.justifyContent='center';
  const tc=o.dark?B.textLight:B.textPrimary;
  db.appendChild(el('div','font-family:'+B.fontEN+';font-size:'+(o.size||160)+'px;font-weight:'+(o.weight||600)+';letter-spacing:2px;color:'+tc+';text-align:center;line-height:1.05;white-space:pre-line;padding:0 80px',o.big,1));
  if(o.sub)db.appendChild(el('div','font-family:'+B.fontCN+';font-size:30px;letter-spacing:10px;margin-top:30px;color:'+tc+';opacity:.8',o.sub,1));
  return db;}

function specBlock(o){const db=mkdb(o.h,o.bg||B.bgLight);
  if(o.title)db.appendChild(el('div','position:absolute;left:70px;top:48px;font-family:'+B.fontEN+';font-size:28px;letter-spacing:6px;color:'+B.textPrimary,o.title,1));
  const pad=70,gap=44,cols=o.items.length;const cw=Math.floor((1440-pad*2-gap*(cols-1))/cols);
  const w=el('div','position:absolute;left:'+pad+'px;right:'+pad+'px;top:'+(o.title?160:90)+'px;display:flex;gap:'+gap+'px');
  o.items.forEach(it=>{const c=el('div','width:'+cw+'px;color:'+B.textPrimary);
    c.appendChild(el('div','font-family:'+B.fontEN+';font-size:96px;font-weight:500;line-height:1;color:'+B.accent,it.num,1));
    c.appendChild(el('div','font-family:'+B.fontCN+';font-size:30px;font-weight:500;margin-top:16px',it.label,1));
    c.appendChild(el('div','font-family:'+B.fontCN+';font-size:22px;line-height:1.9;margin-top:12px;opacity:.75',it.desc,1));w.appendChild(c);});
  db.appendChild(w);return db;}

function swatchStrip(o){const db=mkdb(o.h,o.bg||B.bgLight);db.style.display='flex';db.style.flexDirection='column';db.style.justifyContent='center';db.style.padding='0 70px';
  if(o.title)db.appendChild(el('div','font-family:'+B.fontEN+';font-size:28px;letter-spacing:6px;color:'+B.textPrimary+';margin-bottom:38px',o.title,1));
  const row=el('div','display:flex;gap:0');
  o.swatches.forEach(s=>{const c=el('div','flex:1;display:flex;flex-direction:column;align-items:center');c.appendChild(el('div','width:100%;height:'+(o.sw||190)+'px;background:'+s.color));c.appendChild(el('div','font-family:'+B.fontEN+';font-size:20px;margin-top:16px;color:'+B.textPrimary,s.name,1));row.appendChild(c);});
  db.appendChild(row);return db;}

function lookbookRow(o){const db=mkdb(o.h,o.bg||B.bgLight);
  const w=el('div','position:absolute;inset:0;display:flex;align-items:center;gap:30px;padding:0 70px');
  o.items.forEach(it=>{const c=el('div','flex:1;height:'+(o.h-150)+'px;display:flex;flex-direction:column');const z=imgZone('flex:1');prefill(z,it.img);c.appendChild(z);c.appendChild(el('div','font-family:'+B.fontEN+';font-style:italic;font-size:26px;margin-top:14px;color:'+B.textPrimary,it.label,1));w.appendChild(c);});
  db.appendChild(w);return db;}

function transition(o){const db=mkdb(o.h,o.bg||B.bgMid);db.style.display='flex';db.style.alignItems='center';db.style.justifyContent='center';
  if(o.big)db.appendChild(el('div','position:absolute;inset:0;display:flex;align-items:center;justify-content:center;font-family:'+B.fontEN+';font-size:300px;font-weight:700;color:'+B.textPrimary+';opacity:.06;letter-spacing:12px;white-space:nowrap',o.big));
  const w=el('div','position:relative;z-index:2;display:flex;align-items:center;gap:28px');
  w.appendChild(el('div','width:80px;height:1px;background:'+B.textPrimary+';opacity:.4'));
  w.appendChild(el('div','font-family:'+B.fontCN+';font-size:36px;letter-spacing:10px;color:'+B.textPrimary,o.label,1));
  w.appendChild(el('div','width:80px;height:1px;background:'+B.textPrimary+';opacity:.4'));
  db.appendChild(w);return db;}

function story(o){const db=mkdb(o.h,o.bg||B.bgDark);const z=imgZone();prefill(z,o.img);db.appendChild(z);
  db.appendChild(el('div','position:absolute;inset:0;background:rgba(0,0,0,.52);z-index:2'));
  const t=el('div','position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;z-index:3;color:'+B.textLight+';text-align:center;padding:0 120px');
  t.appendChild(el('div','font-family:'+B.fontEN+';font-style:italic;font-size:62px;line-height:1.35',o.quoteEN,1));
  t.appendChild(el('div','font-family:'+B.fontCN+';font-size:38px;letter-spacing:8px;margin-top:26px;font-weight:300',o.quoteCN,1));
  db.appendChild(t);return db;}

const KIT={heroFull,heroMinimal,heroSplit,featureSplit,editorialAsym,grid,gridMasonry,statement,specBlock,swatchStrip,lookbookRow,transition,story};
// ⚠️ 组装/添加模块的【唯一入口】是前文"编辑器外壳"段的 `addModuleToCanvas`（含拖拽插入+选中+配置面板+↑↓✕），
//    `mod-label/mod-size/mod-actions` 的样式也都在"编辑器外壳"那段 CSS 里。**不要另写 buildAll**，直接用 addModuleToCanvas。
```

### 配置面板（Phase A：逐板块排版/布局）

> **设计原则**：面板控件**直接改 `.db` 里的 DOM/inline 样式**，不重建组件——所以①用户已编辑的文字、已上传的图**不会丢**；②`renderMod` 截的就是改后的 `.db`，**导出自动带上**配置。面板挂在 `.db` 外的 shell 层（`class="cfg-panel"`，renderMod 已加规则隐藏它，不进导出）。

```js
function makeConfigPanel(wrap,db){
  const p=el('div','position:absolute;left:0;top:20px;z-index:60;width:184px;background:#fff;border:1px solid #ccc;border-radius:6px;padding:8px;font:11px sans-serif;box-shadow:0 4px 16px rgba(0,0,0,.15);display:none');
  p.className='cfg-panel';
  const texts=()=>[...db.querySelectorAll('[contenteditable]')];
  const row=(label,ctrl)=>{const r=el('div','display:flex;align-items:center;justify-content:space-between;margin:5px 0;gap:6px');r.appendChild(el('span',null,label));r.appendChild(ctrl);p.appendChild(r);return ctrl;};
  // 高度
  const h=el('input','width:74px');h.type='number';h.value=parseInt(db.style.height)||db.offsetHeight;
  h.oninput=()=>{const v=Math.max(200,Math.min(2000,+h.value||200));db.style.height=v+'px';updateWH(wrap,db);}; row('高度',h);
  // 背景色
  const bg=el('input');bg.type='color'; bg.oninput=()=>{db.style.background=bg.value;}; row('背景',bg);
  // 主文字色
  const fc=el('input');fc.type='color'; fc.oninput=()=>{texts().forEach(t=>t.style.color=fc.value);}; row('文字色',fc);
  // 字号缩放（相对各自原始字号）
  const fs=el('input','width:84px');fs.type='range';fs.min='0.7';fs.max='1.4';fs.step='0.05';fs.value='1';
  fs.oninput=()=>{texts().forEach(t=>{if(!t.dataset.fs0)t.dataset.fs0=parseFloat(getComputedStyle(t).fontSize);t.style.fontSize=(t.dataset.fs0*fs.value)+'px';});updateWH(wrap,db);}; row('字号',fs);
  // 对齐
  const al=el('select');['left','center','right'].forEach(v=>{const o=document.createElement('option');o.value=v;o.textContent=v;al.appendChild(o);});
  al.onchange=()=>{texts().forEach(t=>t.style.textAlign=al.value);}; row('对齐',al);
  // 图文比 + 镜像（仅左右分栏类组件）
  if(db.style.display==='flex'){   // inline 判断（此时 db 可能尚未挂入文档，getComputedStyle 读不到）
    const zone=[...db.children].find(c=>c.style.backgroundSize==='cover');
    if(zone){const rt=el('input','width:84px');rt.type='range';rt.min='30';rt.max='70';rt.value='58';
      rt.oninput=()=>{zone.style.flex='0 0 '+rt.value+'%';}; row('图文比',rt);
      const mir=el('button','width:100%;margin-top:6px',null);mir.textContent='⇄ 镜像左右';
      mir.onclick=()=>{db.style.flexDirection=(db.style.flexDirection==='row-reverse'?'row':'row-reverse');}; p.appendChild(mir);}
  }
  return p;
}
```

> 可按品牌扩充新组件，但**新组件必须过第 6 节导出安全清单**。`mod-actions` 与（Phase A 的）`cfg-panel` 都挂在 `.db` **外**的 shell 层（renderMod 已对二者加了隐藏规则）。

---

## 6. 导出安全清单（现做/扩充组件必须遵守）

- 根节点 `class="db"`；固定高度 `200–2000px`（超 2000 拆成多张）。
- 图片区一律 `background-image`（**不要 `<img>`**，html2canvas 不支持 object-fit）；用 `imgZone()` 接 `makeUploadable`。
- **禁止**高级 SVG（`mask`/`filter`/`foreignObject`）、`object-fit`、`backdrop-filter`、`insertRule`/构造式 stylesheet。
- 文字节点加 `contenteditable`；文案/图片写回 cfg（单一数据源）。
- 颜色取自 `BRAND`；不要用外部图片 URL 占位（不可控且会 taint canvas）。
- **全页签名装饰层**（可选的视觉串联，如胶片颗粒/暖晕/品牌水印）：用**纯 CSS** `radial-gradient`/`linear-gradient` + `mix-blend-mode`（实测 html2canvas 安全），绝对定位铺满、`pointer-events:none`，放在 `.db` 内（会进导出）。**勿用** SVG `filter` 噪点。例：`el('div','position:absolute;inset:0;z-index:3;pointer-events:none;opacity:.06;background:radial-gradient(circle at 30% 20%,#000,transparent 60%);mix-blend-mode:multiply')`。

---

## 7. 必须引入的外部库（国内可达 CDN）
```html
<!-- 主：BootCDN；备选：cdn.staticfile.org -->
<script src="https://cdn.bootcdn.net/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<script src="https://cdn.bootcdn.net/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
```

## 8. 禁止事项速查
| 禁止 | 正确做法 |
|---|---|
| `innerHTML +=` | `createElement` + `appendChild` |
| `<img>` 展示可替换图片 | `background-image` + `background-size:cover` |
| 只靠 `<link>` 等字体就导出 | `ensureFonts` 把 FontFace 注册进 iDoc + `fonts.load` + `check` |
| 库用 cdnjs、字体用 `fonts.googleapis.com`（国内被墙） | BootCDN/staticfile + `fonts.googleapis.cn` 镜像（第 9 节） |
| 复杂 inline SVG（mask/filter/foreignObject）做装饰 | 简单 `path`/基础形状/纯 CSS |
| 模块高度 <200 或 >2000px | 200–2000，超出拆图 |
| 把某个组件/组合当固定模板套所有品牌 | 按概念重新组合（第 5 节） |

## 9. 国内网络可用性
- 字体 host 用 `fonts.googleapis.cn`（CSS 与字体文件都落 `.cn` 域；CJK 走 unicode-range 分片 woff2，短文案只下命中分片）。多 family 用 `&family=` 连接（裸 `&` 会丢第二个 family）。
- 字体栈三段：设计字体 → 同类系统字体 → 通用。serif CJK → `…,"Songti SC","SimSun",serif`；sans CJK → `…,"PingFang SC","Microsoft YaHei",sans-serif`；装饰体(ZCOOL/Ma Shan Zheng 无系统等价) → 接 `"PingFang SC"`/`"Kaiti SC"`。
- 导出烤字依赖 `ensureFonts`（FontFace），不要退回只用 `<link>`。
