<template>
  <view class="app-container">
    <!-- 搜索区域 -->
    <view class="search-container">
      <view class="filter-header" @click="toggleFilter">
        <text>🔍 筛选条件</text>
        <text>{{ showFilter ? '▲' : '▼' }}</text>
      </view>

      <view class="filter-content" v-if="showFilter">
        <view class="filter-item">
          <text class="filter-label">岗位ID</text>
          <input
            type="text"
            class="filter-input"
            placeholder="请输入岗位ID"
            v-model="queryParams.positionId"
            @confirm="handleQuery"
          />
        </view>

        <view class="filter-item">
          <text class="filter-label">岗位名称</text>
          <input
            type="text"
            class="filter-input"
            placeholder="请输入岗位名称"
            v-model="queryParams.positionName"
            @confirm="handleQuery"
          />
        </view>

        <view class="filter-buttons">
          <button class="filter-btn reset" @click="handleResetQuery">重置</button>
          <button class="filter-btn confirm" @click="handleQuery">搜索</button>
        </view>
      </view>
    </view>

    <!-- 操作按钮 -->
    <view class="action-buttons">
      <button
        :v-has-perm="['aioveuPosition:aioveu-position:add']"
        class="action-btn add"
        @click="handleOpenDialog()"
      >
        <text>+</text>
        <text>新增</text>
      </button>
      <button
        :v-has-perm="['aioveuPosition:aioveu-position:delete']"
        class="action-btn delete"
        :disabled="removeIds.length === 0"
        @click="handleDelete()"
      >
        <text>🗑️</text>
        <text>删除</text>
      </button>
    </view>

    <!-- 岗位列表 -->
    <view class="list-container">
      <view v-if="loading" class="loading-state">
        <text>加载中...</text>
      </view>

      <view v-else-if="pageData.length === 0" class="empty-state">
        <text class="empty-icon">📁</text>
        <text class="empty-text">暂无岗位数据</text>
      </view>

      <view
        v-else
        v-for="item in pageData"
        :key="item.positionId"
        class="position-card"
      >
        <view class="card-header">
          <text class="position-name">{{ item.positionName }}</text>
          <text class="position-id">ID: {{ item.positionId }}</text>
        </view>

        <view class="card-content">
          <view class="info-item">
            <text class="info-label">所属部门:</text>
            <text class="info-value">{{ item.deptName || '无' }}</text>
          </view>
          <view class="info-item">
            <text class="info-label">职级:</text>
            <text class="info-value">{{ item.positionLevel }}</text>
          </view>
          <view class="info-item">
            <text class="info-label">岗位描述:</text>
            <text class="info-value">{{ item.description || '无' }}</text>
          </view>
          <view class="info-item">
            <text class="info-label">创建时间:</text>
            <text class="info-value">{{ item.createTime }}</text>
          </view>
        </view>

        <view class="card-footer">
          <button
            :v-has-perm="['aioveuPosition:aioveu-position:edit']"
            class="action-btn edit"
            @click="handleOpenDialog(item.positionId)"
          >
            <text>✏️</text>
            <text>编辑</text>
          </button>
          <button
            :v-has-perm="['aioveuPosition:aioveu-position:delete']"
            class="action-btn delete"
            @click="handleDelete(item.positionId)"
          >
            <text>🗑️</text>
            <text>删除</text>
          </button>
        </view>
      </view>

      <!-- 分页 -->
      <view class="pagination" v-if="total > 0">
        <button
          class="pagination-btn"
          :disabled="queryParams.pageNum <= 1"
          @click="prevPage"
        >
          上一页
        </button>
        <text class="page-info">
          {{ queryParams.pageNum }} / {{ Math.ceil(total / queryParams.pageSize) }}
        </text>
        <button
          class="pagination-btn"
          :disabled="queryParams.pageNum * queryParams.pageSize >= total"
          @click="nextPage"
        >
          下一页
        </button>
      </view>
    </view>

    <!-- 表单弹窗 -->
    <view class="dialog-mask" v-if="dialog.visible">
      <view class="dialog-container">
        <view class="dialog-header">
          <text class="dialog-title">{{ dialog.title }}</text>
          <text class="dialog-close" @click="handleCloseDialog">✕</text>
        </view>

        <view class="dialog-body">
          <view class="form-item">
            <text class="form-label">岗位名称</text>
            <input
              type="text"
              class="form-input"
              placeholder="请输入岗位名称"
              v-model="formData.positionName"
            />
          </view>

          <view class="form-item">
            <text class="form-label">所属部门</text>
            <input
              type="text"
              class="form-input"
              placeholder="请输入所属部门"
              v-model="formData.deptName"
            />
          </view>

          <view class="form-item">
            <text class="form-label">职级</text>
            <input
              type="text"
              class="form-input"
              placeholder="请输入职级"
              v-model="formData.positionLevel"
            />
          </view>

          <view class="form-item">
            <text class="form-label">岗位描述</text>
            <textarea
              class="form-textarea"
              placeholder="请输入岗位描述"
              v-model="formData.description"
            />
          </view>
        </view>

        <view class="dialog-footer">
          <button class="dialog-btn cancel" @click="handleCloseDialog">取消</button>
          <button class="dialog-btn confirm" @click="handleSubmit">提交</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue';
import AioveuPositionAPI, {
  AioveuPositionPageVO,
  AioveuPositionForm,
  AioveuPositionPageQuery
} from "@/packageC/api/aioveuPosition/aioveu-position";

const loading = ref(false);
const removeIds = ref<Array<number | undefined>>([]);
const total = ref(0);
const showFilter = ref(false);

const queryParams = reactive<AioveuPositionPageQuery>({
  pageNum: 1,
  pageSize: 10,
});

const pageData = ref<AioveuPositionPageVO[]>([]);

const dialog = reactive({
  title: "",
  visible: false,
});

const formData = reactive<AioveuPositionForm>({
  positionName: '',
  deptName: '',
  positionLevel: undefined,
  description: undefined
});

// 切换筛选显示
const toggleFilter = () => {
  showFilter.value = !showFilter.value;
};

// 查询岗位
const handleQuery = () => {
  loading.value = true;
  AioveuPositionAPI.getPage(queryParams)
    .then((data) => {
      pageData.value = data.list;
      total.value = data.total;
    })
    .finally(() => {
      loading.value = false;
    });
};

// 重置查询
const handleResetQuery = () => {
  queryParams.pageNum = 1;
  queryParams.positionId = Number(undefined);
  queryParams.positionName = '';
  pageData.value = [];
  handleQuery();
};

// 上一页
const prevPage = () => {
  if (queryParams.pageNum > 1) {
    queryParams.pageNum--;
    handleQuery();
  }
};

// 下一页
const nextPage = () => {
  if (queryParams.pageNum * queryParams.pageSize < total.value) {
    queryParams.pageNum++;
    handleQuery();
  }
};

// 行选择变化
const handleSelectionChange = (selection: AioveuPositionPageVO[]) => {
  removeIds.value = selection
    .map(item => item.positionId)
    .filter((positionId): positionId is number => positionId !== undefined && positionId !== null) as number[];
};

// 打开弹窗
const handleOpenDialog = (positionId?: number) => {

  editingpositionId.value = positionId;

  if (positionId) {
    dialog.title = "修改岗位";
    AioveuPositionAPI.getFormData(positionId).then((data) => {
      Object.assign(formData, data);
      dialog.visible = true;
    });
  } else {
    dialog.title = "新增岗位";
    formData.positionName = '';
    formData.deptName = '';
    formData.positionLevel = undefined;
    formData.description = undefined;
    dialog.visible = true;
  }
};

// 在组件中添加一个变量存储当前编辑的positionId
const editingpositionId = ref<number | undefined>(undefined);

// 关闭弹窗
const handleCloseDialog = () => {
  dialog.visible = false;
};

// 提交表单
const handleSubmit = () => {
  if (!formData.positionName) {
    uni.showToast({
      title: "请输入岗位名称",
      icon: "none"
    });
    return;
  }


  // 确保 positionLevel 是数字类型
  if (formData.positionLevel !== undefined) {
    formData.positionLevel = Number(formData.positionLevel);
  }

  // 确保 deptId 是数字类型
  if (formData.deptId !== undefined) {
    formData.deptId = Number(formData.deptId);
  }

  uni.showLoading({ title: '提交中...' });


  const id = editingpositionId.value; // 使用存储的positionId

  if (id) {
    // 更新
    AioveuPositionAPI.update(id, formData)
      .then(() => {
        uni.showToast({
          title: "修改成功",
          icon: "success"
        });
        handleCloseDialog();
        handleResetQuery();
      });
  } else {
    // 新增
    AioveuPositionAPI.add(formData)
      .then(() => {
        uni.showToast({
          title: "新增成功",
          icon: "success"
        });
        handleCloseDialog();
        handleResetQuery();
      })
      .finally(() => uni.hideLoading());
  }
};

// 删除岗位
const handleDelete = (positionId?: number) => {
  const ids = positionId ? [positionId] : removeIds.value;

  if (ids.length === 0) {
    uni.showToast({
      title: "请选择要删除的岗位",
      icon: "none"
    });
    return;
  }

  uni.showModal({
    title: '提示',
    content: '确认删除选中的岗位吗？',
    success: (res) => {
      if (res.confirm) {
        AioveuPositionAPI.deleteByIds(ids.join(","))
          .then(() => {
            uni.showToast({
              title: "删除成功",
              icon: "success"
            });
            handleResetQuery();
          });
      }
    }
  });
};

onMounted(() => {
  handleQuery();
});
</script>

<style lang="scss" scoped>
.app-container {
  padding: 20rpx;
  background-color: #f5f7fa;
  min-height: 100vh;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

/* 筛选区域 */
.search-container {
  background: white;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  margin-bottom: 24rpx;
  overflow: hidden;
}

.filter-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx;
  font-size: 32rpx;
  font-weight: 600;
  color: #1a1a1a;
  background-color: #f8f9fa;
  border-bottom: 1rpx solid #eaeaea;
}

.filter-content {
  padding: 24rpx;
}

.filter-item {
  margin-bottom: 32rpx;
}

.filter-label {
  display: block;
  font-size: 28rpx;
  color: #666;
  margin-bottom: 12rpx;
  font-weight: 500;
}

.filter-input {
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 20rpx;
  font-size: 28rpx;
  width: 100%;
  background-color: white;
}

.filter-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 24rpx;
}

.filter-btn {
  flex: 1;
  margin: 0 12rpx;
  font-size: 28rpx;
  border-radius: 12rpx;
  padding: 20rpx 0;
  border: none;

  &.reset {
    background-color: #f5f5f5;
    color: #666;
  }

  &.confirm {
    background-color: #5e72e4;
    color: white;
  }
}

/* 操作按钮 */
.action-buttons {
  display: flex;
  gap: 24rpx;
  margin-bottom: 24rpx;
}

.action-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12rpx;
  padding: 24rpx 0;
  border-radius: 12rpx;
  font-size: 28rpx;
  font-weight: 500;
  border: none;

  &.add {
    background: #2dce89;
    color: white;
  }

  &.edit {
    background: #5e72e4;
    color: white;
  }

  &.delete {
    background: #f5365c;
    color: white;

    &[disabled] {
      background: #cccccc;
      opacity: 0.7;
    }
  }
}

/* 岗位列表 */
.list-container {
  background: white;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  overflow: hidden;
}

.loading-state {
  padding: 60rpx;
  text-align: center;
  color: #666;
  font-size: 28rpx;
}

.empty-state {
  padding: 80rpx 24rpx;
  text-align: center;
  color: #666;

  .empty-icon {
    font-size: 100rpx;
    color: #cbd5e0;
    margin-bottom: 24rpx;
    display: block;
  }

  .empty-text {
    font-size: 32rpx;
  }
}

.position-card {
  border-bottom: 1rpx solid #f1f1f1;
  padding: 32rpx;

  &:last-child {
    border-bottom: none;
  }
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24rpx;

  .position-name {
    font-size: 36rpx;
    font-weight: 700;
    color: #1a1a1a;
  }

  .position-id {
    font-size: 28rpx;
    color: #5e72e4;
    background-color: rgba(94, 114, 228, 0.1);
    padding: 8rpx 20rpx;
    border-radius: 30rpx;
  }
}

.card-content {
  margin-bottom: 24rpx;
}

.info-item {
  display: flex;
  margin-bottom: 20rpx;
  font-size: 30rpx;

  .info-label {
    color: #666;
    width: 180rpx;
    font-weight: 500;
  }

  .info-value {
    color: #333;
    flex: 1;
  }
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 24rpx;
  border-top: 1rpx solid #f1f1f1;

  .action-btn {
    flex: 1;
    margin: 0 8rpx;
    padding: 16rpx 0;
    font-size: 26rpx;
  }
}

/* 分页 */
.pagination {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 30rpx;
  border-top: 1rpx solid #f1f1f1;

  .pagination-btn {
    flex: 1;
    margin: 0 16rpx;
    padding: 20rpx 0;
    border-radius: 12rpx;
    font-size: 28rpx;
    background: #f8f9fa;
    color: #333;

    &[disabled] {
      opacity: 0.5;
    }
  }

  .page-info {
    font-size: 28rpx;
    color: #666;
  }
}

/* 弹窗 */
.dialog-mask {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.dialog-container {
  background: white;
  border-radius: 16rpx;
  width: 90%;
  max-width: 600rpx;
  overflow: hidden;
  box-shadow: 0 20rpx 60rpx rgba(0, 0, 0, 0.15);
}

.dialog-header {
  padding: 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1rpx solid #eaeaea;

  .dialog-title {
    font-size: 36rpx;
    font-weight: 600;
    color: #1a1a1a;
  }

  .dialog-close {
    font-size: 40rpx;
    color: #666;
    cursor: pointer;
  }
}

.dialog-body {
  padding: 30rpx;
  max-height: 60vh;
  overflow-y: auto;
}

.form-item {
  margin-bottom: 32rpx;
}

.form-label {
  display: block;
  margin-bottom: 16rpx;
  font-size: 30rpx;
  color: #666;
  font-weight: 500;
}

.form-input, .form-textarea {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 24rpx;
  font-size: 30rpx;
}

.form-textarea {
  height: 200rpx;
}

.dialog-footer {
  padding: 24rpx 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: flex-end;
  gap: 24rpx;
  border-top: 1rpx solid #eaeaea;

  .dialog-btn {
    padding: 20rpx 40rpx;
    border-radius: 12rpx;
    font-size: 30rpx;
    border: none;

    &.cancel {
      background: #f5f5f5;
      color: #666;
    }

    &.confirm {
      background: #5e72e4;
      color: white;
    }
  }
}
</style>
