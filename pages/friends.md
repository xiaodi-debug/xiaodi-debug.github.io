---
title: 友链鱼塘
---

# 友链鱼塘

这里是 **「大神之路」的友链鱼塘** 🐟

欢迎在这里交换友链，一起记录、一起成长。

## 友链申请说明

- **类型**：偏向个人博客 / 技术 / 生活记录。
- **要求**：
  - 站点能正常访问，不是采集站 / 纯广告站；
  - 内容健康，遵守法律法规；
  - 尽量有一定原创内容。
- **申请方式**：
  - 在本页评论区 / 畅所欲言页面留言；
  - 或发邮件至：`liuxiaodi2026@163.com`，附上下面这些信息。

**友链格式（示例）：**

- 站点名称：大神之路
- 站点地址：https://liuxiaodi.icu
- 站点头像：https://你的头像链接
- 一句话描述：记录大神之路上的学习、踩坑与成长

---

## 鱼塘卡片

<div class="friends-grid">

  <!-- 你自己的站点 -->
  <a class="friend-card" href="https://liuxiaodi.icu" target="_blank" rel="noreferrer">
    <div class="friend-avatar">
      <img src="/images/logo/logo.svg" alt="大神之路" />
    </div>
    <div class="friend-info">
      <div class="friend-name">大神之路</div>
      <div class="friend-desc">记录学习、踩坑与成长的个人开发者博客</div>
      <div class="friend-tags">
        <span class="tag">博客</span>
        <span class="tag">技术</span>
        <span class="tag">踩坑记录</span>
      </div>
    </div>
  </a>

  <!-- 示例：阮一峰 -->
  <a class="friend-card" href="https://www.ruanyifeng.com/blog/" target="_blank" rel="noreferrer">
    <div class="friend-avatar">
      <img src="https://via.placeholder.com/80x80?text=R" alt="阮一峰" />
    </div>
    <div class="friend-info">
      <div class="friend-name">阮一峰</div>
      <div class="friend-desc">知名技术博主，大佬中的大佬</div>
      <div class="friend-tags">
        <span class="tag">技术</span>
        <span class="tag">前端</span>
      </div>
    </div>
  </a>

  <!-- 示例：quark / NAS / Docker 相关博客（占位，按你需求改） -->
  <a class="friend-card" href="https://blog.zhheo.com/" target="_blank" rel="noreferrer">
    <div class="friend-avatar">
      <img src="https://via.placeholder.com/80x80?text=H" alt="张洪 Heo" />
    </div>
    <div class="friend-info">
      <div class="friend-name">张洪 Heo</div>
      <div class="friend-desc">产品设计师，独立开发者，设计与科技分享</div>
      <div class="friend-tags">
        <span class="tag">设计</span>
        <span class="tag">独立开发</span>
      </div>
    </div>
  </a>

  <!-- 示例：無名小栈，可以保留为友链 -->
  <a class="friend-card" href="https://blog.imsyy.top/" target="_blank" rel="noreferrer">
    <div class="friend-avatar">
      <img src="https://via.placeholder.com/80x80?text=WM" alt="無名小栈" />
    </div>
    <div class="friend-info">
      <div class="friend-name">無名小栈</div>
      <div class="friend-desc">分享技术与科技生活</div>
      <div class="friend-tags">
        <span class="tag">技术</span>
        <span class="tag">生活</span>
      </div>
    </div>
  </a>

  <!-- 以后新增友链，就按这个结构往下复制一块 friend-card 即可 -->

</div>

<style>
.friends-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 16px;
  margin: 16px 0 32px;
}

.friend-card {
  display: flex;
  align-items: center;
  padding: 14px 16px;
  border-radius: 14px;
  background-color: var(--main-card-background);
  border: 1px solid rgba(0, 0, 0, 0.04);
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.05);
  text-decoration: none;
  color: inherit;
  transition:
    box-shadow 0.2s,
    transform 0.15s,
    border-color 0.2s,
    background-color 0.2s;
}

.friend-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 22px rgba(0, 0, 0, 0.16);
  border-color: rgba(0, 0, 0, 0.08);
  background-color: rgba(255, 255, 255, 0.98);
}

.friend-avatar {
  flex: 0 0 auto;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  overflow: hidden;
  margin-right: 14px;
  background-color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
}

.friend-avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.friend-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.friend-name {
  font-size: 0.95rem;
  font-weight: 600;
  color: var(--main-font-color);
}

.friend-desc {
  font-size: 0.82rem;
  color: var(--main-font-second-color);
  opacity: 0.9;
}

.friend-tags {
  margin-top: 4px;
}

.friend-tags .tag {
  display: inline-block;
  padding: 2px 8px;
  margin-right: 6px;
  border-radius: 999px;
  font-size: 0.7rem;
  color: var(--main-color);
  background-color: var(--main-color-bg);
}
</style>
