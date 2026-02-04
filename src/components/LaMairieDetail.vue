<template>
  <Transition name="detail">
    <div v-if="open" class="detail-overlay" @click.self="close">
      <div class="detail-frame">
        <button class="detail-close" aria-label="Close" @click="close">×</button>
        <div class="detail-layout">
          <div class="detail-left">
            <LaMairieViewer />
          </div>
          <div class="detail-right">
            <h2 class="detail-title">La mairie de Bordj El Bahri</h2>
            <p class="detail-text">
              {{ showFullDescription ? fullDescription : truncatedDescription }}
              <button
                v-if="fullDescription.length > 120"
                type="button"
                class="detail-show-more"
                @click="showFullDescription = !showFullDescription"
              >
                {{ showFullDescription ? 'Show less' : 'Show more' }}
              </button>
            </p>
            <div class="detail-image-wrap">
              <img src="/la%20mairie.jpeg" alt="La mairie" class="detail-image" />
            </div>
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { ref, computed } from 'vue';
import LaMairieViewer from './LaMairieViewer.vue';

defineProps({ open: Boolean });
const emit = defineEmits(['close']);

const fullDescription =
  "La mairie de Bordj El Bahri s'inscrit dans l'histoire administrative de la commune, dont le développement urbain s'est structuré principalement durant la période coloniale, lorsque Bordj El Bahri était connu sous le nom de Fort-de-l'Eau. Le bâtiment de la mairie témoigne de cette phase historique, marquée par l'implantation d'équipements publics destinés à organiser la gestion locale et les services administratifs. Après l'indépendance de l'Algérie en 1962, la mairie a été réappropriée et intégrée au système administratif national en tant que siège de l'Assemblée Populaire Communale (APC), jouant depuis un rôle central dans la gouvernance locale et la vie institutionnelle de Bordj El Bahri.";

const showFullDescription = ref(false);
const truncatedDescription = computed(() =>
  fullDescription.length > 120 ? fullDescription.slice(0, 120).trim() + '…' : fullDescription
);

function close() {
  emit('close');
}
</script>

<style scoped>
.detail-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(6px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  box-sizing: border-box;
}
.detail-frame {
  position: relative;
  width: 100%;
  max-width: 1200px;
  max-height: 90vh;
  background: #fff;
  border-radius: 24px;
  box-shadow: 0 24px 60px rgba(0, 0, 0, 0.22), 0 0 0 1px rgba(0, 0, 0, 0.06);
  overflow-y: auto;
  overflow-x: hidden;
  border: 6px solid #e07b2c;
}
.detail-close {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 44px;
  height: 44px;
  border: none;
  background: rgba(224, 123, 44, 0.9);
  color: #fff;
  font-size: 1.5rem;
  line-height: 1;
  border-radius: 50%;
  cursor: pointer;
  z-index: 10;
  transition: background 0.2s, transform 0.2s;
  box-shadow: 0 4px 12px rgba(160, 70, 10, 0.25);
}
.detail-close:hover {
  background: rgba(237, 122, 12, 0.95);
  transform: scale(1.05);
}
.detail-layout {
  display: grid;
  grid-template-columns: 1fr 400px;
  min-height: 520px;
  align-items: stretch;
}
.detail-left {
  min-height: 360px;
  padding: 1.25rem;
  background: #fdeee0;
  border-right: 4px solid #e07b2c;
  display: flex;
  align-items: center;
  justify-content: center;
}
.detail-left :deep(.viewer) {
  flex: 0 1 auto;
  width: 100%;
  min-height: 320px;
  max-height: 65vh;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12);
}
.detail-right {
  background: linear-gradient(180deg, #2a2520 0%, #1f1c18 100%);
  padding: 2rem 2.25rem;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}
.detail-title {
  margin: 0;
  font-size: 1.85rem;
  font-weight: 700;
  color: #fff;
  line-height: 1.25;
  letter-spacing: -0.02em;
  padding-right: 2rem;
}
.detail-text {
  margin: 0;
  font-size: 0.95rem;
  line-height: 1.65;
  color: rgba(255, 255, 255, 0.88);
}
.detail-show-more {
  margin-left: 0.25rem;
  padding: 0;
  font-size: 0.9rem;
  font-weight: 600;
  color: #f97316;
  background: none;
  border: none;
  cursor: pointer;
  text-decoration: underline;
  text-underline-offset: 2px;
}
.detail-show-more:hover {
  color: #fdba74;
}
.detail-image-wrap {
  margin-top: auto;
  border-radius: 16px;
  overflow: hidden;
  background: #1a1a1a;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.35);
  border: 1px solid rgba(255, 255, 255, 0.06);
}
.detail-image {
  width: 100%;
  height: auto;
  display: block;
}

.detail-enter-active,
.detail-leave-active {
  transition: opacity 0.35s ease;
}
.detail-enter-active .detail-frame,
.detail-leave-active .detail-frame {
  transition: transform 0.35s ease;
}
.detail-enter-from,
.detail-leave-to {
  opacity: 0;
}
.detail-enter-from .detail-frame,
.detail-leave-to .detail-frame {
  transform: scale(0.95) translateY(20px);
}

@media (max-width: 900px) {
  .detail-layout {
    grid-template-columns: 1fr;
    grid-template-rows: 1fr auto;
  }
  .detail-left {
    min-height: 280px;
    border-right: none;
    border-bottom: 4px solid #e07b2c;
  }
}
</style>
