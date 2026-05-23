<template>
  <transition name="modal-fade">
    <div v-if="visible" class="modal-overlay" @click.self="close">
      <div class="modal-window">
        <div class="modal-icon">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"></circle><line x1="12" y1="16" x2="12" y2="12"></line><line x1="12" y1="8" x2="12.01" y2="8"></line></svg>
        </div>
        <div class="modal-content">
          <h3>{{ title }}</h3>
          <p>{{ message }}</p>
        </div>
        <footer class="modal-footer">
          <button @click="close" class="confirm-btn">确定</button>
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
  background-color: rgba(15, 23, 42, 0.6);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
}

.modal-window {
  background: var(--glass-bg);
  border: 1px solid var(--glass-border);
  padding: 2.5rem;
  border-radius: 24px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.2);
  width: 90%;
  max-width: 400px;
  text-align: center;
  transform-origin: center;
}

.modal-icon {
  width: 56px;
  height: 56px;
  background: rgba(99, 102, 241, 0.1);
  color: var(--primary-color);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.5rem;
}

.modal-content h3 {
  margin: 0 0 0.75rem;
  font-size: 1.5rem;
  color: var(--text-main);
}

.modal-content p {
  color: var(--text-muted);
  line-height: 1.6;
  margin-bottom: 2rem;
}

.confirm-btn {
  width: 100%;
  padding: 0.8rem;
  background: var(--primary-color);
  color: white;
  font-size: 1rem;
  border-radius: 12px;
}

.confirm-btn:hover {
  background: var(--primary-hover);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
}

/* 过渡动画 */
.modal-fade-enter-active {
  animation: modal-in 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.modal-fade-leave-active {
  animation: modal-in 0.3s cubic-bezier(0.34, 1.56, 0.64, 1) reverse;
}

@keyframes modal-in {
  0% { opacity: 0; transform: scale(0.8) translateY(20px); }
  100% { opacity: 1; transform: scale(1) translateY(0); }
}
</style>
