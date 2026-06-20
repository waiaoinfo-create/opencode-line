# Skill: GitHub Pages Slide Presentation

## Description
當使用者要求「上傳 GitHub」「開分支」「做報告/簡報/報表」時，遵循此流程建立 GitHub Pages 幻燈片網站。

## Workflow

### Step 1: 建立 Git 倉庫（如尚未初始化）
```bash
git init
git checkout -b main
```

### Step 2: 建立幻燈片簡報 (index.html)
- 單頁 HTML，所有內容在同一個檔案
- 每個 `<section class="slide">` 為一頁
- 方向鍵 ← → 翻頁
- 底部導航列：◀ 頁碼 ▶
- 深色主題封面，淺色主題內頁
- 支援手機 RWD

必要功能：
```html
<div class="slide-nav">
  <button onclick="prevSlide()">◀</button>
  <span id="pageInfo">1 / N</span>
  <button onclick="nextSlide()">▶</button>
</div>
<script>
var i=0,N=幻燈片總數;
function show(n){...}
function nextSlide(){show((i+1)%N)}
function prevSlide(){show((i-1+N)%N)}
document.addEventListener('keydown',function(e){
  if(e.key==='ArrowRight')nextSlide();
  if(e.key==='ArrowLeft')prevSlide()
});
show(0);
</script>
```

### Step 3: 啟用 GitHub Pages
```bash
gh repo create {owner}/{repo} --source="." --public --push
gh api repos/{owner}/{repo}/pages --method POST --input - <<EOF
{"source":{"branch":"main","path":"/"}}
EOF
```

### Step 4: 輸出結果
直接給 GitHub Pages 連結，不要給檔案路徑：
```
https://{owner}.github.io/{repo}/
```

## 使用者偏好
- 不要給 Markdown 或 HTML 檔案路徑
- 不要只存在本機
- 每次開新分支或新專案都要 push 到 GitHub
