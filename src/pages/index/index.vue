<template>
  <view class="container">
    <!-- 语言选择器 -->
    <view class="language-selector" @click="openLanguageModal" v-if="false">
      <view class="language-picker">
        <text class="current-language">{{ currentLanguage.name }}</text>
        <text class="picker-arrow">▼</text>
      </view>
    </view>

    <!-- 计算方法选择区域 -->
    <view class="method-selector">
      <view class="select-item">
        <text class="select-label">{{ $t('method.type.title') }}:</text>
        <picker mode="selector" :range="typeOptions" :range-key="'label'" :model="selectedTypeIndex"
                @change="onTypeChange"
                style="flex: 1">
          <view class="picker-display">
            <text>{{ getTypeLabel(selectedType) }}</text>
            <text class="picker-arrow">▼</text>
          </view>
        </picker>
      </view>

      <view class="select-item mt-10">
        <text class="select-label">{{ $t('method.model.title') }}:</text>
        <picker mode="selector" :range="modelOptions" :range-key="'label'" :model="selectedModelIndex"
                @change="onModelChange"
                style="flex: 1">
          <view class="picker-display">
            <text>{{ getModelLabel(selectedModel) }}</text>
            <text class="picker-arrow">▼</text>
          </view>
        </picker>
      </view>
    </view>

    <button class="upload-btn" @click="handleChooseFile" :disabled="isLoading">{{ $t('action.upload') }}</button>
    <text class="file-name" v-if="fileName">{{ $t('tip.uploadTip') }}: {{ fileName }}</text>

    <!-- 团队介绍组件 -->
    <team-intro :show="!hasResult" @start="onTeamIntroStart"/>

    <team-logo-display v-if="!hasResult"></team-logo-display>

    <!-- 结果展示组件 -->
    <result-display
        :key="displayKey"
        :has-result="hasResult"
        :is-svm-or-tree="isSvmOrTree"
        :is-quantitative="isQuantitative"
        :result="result"
        :probabilities="probabilities"
        :calculation-data="calculationData"
        :quality-info="qualityInfo"
        :file-info="fileInfo"
        :selected-type="selectedType"
        :chart-mode="chartMode"
        @reanalyze="handleResetAnalysis"
        @toggle-chart="toggleChartMode"
    />

    <!-- 自定义语言选择模态框 -->
    <view class="language-modal" v-if="showLanguageModal" @click="closeLanguageModal">
      <view class="modal-content" @click.stop>
        <view class="modal-header">
          <text class="modal-title">{{ $t('language.selectLanguage') || '选择语言' }}</text>
          <text class="modal-close" @click="closeLanguageModal">×</text>
        </view>
        <view class="language-list">
          <view
              class="language-item"
              v-for="item in languageOptions"
              :key="item.locale"
              @click="selectLanguage(item.locale)"
              :class="{ active: currentLanguage.locale === item.locale }"
          >
            <text class="language-name">{{ item.name }}</text>
            <text class="checkmark" v-if="currentLanguage.locale === item.locale">✓</text>
          </view>
        </view>
      </view>
    </view>
    <view style="padding: 10rpx; background: #eee; font-size: 24rpx; width: 100%; text-align: center;">
      当前状态: {{ connectionStatus }}
    </view>
    <view class="result-container" style="padding: 30rpx;">
      <view
          style="background: #f8f9fa; border: 2rpx solid #dee2e6; padding: 40rpx; border-radius: 12rpx; text-align: center;">
        <text style="font-size: 32rpx; color: #333; font-weight: bold;">
          {{ displayResult }}
        </text>
      </view>

      <view style="margin-top: 40rpx;">
        <button type="primary" @click="runScript" :disabled="remainingTime > 0">
          {{ remainingTime > 0 ? '分析中 (' + remainingTime + 's)' : '启动科研分析' }}
        </button>

        <button style="margin-top: 20rpx;" @click="fetchCloudResult">
          手动同步结果
        </button>
      </view>
    </view>
  </view>
</template>

<script>
import TeamGuide from "@/components/TeamGuide.vue";
import TeamIntro from "@/components/TeamIntro.vue";
import TeamLogoDisplay from "@/components/TeamLogoDisplay.vue";
import ResultDisplay from "@/components/ResultDisplay.vue";
import {languageMixin, modelMixin} from "@/mixins";
import {useFileHandler} from "@/composables/useFileHandler";

export default {
  name: 'Index',

  components: {
    TeamGuide,
    TeamIntro,
    TeamLogoDisplay,
    ResultDisplay
  },

  mixins: [languageMixin, modelMixin],

  setup() {
    const {
      fileName,
      isLoading,
      result,
      probabilities,
      fileInfo,
      calculationData,
      qualityInfo,
      chooseFile,
      resetAnalysis
    } = useFileHandler()

    return {
      debugStatus,
      debugData,
      fileName,
      isLoading,
      result,
      probabilities,
      fileInfo,
      calculationData,
      qualityInfo,
      chooseFile,
      resetAnalysis
    }
  },

  data() {
    return {
      socket: null,
      socketTask: null, // 建议使用专门的任务对象
      serverIP: '192.168.1.124', // 提取 IP 方便显示和修改
      connectionStatus: '未连接',
      remainingTime: 0,
      displayResult: '检测结果将在此显示...'
    }
  },

  onLoad() {
    this.initWebSocket();
  },

  methods: {
    // 统一初始化方法
    initWebSocket() {
      this.connectionStatus = `正在连接 ${this.serverIP}...`;

      // 关键：在 App 端使用 uni.connectSocket 返回的任务对象进行管理
      this.socketTask = uni.connectSocket({
        url: `ws://${this.serverIP}:8765`,
        success: () => {
          console.log("Socket 接口调用成功");
        }
      });

      this.socketTask.onOpen(() => {
        this.connectionStatus = `已连接到: ${this.serverIP}`;
        uni.showToast({title: "连接成功", icon: "success"});
      });

      this.socketTask.onError((err) => {
        this.connectionStatus = "连接报错，请检查IP或防火墙";
        console.error("Socket错误:", err);
      });

      this.socketTask.onClose(() => {
        this.connectionStatus = "连接已断开";
      });

      this.socketTask.onMessage((msg) => {
        uni.showToast({title: msg.data, icon: "none"});
      });
    },

    // 发送指令
    sendControlCommand(cmd) {
      if (this.connectionStatus.indexOf('已连接') === -1) {
        uni.showToast({title: "请先连接电脑", icon: "none"});
        this.initWebSocket(); // 尝试重连
        return;
      }

      uni.sendSocketMessage({
        data: cmd,
        fail: (err) => {
          uni.showToast({title: "发送失败", icon: "none"});
          this.initWebSocket();
        }
      });
    },
    runScript() {
      // 1. 启动 WebSocket 指令
      this.sendControlCommand("run");

      // 2. 开启 5 分钟倒计时
      this.remainingTime = 120;
      if (this.autoFetchTimer) clearInterval(this.autoFetchTimer);

      this.autoFetchTimer = setInterval(() => {
        this.remainingTime--;
        if (this.remainingTime <= 0) {
          clearInterval(this.autoFetchTimer);
          this.fetchCloudResult(); // 倒计时结束，自动读 CSV
        }
      }, 1000);
    },

    // 手动获取按键调用此方法
    manualFetch() {
      this.fetchCloudResult();
    },

    // 统一的读取函数
    async fetchCloudResult() {
      uni.request({
        url: 'http://43.138.48.175/result.csv?t=' + new Date().getTime(),
        method: 'GET',
        success: (res) => {
          if (res.statusCode === 200 && res.data) {
            // 1. 拆分行
            const lines = res.data.trim().split(/\r?\n/);

            // 2. 只要行数大于1，就取第二行
            if (lines.length >= 2) {
              const d = lines[1].split(','); // d[0]=1234.38, d[1]=14, d[2]=13.0, d[3]=ug/ml

              // 3. 直接拼成一句话显示
              this.displayResult = `浓度: ${d[2]} ${d[3]} (峰位: ${d[0]}, 强度: ${d[1]})`;

              uni.showToast({title: '同步成功', icon: 'success'});
            } else {
              this.displayResult = '数据计算中，请稍后...';
            }
          }
        },
        fail: () => {
          uni.showToast({title: '网络连接失败', icon: 'none'});
        }
      });
    },

    // 解析 CSV
    parseCsv(csvContent) {
      const lines = csvContent.split('\n');
      if (lines.length > 0) {
        const row = lines[0].split(',');
        // 映射到页面显示的变量
        this.result = row[0];
        this.calculationData = row[1];
        this.fileName = "同步自云端: Quantitation_1.csv";
      }
    },

    shutdownPC() {
      uni.showModal({
        title: '确认操作',
        content: '确定要远程关闭科研电脑吗？',
        success: (res) => {
          // 1. 确保拿到的是字符串并清理前后空格/换行
          let rawData = res.data || "";
          if (typeof rawData !== 'string') rawData = JSON.stringify(rawData);

          // 2. 按换行符切分，并过滤掉空行（防止文件末尾有回车）
          const lines = rawData.split(/\r?\n/).filter(line => line.trim() !== "");

          console.log("解析出的行数:", lines.length);
          console.log("数据行内容:", lines[1]);

          // 3. 必须有至少两行（标题+数据）
          if (lines.length >= 2) {
            const dataValues = lines[1].split(',');

            // 4. 严格对应 CSV 列顺序进行赋值
            this.peakPosition = dataValues[0]; // 1234.38
            this.intensity = dataValues[1]; // 14
            this.result = dataValues[2]; // 13.0 (对应 Concentration)
            this.unit = dataValues[3]; // ug/ml

            uni.showToast({title: '同步成功', icon: 'success'});
          } else {
            uni.showToast({title: '暂无定量结果', icon: 'none'});
          }
        }
      });
    },

    reconnect() {
      this.socket = uni.connectSocket({
        url: 'ws://192.168.1.128:8765',
        complete: () => {
        }
      });
    }
  }
}
</script>

<style scoped>
.container {
  //min-height: 100vh;
  background-color: #f5f7fa;
  padding: 40rpx 30rpx;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

.upload-section {
  width: 100%;
  padding: 40rpx 0 0;
  text-align: center;
}

/* 新增：计算方法选择器样式 */
.method-selector {
  width: 100%;
  margin-bottom: 30rpx;
}

.select-item {
  display: flex;
  align-items: center;
  background-color: #fff;
  border-radius: 12rpx;
  padding: 20rpx;
  box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.05);
}

.select-label {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
  margin-right: 20rpx;
  flex-shrink: 0;
  width: 120rpx;
  text-align: left;
}

.picker-display {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex: 1;
  padding: 10rpx 15rpx;
  border: 1rpx solid #e5e5e5;
  border-radius: 8rpx;
  background-color: #f9f9f9;
}

.picker-display text:first-child {
  font-size: 28rpx;
  color: #333;
  text-align: left;
  flex: 1;
}

.mt-10 {
  margin-top: 10rpx;
}

.upload-btn {
  background: #2f83e5;
  color: #fff !important;
  border: none !important;
  border-radius: 12rpx;
  height: 90rpx;
  font-size: 34rpx;
  font-weight: 600;
  padding: 0 40rpx;
  margin-bottom: 20rpx;
}

.upload-btn:active {
  transform: scale(0.98);
  opacity: 0.9;
}

.upload-btn:disabled {
  opacity: 0.7;
  transform: none;
}

.file-name {
  font-size: 28rpx;
  color: #666;
  background-color: #fff;
  padding: 16rpx 24rpx;
  border-radius: 8rpx;
  box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.05);
  max-width: 100%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: inline-block;
}

.result-section {
  width: 100%;
  margin-bottom: 60rpx;
}

/* 通用卡片样式 */
.result-card, .info-card, .probabilities-card, .file-info-card {
  background-color: #fff;
  border-radius: 12rpx;
  padding: 36rpx 30rpx;
  box-shadow: 0 4rpx 16rpx rgba(0, 0, 0, 0.08);
}

.mt-20 {
  margin-top: 20rpx;
}

/* 主要结果样式 */
.result-title {
  font-size: 30rpx;
  color: #333;
  margin-right: 10rpx;
}

.result-text {
  font-size: 38rpx;
  font-weight: 700;
}

.healthy {
  color: #4CAF50;
}

.sub-healthy {
  color: #FFC107;
}

.ill {
  color: #F44336;
}

/* 四分类饼图样式 */
.four-category-pie {
  background-color: #fff;
  border-radius: 12rpx;
  padding: 36rpx 30rpx;
  box-shadow: 0 4rpx 16rpx rgba(0, 0, 0, 0.08);
}

.section-header {
  margin-bottom: 30rpx;
}

.section-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
  display: block;
}

.pie-chart-container {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20rpx;
}

.pie-wrapper {
  position: relative;
  border-radius: 50%;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.1);
}

.pie-chart {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  position: relative;
}

.pie-center {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 20%;
  height: 20%;
  background-color: white;
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-shadow: inset 0 0 10rpx rgba(0, 0, 0, 0.05);
}

.pie-total {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
}

.pie-total-value {
  font-size: 32rpx;
  color: #2c3e50;
  font-weight: bold;
  margin-top: 5rpx;
}

.pie-legend {
  width: 320rpx;
}

.legend-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6rpx;
  margin-bottom: 6rpx;
  border-radius: 12rpx;
  background-color: #f8f9fa;
  transition: all 0.3s ease;
  border: 2rpx solid transparent;
  font-size: 12px;
}

.legend-item.active {
  border-color: #3498db;
  background-color: #e8f4fc;
  transform: translateX(5rpx);
}

.legend-item:active {
  opacity: 0.8;
}

.legend-info {
  display: flex;
  align-items: center;
  flex: 1;
}

.legend-color {
  width: 24rpx;
  height: 24rpx;
  border-radius: 50%;
  margin-right: 20rpx;
  box-shadow: 0 2rpx 6rpx rgba(0, 0, 0, 0.1);
}

.legend-details {
  display: flex;
  flex-direction: column;
}

.legend-name {
  font-size: 28rpx;
  font-weight: 500;
  color: #2c3e50;
  margin-bottom: 3rpx;
}

.legend-percentage {
  font-size: 24rpx;
  color: #7f8c8d;
}

.legend-angle {
  font-size: 26rpx;
  color: #2c3e50;
  font-weight: 500;
}

/* 信息卡片样式 */
.info-title, .prob-title, .file-info-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
  margin-bottom: 24rpx;
  display: block;
}

.info-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16rpx;
}

.info-label {
  font-size: 28rpx;
  color: #666;
}

.info-value {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
}

/* 概率卡片样式 */
.prob-item {
  margin-bottom: 24rpx;
}

.prob-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12rpx;
}

.prob-name {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
}

.prob-percent {
  font-size: 26rpx;
  color: #666;
  font-weight: 500;
}

.prob-bar {
  height: 12rpx;
  background-color: #f0f0f0;
  border-radius: 6rpx;
  overflow: hidden;
}

.prob-bar-fill {
  height: 100%;
  border-radius: 6rpx;
  transition: width 0.5s ease-in-out;
}

.carotene-bar {
  background: linear-gradient(90deg, #FF6B6B, #FF6B6B);
}

.k3-bar {
  background: linear-gradient(90deg, #06D6A0, #06D6A0);
}

.d3-bar {
  background: linear-gradient(90deg, #FFD166, #FFD166);
}

.retinol-bar {
  background: linear-gradient(90deg, #4e63cd, #4e63cd);
}

.healthy-bar {
  background: linear-gradient(90deg, #4CAF50, #66BB6A);
}

.sub-healthy-bar {
  background: linear-gradient(90deg, #FFC107, #FFD54F);
}

.ill-bar {
  background: linear-gradient(90deg, #F44336, #EF5350);
}

.default-bar {
  background: linear-gradient(90deg, #2196F3, #42A5F5);
}

/* 文件信息样式 */
.file-info-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 16rpx;
}

.file-info-label {
  font-size: 28rpx;
  color: #666;
  flex-shrink: 0;
  margin-right: 20rpx;
}

.file-info-value {
  font-size: 28rpx;
  color: #333;
  font-weight: 500;
  text-align: right;
  word-break: break-all;
}

/* 语言选择器样式 */
.language-selector {
  position: fixed;
  top: 0;
  height: 44px;
  display: flex;
  align-items: center;
  right: 30rpx;
  z-index: 1000;
}

.language-picker {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.9);
  border: 2rpx solid #409EFF;
  border-radius: 20rpx;
  padding: 8rpx 12rpx;
  backdrop-filter: blur(10px);
}

.current-language {
  font-size: 26rpx;
  color: #409EFF;
  font-weight: 500;
  margin-right: 8rpx;
}

.picker-arrow {
  font-size: 20rpx;
  color: #409EFF;
}

/* 语言选择模态框样式 */
.language-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.modal-content {
  background-color: #fff;
  border-radius: 20rpx;
  width: 600rpx;
  max-height: 80vh;
  overflow: hidden;
  animation: modalShow 0.3s ease;
}

@keyframes modalShow {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 40rpx 30rpx 20rpx;
  border-bottom: 2rpx solid #f0f0f0;
}

.modal-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
}

.modal-close {
  font-size: 40rpx;
  color: #999;
  line-height: 1;
  padding: 10rpx;
}

.language-list {
  max-height: 600rpx;
  overflow-y: auto;
}

.language-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 30rpx;
  border-bottom: 1rpx solid #f5f5f5;
  transition: background-color 0.2s;
}

.language-item:active {
  background-color: #f8f8f8;
}

.language-item.active {
  background-color: #f0f7ff;
}

.language-name {
  font-size: 30rpx;
  color: #333;
}

.checkmark {
  color: #409EFF;
  font-size: 28rpx;
  font-weight: bold;
}

/* 定量分析计算详情样式 */
.calc-detail-text {
  font-size: 28rpx;
  color: #333;
  line-height: 1.6;
  margin-top: 10rpx;
  display: block;
}

.calc-data-section {
  margin-top: 20rpx;
  padding-top: 20rpx;
  border-top: 1rpx solid #f0f0f0;
}

.calc-data-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12rpx;
}

.calc-data-label {
  font-size: 26rpx;
  color: #666;
}

.calc-data-value {
  font-size: 26rpx;
  color: #333;
  font-weight: 500;
}

/* 响应式设计 */
@media (max-width: 320px) {
  .container {
    padding: 30rpx 20rpx;
  }

  .upload-btn {
    height: 80rpx;
    font-size: 30rpx;
  }

  .result-text {
    font-size: 34rpx;
  }

  .info-title, .prob-title, .file-info-title {
    font-size: 30rpx;
  }

  .info-label, .info-value, .prob-name, .prob-percent, .file-info-label, .file-info-value {
    font-size: 26rpx;
  }

  .current-language {
    font-size: 24rpx;
  }

  .modal-content {
    width: 560rpx;
  }

  .modal-title {
    font-size: 30rpx;
  }

  .language-name {
    font-size: 28rpx;
  }

  .select-label {
    font-size: 26rpx;
    width: 120rpx;
  }

  .picker-display text:first-child {
    font-size: 26rpx;
  }

  .calc-detail-text {
    font-size: 26rpx;
  }

  .calc-data-label, .calc-data-value {
    font-size: 24rpx;
  }

  /* 饼图响应式调整 */
  .pie-wrapper {
    width: 180rpx !important;
    height: 180rpx !important;
  }

  .legend-name {
    font-size: 26rpx;
  }

  .legend-percentage {
    font-size: 22rpx;
  }

  .legend-angle {
    font-size: 24rpx;
  }
}

.reanalyze-btn {
  background-color: #2f83e5;
  color: #fff;
  border: none;
  border-radius: 12rpx;
  height: 80rpx;
  font-size: 32rpx;
  padding: 0 40rpx;
  width: 100%;
}

.reanalyze-btn:active {
  opacity: 0.9;
  transform: scale(0.98);
}

.prob-chart {
  width: 100%;
}

.prob-header-line {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20rpx;
}

.view-hint {
  font-size: 24rpx;
  color: #999;
}

.prob-bar-wrapper {
  margin-top: 20rpx;
}

.chart-switch {
  display: flex;
  gap: 12rpx;
}

.chart-icon {
  width: 48rpx;
  height: 48rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8rpx;
  font-size: 32rpx;
  color: #999;
  background: #f5f5f5;
  transition: all 0.2s;
}

.chart-icon.active {
  color: #fff;
  background: #4c7ef3;
}
</style>