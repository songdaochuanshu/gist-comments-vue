<template>
  <transition name="modal-reveal">
    <div v-if="visible" class="modal-overlay" @click.self="close">
      <div class="modal-window">
        <div class="modal-icon">
          <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="16" x2="12" y2="12"></line>
            <line x1="12" y1="8" x2="12.01" y2="8"></line>
          </svg>
        </div>
        <div class="modal-content">
          <h3>{{ title }}</h3>
          <p>{{ message }}</p>
        </div>
        <footer class="modal-footer">
          <button @click="close" class="confirm-btn">知悉</button>
        </footer>
      </div>
    </div>
  </transition>
</template>

<script setup>
import { ref, defineProps, defineEmits } from 'vue';

const props = defineProps({
  title: { type: String, required: true },
  message: { type: String, required: true }
});

const emit = defineEmits(['close']);
const visible = ref(true);

function close() {
  visible.value = false;
  setTimeout(() => {
    emit('close');
  }, 400);
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(44, 36, 56, 0.4);
  backdrop-filter: blur(12px) saturate(0.8);
  -webkit-backdrop-filter: blur(12px) saturate(0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
}

.modal-window {
  background: var(--glass-bg);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid var(--glass-border);
  padding: 2.5rem 2.5rem 2rem;
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-elevated);
  width: 90%;
  max-width: 380px;
  text-align: center;
  position: relative;
  overflow: hidden;
}

/* 顶部装饰线 */
.modal-window::before {
  content: '';
  position: absolute;
  top: 0;
  left: 20%;
  right: 20%;
  height: 2px;
  background: linear-gradient(90deg, transparent, var(--primary-light), transparent);
  border-radius: 1px;
}

.modal-icon {
  width: 52px;
  height: 52px;
  background: var(--primary-glow);
  color: var(--primary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.5rem;
  border: 1px solid var(--glass-border);
}

.modal-content h3 {
  margin: 0 0 0.75rem;
  font-family: var(--font-display);
  font-size: 1.3rem;
  font-weight: 400;
  color: var(--ink-dark);
  letter-spacing: 0.1em;
}

.modal-content p {
  color: var(--ink-light);
  line-height: 1.8;
  margin-bottom: 2rem;
  font-size: 0.9rem;
}

.confirm-btn {
  width: 100%;
  padding: 0.75rem;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-light) 100%);
  color: white;
  font-size: 0.9rem;
  font-family: var(--font-body);
  border-radius: var(--radius-sm);
  letter-spacing: 0.15em;
}

.confirm-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 16px var(--primary-glow);
}

/* 过渡动画 */
.modal-reveal-enter-active {
  animation: modal-in 0.5s cubic-bezier(0.23, 1, 0.32, 1);
}

.modal-reveal-leave-active {
  animation: modal-in 0.35s cubic-bezier(0.4, 0, 1, 1) reverse;
}

@keyframes modal-in {
  0% {
    opacity: 0;
    transform: scale(0.92) translateY(16px);
  }
  100% {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}
</style>
