<script setup lang="ts">
import RenderCard from './RenderCard.vue';
import AnimCard from './AnimCard.vue';
import PhysicsCard from './PhysicsCard.vue';
import BoneCard from './BoneCard.vue';
</script>

<template>
  <section class="TechFieldsSection ScrollReveal" id="TechFields">
    <div class="SectionHeader">
      <h2 class="SectionTitle">核心技术沉淀</h2>
      <p class="SectionSubtitle">
        Deeply focusing on handy techniques like rendering, animation, collisions, and dynamic bones.
      </p>
    </div>

    <!-- 交互式技术卡片网格，采用 PascalCase 类名 -->
    <div class="CardsGrid">
      <div class="GridItem">
        <RenderCard />
      </div>
      <div class="GridItem">
        <AnimCard />
      </div>
      <div class="GridItem">
        <PhysicsCard />
      </div>
      <div class="GridItem">
        <BoneCard />
      </div>
    </div>
  </section>
</template>

<style scoped>
.TechFieldsSection {
  display: flex;
  flex-direction: column;
  gap: 40px;
  padding: 40px 0;
}

.SectionHeader {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.SectionTitle {
  font-size: 32px;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.SectionSubtitle {
  font-size: 16px;
  color: var(--textColor);
  max-width: 600px;
  line-height: 1.6;
  font-family: var(--fontSans);
}

.CardsGrid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 24px;
  width: 100%;
}

.GridItem {
  display: flex;
  flex-direction: column;
}

@media (max-width: 1024px) {
  .CardsGrid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 640px) {
  .CardsGrid {
    grid-template-columns: 1fr;
  }
  .SectionTitle {
    font-size: 26px;
  }
  .SectionSubtitle {
    font-size: 14px;
  }
}
</style>
