<template>
  <div class="comment-board">
    <h1>留言板</h1>

    <form @submit.prevent="submitComment">
      <input v-model="name" placeholder="你的名字" required />
      <textarea v-model="message" placeholder="留言内容" required></textarea>
      <button type="submit">提交留言</button>
    </form>

    <div id="comments-container">
      <div v-for="comment in comments" :key="comment.id" class="comment">
        <strong>{{ comment.name }}</strong>: {{ comment.message }} ({{ formatDate(comment.timestamp) }})
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';

// 替换为你的 Gist ID 和 GitHub Personal Access Token
const gistId =import.meta.env.VITE_GIST_ID;
const token = import.meta.env.VITE_GITHUB_TOKEN; // 使用环境变量来存储敏感信息

const name = ref('');
const message = ref('');
const comments = ref([]);

// 获取留言
async function fetchComments() {
  try {
    const response = await axios.get(`https://api.github.com/gists/${gistId}`);
    comments.value = JSON.parse(response.data.files['comments.json'].content).comments || [];
  } catch (error) {
    console.error('Failed to fetch comments:', error);
  }
}

// 提交留言
async function submitComment() {
  if (!name.value.trim() || !message.value.trim()) {
    alert('请填写完整的信息！');
    return;
  }

  try {
    // 获取现有留言
    const response = await axios.get(`https://api.github.com/gists/${gistId}`);
    let existingComments = JSON.parse(response.data.files['comments.json'].content).comments || [];

    // 添加新留言
    const newComment = {
      id: Date.now(),
      name: name.value,
      message: message.value,
      timestamp: new Date().toISOString()
    };
    existingComments.push(newComment);

    // 更新 Gist
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

    alert('留言提交成功！');
    name.value = '';
    message.value = '';
    fetchComments(); // 刷新留言列表
  } catch (error) {
    console.error('Failed to submit comment:', error);
    alert('留言提交失败，请重试。');
  }
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
</style>