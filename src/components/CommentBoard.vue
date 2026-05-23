<template>
  <div class="comment-board-container">
    <div class="glass-card">
      <h1>留言板</h1>

      <form @submit.prevent="submitComment" class="input-section">
        <div class="input-group">
          <input 
            v-model="name" 
            placeholder="您的称呼" 
            maxlength="20" 
            required 
            class="styled-input"
          />
        </div>
        <div class="input-group">
          <textarea 
            v-model="message" 
            placeholder="分享您的想法..." 
            maxlength="200" 
            required 
            class="styled-textarea"
          ></textarea>
        </div>
        <div class="form-footer">
          <button type="submit" :disabled="isSubmitting || isCoolingDown" class="submit-btn">
            <span v-if="isSubmitting" class="loader"></span>
            {{ isSubmitting ? '发送中' : '发布留言' }}
          </button>
          <transition name="fade">
            <span v-if="isCoolingDown" class="cooldown-text">
              休息一下，{{ cooldownTime }}s 后可再发
            </span>
          </transition>
        </div>
      </form>

      <div class="divider"></div>

      <div v-if="isLoading" class="loading-state">
        <div class="pulse-loader"></div>
        <p>正在获取灵感...</p>
      </div>

      <div v-else id="comments-container">
        <transition-group name="list" tag="div" class="comments-list">
          <div v-for="comment in paginatedComments" :key="comment.id" class="comment-item">
            <div class="comment-header">
              <span class="comment-author">{{ comment.name }}</span>
              <span class="comment-date">{{ formatDate(comment.timestamp) }}</span>
            </div>
            <p class="comment-body">{{ comment.message }}</p>
          </div>
        </transition-group>

        <div v-if="totalPages > 1" class="pagination">
          <button @click="prevPage" :disabled="currentPage === 1" class="nav-btn">
            &larr;
          </button>
          <span class="page-indicator">{{ currentPage }} / {{ totalPages }}</span>
          <button @click="nextPage" :disabled="currentPage === totalPages" class="nav-btn">
            &rarr;
          </button>
        </div>
      </div>
    </div>

    <Modal v-if="showModal" :title="modalTitle" :message="modalMessage" @close="hideModal" />
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import Modal from './Modal.vue';

const gistId = import.meta.env.VITE_GIST_ID;
const token = import.meta.env.VITE_GITHUB_TOKEN;

const name = ref('');
const message = ref('');
const comments = ref([]);
const localComments = ref([]);
const isSubmitting = ref(false);
const isLoading = ref(false);
const showModal = ref(false);
const modalTitle = ref('');
const modalMessage = ref('');
const currentPage = ref(1);
const pageSize = 5; // 减少每页数量，增加艺术感
const cooldownDuration = 5000;
const cooldownTime = ref(0);
const isCoolingDown = ref(false);

let etag;

function validateInput() {
  if (!name.value.trim() || !message.value.trim()) {
    showNotification('提示', '请留下您的姓名和留言内容。');
    return false;
  }
  return true;
}

async function fetchComments() {
  isLoading.value = true;
  try {
    const headers = etag ? { 'If-None-Match': etag } : {};
    const response = await axios.get(`https://api.github.com/gists/${gistId}`, { headers });

    if (response.status === 200) {
      const commentsData = JSON.parse(response.data.files['comments.json'].content).comments || [];
      comments.value = commentsData.reverse();
      etag = response.headers.etag;
    }
  } catch (error) {
    if (error.response && error.response.status !== 304) {
      showNotification('加载失败', '无法获取留言，请稍后再试。');
    }
  } finally {
    isLoading.value = false;
  }
}

async function submitComment() {
  if (!validateInput()) return;
  isSubmitting.value = true;

  try {
    const newComment = {
      id: Date.now(),
      name: name.value.trim(),
      message: message.value.trim(),
      timestamp: new Date().toISOString()
    };

    localComments.value.unshift(newComment);

    const response = await axios.get(`https://api.github.com/gists/${gistId}`);
    let existingComments = JSON.parse(response.data.files['comments.json'].content).comments || [];
    existingComments.push(newComment);

    await axios.patch(`https://api.github.com/gists/${gistId}`, {
      files: {
        'comments.json': {
          content: JSON.stringify({ comments: existingComments }, null, 2)
        }
      }
    }, {
      headers: { Authorization: `token ${token}` }
    });

    name.value = '';
    message.value = '';
    etag = null;
    await fetchComments();
  } catch (error) {
    showNotification('提交失败', '留言未能成功发布，请检查网络。');
  } finally {
    startCooldown();
    isSubmitting.value = false;
  }
}

const displayedComments = computed(() => {
  const allComments = [...localComments.value, ...comments.value];
  return allComments.filter((comment, index, self) =>
    index === self.findIndex((c) => c.id === comment.id)
  );
});

const totalPages = computed(() => Math.ceil(displayedComments.value.length / pageSize));
const paginatedComments = computed(() => {
  const start = (currentPage.value - 1) * pageSize;
  const end = start + pageSize;
  return displayedComments.value.slice(start, end);
});

function prevPage() { if (currentPage.value > 1) currentPage.value--; }
function nextPage() { if (currentPage.value < totalPages.value) currentPage.value++; }

function startCooldown() {
  isCoolingDown.value = true;
  cooldownTime.value = cooldownDuration / 1000;
  const interval = setInterval(() => {
    cooldownTime.value--;
    if (cooldownTime.value <= 0) {
      clearInterval(interval);
      isCoolingDown.value = false;
    }
  }, 1000);
}

function showNotification(title, message) {
  modalTitle.value = title;
  modalMessage.value = message;
  showModal.value = true;
}

function hideModal() { showModal.value = false; }

function formatDate(dateString) {
  const date = new Date(dateString);
  return new Intl.DateTimeFormat('zh-CN', {
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  }).format(date);
}

onMounted(fetchComments);
</script>

<style scoped>
.comment-board-container {
  perspective: 1000px;
}

.glass-card {
  background: var(--glass-bg);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid var(--glass-border);
  border-radius: 24px;
  padding: 3rem;
  box-shadow: var(--shadow);
}

.input-section {
  margin-bottom: 3rem;
}

.input-group {
  margin-bottom: 1.5rem;
}

.styled-input, .styled-textarea {
  width: 100%;
  padding: 1rem 1.25rem;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid var(--glass-border);
  border-radius: 12px;
  color: var(--text-main);
  font-size: 1rem;
  transition: all 0.3s ease;
  box-sizing: border-box;
}

.styled-textarea {
  min-height: 120px;
  resize: vertical;
}

.styled-input:focus, .styled-textarea:focus {
  outline: none;
  border-color: var(--primary-color);
  background: rgba(255, 255, 255, 0.1);
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.1);
}

.form-footer {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.submit-btn {
  padding: 0.8rem 2rem;
  background: var(--primary-color);
  color: white;
  font-size: 1rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.submit-btn:hover:not(:disabled) {
  background: var(--primary-hover);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.cooldown-text {
  font-size: 0.875rem;
  color: var(--text-muted);
}

.divider {
  height: 1px;
  background: linear-gradient(to right, transparent, var(--glass-border), transparent);
  margin: 3rem 0;
}

.comment-item {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid var(--glass-border);
  border-radius: 16px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  transition: transform 0.3s ease;
}

.comment-item:hover {
  transform: translateX(8px);
  background: rgba(255, 255, 255, 0.05);
}

.comment-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}

.comment-author {
  font-weight: 700;
  color: var(--primary-color);
}

.comment-date {
  font-size: 0.8rem;
  color: var(--text-muted);
}

.comment-body {
  margin: 0;
  line-height: 1.6;
  color: var(--text-main);
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 2rem;
  margin-top: 3rem;
}

.nav-btn {
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--glass-bg);
  border: 1px solid var(--glass-border);
  border-radius: 50%;
  color: var(--text-main);
}

.nav-btn:hover:not(:disabled) {
  background: var(--primary-color);
  color: white;
}

.page-indicator {
  font-weight: 600;
  color: var(--text-muted);
}

/* Animations */
.list-enter-active, .list-leave-active {
  transition: all 0.5s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: translateY(30px);
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}

.loading-state {
  text-align: center;
  padding: 3rem;
  color: var(--text-muted);
}

.pulse-loader {
  width: 40px;
  height: 40px;
  background-color: var(--primary-color);
  border-radius: 50%;
  margin: 0 auto 1rem;
  animation: pulse 1.5s infinite ease-in-out;
}

@keyframes pulse {
  0% { transform: scale(0.8); opacity: 0.5; }
  50% { transform: scale(1.2); opacity: 1; }
  100% { transform: scale(0.8); opacity: 0.5; }
}

.loader {
  width: 16px;
  height: 16px;
  border: 2px solid #fff;
  border-bottom-color: transparent;
  border-radius: 50%;
  display: inline-block;
  animation: rotation 1s linear infinite;
}

@keyframes rotation {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>
