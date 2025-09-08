<template>
  <view class="form-container">
    <view class="form-header">
      <text class="form-title">{{ formTitle }}</text>
    </view>

    <scroll-view class="form-body" scroll-y>
      <!-- 员工 -->
      <view class="form-item">
        <text class="form-label">员工</text>
        <picker
          class="form-picker"
          mode="selector"
          :range="employeeOptions"
          range-key="employeeName"
          :value="employeeIndex"
          @change="onEmployeeChange"
        >
          <view class="picker-view">
            {{ employeeIndex >= 0 ? employeeOptions[employeeIndex].employeeName : '请选择员工' }}
          </view>
        </picker>
      </view>

      <!-- 考核年份 -->
      <view class="form-item">
        <text class="form-label">考核年份</text>
        <input
          type="number"
          class="form-input"
          placeholder="请输入考核年份"
          v-model="formData.periodYear"
        />
      </view>

      <!-- 考核季度 -->
      <view class="form-item">
        <text class="form-label">考核季度</text>
        <picker
          class="form-picker"
          mode="selector"
          :range="periodQuarterOptions"
          range-key="label"
          :value="quarterIndex"
          @change="onQuarterChange"
        >
          <view class="picker-view">
            {{ quarterIndex >= 0 ? periodQuarterOptions[quarterIndex].label : '请选择季度' }}
          </view>
        </picker>
      </view>

      <!-- KPI评分 -->
      <view class="form-item">
        <text class="form-label">KPI评分</text>
        <input
          type="number"
          class="form-input"
          placeholder="请输入KPI评分(1-100)"
          v-model="formData.kpiScore"
        />
      </view>

      <!-- 生产率 -->
      <view class="form-item">
        <text class="form-label">生产率</text>
        <input
          type="number"
          class="form-input"
          placeholder="请输入生产率百分比(%)"
          v-model="formData.productivity"
        />
      </view>

      <!-- 主管评语 -->
      <view class="form-item">
        <text class="form-label">主管评语</text>
        <textarea
          class="form-textarea"
          placeholder="请输入主管评语"
          v-model="formData.review"
        />
      </view>

<!--      &lt;!&ndash; 绩效等级 &ndash;&gt;-->
<!--      <view class="form-item">-->
<!--        <text class="form-label">绩效等级</text>-->
<!--        <picker-->
<!--          class="form-picker"-->
<!--          mode="selector"-->
<!--          :range="performanceGradeOptions"-->
<!--          range-key="label"-->
<!--          :value="gradeIndex"-->
<!--          @change="onGradeChange"-->
<!--        >-->
<!--          <view class="picker-view">-->
<!--            {{ gradeIndex >= 0 ? performanceGradeOptions[gradeIndex].label : '请选择绩效等级' }}-->
<!--          </view>-->
<!--        </picker>-->
<!--      </view>-->
    </scroll-view>

    <view class="form-footer">
      <button class="form-btn cancel" @click="handleCancel">取消</button>
      <button class="form-btn confirm" @click="handleSubmit">提交</button>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue';
import { onLoad } from '@dcloudio/uni-app';
import AioveuPerformanceAPI, {
  AioveuPerformanceForm
} from "@/packageC/api/aioveuPerformance/aioveu-performance";
import AioveuEmployeeAPI, { EmployeeOptionVO } from "@/packageC/api/aioveuEmployee/aioveu-employee";
import DictAPI, { DictItemOption } from '@/api/system/dict';

const formTitle = ref('新增绩效');
const performanceId = ref<number | undefined>(undefined);
const loading = ref(false);

const formData = reactive<AioveuPerformanceForm>({
  employeeName: '',
  periodYear: undefined,
  periodQuarter: undefined,
  kpiScore: undefined,
  productivity: undefined,
  review: '',
  // performanceGrade: undefined
});

const employeeOptions = ref<EmployeeOptionVO[]>([]);
const periodQuarterOptions = ref<DictItemOption[]>([]);
const performanceGradeOptions = ref<DictItemOption[]>([]);

const employeeIndex = ref(-1);
const quarterIndex = ref(-1);
const gradeIndex = ref(-1);

onLoad((options: any) => {
  console.log('页面参数:', options);

  if (options.id) {
    performanceId.value = Number(options.id);
    formTitle.value = '编辑绩效';
    loadPerformanceData();
  } else {
    formTitle.value = '新增绩效';
  }

  loadEmployees();
  loadPeriodQuarterOptions();
  loadPerformanceGradeOptions();
});

// 加载绩效数据
const loadPerformanceData = () => {
  if (!performanceId.value) return;

  loading.value = true;
  AioveuPerformanceAPI.getFormData(performanceId.value)
    .then((data) => {
      Object.assign(formData, data);

      // 设置员工索引
      if (formData.employeeName) {
        const index = employeeOptions.value.findIndex(
          emp => emp.employeeName === formData.employeeName
        );
        employeeIndex.value = index;
      }

      // 设置季度索引
      if (formData.periodQuarter !== undefined) {
        const index = periodQuarterOptions.value.findIndex(
          item => Number(item.value) === formData.periodQuarter
        );
        quarterIndex.value = index;
      }

      // // 设置绩效等级索引
      // if (formData.performanceGrade) {
      //   const index = performanceGradeOptions.value.findIndex(
      //     item => item.value === formData.performanceGrade
      //   );
      //   gradeIndex.value = index;
      // }
    })
    .finally(() => {
      loading.value = false;
    });
};

// 加载员工选项
const loadEmployees = () => {
  AioveuEmployeeAPI.getAllEmployeeOptions()
    .then(response => {
      if (Array.isArray(response)) {
        employeeOptions.value = response.map(emp => ({
          employeeId: Number(emp.employeeId),
          employeeName: emp.employeeName
        }));
      }
    });
};

// 加载季度选项
const loadPeriodQuarterOptions = () => {
  DictAPI.getDictItems('performance_period_quarter')
    .then(response => {
      periodQuarterOptions.value = response;
    });
};

// 加载绩效等级选项
const loadPerformanceGradeOptions = () => {
  DictAPI.getDictItems('performance_grade')
    .then(response => {
      performanceGradeOptions.value = response;
    });
};

// 员工选择变化
const onEmployeeChange = (e: any) => {
  const index = e.detail.value;
  employeeIndex.value = index;
  if (employeeOptions.value[index]) {
    formData.employeeName = employeeOptions.value[index].employeeName;
  }
};

// 季度选择变化
const onQuarterChange = (e: any) => {
  const index = e.detail.value;
  quarterIndex.value = index;
  if (periodQuarterOptions.value[index]) {
    formData.periodQuarter = Number(periodQuarterOptions.value[index].value);
  }
};

// // 绩效等级选择变化
// const onGradeChange = (e: any) => {
//   const index = e.detail.value;
//   gradeIndex.value = index;
//   if (performanceGradeOptions.value[index]) {
//     formData.performanceGrade = performanceGradeOptions.value[index].value;
//   }
// };

// 提交表单
const handleSubmit = () => {
  if (!validateForm()) return;

  uni.showLoading({ title: '提交中...' });

  if (performanceId.value) {
    // 更新
    AioveuPerformanceAPI.update(performanceId.value, formData)
      .then(() => {
        uni.showToast({
          title: "修改成功",
          icon: "success"
        });
        uni.navigateBack();
      })
      .finally(() => uni.hideLoading());
  } else {
    // 新增
    AioveuPerformanceAPI.add(formData)
      .then(() => {
        uni.showToast({
          title: "新增成功",
          icon: "success"
        });
        uni.navigateBack();
      })
      .finally(() => uni.hideLoading());
  }
};

// 表单验证
const validateForm = () => {
  if (!formData.employeeName) {
    uni.showToast({
      title: "请选择员工",
      icon: "none"
    });
    return false;
  }

  if (!formData.periodYear) {
    uni.showToast({
      title: "请输入考核年份",
      icon: "none"
    });
    return false;
  }

  if (formData.periodQuarter === undefined) {
    uni.showToast({
      title: "请选择考核季度",
      icon: "none"
    });
    return false;
  }

  if (!formData.kpiScore) {
    uni.showToast({
      title: "请输入KPI评分",
      icon: "none"
    });
    return false;
  }

  if (!formData.productivity) {
    uni.showToast({
      title: "请输入生产率",
      icon: "none"
    });
    return false;
  }

  return true;
};

// 取消
const handleCancel = () => {
  uni.navigateBack();
};
</script>

<style lang="scss" scoped>
.form-container {
  padding: 20rpx;
  background-color: #f5f7fa;
  min-height: 100vh;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

.form-header {
  padding: 30rpx;
  background: white;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  margin-bottom: 24rpx;

  .form-title {
    font-size: 36rpx;
    font-weight: 600;
    color: #1a1a1a;
    text-align: center;
  }
}

.form-body {
  background: white;
  border-radius: 16rpx;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  padding: 24rpx;
  max-height: 70vh;
}

.form-item {
  margin-bottom: 32rpx;
}

.form-label {
  display: block;
  font-size: 30rpx;
  color: #666;
  margin-bottom: 16rpx;
  font-weight: 500;
}

.form-input, .form-picker {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 24rpx;
  font-size: 30rpx;
}

.form-textarea {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 24rpx;
  font-size: 30rpx;
  height: 200rpx;
}

.picker-view {
  height: 44rpx;
  line-height: 44rpx;
}

.form-footer {
  padding: 24rpx 30rpx;
  background: #f8f9fa;
  display: flex;
  justify-content: space-between;
  gap: 24rpx;
  margin-top: 24rpx;
  border-top: 1rpx solid #eaeaea;

  .form-btn {
    flex: 1;
    padding: 20rpx 0;
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
