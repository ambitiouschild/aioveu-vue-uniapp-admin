<template>
  <view class="form-container">
    <view class="form-header">
      <text class="form-title">{{ formTitle }}</text>
    </view>

    <scroll-view class="form-body" scroll-y>
      <!-- 仓库 -->
      <view class="form-item">
        <text class="form-label">仓库</text>
        <picker
          class="form-picker"
          mode="selector"
          :range="warehouseOptions"
          range-key="warehouseName"
          :value="warehouseIndex"
          @change="onWarehouseChange"
        >
          <view class="picker-view">
            {{ warehouseIndex >= 0 ? warehouseOptions[warehouseIndex].warehouseName : '请选择仓库' }}
          </view>
        </picker>
      </view>

      <!-- 物资 -->
      <view class="form-item">
        <text class="form-label">物资</text>
        <picker
          class="form-picker"
          mode="selector"
          :range="materialOptions"
          range-key="materialName"
          :value="materialIndex"
          @change="onMaterialChange"
        >
          <view class="picker-view">
            {{ materialIndex >= 0 ? materialOptions[materialIndex].materialName : '请选择物资' }}
          </view>
        </picker>
      </view>

      <!-- 当前库存量 -->
      <view class="form-item">
        <text class="form-label">当前库存量</text>
        <input
          type="number"
          class="form-input"
          placeholder="请输入当前库存量"
          v-model="formData.quantity"
        />
      </view>

      <!-- 批次号 -->
      <view class="form-item">
        <text class="form-label">批次号</text>
        <input
          type="text"
          class="form-input"
          placeholder="请输入批次号"
          v-model="formData.batchNumber"
        />
      </view>

      <!-- 生产日期 -->
      <view class="form-item">
        <text class="form-label">生产日期</text>
        <picker
          class="form-picker"
          mode="date"
          :value="formData.productionDate"
          @change="onProductionDateChange"
        >
          <view class="picker-view">
            {{ formData.productionDate || '请选择生产日期' }}
          </view>
        </picker>
      </view>

      <!-- 有效期至 -->
      <view class="form-item">
        <text class="form-label">有效期至</text>
        <picker
          class="form-picker"
          mode="date"
          :value="formData.expiryDate"
          @change="onExpiryDateChange"
        >
          <view class="picker-view">
            {{ formData.expiryDate || '请选择有效期至' }}
          </view>
        </picker>
      </view>

      <!-- 最后入库时间 -->
      <view class="form-item">
        <text class="form-label">最后入库时间</text>
        <picker
          class="form-picker"
          mode="datetime"
          :value="formData.lastInbound"
          @change="onLastInboundChange"
        >
          <view class="picker-view">
            {{ formData.lastInbound || '请选择最后入库时间' }}
          </view>
        </picker>
      </view>

      <!-- 最后出库时间 -->
      <view class="form-item">
        <text class="form-label">最后出库时间</text>
        <picker
          class="form-picker"
          mode="datetime"
          :value="formData.lastOutbound"
          @change="onLastOutboundChange"
        >
          <view class="picker-view">
            {{ formData.lastOutbound || '请选择最后出库时间' }}
          </view>
        </picker>
      </view>
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
import AioveuInventoryAPI, {
  AioveuInventoryForm
} from "@/packageD/api/aioveuInventory/aioveu-inventory";
import AioveuWarehouseAPI, { WarehouseOptionVO } from "@/packageD/api/aioveuWarehouse/aioveu-warehouse";
import AioveuMaterialAPI, { MaterialOptionVO } from "@/packageD/api/aioveuMaterial/aioveu-material";

const formTitle = ref('新增库存');
const inventoryId = ref<number | undefined>(undefined);
const loading = ref(false);

const formData = reactive<AioveuInventoryForm>({
  warehouseName: '',
  materialName: '',
  quantity: undefined,
  batchNumber: '',
  // productionDate: '',
  // expiryDate: '',
  // lastInbound: '',
  // lastOutbound: ''
});

const warehouseOptions = ref<WarehouseOptionVO[]>([]);
const materialOptions = ref<MaterialOptionVO[]>([]);

const warehouseIndex = ref(-1);
const materialIndex = ref(-1);

onLoad((options: any) => {
  console.log('页面参数:', options);

  if (options.id) {
    inventoryId.value = Number(options.id);
    formTitle.value = '编辑库存';
    loadInventoryData();
  } else {
    formTitle.value = '新增库存';
  }

  // 加载选项数据
  loadWarehouseOptions();
  loadMaterialOptions();
});

// 加载库存数据
const loadInventoryData = async () => {
  if (!inventoryId.value) return;

  try {
    loading.value = true;
    const data = await AioveuInventoryAPI.getFormData(inventoryId.value);
    Object.assign(formData, data);

    // 设置仓库索引
    if (formData.warehouseName) {
      const index = warehouseOptions.value.findIndex(
        wh => wh.warehouseName === formData.warehouseName
      );
      warehouseIndex.value = index;
    }

    // 设置物资索引
    if (formData.materialName) {
      const index = materialOptions.value.findIndex(
        mat => mat.materialName === formData.materialName
      );
      materialIndex.value = index;
    }
  } catch (error) {
    console.error('加载库存数据失败:', error);
    uni.showToast({
      title: '加载数据失败',
      icon: 'none'
    });
  } finally {
    loading.value = false;
  }
};

// 加载仓库选项
const loadWarehouseOptions = async () => {
  try {
    const response = await AioveuWarehouseAPI.getAllWarehouseOptions();
    if (Array.isArray(response)) {
      warehouseOptions.value = response.map(wh => ({
        warehouseId: Number(wh.warehouseId),
        warehouseName: wh.warehouseName
      }));
    }
  } catch (error) {
    console.error('加载仓库列表失败:', error);
    uni.showToast({
      title: '加载仓库列表失败',
      icon: 'none'
    });
  }
};

// 加载物资选项
const loadMaterialOptions = async () => {
  try {
    const response = await AioveuMaterialAPI.getAllMaterialOptions();
    if (Array.isArray(response)) {
      materialOptions.value = response.map(mat => ({
        materialId: Number(mat.materialId),
        materialName: mat.materialName
      }));
    }
  } catch (error) {
    console.error('加载物资列表失败:', error);
    uni.showToast({
      title: '加载物资列表失败',
      icon: 'none'
    });
  }
};

// 仓库选择变化
const onWarehouseChange = (e: any) => {
  const index = e.detail.value;
  warehouseIndex.value = index;
  if (warehouseOptions.value[index]) {
    formData.warehouseName = warehouseOptions.value[index].warehouseName;
  }
};

// 物资选择变化
const onMaterialChange = (e: any) => {
  const index = e.detail.value;
  materialIndex.value = index;
  if (materialOptions.value[index]) {
    formData.materialName = materialOptions.value[index].materialName;
  }
};

// 生产日期选择变化
const onProductionDateChange = (e: any) => {
  formData.productionDate = e.detail.value;
};

// 有效期至选择变化
const onExpiryDateChange = (e: any) => {
  formData.expiryDate = e.detail.value;
};

// 最后入库时间选择变化
const onLastInboundChange = (e: any) => {
  formData.lastInbound = e.detail.value;
};

// 最后出库时间选择变化
const onLastOutboundChange = (e: any) => {
  formData.lastOutbound = e.detail.value;
};

// 提交表单
const handleSubmit = async () => {
  if (!validateForm()) return;

  try {
    uni.showLoading({ title: '提交中...' });

    if (inventoryId.value) {
      // 更新库存
      await AioveuInventoryAPI.update(inventoryId.value, formData);
      uni.showToast({
        title: "修改成功",
        icon: "success"
      });
    } else {
      // 新增库存
      await AioveuInventoryAPI.add(formData);
      uni.showToast({
        title: "新增成功",
        icon: "success"
      });
    }

    // 返回列表页
    uni.navigateBack();
  } catch (error) {
    console.error('提交表单失败:', error);
    uni.showToast({
      title: "提交失败",
      icon: "none"
    });
  } finally {
    uni.hideLoading();
  }
};

// 表单验证
const validateForm = () => {
  if (!formData.warehouseName) {
    uni.showToast({
      title: "请选择仓库",
      icon: "none"
    });
    return false;
  }

  if (!formData.materialName) {
    uni.showToast({
      title: "请选择物资",
      icon: "none"
    });
    return false;
  }

  if (!formData.quantity) {
    uni.showToast({
      title: "请输入库存数量",
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

.form-input, .form-picker, .form-textarea {
  width: 100%;
  border: 1rpx solid #e2e8f0;
  border-radius: 12rpx;
  padding: 24rpx;
  font-size: 30rpx;
}

.form-textarea {
  height: 200rpx;
}

.picker-view {
  height: 44rpx;
  line-height: 44rpx;
}

/* 表单底部按钮容器 */
.form-footer {
  display: flex;
  justify-content: space-between;
  padding: 24rpx 30rpx;
  background: #f8f9fa;
  gap: 24rpx;
  margin-top: 24rpx;
  border-top: 1rpx solid #eaeaea;

  .form-btn {
    flex: 1;
    padding: 20rpx 0;
    border-radius: 12rpx;
    font-size: 30rpx;
    border: none;
    display: flex;
    align-items: center;
    justify-content: center;

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
