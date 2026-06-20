<script setup lang="ts">
// 常量定义，使用 UPPER_SNAKE_CASE
const PUBLISHER_PROFILE_URL = 'https://publisher.unity.com/account/profile';
const EMAIL_ADDRESS = 'mailto:your-email@example.com'; // 占位但完整可用，后续用户可替换
const GITHUB_PROFILE = 'https://github.com';
</script>

<template>
  <footer class="ContactFooter">
    <div class="FooterGrid">
      <div class="BrandInfo">
        <h4 class="BrandTitle">Unity Wooing</h4>
        <p class="BrandDesc">
          专注于发现和实现实用的游戏引擎算法，为 Unity 开发者提供高性能资产套件。
        </p>
      </div>

      <div class="QuickLinks">
        <h5 class="LinkHeader">导航</h5>
        <ul class="LinkList">
          <li><a href="#TechFields" class="FooterLink">技术领域</a></li>
          <li><a href="#AnimationMirror" class="FooterLink">AnimationMirror</a></li>
        </ul>
      </div>

      <div class="SocialChannels">
        <h5 class="LinkHeader">开发者社群</h5>
        <ul class="LinkList">
          <li>
            <a :href="PUBLISHER_PROFILE_URL" target="_blank" rel="noopener noreferrer" class="FooterLink ActionLink">
              Unity Asset Store Profile
            </a>
          </li>
          <li>
            <a :href="GITHUB_PROFILE" target="_blank" rel="noopener noreferrer" class="FooterLink">
              GitHub 仓库
            </a>
          </li>
          <li>
            <a :href="EMAIL_ADDRESS" class="FooterLink">
              联系电子邮箱
            </a>
          </li>
        </ul>
      </div>
    </div>

    <div class="FooterBottom">
      <p class="CopyrightText">
        &copy; {{ new Date().getFullYear() }} Unity Wooing. All rights reserved.
      </p>
      <p class="SloganFooter">
        Slogan: "Found a really useful game engine algorithm"
      </p>
    </div>
  </footer>
</template>

<style scoped>
.ContactFooter {
  background-color: rgba(10, 11, 14, 0.9);
  border-top: 1px solid rgba(255, 255, 255, 0.05);
  padding: 60px 24px 30px;
  width: 100%;
  margin-top: auto;
  position: relative;
  z-index: 10;
}

.FooterGrid {
  max-width: 1200px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: 2fr 1fr 1.5fr;
  gap: 40px;
  padding-bottom: 40px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

.BrandInfo {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.BrandTitle {
  font-size: 18px;
  color: var(--textHighlightColor);
  font-weight: 700;
}

.BrandDesc {
  font-size: 14px;
  color: var(--textColor);
  line-height: 1.6;
  max-width: 360px;
}

.LinkHeader {
  font-size: 12px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--textHighlightColor);
  margin-bottom: 16px;
}

.LinkList {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.FooterLink {
  color: var(--textColor);
  font-size: 14px;
  transition: all 0.3s ease;
}

.FooterLink:hover {
  color: var(--primaryColor);
  padding-left: 4px;
}

.ActionLink {
  color: var(--primaryColor);
  font-weight: 600;
}

.FooterBottom {
  max-width: 1200px;
  margin: 0 auto;
  padding-top: 30px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 16px;
}

.CopyrightText {
  font-size: 13px;
  color: var(--textColor);
  opacity: 0.6;
}

.SloganFooter {
  font-size: 13px;
  font-family: var(--fontMono);
  color: var(--secondaryColor);
  opacity: 0.8;
}

@media (max-width: 768px) {
  .FooterGrid {
    grid-template-columns: 1fr;
    gap: 30px;
  }
  .FooterBottom {
    flex-direction: column;
    text-align: center;
  }
}
</style>
