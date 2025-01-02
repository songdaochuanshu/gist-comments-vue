<!-- Modal.vue -->
<template>
    <transition name="modal">
      <div v-if="visible" class="modal-overlay" @click.self="close">
        <div class="modal-window">
          <div class="modal-content">
            <h3>{{ title }}</h3>
            <p>{{ message }}</p>
          </div>
          <footer class="modal-footer">
            <button @click="close">确定</button>
          </footer>
        </div>
      </div>
    </transition>
  </template>
  
  <script setup>
  import { ref, defineProps, defineEmits } from 'vue';
  
  const props = defineProps({
    title: {
      type: String,
      required: true
    },
    message: {
      type: String,
      required: true
    }
  });
  
  const emit = defineEmits(['close']);
  const visible = ref(true);
  
  function close() {
    visible.value = false;
    setTimeout(() => {
      emit('close');
    }, 300); // 等待过渡动画结束
  }
  </script>
  
  <style scoped>
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }
  
  .modal-window {
    background-color: white;
    padding: 20px;
    border-radius: 5px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.33);
    width: 80%;
    max-width: 400px;
  }
  
  .modal-content {
    margin-bottom: 20px;
  }
  
  .modal-footer {
    text-align: right;
  }
  
  button {
    padding: 10px 20px;
    background-color: #007BFF;
    color: white;
    border: none;
    cursor: pointer;
  }
  
  button:hover {
    background-color: #0056b3;
  }
  
  /* 过渡效果 */
  .modal-enter-active,
  .modal-leave-active {
    transition: opacity 0.3s ease;
  }
  
  .modal-enter-from,
  .modal-leave-to {
    opacity: 0;
  }
  </style>