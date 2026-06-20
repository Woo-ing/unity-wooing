<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import logoUrl from '../assets/logo.png';

// 常量定义，采用 UPPER_SNAKE_CASE
const PROFILE_URL = 'https://publisher.unity.com/account/profile';

// 响应式变量，采用 camelCase
const isScrolled = ref(false);

// 滚动监听事件处理函数，采用 camelCase
const handleScroll = () => {
  isScrolled.value = window.scrollY > 20;
};

onMounted(() => {
  window.addEventListener('scroll', handleScroll);
});

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll);
});
</script>

<template>
  <header :class="['AppHeader', { 'HeaderScrolled': isScrolled }]">
    <div class="HeaderContainer">
      <div class="LogoSection">
        <img :src="logoUrl" alt="Publisher Logo" class="PublisherLogo" />
        <span class="BrandName">Unity <span class="PrimaryGradientText">Wooing</span></span>
      </div>
      
      <nav class="NavigationLinks">
        <a href="#TechFields" class="NavLink">技术领域</a>
        <a href="#AnimationMirror" class="NavLink">作品展示</a>
      </nav>

      <div class="ActionSection">
        <a 
          :href="PROFILE_URL" 
          target="_blank" 
          rel="noopener noreferrer" 
          class="ProfileLinkButton PrimaryButton"
          id="HeaderProfileBtn"
        >
          <span>Unity Profile</span>
          <svg class="ButtonIcon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
            <polyline points="15 3 21 3 21 9" />
            <line x1="10" y1="14" x2="21" y2="3" />
          </svg>
        </a>
      </div>
    </div>
  </header>
</template>

<style scoped>
.AppHeader {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 70px;
  z-index: 100;
  transition: all 0.3s ease;
  border-bottom: 1px solid transparent;
}

.HeaderScrolled {
  background: rgba(10, 11, 14, 0.75);
  backdrop-filter: blur(12px) saturate(180%);
  -webkit-backdrop-filter: blur(12px) saturate(180%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  height: 64px;
}

.HeaderContainer {
  max-width: 1200px;
  height: 100%;
  margin: 0 auto;
  padding: 0 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.LogoSection {
  display: flex;
  align-items: center;
  gap: 12px;
}

.PublisherLogo {
  height: 36px;
  width: auto;
  border-radius: 6px;
}

.BrandName {
  font-size: 20px;
  font-weight: 700;
  color: var(--textHighlightColor);
  letter-spacing: -0.03em;
}

.NavigationLinks {
  display: flex;
  gap: 32px;
}

.NavLink {
  color: var(--textColor);
  font-weight: 500;
  font-size: 15px;
  transition: color 0.3s ease;
}

.NavLink:hover {
  color: var(--primaryColor);
}

.ButtonIcon {
  width: 16px;
  height: 16px;
}

@media (max-width: 768px) {
  .NavigationLinks {
    display: none;
  }
}
</style>
