<template>
  <view class="mine-container">
    <!-- 标题栏 -->
    <view class="header">
      <view class="header-icon">👤</view>
      <text class="header-subtitle">个人中心</text>
    </view>
    
    <!-- 功能列表 -->
    <view class="function-section">
      <view class="function-card">
        <view class="function-item" @click="showVersion">
          <text class="function-title">版本信息</text>
          <text class="function-value">{{ appVersion }}</text>
          <text class="function-arrow">›</text>
        </view>
        
        <view class="function-item" @click="contactUs">
          <text class="function-title">联系我们</text>
          <text class="function-arrow">›</text>
        </view>
        
        <view class="function-item" @click="clearCache">
          <text class="function-title">清除缓存</text>
          <text class="function-value">{{ cacheSize }}</text>
          <text class="function-arrow">›</text>
        </view>
      </view>
    </view>
    
    <!-- 版权信息 -->
    <view class="copyright">
      <text>© 2026 美剧学习播放器</text>
    </view>
    

  </view>
</template>

<script>
/**
 * 我的页面
 * 功能：
 * 1. 显示版本信息
 * 2. 联系我们
 * 3. 清除缓存
 */
export default {
  data() {
    return {
      appVersion: '',
      cacheSize: '0 KB'
    }
  },
  
  onLoad() {
    this.getAppVersion();
    this.getCacheSize();
  },
  
  methods: {
    /**
     * 获取应用版本号
     */
    getAppVersion() {
      // 直接使用manifest.json中的版本号
      this.appVersion = '1.0.4';
    },
    
    /**
     * 显示版本信息
     */
    showVersion() {
      uni.showModal({
        title: '版本信息',
        content: `当前版本：${this.appVersion}`,
        showCancel: false
      });
    },
    
    /**
     * 联系我们
     */
    contactUs() {
      uni.showModal({
        title: '联系我们',
        content: '如果您有任何问题或建议，请通过以下方式联系我们：\n\n邮箱：contact@example.com',
        showCancel: false
      });
    },
    
    /**
     * 获取缓存大小
     */
    getCacheSize() {
      // 在H5环境下无法获取准确的缓存大小
      if (typeof document !== 'undefined') {
        this.cacheSize = '未知';
      } else {
        // 在App环境下尝试获取缓存大小
        try {
          plus.cache.calculate(function(size) {
            let cacheSize = (size / 1024 / 1024).toFixed(2);
            this.cacheSize = cacheSize + ' MB';
          }.bind(this));
        } catch (e) {
          this.cacheSize = '未知';
        }
      }
    },
    
    /**
     * 清除缓存
     */
    clearCache() {
      uni.showModal({
        title: '清除缓存',
        content: '确定要清除所有缓存数据吗？',
        success: (res) => {
          if (res.confirm) {
            // 在H5环境下无法清除缓存
            if (typeof document !== 'undefined') {
              uni.showToast({
                title: 'H5环境暂不支持清除缓存',
                icon: 'none'
              });
            } else {
              // 在App环境下清除缓存
              try {
                plus.cache.clear(function() {
                  uni.showToast({
                    title: '缓存清除成功',
                    icon: 'success'
                  });
                  this.cacheSize = '0 KB';
                }.bind(this));
              } catch (e) {
                uni.showToast({
                  title: '缓存清除失败',
                  icon: 'none'
                });
              }
            }
          }
        }
      });
    },
    

  }
}
</script>

<style>
/* 全局样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* 应用容器样式 */
.mine-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #1a1f2e 0%, #2d3748 100%);
  color: #ffffff;
  padding: 20rpx;
}

/* 标题栏样式 */
.header {
  text-align: center;
  margin-bottom: 40rpx;
  padding: 30rpx 0;
}

.header-icon {
  font-size: 48rpx;
  margin-bottom: 10rpx;
}

.header-title {
  display: block;
  font-size: 40rpx;
  font-weight: bold;
  color: #63b3ed;
  margin-bottom: 10rpx;
}

.header-subtitle {
  display: block;
  font-size: 24rpx;
  color: #a0aec0;
}

/* 功能区域样式 */
.function-section {
  margin-bottom: 40rpx;
}

.function-card {
  background-color: rgba(255, 255, 255, 0.05);
  border: 1rpx solid rgba(255, 255, 255, 0.1);
  border-radius: 12rpx;
  overflow: hidden;
}

.function-item {
  display: flex;
  align-items: center;
  padding: 20rpx;
  border-bottom: 1rpx solid rgba(255, 255, 255, 0.05);
}

.function-item:last-child {
  border-bottom: none;
}

.function-icon {
  font-size: 32rpx;
  margin-right: 20rpx;
}

.function-title {
  flex: 1;
  font-size: 28rpx;
  color: #e2e8f0;
}

.function-value {
  font-size: 24rpx;
  color: #718096;
  margin-right: 10rpx;
}

.function-arrow {
  font-size: 32rpx;
  color: #718096;
}

/* 版权信息样式 */
.copyright {
  text-align: center;
  padding: 40rpx 0;
  color: #718096;
  font-size: 22rpx;
}


</style>