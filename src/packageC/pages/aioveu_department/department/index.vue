<template>
  <view class="app-container">
    <!-- 筛选区域 -->
    <view class="filter-container">
      <view class="filter-header" @click="toggleFilter">
        <text class="filter-title">筛选条件</text>
        <text class="filter-icon">{{ showFilter ? '▲' : '▼' }}</text>
      </view>

      <view class="filter-content" v-if="showFilter">
        <view class="filter-item">
          <text class="filter-label">部门ID</text>
          <input
            type="text"
            class="filter-input"
            placeholder="请输入部门ID"
            v-model="queryParams.deptId"
            @confirm="handleQuery"
          />
        </view>

        <view class="filter-item">
          <text class="filter-label">部门名称</text>
          <picker
            class="filter-picker"
            mode="selector"
            :range="deptOptions"
            range-key="deptName"
            :value="deptNameIndex"
            @change="onDeptNameChange"
          >
            <view class="picker-view">
              {{ deptNameIndex >= 0 ? deptOptions[deptNameIndex].deptName : '请选择部门' }}
            </view>
          </picker>
        </view>

        <view class="filter-buttons">
          <button class="filter-btn reset" @click="handleResetQuery">重置</button>
          <button class="filter-btn confirm" @click="handleQuery">确定</button>
        </view>
      </view>
    </view>

    <!-- 操作按钮 -->
    <view class="action-buttons">
      <button
        class="action-btn add"
        :v-has-perm="['aioveuDepartment:aioveu-department:add']"
        @click="handleOpenDialog()"
      >
        <text>+</text>
        <text>新增部门</text>
      </button>
      <button
        class="action-btn delete"
        :v-has-perm="['aioveuDepartment:aioveu-department:delete']"
        :disabled="removeIds.length === 0"
        @click="handleDelete()"
      >
        <text>🗑️</text>
        <text>删除选中</text>
      </button>
    </view>

    <!-- 部门列表 -->
    <view class="list-container">
      <view v-if="pageData.length === 0" class="empty-state">
        <text class="empty-icon">📁</text>
        <text class="empty-text">暂无部门数据</text>
      </view>

      <view
        v-for="item in pageData"
        :key="item.deptId"
        class="department-card"
      >
        <view class="card-header">
          <text class="dept-name">{{ item.deptName }}</text>
          <text class="dept-id">ID: {{ item.deptId }}</text>
        </view>

        <view class="card-content">
          <view class="info-item">
            <text class="info-label">上级部门:</text>
            <text class="info-value">{{ item.parentDeptName || '无' }}</text>
          </view>
          <view class="info-item">
            <text class="info-label">创建时间:</text>
            <text class="info-value">{{ item.createTime }}</text>
          </view>
          <view class="info-item">
            <text class="info-label">更新时间:</text>
            <text class="info-value">{{ item.updateTime }}</text>
          </view>
        </view>

        <view class="card-footer">
          <button
            class="action-btn edit"
            :v-has-perm="['aioveuDepartment:aioveu-department:edit']"
            @click="handleOpenDialog(item.deptId)"
          >
            <text>✏️</text>
            <text>编辑</text>
          </button>
          <button
            class="action-btn delete"
            :v-has-perm="['aioveuDepartment:aioveu-department:delete']"
            @click="handleDelete(item.deptId)"
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
          第 {{ queryParams.pageNum }} 页 / 共 {{ Math.ceil(total / queryParams.pageSize) }} 页
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
            <text class="form-label">部门名称</text>
            <input
              type="text"
              class="form-input"
              placeholder="请输入部门名称"
              v-model="formData.deptName"
            />
          </view>

          <view class="form-item">
            <text class="form-label">上级部门</text>
            <picker
              class="form-picker"
              mode="selector"
              :range="deptOptions"
              range-key="deptName"
              :value="parentDeptIndex"
              @change="onParentDeptChange"
            >
              <view class="picker-view">
                {{ parentDeptIndex >= 0 ? deptOptions[parentDeptIndex].deptName : '请选择上级部门' }}
              </view>
            </picker>
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
import AioveuDepartmentAPI, {
  AioveuDepartmentPageVO,
  AioveuDepartmentForm,
  AioveuDepartmentPageQuery,
  DeptOptionVO
} from "@/packageC/api/aioveuDepartment/aioveu-department";
// } from "../../../api/aioveuDepartment/aioveu-department.js";


const showFilter = ref(false);
const queryParams = reactive<AioveuDepartmentPageQuery>({
  pageNum: 1,
  pageSize: 10,
});
const deptNameIndex = ref(-1);
const parentDeptIndex = ref(-1);

const loading = ref(false);
const removeIds = ref<number[]>([]);
const total = ref(0);

const pageData = ref<AioveuDepartmentPageVO[]>([]);
const deptOptions = ref<DeptOptionVO[]>([]);

const dialog = reactive({
  title: "",
  visible: false,
});

const formData = reactive<AioveuDepartmentForm>({});
const editingDeptId = ref<number | undefined>(undefined);

// 切换筛选显示
const toggleFilter = () => {
  showFilter.value = !showFilter.value;
};

// 部门名称选择变化
const onDeptNameChange = (e: any) => {
  const index = e.detail.value;
  deptNameIndex.value = index;
  queryParams.deptName = deptOptions.value[index]?.deptName || '';
};

// 上级部门选择变化
const onParentDeptChange = (e: any) => {
  const index = e.detail.value;
  parentDeptIndex.value = index;
  const selectedDept = deptOptions.value[index];

  if (dialog.visible) {
    formData.parentDeptName = selectedDept?.deptName || '';
  } else {
    queryParams.parentDeptName = selectedDept?.deptName || '';
  }
};

// 重置查询
const handleResetQuery = () => {
  queryParams.pageNum = 1;
  queryParams.deptId =undefined;
  queryParams.deptName = '';
  queryParams.parentDeptName = '';
  deptNameIndex.value = -1;
  parentDeptIndex.value = -1;
  pageData.value = [];
  handleQuery();
};

// 执行查询
const handleQuery = () => {
  loading.value = true;

  AioveuDepartmentAPI.getPage(queryParams)
    .then((data) => {
      pageData.value = data.list;
      total.value = data.total;
    })
    .finally(() => {
      loading.value = false;
    });
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

// 打开弹窗
const handleOpenDialog = (deptId?: number) => {
  editingDeptId.value = deptId;

  if (deptId) {
    dialog.title = "修改部门";
    AioveuDepartmentAPI.getFormData(deptId).then((data) => {
      Object.assign(formData, data);
      dialog.visible = true;
    });
  } else {
    dialog.title = "新增部门";
    Object.assign(formData, {
      deptName: '',
      parentDeptName: ''
    });
    dialog.visible = true;
  }
};

// 关闭弹窗
const handleCloseDialog = () => {
  dialog.visible = false;
  setTimeout(() => {
    Object.assign(formData, {
      deptName: '',
      parentDeptName: ''
    });
    editingDeptId.value = undefined;
  }, 300);
};

// 提交表单
const handleSubmit = () => {
  if (!formData.deptName) {
    uni.showToast({
      title: '请输入部门名称',
      icon: 'none'
    });
    return;
  }

  uni.showLoading({ title: '提交中...' });

  if (editingDeptId.value) {
    // 更新部门
    AioveuDepartmentAPI.update(editingDeptId.value, formData)
      .then(() => {
        uni.showToast({
          title: '修改成功',
          icon: 'success'
        });
        handleCloseDialog();
        handleQuery();
        loadDepartments();
      })
      .finally(() => {
        uni.hideLoading();
      });
  } else {
    // 新增部门
    AioveuDepartmentAPI.add(formData)
      .then(() => {
        uni.showToast({
          title: '新增成功',
          icon: 'success'
        });
        handleCloseDialog();
        handleQuery();
      })
      .finally(() => {
        uni.hideLoading();
      });
  }
};

// 删除部门
const handleDelete = (deptId?: number) => {
  const ids = deptId ? [deptId] : removeIds.value;

  if (ids.length === 0) {
    uni.showToast({
      title: '请选择要删除的部门',
      icon: 'none'
    });
    return;
  }

  uni.showModal({
    title: '提示',
    content: '确认删除选中的部门吗？',
    success: (res) => {
      if (res.confirm) {
        uni.showLoading({ title: '删除中...' });
        AioveuDepartmentAPI.deleteByIds(ids.join(','))
          .then(() => {
            uni.showToast({
              title: '删除成功',
              icon: 'success'
            });
            handleQuery();
          })
          .finally(() => {
            uni.hideLoading();
          });
      }
    }
  });
};

// 加载部门选项
const loadDepartments = () => {
  AioveuDepartmentAPI.getAllDepartmentOptions()
    .then(response => {
      if (Array.isArray(response)) {
        deptOptions.value = response.map(dept => ({
          deptId: Number(dept.deptId),
          deptName: dept.deptName
        }));
      }
    });
};

onMounted(() => {
  handleQuery();
  loadDepartments();
});
</script>

<style lang="scss" scoped>
.app-container {
  padding: 20rpx;
  background-color: #f5f7fa;
  min-height: 100vh;
}

/* 筛选区域 */
.filter-container {
  background: white;
  border-radius: 12rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  margin-bottom: 24rpx;
  overflow: hidden;
}

.filter-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20rpx;
  font-size: 30rpx;
  font-weight: 600;
  color: #1a1a1a;
  background-color: #f8f9fa;
  border-bottom: 1rpx solid #eaeaea;
}

.filter-content {
  padding: 20rpx;
}

.filter-item {
  margin-bottom: 30rpx;
}

.filter-label {
  display: block;
  font-size: 28rpx;
  color: #666;
  margin-bottom: 10rpx;
}

.filter-input, .filter-picker {
  border: 1rpx solid #e2e8f0;
  border-radius: 8rpx;
  padding: 15rpx 20rpx;
  font-size: 28rpx;
  width: 100%;
  background-color: white;
}

.picker-view {
  height: 40rpx;
  line-height: 40rpx;
}

.filter-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 20rpx;
}

.filter-btn {
  flex: 1;
  margin: 0 10rpx;
  font-size: 28rpx;
  border-radius: 8rpx;
  padding: 15rpx 0;
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
  gap: 20rpx;
  margin-bottom: 24rpx;

  .action-btn {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8rpx;
    padding: 20rpx 0;
    border-radius: 8rpx;
    font-size: 28rpx;
    font-weight: 500;
    border: none;

    &.add {
      background: #2dce89;
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
}

/* 部门列表 */
.list-container {
  background: white;
  border-radius: 12rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  overflow: hidden;
}

.empty-state {
  padding: 60rpx 20rpx;
  text-align: center;
  color: #666;

  .empty-icon {
    font-size: 80rpx;
    color: #cbd5e0;
    margin-bottom: 16rpx;
    display: block;
  }

  .empty-text {
    font-size: 30rpx;
  }
}

.department-card {
  border-bottom: 1rpx solid #f1f1f1;
  padding: 30rpx;

  &:last-child {
    border-bottom: none;
  }
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20rpx;

  .dept-name {
    font-size: 32rpx;
    font-weight: 700;
    color: #1a1a1a;
  }

  .dept-id {
    font-size: 26rpx;
    color: #5e72e4;
    background-color: rgba(94, 114, 228, 0.1);
    padding: 6rpx 16rpx;
    border-radius: 20rpx;
  }
}

.card-content {
  margin-bottom: 20rpx;
}

.info-item {
  display: flex;
  margin-bottom: 15rpx;
  font-size: 28rpx;

  .info-label {
    color: #666;
    width: 160rpx;
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
  gap: 20rpx;
  padding-top: 20rpx;
  border-top: 1rpx solid #f1f1f1;

  .action-btn {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8rpx;
    padding: 15rpx 0;
    border-radius: 8rpx;
    font-size: 28rpx;
    border: none;

    &.edit {
      background: rgba(94, 114, 228, 0.1);
      color: #5e72e4;
    }

    &.delete {
      background: rgba(245, 54, 92, 0.1);
      color: #f5365c;
    }
  }
}

/* 分页 */
.pagination {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 30rpx;

  .pagination-btn {
    padding: 15rpx 30rpx;
    border-radius: 8rpx;
    background: #f5f5f5;
    color: #666;
    font-size: 28rpx;
    border: none;

    &[disabled] {
      opacity: 0.5;
    }
  }

  .page-info {
    font-size: 26rpx;
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
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.15);
}

.dialog-header {
  padding: 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1rpx solid #eaeaea;

  .dialog-title {
    font-size: 32rpx;
    font-weight: 600;
    color: #1a1a1a;
  }

  .dialog-close {
    font-size: 36rpx;
    color: #666;
  }
}

.dialog-body {
  padding: 30rpx;
}

.form-item {
  margin-bottom: 30rpx;
}

.form-label {
  display: block;
  margin-bottom: 10rpx;
  font-size: 28rpx;
  color: #666;
  font-weight: 500;
}

.form-input, .form-picker {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 8rpx;
  padding: 15rpx 20rpx;
  font-size: 28rpx;
}

.dialog-footer {
  padding: 20rpx 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: flex-end;
  gap: 20rpx;
  border-top: 1rpx solid #eaeaea;

  .dialog-btn {
    padding: 15rpx 30rpx;
    border-radius: 8rpx;
    font-size: 28rpx;
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
