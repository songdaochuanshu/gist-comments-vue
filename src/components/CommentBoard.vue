<template>
  <div class="comment-board">
    <h1>留言板</h1>

    <form @submit.prevent="submitComment">
      <input v-model="name" placeholder="你的名字（最多20个字符）" maxlength="20" required />
      <textarea v-model="message" placeholder="留言内容（最多200个字符）" maxlength="200" required></textarea>
      <button type="submit" :disabled="isSubmitting || isCoolingDown">
        {{ isSubmitting ? '提交中...' : '提交留言' }}
      </button>
      <span v-if="isCoolingDown" class="cooldown">请等待 {{ cooldownTime }} 秒后再次提交</span>
    </form>

    <div v-if="isLoading" class="loading">加载中...</div>
    <div v-else id="comments-container">
      <div v-for="comment in paginatedComments" :key="comment.id" class="comment">
        <strong>{{ comment.name }}</strong>: {{ comment.message }} ({{ formatDate(comment.timestamp) }})
      </div>
      <div class="pagination">
        <button @click="prevPage" :disabled="currentPage === 1">上一页</button>
        <span>第 {{ currentPage }} 页</span>
        <button @click="nextPage" :disabled="currentPage === totalPages">下一页</button>
      </div>
    </div>

    <!-- 模态框组件 -->
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
const comments = ref([]); // 从服务器获取的留言
const localComments = ref([]); // 本地缓存的留言
const isSubmitting = ref(false);
const isLoading = ref(false);
const showModal = ref(false);
const modalTitle = ref('');
const modalMessage = ref('');
const currentPage = ref(1);
const pageSize = 10; // 每页显示10条留言
const cooldownDuration = 5000; // 冷却时间5秒
const cooldownTime = ref(0);
const isCoolingDown = ref(false);

let etag;

// 输入验证函数
function validateInput() {
  if (!name.value.trim() || !message.value.trim()) {
    showNotification('提示', '请填写完整的信息！');
    return false;
  }
  if (name.value.trim().length > 20 || message.value.trim().length > 200) {
    showNotification('提示', '用户名或留言内容超出长度限制！');
    return false;
  }
  return true;
}

// 获取留言
async function fetchComments() {
  isLoading.value = true;
  try {
    const headers = etag ? { 'If-None-Match': etag } : {};
    const response = await axios.get(`https://api.github.com/gists/${gistId}`, { headers });

    if (response.status === 200) {
      // 获取服务器留言并反转
      const commentsData = JSON.parse(response.data.files['comments.json'].content).comments || [];
      comments.value = commentsData.reverse(); // 反转数组
      etag = response.headers.etag; // 更新 ETag
    } else if (response.status === 304) {
      console.log('数据未修改，使用缓存');
    }
  } catch (error) {
    if (error.response && error.response.status === 304) {
      console.log('数据未修改，使用缓存');
    } else {
      console.error('Failed to fetch comments:', error);
      showNotification('错误', '加载留言失败，请重试。');
    }
  } finally {
    isLoading.value = false;
  }
}

// 提交留言
async function submitComment() {
  if (!validateInput()) return; // 调用 validateInput 函数

  isSubmitting.value = true;

  try {
    const newComment = {
      id: Date.now(), // 使用时间戳作为唯一 ID
      name: name.value.trim(),
      message: message.value.trim(),
      timestamp: new Date().toISOString()
    };

    // 将新留言添加到本地缓存
    localComments.value.unshift(newComment); // 添加到数组开头

    // 提交到服务器
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
      headers: {
        Authorization: `token ${token}`
      }
    });

    showNotification('成功', '留言提交成功！');
    name.value = '';
    message.value = '';

    // 提交成功后强制更新 ETag
    etag = null; // 清除旧的 ETag
    await fetchComments(); // 重新获取留言
  } catch (error) {
    console.error('Failed to submit comment:', error);
    showNotification('错误', '留言提交失败，请重试。');
  } finally {
    startCooldown();
    isSubmitting.value = false;
  }
}

// 合并显示的留言
const displayedComments = computed(() => {
  // 合并本地缓存和服务器留言，并去重
  const allComments = [...localComments.value, ...comments.value];
  const uniqueComments = allComments.filter((comment, index, self) =>
    index === self.findIndex((c) => c.id === comment.id)
  );
  return uniqueComments;
});

// 分页逻辑
const totalPages = computed(() => Math.ceil(displayedComments.value.length / pageSize));
const paginatedComments = computed(() => {
  const start = (currentPage.value - 1) * pageSize;
  const end = start + pageSize;
  return displayedComments.value.slice(start, end);
});

function prevPage() {
  if (currentPage.value > 1) currentPage.value--;
}

function nextPage() {
  if (currentPage.value < totalPages.value) currentPage.value++;
}

// 冷却时间
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

// 模态框
function showNotification(title, message) {
  modalTitle.value = title;
  modalMessage.value = message;
  showModal.value = true;
}

function hideModal() {
  showModal.value = false;
}

// 格式化日期
function formatDate(dateString) {
  return new Date(dateString).toLocaleString();
}

// 初始化加载留言
onMounted(fetchComments);
</script>

<style scoped>
.comment-board {
  font-family: Arial, sans-serif;
  margin: 20px;
}

#comments-container {
  margin-top: 20px;
}

.comment {
  border-bottom: 1px solid #ccc;
  padding: 10px 0;
}

form {
  margin-bottom: 20px;
}

input, textarea {
  width: 100%;
  padding: 8px;
  margin: 5px 0;
  box-sizing: border-box;
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

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.loading {
  text-align: center;
  margin: 20px 0;
}

.cooldown {
  margin-left: 10px;
  color: #888;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
}

.pagination button {
  margin: 0 10px;
}
</style>