<template>
  <view class="app-container">


<!--    在Uniapp中，列表渲染通常使用<view>组件配合v-for指令来实现。移动端界面设计通常采用垂直列表布局，每个列表项独立显示，这样更符合移动设备的交互习惯。但是，Uniapp也支持更灵活的布局方式，比如网格布局（Grid），可以根据需要展示多行多列。

根据您的需求，您希望考勤管理列表在手机上展示，并且要求一行一行渲染（即每个考勤记录占一行）。这种设计在移动端非常常见，因为手机屏幕较窄，单行显示可以清晰展示信息。

但是，如果您希望以卡片形式展示，并且每行显示多个卡片（比如每行两个卡片），也是可以实现的。这可以通过使用网格布局（Grid）或者Flex布局来实现。

考虑到您之前提供的PC端页面是表格形式，在移动端转换为卡片列表是常见的做法。每个卡片包含一个考勤记录的详细信息。

在之前的实现中，我采用了卡片列表的形式，每个考勤记录用一个卡片展示，卡片内部分为多个行（每个信息项一行）。这种设计在移动端用户体验良好。

如果您希望改变布局，比如每行显示多个卡片，可以调整样式，使用网格布局。但是，由于考勤记录的信息量较大（包含多个字段），在手机屏幕上每行显示多个卡片可能会导致信息显示不全，影响用户体验。

因此，我建议保持一行一个卡片的布局。这样可以在卡片内清晰地展示所有信息，并且方便用户操作（每个卡片有操作按钮）。-->

<!--    目前，我提供的列表页代码中，每个考勤记录使用一个卡片（.attendance-card）展示，卡片之间是垂直排列的。这是移动端常见的列表布局。

如果您希望每行显示两个卡片，可以修改样式如下：

1.
将卡片容器设置为flex布局，并允许换行。

2.
设置每个卡片的宽度为50%减去间距。-->
    <!-- 搜索区域 -->
    <view class="search-container">
      <view class="filter-header" @click="toggleFilter">
        <text class="filter-icon">🔍</text>
        <text class="filter-title">筛选条件</text>
        <text class="filter-toggle">{{ showFilter ? '▲' : '▼' }}</text>
      </view>

      <view v-if="showFilter" class="filter-content">
        <view class="filter-item">
          <text class="filter-label">部门ID</text>
          <input
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
        v-hasPerm="['aioveuDepartment:aioveu-department:add']"
        class="action-btn add"
        @click="handleOpenDialog()"
      >
        <text class="btn-icon">+</text>
        <text>新增部门</text>
      </button>
      <button
        v-hasPerm="['aioveuDepartment:aioveu-department:delete']"
        class="action-btn delete"
        :disabled="removeIds.length === 0"
        @click="handleDelete()"
      >
        <text class="btn-icon">🗑️</text>
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
          <view class="update-time">更新时间: {{ item.updateTime }}</view>
          <view class="card-actions">
            <button
              v-hasPerm="['aioveuDepartment:aioveu-department:edit']"
              class="action-btn edit"
              @click="handleOpenDialog(item.deptId)"
            >
              <text class="btn-icon">✏️</text>
              <text>编辑</text>
            </button>
            <button
              v-hasPerm="['aioveuDepartment:aioveu-department:delete']"
              class="action-btn delete"
              @click="handleDelete(item.deptId)"
            >
              <text class="btn-icon">🗑️</text>
              <text>删除</text>
            </button>
          </view>
        </view>
      </view>

      <!-- 加载更多 -->
      <view class="loadmore">
        <button
          v-if="loadMoreState === 'loading'"
          class="load-btn"
          @click="handleQuery"
        >
          <text>加载中...</text>
        </button>
        <text v-else-if="loadMoreState === 'finished'">没有更多数据了</text>
        <view v-else-if="loadMoreState === 'error'">
          <text>加载失败</text>
          <button class="reload-btn" @click="handleQuery">重新加载</button>
        </view>
      </view>
    </view>

    <!-- 表单弹窗 -->
    <view v-if="dialog.visible" class="dialog-mask">
      <view class="dialog-container">
        <view class="dialog-header">
          <text class="dialog-title">{{ dialog.title }}</text>
          <text class="dialog-close" @click="handleCloseDialog">✕</text>
        </view>

        <view class="dialog-body">
          <view class="form-item">
            <text class="form-label">部门名称</text>
            <input
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
          <button class="dialog-btn confirm" @click="handleSubmit">确定</button>
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
} from "@/api/aioveuDepartment/aioveu-department";

// 筛选相关状态
const showFilter = ref(false);
const queryParams = reactive<AioveuDepartmentPageQuery>({
  pageNum: 1,
  pageSize: 10,
  deptId: '',
  deptName: '',
  parentDeptName: ''
});
const deptNameIndex = ref(-1);
const parentDeptIndex = ref(-1);

// 列表相关状态
const pageData = ref<AioveuDepartmentPageVO[]>([]);
const total = ref(0);
const loadMoreState = ref<'loading' | 'finished' | 'error'>('loading');
const removeIds = ref<Array<number | undefined>>([]);

// 弹窗相关状态
const dialog = reactive({
  title: '',
  visible: false
});
const formData = reactive<AioveuDepartmentForm>({});
const editingDeptId = ref<number | null>(null);

// 部门选项
const deptOptions = ref<DeptOptionVO[]>([]);

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
  queryParams.deptId = '';
  queryParams.deptName = '';
  queryParams.parentDeptName = '';
  deptNameIndex.value = -1;
  parentDeptIndex.value = -1;
  pageData.value = [];
  handleQuery();
};

// 执行查询
const handleQuery = () => {
  loadMoreState.value = 'loading';

  // 模拟API调用
  setTimeout(() => {
    try {
      const mockData = generateMockData();

      if (queryParams.pageNum === 1) {
        pageData.value = mockData;
      } else {
        pageData.value = [...pageData.value, ...mockData];
      }

      total.value = 50;
      loadMoreState.value = mockData.length < queryParams.pageSize ? 'finished' : 'loading';
    } catch (error) {
      loadMoreState.value = 'error';
      console.error('查询失败:', error);
    }
  }, 1000);
};

// 打开弹窗
const handleOpenDialog = (deptId?: number) => {
  editingDeptId.value = deptId || null;

  if (deptId) {
    dialog.title = '修改部门';
    // 模拟获取表单数据
    const item = pageData.value.find(d => d.deptId === deptId);
    if (item) {
      formData.deptName = item.deptName;
      formData.parentDeptName = item.parentDeptName || '';

      // 设置上级部门索引
      const index = deptOptions.value.findIndex(
        d => d.deptName === item.parentDeptName
      );
      parentDeptIndex.value = index;
    }
  } else {
    dialog.title = '新增部门';
    formData.deptName = '';
    formData.parentDeptName = '';
    parentDeptIndex.value = -1;
  }

  dialog.visible = true;
};

// 关闭弹窗
const handleCloseDialog = () => {
  dialog.visible = false;
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

  // 模拟提交操作
  setTimeout(() => {
    if (editingDeptId.value) {
      // 更新部门
      pageData.value = pageData.value.map(item => {
        if (item.deptId === editingDeptId.value) {
          return {
            ...item,
            deptName: formData.deptName,
            parentDeptName: formData.parentDeptName,
            updateTime: formatDate(new Date())
          };
        }
        return item;
      });

      uni.showToast({
        title: '修改成功',
        icon: 'success'
      });
    } else {
      // 新增部门
      const newDept: Department = {
        deptId: Math.max(...pageData.value.map(d => d.deptId), 0) + 1,
        deptName: formData.deptName,
        parentDeptName: formData.parentDeptName,
        createTime: formatDate(new Date()),
        updateTime: formatDate(new Date())
      };

      pageData.value = [newDept, ...pageData.value];
      uni.showToast({
        title: '新增成功',
        icon: 'success'
      });
    }

    handleCloseDialog();
    loadDepartments();
    uni.hideLoading();
  }, 1000);
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
        pageData.value = pageData.value.filter(
          item => !ids.includes(item.deptId)
        );
        removeIds.value = [];
        uni.showToast({
          title: '删除成功',
          icon: 'success'
        });
        loadDepartments();
      }
    }
  });
};

// 加载部门选项
const loadDepartments = () => {
  // 模拟获取部门选项
  deptOptions.value = [
    { deptId: 1, deptName: '技术部' },
    { deptId: 2, deptName: '市场部' },
    { deptId: 3, deptName: '财务部' },
    { deptId: 4, deptName: '人力资源部' },
    { deptId: 5, deptName: '行政部' }
  ];
};

// 生成模拟数据
const generateMockData = (): AioveuDepartmentPageVO[] => {
  const data: AioveuDepartmentPageVO[] = [];
  const count = queryParams.pageSize;
  const startIndex = (queryParams.pageNum - 1) * count;

  for (let i = 1; i <= count; i++) {
    const id = startIndex + i;
    data.push({
      deptId: id,
      deptName: `部门${id}`,
      parentDeptName: id % 3 === 0 ? '技术部' : id % 3 === 1 ? '市场部' : '行政部',
      createTime: formatDate(new Date(Date.now() - Math.random() * 30 * 24 * 60 * 60 * 1000)),
      updateTime: formatDate(new Date(Date.now() - Math.random() * 7 * 24 * 60 * 60 * 1000))
    });
  }

  return data;
};

// 格式化日期
const formatDate = (date: Date): string => {
  const year = date.getFullYear();
  const month = (date.getMonth() + 1).toString().padStart(2, '0');
  const day = date.getDate().toString().padStart(2, '0');
  const hours = date.getHours().toString().padStart(2, '0');
  const minutes = date.getMinutes().toString().padStart(2, '0');

  return `${year}-${month}-${day} ${hours}:${minutes}`;
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
  font-family: 'PingFang SC', 'Helvetica Neue', Arial, sans-serif;
}

/* 搜索区域 */
.search-container {
  background: #ffffff;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  margin-bottom: 24rpx;
  overflow: hidden;
}

.filter-header {
  display: flex;
  align-items: center;
  padding: 24rpx;
  background: #f8f9fa;
  border-bottom: 1rpx solid #eaeaea;
}

.filter-icon {
  font-size: 36rpx;
  margin-right: 12rpx;
}

.filter-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #1a1a1a;
  flex: 1;
}

.filter-toggle {
  font-size: 32rpx;
  color: #666;
}

.filter-content {
  padding: 24rpx;
}

.filter-item {
  margin-bottom: 30rpx;
}

.filter-label {
  display: block;
  font-size: 28rpx;
  color: #666;
  margin-bottom: 12rpx;
  font-weight: 500;
}

.filter-input, .filter-picker {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 20rpx;
  font-size: 28rpx;
  background: #ffffff;
}

.picker-view {
  height: 40rpx;
  line-height: 40rpx;
}

.filter-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 30rpx;
}

.filter-btn {
  flex: 1;
  margin: 0 12rpx;
  font-size: 28rpx;
  border-radius: 12rpx;
  padding: 20rpx 0;
  border: none;

  &.reset {
    background: #f5f5f5;
    color: #666;
  }

  &.confirm {
    background: #5e72e4;
    color: white;
  }
}

/* 操作按钮 */
.action-buttons {
  display: flex;
  gap: 20rpx;
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

  &.delete {
    background: #f5365c;
    color: white;

    &[disabled] {
      background: #cccccc;
      opacity: 0.7;
    }
  }
}

.btn-icon {
  font-size: 32rpx;
}

/* 部门列表 */
.list-container {
  background: white;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  overflow: hidden;
  margin-bottom: 40rpx;
}

.empty-state {
  padding: 60rpx 20rpx;
  text-align: center;
  color: #666;
}

.empty-icon {
  font-size: 80rpx;
  color: #cbd5e0;
  margin-bottom: 16rpx;
  display: block;
}

.empty-text {
  font-size: 30rpx;
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
}

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

.card-content {
  margin-bottom: 20rpx;
}

.info-item {
  display: flex;
  margin-bottom: 15rpx;
  font-size: 28rpx;
}

.info-label {
  color: #666;
  width: 160rpx;
  font-weight: 500;
}

.info-value {
  color: #333;
  flex: 1;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 20rpx;
  border-top: 1rpx solid #f1f1f1;
}

.update-time {
  font-size: 24rpx;
  color: #999;
  flex: 1;
}

.card-actions {
  display: flex;
  gap: 20rpx;
}

.action-btn {
  padding: 12rpx 24rpx;
  border-radius: 8rpx;
  font-size: 26rpx;
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

/* 加载更多 */
.loadmore {
  padding: 30rpx;
  text-align: center;
  color: #666;
  font-size: 26rpx;
}

.load-btn {
  width: 100%;
  padding: 20rpx;
  background: #f8f9fa;
  border-radius: 12rpx;
  border: none;
}

.reload-btn {
  margin-top: 20rpx;
  padding: 15rpx 30rpx;
  background: #5e72e4;
  color: white;
  border-radius: 12rpx;
  border: none;
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
  border-radius: 24rpx;
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
}

.dialog-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #1a1a1a;
}

.dialog-close {
  font-size: 36rpx;
  color: #666;
}

.dialog-body {
  padding: 30rpx;
}

.form-item {
  margin-bottom: 30rpx;
}

.form-label {
  display: block;
  margin-bottom: 12rpx;
  font-size: 28rpx;
  color: #666;
  font-weight: 500;
}

.form-input, .form-picker {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 20rpx;
  font-size: 28rpx;
  background: #ffffff;
}

.dialog-footer {
  padding: 20rpx 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: flex-end;
  gap: 20rpx;
  border-top: 1rpx solid #eaeaea;
}

.dialog-btn {
  padding: 20rpx 40rpx;
  border-radius: 12rpx;
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
</style>
