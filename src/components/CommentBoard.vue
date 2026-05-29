<template>
  <div class="comment-board-container">
    <!-- 装饰性浮动元素 -->
    <div class="deco-float deco-float-1"></div>
    <div class="deco-float deco-float-2"></div>

    <div class="glass-card">
      <!-- 标题区 -->
      <header class="board-header">
        <h1>留言板</h1>
        <p class="subtitle">执笔留痕，岁月可期</p>
      </header>

      <!-- 输入区 -->
      <form @submit.prevent="submitComment" class="input-section">
        <div class="input-row">
          <div class="input-group name-group">
            <label class="input-label">称呼</label>
            <input
              v-model="name"
              placeholder="您的雅号"
              maxlength="20"
              required
              class="styled-input"
            />
            <div class="input-line"></div>
          </div>
        </div>
        <div class="input-group">
          <label class="input-label">留言</label>
          <textarea
            v-model="message"
            placeholder="落笔生花，留几行心语……"
            maxlength="200"
            required
            class="styled-textarea"
          ></textarea>
          <div class="textarea-counter">
            <span :class="{ 'counter-warn': message.length > 180 }">{{ message.length }}</span>
            <span class="counter-sep">/</span>
            <span>200</span>
          </div>
        </div>
        <div class="form-footer">
          <button type="submit" :disabled="isSubmitting || isCoolingDown" class="submit-btn">
            <span v-if="isSubmitting" class="btn-loader"></span>
            <span v-else class="btn-icon">✦</span>
            {{ isSubmitting ? '发送中…' : '落笔' }}
          </button>
          <transition name="fade-slide">
            <span v-if="isCoolingDown" class="cooldown-text">
              <span class="cooldown-icon">⏳</span>
              {{ cooldownTime }}s
            </span>
          </transition>
        </div>
      </form>

      <!-- 分隔线 -->
      <div class="art-divider">
        <span class="divider-dot"></span>
        <span class="divider-line"></span>
        <span class="divider-char">言</span>
        <span class="divider-line"></span>
        <span class="divider-dot"></span>
      </div>

      <!-- 加载状态 -->
      <div v-if="isLoading" class="loading-state">
        <div class="ink-loader">
          <span></span><span></span><span></span>
        </div>
        <p class="loading-text">展卷中…</p>
      </div>

      <!-- 留言列表 -->
      <div v-else id="comments-container">
        <transition-group name="comment-reveal" tag="div" class="comments-list">
          <article
            v-for="(comment, index) in paginatedComments"
            :key="comment.id"
            class="comment-item"
            :style="{ '--delay': index * 0.08 + 's' }"
          >
            <div class="comment-accent"></div>
            <div class="comment-content">
              <header class="comment-header">
                <div class="author-avatar">
                  {{ comment.name.charAt(0) }}
                </div>
                <div class="comment-meta">
                  <span class="comment-author">{{ comment.name }}</span>
                  <time class="comment-date">{{ formatDate(comment.timestamp) }}</time>
                </div>
              </header>
              <p class="comment-body">{{ comment.message }}</p>
            </div>
          </article>
        </transition-group>

        <!-- 空状态 -->
        <div v-if="displayedComments.length === 0" class="empty-state">
          <div class="empty-icon">☽</div>
          <p>尚无留言</p>
          <p class="empty-sub">执笔落墨，留下第一行文字</p>
        </div>

        <!-- 分页 -->
        <div v-if="totalPages > 1" class="pagination">
          <button @click="prevPage" :disabled="currentPage === 1" class="nav-btn" aria-label="上一页">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="15 18 9 12 15 6"></polyline></svg>
          </button>
          <div class="page-dots">
            <span
              v-for="p in totalPages"
              :key="p"
              :class="['page-dot', { active: p === currentPage }]"
              @click="currentPage = p"
            ></span>
          </div>
          <button @click="nextPage" :disabled="currentPage === totalPages" class="nav-btn" aria-label="下一页">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"></polyline></svg>
          </button>
        </div>
      </div>
    </div>

    <!-- 页脚 -->
    <footer class="board-footer">
      <span>Powered by GitHub Gist</span>
      <span class="footer-sep">·</span>
      <span>Built with ♡</span>
    </footer>

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
const pageSize = 5;
const cooldownDuration = 5000;
const cooldownTime = ref(0);
const isCoolingDown = ref(false);

let etag;

function validateInput() {
  if (!name.value.trim() || !message.value.trim()) {
    showNotification('提示', '请留下您的称呼和留言内容。');
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

function showNotification(title, msg) {
  modalTitle.value = title;
  modalMessage.value = msg;
  showModal.value = true;
}

function hideModal() { showModal.value = false; }

function formatDate(dateString) {
  const date = new Date(dateString);
  const now = new Date();
  const diff = now - date;
  const minutes = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);

  if (minutes < 1) return '刚刚';
  if (minutes < 60) return `${minutes} 分钟前`;
  if (hours < 24) return `${hours} 小时前`;

  return new Intl.DateTimeFormat('zh-CN', {
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  }).format(date);
}

onMounted(fetchComments);
</script>

<style scoped>
/* ─── 容器 ─── */
.comment-board-container {
  position: relative;
  animation: board-enter 1s cubic-bezier(0.23, 1, 0.32, 1) both;
}

@keyframes board-enter {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
}

/* ─── 装饰浮动元素 ─── */
.deco-float {
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
  z-index: -1;
}

.deco-float-1 {
  width: 200px;
  height: 200px;
  background: radial-gradient(circle, var(--primary-glow), transparent 70%);
  top: -60px;
  left: -80px;
  animation: deco-drift-1 12s ease-in-out infinite;
}

.deco-float-2 {
  width: 150px;
  height: 150px;
  background: radial-gradient(circle, rgba(196, 149, 106, 0.06), transparent 70%);
  bottom: 100px;
  right: -60px;
  animation: deco-drift-2 10s ease-in-out infinite;
}

@keyframes deco-drift-1 {
  0%, 100% { transform: translate(0, 0); }
  50% { transform: translate(20px, 15px); }
}

@keyframes deco-drift-2 {
  0%, 100% { transform: translate(0, 0); }
  50% { transform: translate(-15px, -20px); }
}

/* ─── 毛玻璃卡片 ─── */
.glass-card {
  background: var(--glass-bg);
  backdrop-filter: blur(20px) saturate(1.2);
  -webkit-backdrop-filter: blur(20px) saturate(1.2);
  border: 1px solid var(--glass-border);
  border-top: 1px solid var(--glass-highlight);
  border-radius: var(--radius-xl);
  padding: var(--space-xl) var(--space-lg);
  box-shadow: var(--shadow-medium);
  position: relative;
  overflow: hidden;
}

/* 卡片内纹理 */
.glass-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 200px;
  background: linear-gradient(180deg, rgba(91, 74, 138, 0.02) 0%, transparent 100%);
  pointer-events: none;
}

/* ─── 标题区 ─── */
.board-header {
  text-align: center;
  margin-bottom: var(--space-xl);
  position: relative;
}

.board-header h1 {
  font-family: var(--font-display);
  font-size: 2.8rem;
  font-weight: 400;
  letter-spacing: 0.2em;
  margin: 0;
  color: var(--ink-dark);
  position: relative;
  display: inline-block;
}

.board-header h1::before,
.board-header h1::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 40px;
  height: 1px;
  background: linear-gradient(90deg, transparent, var(--ink-faint), transparent);
}

.board-header h1::before {
  right: calc(100% + 16px);
}

.board-header h1::after {
  left: calc(100% + 16px);
}

.subtitle {
  font-family: var(--font-serif);
  font-size: 0.9rem;
  color: var(--ink-faint);
  margin: var(--space-xs) 0 0;
  letter-spacing: 0.3em;
  font-style: italic;
}

/* ─── 输入区 ─── */
.input-section {
  margin-bottom: var(--space-lg);
  position: relative;
}

.input-row {
  display: flex;
  gap: var(--space-md);
  margin-bottom: var(--space-md);
}

.name-group {
  flex: 0 0 40%;
}

.input-label {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  color: var(--ink-faint);
  margin-bottom: var(--space-xs);
}

.styled-input,
.styled-textarea {
  width: 100%;
  padding: 0.9rem 1rem;
  background: rgba(255, 255, 255, 0.4);
  border: 1px solid var(--glass-border);
  border-radius: var(--radius-sm);
  color: var(--ink-dark);
  font-size: 0.95rem;
  font-family: var(--font-body);
  transition: all 0.4s cubic-bezier(0.23, 1, 0.32, 1);
  box-sizing: border-box;
  position: relative;
}

.styled-textarea {
  min-height: 110px;
  resize: vertical;
  line-height: 1.8;
}

.styled-input:focus,
.styled-textarea:focus {
  outline: none;
  border-color: var(--primary-light);
  background: rgba(255, 255, 255, 0.6);
  box-shadow: 0 0 0 3px var(--primary-glow);
}

.styled-input::placeholder,
.styled-textarea::placeholder {
  color: var(--ink-faint);
  font-style: italic;
}

/* 字数统计 */
.textarea-counter {
  text-align: right;
  font-size: 0.75rem;
  color: var(--ink-faint);
  margin-top: 4px;
  font-variant-numeric: tabular-nums;
}

.counter-sep {
  margin: 0 2px;
  opacity: 0.5;
}

.counter-warn {
  color: var(--accent);
}

/* ─── 提交按钮 ─── */
.form-footer {
  display: flex;
  align-items: center;
  gap: var(--space-md);
  margin-top: var(--space-md);
}

.submit-btn {
  padding: 0.8rem 2.5rem;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-light) 100%);
  color: white;
  font-size: 0.95rem;
  font-family: var(--font-body);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  gap: 0.6rem;
  box-shadow: 0 4px 16px var(--primary-glow);
  letter-spacing: 0.1em;
}

.submit-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(91, 74, 138, 0.2);
}

.submit-btn:active:not(:disabled) {
  transform: translateY(0);
}

.submit-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-icon {
  font-size: 0.8em;
  opacity: 0.8;
}

.btn-loader {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.cooldown-text {
  font-size: 0.85rem;
  color: var(--ink-faint);
  display: flex;
  align-items: center;
  gap: 4px;
}

.cooldown-icon {
  font-size: 0.9em;
}

/* ─── 艺术分隔线 ─── */
.art-divider {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin: var(--space-xl) 0;
  opacity: 0.4;
}

.divider-line {
  width: 60px;
  height: 1px;
  background: linear-gradient(90deg, transparent, var(--ink-faint), transparent);
}

.divider-dot {
  width: 3px;
  height: 3px;
  border-radius: 50%;
  background: var(--ink-faint);
}

.divider-char {
  font-family: var(--font-display);
  font-size: 0.85rem;
  color: var(--ink-faint);
  letter-spacing: 0;
}

/* ─── 加载状态 ─── */
.loading-state {
  text-align: center;
  padding: var(--space-xl) 0;
}

.ink-loader {
  display: flex;
  justify-content: center;
  gap: 8px;
  margin-bottom: var(--space-md);
}

.ink-loader span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--primary);
  animation: ink-pulse 1.4s ease-in-out infinite;
}

.ink-loader span:nth-child(2) { animation-delay: 0.2s; }
.ink-loader span:nth-child(3) { animation-delay: 0.4s; }

@keyframes ink-pulse {
  0%, 80%, 100% {
    transform: scale(0.6);
    opacity: 0.3;
  }
  40% {
    transform: scale(1);
    opacity: 1;
  }
}

.loading-text {
  font-family: var(--font-serif);
  font-size: 0.9rem;
  color: var(--ink-faint);
  letter-spacing: 0.2em;
  margin: 0;
}

/* ─── 留言列表 ─── */
.comments-list {
  display: flex;
  flex-direction: column;
  gap: var(--space-md);
}

.comment-item {
  position: relative;
  background: rgba(255, 255, 255, 0.35);
  border: 1px solid var(--glass-border);
  border-radius: var(--radius-md);
  overflow: hidden;
  transition: all 0.4s cubic-bezier(0.23, 1, 0.32, 1);
  animation: comment-enter 0.6s cubic-bezier(0.23, 1, 0.32, 1) both;
  animation-delay: var(--delay);
}

@keyframes comment-enter {
  from {
    opacity: 0;
    transform: translateY(16px);
  }
}

.comment-item:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-soft);
  border-color: var(--primary-glow);
}

/* 左侧装饰条 */
.comment-accent {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 3px;
  background: linear-gradient(180deg, var(--primary-light), var(--accent));
  opacity: 0;
  transition: opacity 0.4s;
}

.comment-item:hover .comment-accent {
  opacity: 1;
}

.comment-content {
  padding: var(--space-md) var(--space-lg);
}

/* ─── 留言头部 ─── */
.comment-header {
  display: flex;
  align-items: center;
  gap: var(--space-sm);
  margin-bottom: var(--space-sm);
}

.author-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--primary-glow), rgba(196, 149, 106, 0.1));
  border: 1px solid var(--glass-border);
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: var(--font-display);
  font-size: 0.85rem;
  color: var(--primary);
  flex-shrink: 0;
}

.comment-meta {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.comment-author {
  font-weight: 700;
  font-size: 0.95rem;
  color: var(--ink-dark);
  letter-spacing: 0.02em;
}

.comment-date {
  font-size: 0.75rem;
  color: var(--ink-faint);
  font-variant-numeric: tabular-nums;
}

/* ─── 留言内容 ─── */
.comment-body {
  margin: 0;
  line-height: 1.9;
  color: var(--ink-medium);
  font-size: 0.95rem;
  padding-left: calc(36px + var(--space-sm));
}

/* ─── 空状态 ─── */
.empty-state {
  text-align: center;
  padding: var(--space-xl) 0;
}

.empty-icon {
  font-size: 2.5rem;
  margin-bottom: var(--space-sm);
  opacity: 0.3;
}

.empty-state p {
  margin: 4px 0;
  color: var(--ink-faint);
}

.empty-sub {
  font-size: 0.85rem;
  font-style: italic;
}

/* ─── 分页 ─── */
.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: var(--space-md);
  margin-top: var(--space-xl);
}

.nav-btn {
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 1px solid var(--glass-border);
  border-radius: 50%;
  color: var(--ink-light);
  padding: 0;
}

.nav-btn:hover:not(:disabled) {
  background: var(--primary-glow);
  border-color: var(--primary-light);
  color: var(--primary);
}

.nav-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.page-dots {
  display: flex;
  gap: 8px;
  align-items: center;
}

.page-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--glass-border);
  cursor: pointer;
  transition: all 0.3s;
}

.page-dot.active {
  background: var(--primary);
  transform: scale(1.3);
  box-shadow: 0 0 8px var(--primary-glow);
}

.page-dot:hover:not(.active) {
  background: var(--ink-faint);
}

/* ─── 页脚 ─── */
.board-footer {
  text-align: center;
  margin-top: var(--space-lg);
  font-size: 0.75rem;
  color: var(--ink-faint);
  letter-spacing: 0.1em;
}

.footer-sep {
  margin: 0 8px;
  opacity: 0.5;
}

/* ─── 动画 ─── */
.comment-reveal-enter-active {
  transition: all 0.5s cubic-bezier(0.23, 1, 0.32, 1);
}

.comment-reveal-leave-active {
  transition: all 0.3s cubic-bezier(0.4, 0, 1, 1);
}

.comment-reveal-enter-from {
  opacity: 0;
  transform: translateY(20px);
}

.comment-reveal-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.fade-slide-enter-active {
  transition: all 0.4s cubic-bezier(0.23, 1, 0.32, 1);
}

.fade-slide-leave-active {
  transition: all 0.3s ease;
}

.fade-slide-enter-from {
  opacity: 0;
  transform: translateX(-10px);
}

.fade-slide-leave-to {
  opacity: 0;
}

/* ─── 深色模式覆盖 ─── */
@media (prefers-color-scheme: dark) {
  .comment-item {
    background: rgba(30, 25, 40, 0.5);
  }

  .styled-input,
  .styled-textarea {
    background: rgba(30, 25, 40, 0.4);
  }

  .styled-input:focus,
  .styled-textarea:focus {
    background: rgba(30, 25, 40, 0.6);
  }

  .glass-card::before {
    background: linear-gradient(180deg, rgba(160, 143, 212, 0.03) 0%, transparent 100%);
  }
}

/* ─── 响应式 ─── */
@media (max-width: 640px) {
  .glass-card {
    padding: var(--space-lg) var(--space-md);
    border-radius: var(--radius-lg);
  }

  .board-header h1 {
    font-size: 2rem;
  }

  .board-header h1::before,
  .board-header h1::after {
    width: 24px;
  }

  .input-row {
    flex-direction: column;
    gap: var(--space-sm);
  }

  .name-group {
    flex: none;
  }

  .comment-content {
    padding: var(--space-sm) var(--space-md);
  }

  .comment-body {
    padding-left: 0;
  }

  .submit-btn {
    width: 100%;
    justify-content: center;
  }

  .form-footer {
    flex-direction: column;
    gap: var(--space-sm);
  }
}
</style>
