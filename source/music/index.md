---
title: 音乐
date: 2024-10-23 11:29:59
---
<!-- 音乐播放器 -->
<div class="music-page">
  {% meting "2474029461" "netease" "playlist" 
"autoplay" 
"mutex:false" 
"listmaxheight:800px" 
"preload:auto" 
"theme:#f9d3e3" %}
</div>

<style>
  body {
    background: linear-gradient(to right, #c9d6ff, #e2e2e2);
    background-attachment: fixed;
    font-family: 'Lora', 'Helvetica Neue', Helvetica, Arial, sans-serif;
  }
  
 .music-page {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 50px;
    padding: 20px;
    background: rgba(255, 255, 255, 0.8);
    box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
    backdrop-filter: blur(8.5px);
    border-radius: 15px;
    width: 80%;
    margin-left: auto;
    margin-right: auto;
    height: 1000px;
  }

  h1, h2 {
    text-align: center;
    color: #6a11cb;
    margin-top: 20px;
  }

  .aplayer {
    border-radius: 10px;
    overflow: hidden;
  }
</style>

<script>
 // js 修改部分
 document.addEventListener('DOMContentLoaded', function() {
  const player = document.querySelector('.music-page');
  player.style.opacity = 0;
  player.style.transition = "opacity 2s";
  setTimeout(() => {
    player.style.opacity = 1;
  }, 200);
});
</script>