<script setup lang="ts">
import { onMounted } from 'vue';
import AppHeader from './components/AppHeader.vue';
import HeroSection from './components/HeroSection.vue';
import TechFields from './components/TechFields.vue';
import AnimationMirror from './components/AnimationMirror.vue';
import ContactFooter from './components/ContactFooter.vue';

// 采用 camelCase 命名变量与方法
onMounted(() => {
  if (!CSS.supports('(animation-timeline: view()) and (animation-range: entry)')) {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            entry.target.classList.add('Visible');
          }
        });
      },
      {
        // 降低阈值以确保在小屏幕上更早触发过渡
        threshold: 0.05,
        rootMargin: '0px 0px -50px 0px'
      }
    );

    document.querySelectorAll('.ScrollReveal').forEach((el) => {
      observer.observe(el);
    });
  }
});
</script>

<template>
  <div class="MainLayout">
    <!-- 背景网格，类名采用 PascalCase -->
    <div class="GridBackground"></div>
    
    <!-- 头部栏 -->
    <AppHeader />
    
    <!-- 主体区域，增加顶部 padding 避开 Header -->
    <main class="MainContent">
      <HeroSection />
      <TechFields />
      <AnimationMirror />
    </main>
    
    <!-- 页脚栏 -->
    <ContactFooter />
  </div>
</template>

<style scoped>
.MainContent {
  padding-top: 70px; /* 避开固定头部的空间 */
}
</style>
