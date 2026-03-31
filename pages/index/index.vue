<template>
  <view class="app-container">
    <!-- 标题栏 -->
    <view class="header">
      <view class="header-icon">▶️</view>
      <text class="header-title">美剧学习播放器</text>
      <text class="header-subtitle">支持SRT/VTT格式 | 点击字幕跳转 | A-B片段循环</text>
    </view>
    
    <!-- 视频文件上传区域 -->
    <view class="upload-card">
      <view class="upload-label">
        <text class="upload-icon">📹</text>
        <text class="upload-title">视频文件</text>
      </view>
      <view class="upload-area" @click="chooseVideoFile">
        <view class="upload-content">
          <text class="upload-icon-large">☁️</text>
          <text class="upload-text">点击或拖拽视频文件到此处</text>
          <text class="upload-hint">支持 MP4, MOV 等格式</text>
        </view>
      </view>
    </view>
    
    <!-- 字幕文件上传区域 -->
    <view class="upload-card">
      <view class="upload-label">
        <text class="upload-icon">📄</text>
        <text class="upload-title">字幕文件</text>
      </view>
      <view class="upload-area" @click="chooseSubtitleFile">
        <view class="upload-content">
          <text class="upload-icon-large">📋</text>
          <text class="upload-text">点击或拖拽字幕文件到此处</text>
          <text class="upload-hint">支持 SRT, VTT 格式</text>
        </view>
      </view>
    </view>
    
    <!-- 视频播放区域 -->
    <view class="video-section">
      <view class="video-container">
        <!-- 使用uniapp-video-player插件 -->
        <DomVideoPlayer 
          v-if="videoUrl"
          ref="domVideoPlayer"
          :src="videoUrl"
          :autoplay="false"
          :loop="false"
          :controls="true"
          :muted="false"
          :isLoading="true"
          :objectFit="'contain'"
          :poster="''"
          style="width: 100%; height: 100%;"
          @timeupdate="onVideoTimeUpdate"
          @loadedmetadata="onVideoLoaded"
          @error="onVideoError"
          @loadeddata="onVideoLoaded"
          @canplay="onVideoLoaded"
          @play="onPlay"
          :key="videoUrl"
        ></DomVideoPlayer>
        <!-- 视频空状态 -->
        <view v-else class="video-placeholder">
          <text class="video-placeholder-icon">🎬</text>
          <text class="video-placeholder-text">请加载本地视频文件</text>
        </view>
      </view>
      
      <!-- 播放控制栏 -->
      <view class="control-section">
        <view class="time-display">{{ currentTime }} / {{ duration }}</view>
        <view class="control-buttons">
          <button @click="toggleLoop" class="control-btn loop-btn">
            <text class="btn-icon">🔄</text>
            <text>循环: {{ loopMode ? '开' : '关' }}</text>
          </button>
        </view>
      </view>
      
      <!-- A-B循环控制 -->
      <view class="ab-loop-section">
        <text class="ab-status">{{ abLoopStatus }}</text>
        <view class="ab-buttons">
          <button @click="setStartPoint" class="ab-btn" :class="{ active: loopStart > 0 }">起点 A</button>
          <button @click="setEndPoint" class="ab-btn" :class="{ active: loopEnd > 0 }">终点 B</button>
          <button @click="clearLoopPoints" class="ab-btn clear-btn">🗑️</button>
        </view>
      </view>
    </view>
    
    <!-- 字幕列表区域 -->
    <view class="subtitle-section">
      <view class="subtitle-header">
        <text class="subtitle-title">字幕列表</text>
        <text class="subtitle-count">{{ subtitles.length }}条</text>
      </view>
      <scroll-view 
        ref="subtitleScroll" 
        class="subtitle-list"
        scroll-y
        :scroll-into-view="scrollIntoViewId"
        scroll-with-animation
      >
        <view v-if="subtitles.length === 0" class="empty-subtitle">
          <text class="empty-icon">☰</text>
          <text class="empty-text">暂无字幕数据</text>
          <text class="empty-hint">请上传本地字幕文件</text>
        </view>
        <view v-else class="subtitle-items">
          <view 
            v-for="(subtitle, index) in subtitles" 
            :key="index"
          >
            <!-- 锚点元素：用于 scroll-into-view 定位，使字幕项居中显示 -->
            <view :id="'subtitle-anchor-' + index" class="subtitle-anchor"></view>
            <view 
              @click="jumpToTime(subtitle.startTime)" 
              class="subtitle-item"
              :class="{ active: currentSubtitleIndex === index }"
              :id="'subtitle-item-' + index"
            >
              <text class="subtitle-time">{{ formatTime(subtitle.startTime) }}</text>
              <text class="subtitle-text">{{ subtitle.text }}</text>
            </view>
          </view>
        </view>
      </scroll-view>
    </view>
  </view>
</template>

<script>
/**
 * 美剧学习播放器
 * 功能：
 * 1. 本地视频文件导入（MP4/MOV等格式）
 * 2. 本地字幕文件导入（SRT/VTT格式）
 * 3. 智能字幕交互 - 点击跳转+当前字幕高亮
 * 4. 灵活播放控制 - 整段循环+A-B片段重复
 */
import DomVideoPlayer from 'uniapp-video-player'

export default {
  components: {
    DomVideoPlayer
  },
  data() {
    return {
      // 环境相关
      isApp: false, // 是否在 app 环境下
      
      // 视频相关
      videoUrl: '', // 视频地址（本地临时路径）
      videoFileName: '', // 视频文件名
      currentTime: '00:00:00', // 当前播放时间
      duration: '00:00:00', // 视频总时长
      currentTimeSec: 0, // 当前时间（秒）
      durationSec: 0, // 总时长（秒）
      videoContext: null, // 视频上下文
      isVideoReadyForSeek: false, // 视频是否准备好进行跳转
      
      // 字幕相关
      subtitleFileName: '', // 字幕文件名
      subtitles: [], // 解析后的字幕列表
      currentSubtitleIndex: -1, // 当前播放的字幕索引
      subtitleRefs: [], // 字幕项的引用
      // subtitleScrollTop: 0, // 字幕列表滚动位置（已弃用，改用 scroll-into-view）
      scrollIntoViewId: '', // scroll-into-view 目标元素 id
      
      // 循环播放相关
      loopMode: false, // 循环模式
      loopStart: 0, // 循环开始时间
      loopEnd: 0, // 循环结束时间
      
      // 播放状态
      isPlaying: false // 是否正在播放
    }
  },
  
  computed: {
    /**
     * A-B循环状态文本
     */
    abLoopStatus() {
      if (this.loopStart > 0 && this.loopEnd > 0) {
        return `A: ${this.formatTime(this.loopStart)} - B: ${this.formatTime(this.loopEnd)}`;
      } else if (this.loopStart > 0) {
        return `A: ${this.formatTime(this.loopStart)} - B: 未设置`;
      }
      return '区间循环未设置';
    }
  },
  
  /**
   * 页面加载时触发
   */
  onLoad() {
    // 页面加载时获取视频上下文
    this.getVideoContext();
  },
  
  /**
   * 页面显示时触发
   */
  onShow() {
    // 页面显示时重新获取视频上下文，确保组件已渲染
    this.getVideoContext();
  },
  
  methods: {
    /**
     * 获取视频上下文
     */
    getVideoContext() {
      // 使用uniapp-video-player插件的API获取视频上下文
      if (this.$refs.domVideoPlayer) {
        this.videoContext = this.$refs.domVideoPlayer;
        console.log('获取视频上下文:', this.videoContext);
      } else {
        console.log('视频组件未找到（可能未渲染）');
        this.videoContext = null;
      }
    },
    
    /**
     * 选择视频文件
     */
    chooseVideoFile() {
      // 检查是否在H5环境且存在document对象
      if (typeof document !== 'undefined') {
        // 在H5端，使用input[type=file]来选择文件
        const input = document.createElement('input');
        input.type = 'file';
        input.accept = '.mp4,.mov,.avi,.mkv,.webm';
        input.onchange = (e) => {
          const file = e.target.files[0];
          if (file) {
            // 检查文件类型
            if (!file.type.startsWith('video/')) {
              console.warn('文件类型可能不是视频:', file.type);
            }
            
            this.videoFileName = file.name;
            
            // 释放之前的URL
            if (this.videoUrl) {
              URL.revokeObjectURL(this.videoUrl);
            }
            
            this.videoUrl = URL.createObjectURL(file);
                        
            // 直接操作原生video元素
            const videoElement = document.querySelector('video');
            if (videoElement) {
                            
              // 重置视频元素
              videoElement.pause();
              videoElement.removeAttribute('src');
              videoElement.load();
              
              // 设置视频源
              videoElement.src = this.videoUrl;
              
              // 定义事件处理函数
              const onVideoLoaded = () => {
                                                this.durationSec = videoElement.duration;
                this.duration = this.formatTime(videoElement.duration);
                uni.showToast({ title: '视频加载成功', icon: 'success' });
              };
              
              const onVideoCanPlay = () => {
                                                this.durationSec = videoElement.duration;
                this.duration = this.formatTime(videoElement.duration);
              };
              
              const onVideoError = (err) => {
                
                let errorMessage = '视频加载失败';
                if (videoElement.error) {
                                                         
                  switch (videoElement.error.code) {
                    case MediaError.MEDIA_ERR_ABORTED:
                      errorMessage = '视频加载被中止';
                      break;
                    case MediaError.MEDIA_ERR_NETWORK:
                      errorMessage = '网络错误导致视频加载失败';
                      break;
                    case MediaError.MEDIA_ERR_DECODE:
                      errorMessage = '视频解码失败，请检查文件格式';
                      break;
                    case MediaError.MEDIA_ERR_SRC_NOT_SUPPORTED:
                      errorMessage = '视频格式不支持，请尝试MP4格式';
                      break;
                    default:
                      errorMessage = '视频加载失败，请检查文件';
                  }
                }
                
                                uni.showToast({ title: errorMessage, icon: 'none' });
              };
              
              // 添加事件监听器（使用once选项，只触发一次）
              videoElement.addEventListener('loadedmetadata', onVideoLoaded, { once: true });
              videoElement.addEventListener('canplay', onVideoCanPlay, { once: true });
              videoElement.addEventListener('error', onVideoError, { once: true });
              
              // 添加时间更新事件监听
              videoElement.addEventListener('timeupdate', () => {
                this.onTimeUpdate();
              });
              
              // 加载视频
              videoElement.load();
            } else {
              uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
            }
          }
        };
        input.click();
      } else {
        // 检查是否是Android平台
        const systemInfo = uni.getSystemInfoSync();
        if (systemInfo.platform === 'android' && typeof plus !== 'undefined') {
          // 在Android端，使用Native.js调用原生文件选择器
          this.chooseFileWithNativeJS('video');
        } else if (uni.chooseMessageFile) {
          // 在其他移动端平台，使用uni.chooseMessageFile
          uni.chooseMessageFile({
            count: 1,
            type: 'all',
            extension: ['.mp4', '.mov', '.avi', '.mkv', '.webm'],
            success: (res) => {
              if (res.tempFiles && res.tempFiles.length > 0) {
                const selectedFile = res.tempFiles[0];
                // 检查文件扩展名
                const fileName = selectedFile.name.toLowerCase();
                const videoExtensions = ['.mp4', '.mov', '.avi', '.mkv', '.webm'];
                const hasValidExtension = videoExtensions.some(ext => fileName.endsWith(ext));
                
                if (!hasValidExtension) {
                  uni.showToast({ title: '请选择有效的视频文件', icon: 'none' });
                  return;
                }
                
                this.videoFileName = selectedFile.name;
                this.videoUrl = selectedFile.path;
                
                // 在所有环境下，使用uni-app的videoContext
                const videoContext = uni.createVideoContext('videoPlayer');
                if (videoContext) {
                  // 尝试使用videoContext播放视频
                  videoContext.play();
                  uni.showToast({ title: '视频加载成功', icon: 'success' });
                } else {
                  uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                }
              }
            },
            fail: (err) => {
              uni.showToast({ title: '选择视频失败', icon: 'none' });
            }
          });
        } else {
          // 如果不支持chooseMessageFile，显示错误信息
          uni.showToast({ title: '当前环境不支持文件选择', icon: 'none' });
        }
      }
    },
    
    /**
     * 使用Native.js选择文件（Android专用）
     */
    chooseFileWithNativeJS(type) {
      const _this = this;
      
      try {
        // 检查plus对象是否存在
        if (typeof plus === 'undefined') {
          throw new Error('plus对象不存在');
        }
        
        // Android文件选择器相关类
        const Intent = plus.android.importClass('android.content.Intent');
        const Activity = plus.android.importClass('android.app.Activity');
        const MainActivity = plus.android.runtimeMainActivity();
        
        if (!MainActivity) {
          throw new Error('无法获取MainActivity');
        }
        
        // 检查并请求存储权限
        const requestStoragePermission = function(callback) {
          // 检查Android版本
          const Build = plus.android.importClass('android.os.Build');
          const VERSION_CODES = plus.android.importClass('android.os.Build$VERSION_CODES');
          
          // Android 6.0及以上需要动态请求权限
          if (Build.VERSION.SDK_INT >= VERSION_CODES.M) {
            const ContextCompat = plus.android.importClass('androidx.core.content.ContextCompat');
            const Manifest = plus.android.importClass('android.Manifest');
            
            // 检查是否已经有存储权限
            const hasPermission = ContextCompat.checkSelfPermission(
              MainActivity,
              Manifest.permission.READ_EXTERNAL_STORAGE
            ) === plus.android.importClass('android.content.pm.PackageManager').PERMISSION_GRANTED;
            
            if (hasPermission) {
              console.log('已经有存储权限');
              callback(true);
            } else {
              // 请求存储权限
              console.log('请求存储权限');
              const ActivityCompat = plus.android.importClass('androidx.core.app.ActivityCompat');
              ActivityCompat.requestPermissions(
                MainActivity,
                [Manifest.permission.READ_EXTERNAL_STORAGE],
                1002
              );
              
              // 监听权限请求结果
              const oldOnRequestPermissionsResult = MainActivity.onRequestPermissionsResult;
              MainActivity.onRequestPermissionsResult = function(requestCode, permissions, grantResults) {
                if (requestCode === 1002) {
                  const PackageManager = plus.android.importClass('android.content.pm.PackageManager');
                  const granted = grantResults.length > 0 && grantResults[0] === PackageManager.PERMISSION_GRANTED;
                  console.log('存储权限请求结果:', granted);
                  callback(granted);
                  // 恢复原始的回调
                  MainActivity.onRequestPermissionsResult = oldOnRequestPermissionsResult;
                } else {
                  // 调用原始的回调
                  if (oldOnRequestPermissionsResult) {
                    oldOnRequestPermissionsResult.call(MainActivity, requestCode, permissions, grantResults);
                  }
                }
              };
            }
          } else {
            // Android 6.0以下不需要动态请求权限
            console.log('Android 6.0以下，不需要动态请求权限');
            callback(true);
          }
        };
        
        // 请求存储权限后打开文件选择器
        requestStoragePermission(function(granted) {
          if (granted) {
            // 创建文件选择意图
            const intent = new Intent(Intent.ACTION_GET_CONTENT);
            
            // 设置文件类型
            if (type === 'video') {
              intent.setType('video/*');
            } else if (type === 'subtitle') {
              // 为字幕文件设置更通用的MIME类型，确保能看到所有文件
              intent.setType('*/*');
              // 直接设置文件类型，不使用EXTRA_MIME_TYPES
            } else {
              intent.setType('*/*');
            }
            
            // 设置选择模式
            intent.putExtra(Intent.EXTRA_ALLOW_MULTIPLE, false);
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            
            // 启动文件选择器
            MainActivity.startActivityForResult(intent, 1001);
          } else {
            // 没有获得存储权限
            uni.showToast({ title: '需要存储权限才能读取文件', icon: 'none' });
          }
        });
        
        // 监听活动结果
        const oldOnActivityResult = MainActivity.onActivityResult;
        MainActivity.onActivityResult = function(requestCode, resultCode, data) {
          if (requestCode === 1001 && resultCode === Activity.RESULT_OK && data) {
            try {
              // 获取选择的文件URI
              const uri = data.getData();
              if (uri) {
                // 使用plus.android.invoke调用URI方法
                const uriString = plus.android.invoke(uri, 'toString');
                console.log('URI字符串:', uriString);
                
                // 从ContentResolver获取实际的文件名
                let fileName = 'temp_file';
                let fileExtension = type === 'video' ? '.mp4' : '.srt';
                let ContentResolver = null;
                
                try {
                  // 获取ContentResolver
                  ContentResolver = MainActivity.getContentResolver();
                  console.log('成功获取ContentResolver');
                  
                  // 尝试获取文件名
                  const Cursor = plus.android.importClass('android.database.Cursor');
                  const projection = ['_display_name', '_data'];
                  
                  // 注意：这里不使用query方法，而是尝试其他方式获取文件名
                  try {
                    // 尝试从URI中获取文件名
                    if (uriString) {
                      // 处理不同类型的URI
                      if (uriString.includes('/document/') || uriString.includes('/file/')) {
                        // 对于document和file URI，尝试从ContentResolver获取文件名
                        console.log('尝试查询ContentResolver获取文件名');
                        try {
                          const cursor = plus.android.invoke(ContentResolver, 'query', uri, projection, null, null, null);
                          if (cursor) {
                            try {
                              if (plus.android.invoke(cursor, 'moveToFirst')) {
                                const displayNameIndex = plus.android.invoke(cursor, 'getColumnIndex', '_display_name');
                                if (displayNameIndex !== -1) {
                                  fileName = plus.android.invoke(cursor, 'getString', displayNameIndex);
                                  // 增加对null的检查
                                  if (!fileName) {
                                    console.error('从ContentResolver获取的文件名为null');
                                    fileName = 'temp_file'; // 重置为临时文件名，以便后续使用默认文件名
                                  } else {
                                    console.log('从ContentResolver获取的文件名:', fileName);
                                  }
                                } else {
                                  console.error('未找到_display_name列');
                                }
                              } else {
                                console.error('Cursor为空');
                              }
                            } catch (e) {
                              console.error('获取文件名失败:', e);
                            } finally {
                              if (cursor) {
                                plus.android.invoke(cursor, 'close');
                              }
                            }
                          } else {
                            console.error('Cursor为null');
                          }
                        } catch (e) {
                          console.error('查询ContentResolver失败:', e);
                        }
                      }
                    }
                  } catch (e) {
                    console.error('获取文件名失败:', e);
                  }
                  
                  // 如果还是没有获取到文件名，使用默认文件名
                  if (fileName === 'temp_file' || !fileName) {
                    // 根据类型生成默认文件名
                    if (type === 'video') {
                      fileName = 'video_' + Date.now() + '.mp4';
                    } else if (type === 'subtitle') {
                      fileName = 'subtitle_' + Date.now() + '.srt';
                    }
                    console.log('使用默认文件名:', fileName);
                  }
                  
                  console.log('最终文件名:', fileName);
                  
                  // 使用plus.io模块处理content:// URI
                  if (uriString.startsWith('content://')) {
                    console.log('使用plus.io处理content:// URI');
                    
                    // 生成uni-app沙箱路径
                    const sandboxPath = '_doc/' + fileName;
                    console.log('沙箱路径:', sandboxPath);
                    
                    try {
                      // 尝试使用plus.io的文件系统操作来处理content:// URI
                      console.log('尝试使用plus.io处理content:// URI');
                      
                      // 验证文件类型
                      if (type === 'video') {
                        const videoExtensions = ['.mp4', '.mov', '.avi', '.mkv', '.webm'];
                        const hasValidExtension = videoExtensions.some(ext => fileName.toLowerCase().endsWith(ext));
                        if (hasValidExtension) {
                          // 尝试将content:// URI复制到应用沙箱
                          try {
                            const ContentResolver = MainActivity.getContentResolver();
                            const inputStream = plus.android.invoke(ContentResolver, 'openInputStream', uri);
                            
                            if (inputStream) {
                              console.log('成功打开视频输入流');
                              
                              // 使用plus.io的文件系统操作
                              plus.io.requestFileSystem(plus.io.PRIVATE_DOC, function(fs) {
                                fs.root.getFile(fileName, {create: true}, function(fileEntry) {
                                  fileEntry.createWriter(function(writer) {
                                    writer.onwriteend = function() {
                                      console.log('视频文件复制成功:', fileEntry.toURL());
                                      _this.videoFileName = fileName;
                                      _this.videoUrl = fileEntry.toURL();
                                      console.log('视频文件路径:', _this.videoUrl);
                                      
                                      // 在所有环境下，使用uni-app的videoContext
                                      const videoContext = uni.createVideoContext('videoPlayer');
                                      if (videoContext) {
                                        // 尝试使用videoContext播放视频
                                        videoContext.play();
                                        uni.showToast({ title: '视频加载成功', icon: 'success' });
                                      } else {
                                        uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                                      }
                                    };
                                    
                                    writer.onerror = function(e) {
                                      console.error('文件写入失败:', e);
                                      // 回退到直接使用content:// URI
                                      _this.videoFileName = fileName;
                                      _this.videoUrl = uriString;
                                      console.log('回退到直接使用content:// URI:', uriString);
                                      
                                      // 在所有环境下，使用uni-app的videoContext
                                      const videoContext = uni.createVideoContext('videoPlayer');
                                      if (videoContext) {
                                        // 尝试使用videoContext播放视频
                                        videoContext.play();
                                        uni.showToast({ title: '视频加载成功', icon: 'success' });
                                      } else {
                                        uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                                      }
                                    };
                                    
                                    // 读取inputStream并写入文件
                                    try {
                                      const ByteArrayOutputStream = plus.android.importClass('java.io.ByteArrayOutputStream');
                                      const byteArrayOutputStream = new ByteArrayOutputStream();
                                      const buffer = plus.android.newArray('byte', 1024);
                                      let bytesRead;
                                      
                                      while ((bytesRead = plus.android.invoke(inputStream, 'read', buffer)) !== -1) {
                                        plus.android.invoke(byteArrayOutputStream, 'write', buffer, 0, bytesRead);
                                      }
                                      
                                      plus.android.invoke(inputStream, 'close');
                                      
                                      const bytes = plus.android.invoke(byteArrayOutputStream, 'toByteArray');
                                      const blob = new Blob([bytes], {type: 'video/mp4'});
                                      writer.write(blob);
                                    } catch (e) {
                                      console.error('读取视频流失败:', e);
                                      writer.onerror(e);
                                    }
                                  }, function(error) {
                                    console.error('创建文件写入器失败:', error);
                                    // 回退到直接使用content:// URI
                                    _this.videoFileName = fileName;
                                    _this.videoUrl = uriString;
                                    console.log('回退到直接使用content:// URI:', uriString);
                                    
                                    // 在所有环境下，使用uni-app的videoContext
                                    const videoContext = uni.createVideoContext('videoPlayer');
                                    if (videoContext) {
                                      // 尝试使用videoContext播放视频
                                      videoContext.play();
                                      uni.showToast({ title: '视频加载成功', icon: 'success' });
                                    } else {
                                      uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                                    }
                                  });
                                }, function(error) {
                                  console.error('获取文件失败:', error);
                                  // 回退到直接使用content:// URI
                                  _this.videoFileName = fileName;
                                  _this.videoUrl = uriString;
                                  console.log('回退到直接使用content:// URI:', uriString);
                                  
                                  // 在所有环境下，使用uni-app的videoContext
                                  const videoContext = uni.createVideoContext('videoPlayer');
                                  if (videoContext) {
                                    // 尝试使用videoContext播放视频
                                    videoContext.play();
                                    uni.showToast({ title: '视频加载成功', icon: 'success' });
                                  } else {
                                    uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                                  }
                                });
                              }, function(error) {
                                console.error('请求文件系统失败:', error);
                                // 回退到直接使用content:// URI
                                _this.videoFileName = fileName;
                                _this.videoUrl = uriString;
                                console.log('回退到直接使用content:// URI:', uriString);
                                
                                // 在所有环境下，使用uni-app的videoContext
                                const videoContext = uni.createVideoContext('videoPlayer');
                                if (videoContext) {
                                  // 尝试使用videoContext播放视频
                                  videoContext.play();
                                  uni.showToast({ title: '视频加载成功', icon: 'success' });
                                } else {
                                  uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                                }
                              });
                            } else {
                              console.error('无法打开输入流，尝试使用备用方案');
                              // 备用方案：直接使用content:// URI
                              _this.videoFileName = fileName;
                              _this.videoUrl = uriString;
                              console.log('回退到直接使用content:// URI:', uriString);
                              
                              // 在所有环境下，使用uni-app的videoContext
                              const videoContext = uni.createVideoContext('videoPlayer');
                              if (videoContext) {
                                // 尝试使用videoContext播放视频
                                videoContext.play();
                                uni.showToast({ title: '视频加载成功', icon: 'success' });
                              } else {
                                uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                              }
                            }
                          } catch (e) {
                            console.error('复制文件失败:', e);
                            // 回退到直接使用content:// URI
                            _this.videoFileName = fileName;
                            _this.videoUrl = uriString;
                            console.log('回退到直接使用content:// URI:', uriString);
                            
                            // 在所有环境下，使用uni-app的videoContext
                            const videoContext = uni.createVideoContext('videoPlayer');
                            if (videoContext) {
                              // 尝试使用videoContext播放视频
                              videoContext.play();
                              uni.showToast({ title: '视频加载成功', icon: 'success' });
                            } else {
                              uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                            }
                          }
                        } else {
                          uni.showToast({ title: '请选择有效的视频文件', icon: 'none' });
                        }
                      } else if (type === 'subtitle') {
                        const subtitleExtensions = ['.srt', '.vtt', '.ass', '.ssa'];
                        const hasValidExtension = subtitleExtensions.some(ext => fileName.toLowerCase().endsWith(ext));
                        if (hasValidExtension) {
                          _this.subtitleFileName = fileName;
                          console.log('字幕文件路径:', uriString);
                          
                          // 尝试使用Native.js直接读取输入流内容
                          try {
                            const ContentResolver = MainActivity.getContentResolver();
                            console.log('尝试打开输入流，URI:', uriString);
                            const inputStream = plus.android.invoke(ContentResolver, 'openInputStream', uri);
                            
                            if (inputStream) {
                              console.log('成功打开输入流，开始读取内容');
                              
                              // 使用Java的Scanner来读取文本内容
                              try {
                                const Scanner = plus.android.importClass('java.util.Scanner');
                                const scanner = new Scanner(inputStream, 'UTF-8');
                                let content = '';
                                
                                // 使用useDelimiter读取所有内容
                                scanner.useDelimiter('\\A');
                                if (scanner.hasNext()) {
                                  content = scanner.next();
                                }
                                
                                scanner.close();
                                plus.android.invoke(inputStream, 'close');
                                
                                if (content) {
                                  console.log('成功读取字幕内容，长度:', content.length);
                                  console.log('字幕内容前100字符:', content.substring(0, 100));
                                  _this.parseSubtitles(content);
                                  uni.showToast({ title: '字幕解析成功', icon: 'success' });
                                } else {
                                  console.error('读取到的内容为空');
                                  uni.showToast({ title: '字幕内容为空', icon: 'none' });
                                }
                              } catch (scannerError) {
                                console.error('Scanner读取失败:', scannerError);
                                // 备用方案：使用BufferedReader
                                try {
                                  const BufferedReader = plus.android.importClass('java.io.BufferedReader');
                                  const InputStreamReader = plus.android.importClass('java.io.InputStreamReader');
                                  const reader = new BufferedReader(new InputStreamReader(inputStream, 'UTF-8'));
                                  
                                  let line;
                                  let content = '';
                                  while ((line = plus.android.invoke(reader, 'readLine')) !== null) {
                                    content += line + '\n';
                                  }
                                  
                                  reader.close();
                                  plus.android.invoke(inputStream, 'close');
                                  
                                  if (content) {
                                    console.log('使用BufferedReader成功读取字幕内容，长度:', content.length);
                                    _this.parseSubtitles(content);
                                    uni.showToast({ title: '字幕解析成功', icon: 'success' });
                                  } else {
                                    console.error('BufferedReader读取到的内容为空');
                                    uni.showToast({ title: '字幕内容为空', icon: 'none' });
                                  }
                                } catch (bufferedReaderError) {
                                  console.error('BufferedReader读取失败:', bufferedReaderError);
                                  uni.showToast({ title: '字幕读取失败', icon: 'none' });
                                }
                              }
                            } else {
                              console.error('无法打开输入流，尝试使用备用方案');
                              // 备用方案：尝试使用DocumentFile API
                              try {
                                const DocumentFile = plus.android.importClass('androidx.documentfile.provider.DocumentFile');
                                const docFile = DocumentFile.fromSingleUri(MainActivity, uri);
                                if (docFile && docFile.exists()) {
                                  console.log('使用DocumentFile API成功获取文件');
                                  // 尝试使用FileDescriptor读取
                                  const ParcelFileDescriptor = plus.android.importClass('android.os.ParcelFileDescriptor');
                                  const pfd = plus.android.invoke(ContentResolver, 'openFileDescriptor', uri, 'r');
                                  if (pfd) {
                                    const fileDescriptor = plus.android.invoke(pfd, 'getFileDescriptor');
                                    const FileInputStream = plus.android.importClass('java.io.FileInputStream');
                                    const fis = new FileInputStream(fileDescriptor);
                                    
                                    const BufferedReader = plus.android.importClass('java.io.BufferedReader');
                                    const InputStreamReader = plus.android.importClass('java.io.InputStreamReader');
                                    const reader = new BufferedReader(new InputStreamReader(fis, 'UTF-8'));
                                    
                                    let line;
                                    let content = '';
                                    while ((line = plus.android.invoke(reader, 'readLine')) !== null) {
                                      content += line + '\n';
                                    }
                                    
                                    reader.close();
                                    fis.close();
                                    plus.android.invoke(pfd, 'close');
                                    
                                    if (content) {
                                      console.log('使用FileDescriptor成功读取字幕内容，长度:', content.length);
                                      _this.parseSubtitles(content);
                                      uni.showToast({ title: '字幕解析成功', icon: 'success' });
                                    } else {
                                      console.error('FileDescriptor读取到的内容为空');
                                      uni.showToast({ title: '字幕内容为空', icon: 'none' });
                                    }
                                  } else {
                                    console.error('无法打开FileDescriptor');
                                    uni.showToast({ title: '无法读取字幕文件', icon: 'none' });
                                  }
                                } else {
                                  console.error('DocumentFile API无法获取文件');
                                  uni.showToast({ title: '无法读取字幕文件', icon: 'none' });
                                }
                              } catch (docFileError) {
                                console.error('DocumentFile API失败:', docFileError);
                                uni.showToast({ title: '无法读取字幕文件', icon: 'none' });
                              }
                            }
                          } catch (e) {
                            console.error('读取字幕文件失败:', e);
                            uni.showToast({ title: '字幕读取失败', icon: 'none' });
                          }
                        } else {
                          uni.showToast({ title: '请选择有效的字幕文件', icon: 'none' });
                        }
                      }
                    } catch (e) {
                      console.error('文件保存过程异常:', e);
                      uni.showToast({ title: '文件保存失败', icon: 'none' });
                    }
                  } else if (uriString.startsWith('file://')) {
                    // 直接从file:// URI获取路径
                    const filePath = uriString.substring(7); // 移除file://前缀
                    console.log('直接使用文件路径:', filePath);
                    
                    // 验证文件类型
                    if (type === 'video') {
                      const videoExtensions = ['.mp4', '.mov', '.avi', '.mkv', '.webm'];
                      const hasValidExtension = videoExtensions.some(ext => fileName.toLowerCase().endsWith(ext));
                      if (hasValidExtension) {
                        _this.videoFileName = fileName;
                        _this.videoUrl = filePath;
                        
                        // 检查是否存在document对象
                        // 在所有环境下，使用uni-app的videoContext
                        const videoContext = uni.createVideoContext('videoPlayer');
                        if (videoContext) {
                          // 尝试使用videoContext播放视频
                          videoContext.play();
                          uni.showToast({ title: '视频加载成功', icon: 'success' });
                        } else {
                          uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
                        }
                      } else {
                        uni.showToast({ title: '请选择有效的视频文件', icon: 'none' });
                      }
                    } else if (type === 'subtitle') {
                      const subtitleExtensions = ['.srt', '.vtt', '.ass', '.ssa'];
                      const hasValidExtension = subtitleExtensions.some(ext => fileName.toLowerCase().endsWith(ext));
                      if (hasValidExtension) {
                        _this.subtitleFileName = fileName;
                        
                        // 读取文件内容
                        console.log('尝试读取字幕文件:', filePath);
                        uni.readFile({
                          filePath: filePath,
                          encoding: 'utf-8',
                          success: (readRes) => {
                            console.log('读取文件成功，内容长度:', readRes.data.length);
                            console.log('文件内容前100字符:', readRes.data.substring(0, 100));
                            _this.parseSubtitles(readRes.data);
                            uni.showToast({ title: '字幕解析成功', icon: 'success' });
                          },
                          fail: (err) => {
                            console.error('读取字幕文件失败:', err);
                            uni.showToast({ title: '字幕读取失败: ' + JSON.stringify(err), icon: 'none' });
                          }
                        });
                      } else {
                        uni.showToast({ title: '请选择有效的字幕文件', icon: 'none' });
                      }
                    }
                  }
                } catch (e) {
                  console.error('处理文件选择失败:', e);
                  // 尝试回退方案
                  uni.showToast({ title: '尝试使用系统文件选择器', icon: 'none' });
                  if (uni.chooseMessageFile) {
                    uni.chooseMessageFile({
                      count: 1,
                      type: type === 'video' ? 'all' : 'file',
                      extension: type === 'video' ? ['.mp4', '.mov', '.avi', '.mkv', '.webm'] : ['.srt', '.vtt', '.ass', '.ssa'],
                      success: (res) => {
                        if (res.tempFiles && res.tempFiles.length > 0) {
                          const selectedFile = res.tempFiles[0];
                          if (type === 'video') {
                            _this.videoFileName = selectedFile.name;
                            _this.videoUrl = selectedFile.path;
                            
                            if (typeof document !== 'undefined') {
                              const videoElement = document.querySelector('video');
                              if (videoElement) {
                                videoElement.src = selectedFile.path;
                                videoElement.load();
                              }
                            }
                            uni.showToast({ title: '视频加载成功', icon: 'success' });
                          } else if (type === 'subtitle') {
                            _this.subtitleFileName = selectedFile.name;
                            uni.readFile({
                              filePath: selectedFile.path,
                              encoding: 'utf-8',
                              success: (readRes) => {
                                _this.parseSubtitles(readRes.data);
                                uni.showToast({ title: '字幕解析成功', icon: 'success' });
                              },
                              fail: (err) => {
                                console.error('读取字幕文件失败:', err);
                                uni.showToast({ title: '字幕读取失败', icon: 'none' });
                              }
                            });
                          }
                        }
                      },
                      fail: (err) => {
                        console.error('uni.chooseMessageFile失败:', err);
                        uni.showToast({ title: '选择文件失败', icon: 'none' });
                      }
                    });
                  } else {
                    uni.showToast({ title: '当前环境不支持文件选择', icon: 'none' });
                  }
                }
              } else {
                console.error('URI为null');
                uni.showToast({ title: '选择文件失败', icon: 'none' });
              }
            } catch (e) {
              console.error('处理文件选择结果失败:', e);
              uni.showToast({ title: '选择文件失败', icon: 'none' });
            }
          } else if (requestCode === 1001 && resultCode === Activity.RESULT_CANCELED) {
            // 用户取消选择
            console.log('用户取消选择文件');
          } else {
            console.error('文件选择结果错误:', 'requestCode=' + requestCode, 'resultCode=' + resultCode);
            uni.showToast({ title: '选择文件失败', icon: 'none' });
          }
          
          // 恢复原始的onActivityResult
          MainActivity.onActivityResult = oldOnActivityResult;
        };
      } catch (e) {
        console.error('Native.js调用失败:', e);
        // 如果Native.js调用失败，回退到uni.chooseMessageFile
        if (uni.chooseMessageFile) {
          uni.showToast({ title: '使用系统文件选择器', icon: 'none' });
          uni.chooseMessageFile({
            count: 1,
            type: type === 'video' ? 'all' : 'file',
            extension: type === 'video' ? ['.mp4', '.mov', '.avi', '.mkv', '.webm'] : ['.srt', '.vtt', '.ass', '.ssa'],
            success: (res) => {
              if (res.tempFiles && res.tempFiles.length > 0) {
                const selectedFile = res.tempFiles[0];
                if (type === 'video') {
                  _this.videoFileName = selectedFile.name;
                  _this.videoUrl = selectedFile.path;
                  
                  if (typeof document !== 'undefined') {
                    const videoElement = document.querySelector('video');
                    if (videoElement) {
                      videoElement.src = selectedFile.path;
                      videoElement.load();
                    }
                  }
                  uni.showToast({ title: '视频加载成功', icon: 'success' });
                } else if (type === 'subtitle') {
                  _this.subtitleFileName = selectedFile.name;
                  uni.readFile({
                    filePath: selectedFile.path,
                    encoding: 'utf-8',
                    success: (readRes) => {
                      _this.parseSubtitles(readRes.data);
                      uni.showToast({ title: '字幕解析成功', icon: 'success' });
                    },
                    fail: (err) => {
                      console.error('读取字幕文件失败:', err);
                      uni.showToast({ title: '字幕读取失败', icon: 'none' });
                    }
                  });
                }
              }
            },
            fail: (err) => {
              console.error('uni.chooseMessageFile失败:', err);
              uni.showToast({ title: '选择文件失败', icon: 'none' });
            }
          });
        } else {
          uni.showToast({ title: '当前环境不支持文件选择', icon: 'none' });
        }
      }
    },
    
    /**
     * 选择字幕文件
     */
    chooseSubtitleFile() {
      // 检查是否在H5环境且存在document对象
      if (typeof document !== 'undefined') {
        // 在H5端，使用input[type=file]来选择文件
        const input = document.createElement('input');
        input.type = 'file';
        input.accept = '.srt,.vtt,.ass,.ssa';
        input.onchange = (e) => {
          const file = e.target.files[0];
          if (file) {
            this.subtitleFileName = file.name;
            const reader = new FileReader();
            reader.onload = (event) => {
              const content = event.target.result;
              this.parseSubtitles(content);
              uni.showToast({ title: '字幕解析成功', icon: 'success' });
            };
            reader.onerror = () => {
              uni.showToast({ title: '字幕读取失败', icon: 'none' });
            };
            reader.readAsText(file, 'utf-8');
          }
        };
        input.click();
      } else {
        // 检查是否是Android平台
        const systemInfo = uni.getSystemInfoSync();
        if (systemInfo.platform === 'android' && typeof plus !== 'undefined') {
          // 在Android端，使用Native.js调用原生文件选择器
          this.chooseFileWithNativeJS('subtitle');
        } else if (uni.chooseMessageFile) {
          // 在其他移动端平台，使用uni.chooseMessageFile
          uni.chooseMessageFile({
            count: 1,
            type: 'file',
            extension: ['.srt', '.vtt', '.ass', '.ssa'],
            success: (res) => {
              console.log('选择字幕文件成功:', res);
              if (res.tempFiles && res.tempFiles.length > 0) {
                const selectedFile = res.tempFiles[0];
                console.log('选中的文件:', selectedFile);
                // 检查文件扩展名
                const fileName = selectedFile.name.toLowerCase();
                const subtitleExtensions = ['.srt', '.vtt', '.ass', '.ssa'];
                const hasValidExtension = subtitleExtensions.some(ext => fileName.endsWith(ext));
                
                if (!hasValidExtension) {
                  uni.showToast({ title: '请选择有效的字幕文件', icon: 'none' });
                  return;
                }
                
                this.subtitleFileName = selectedFile.name;
                // 读取文件内容
                uni.readFile({
                  filePath: selectedFile.path,
                  encoding: 'utf-8',
                  success: (readRes) => {
                    console.log('读取文件成功，内容长度:', readRes.data.length);
                    this.parseSubtitles(readRes.data);
                    uni.showToast({ title: '字幕解析成功', icon: 'success' });
                  },
                  fail: (err) => {
                    console.error('读取文件失败:', err);
                    uni.showToast({ title: '字幕读取失败: ' + JSON.stringify(err), icon: 'none' });
                  }
                });
              }
            },
            fail: (err) => {
              console.error('选择字幕失败:', err);
              uni.showToast({ title: '选择字幕失败: ' + JSON.stringify(err), icon: 'none' });
            }
          });
        } else {
          // 如果不支持chooseMessageFile，显示错误信息
          uni.showToast({ title: '当前环境不支持文件选择', icon: 'none' });
        }
      }
    },
    

    
    
    
    /**
     * 解析字幕文件
     */
    parseSubtitles(subtitleContent) {
      const subtitles = [];
      
      try {
        console.log('开始解析字幕，内容长度:', subtitleContent.length);
        console.log('字幕内容前200字符:', subtitleContent.substring(0, 200));
                
        // 处理不同的换行符
        const content = subtitleContent.replace(/\r\n/g, '\n').replace(/\r/g, '\n');
        console.log('处理换行符后内容长度:', content.length);
        
        // 检查是否是 ASS/SSA 格式
        if (content.includes('[Events]')) {
          console.log('检测到 ASS/SSA 格式字幕');
                    
          // 找到 Events 部分
          const eventsIndex = content.indexOf('[Events]');
          if (eventsIndex !== -1) {
            const eventsContent = content.substring(eventsIndex);
            const lines = eventsContent.split('\n');
            console.log('ASS/SSA 格式行数:', lines.length);
            
                        
            for (const line of lines) {
              if (line.startsWith('Dialogue:')) {
                // 解析 Dialogue 行
                const parts = line.split(',');
                if (parts.length >= 10) {
                  const startTimeStr = parts[1].trim();
                  const endTimeStr = parts[2].trim();
                  const textParts = parts.slice(9);
                  const text = textParts.join(',').trim();
                  
                  // 解析时间戳
                  const parseTime = (timeStr) => {
                    const parts = timeStr.split(':');
                    if (parts.length === 3) {
                      const hours = parseInt(parts[0]);
                      const minutes = parseInt(parts[1]);
                      const secondsParts = parts[2].split('.');
                      const seconds = parseInt(secondsParts[0]);
                      const centiseconds = parseInt(secondsParts[1] || '00');
                      return hours * 3600 + minutes * 60 + seconds + centiseconds / 100;
                    }
                    return 0;
                  };
                  
                  const startTime = parseTime(startTimeStr);
                  const endTime = parseTime(endTimeStr);
                  
                  // 移除 ASS 标签
                  let cleanText = text.replace(/\{[^}]+\}/g, '').trim();
                  
                  // 处理 \N 换行符
                  cleanText = cleanText.replace(/\\N/g, ' ').trim();
                  // 清理广告内容和网址
                  cleanText = this.cleanSubtitleText(cleanText);
                  // 清理乱码：移除类似 "m 0 0 1 0" 这样的乱码
                  cleanText = cleanText.replace(/\b[a-zA-Z]\s+[\d\s]+\b/g, '').trim();
                  // 清理多余的空格
                  cleanText = cleanText.replace(/\s+/g, ' ').trim();
                                    
                  if (cleanText) {
                    const subtitle = {
                      startTime,
                      endTime,
                      text: cleanText
                    };
                    subtitles.push(subtitle);
                                      }
                }
              }
            }
          }
        } else {
          // 按 SRT 格式解析
          console.log('检测到 SRT 格式字幕');
                    
          // 分割字幕块
          const blocks = content.split(/\n\s*\n/);
          console.log('SRT 格式字幕块数:', blocks.length);
                    
          // 找到第一个有效字幕块的索引
          let startBlockIndex = 0;
          for (let i = 0; i < blocks.length; i++) {
            const block = blocks[i];
            if (!block.trim()) continue;
            
            const lines = block.split('\n').filter(line => line.trim());
                        
            if (lines.length < 2) continue;
            
            let timeLineIndex = 0;
            // 跳过序号行
            if (!isNaN(parseInt(lines[0]))) {
              timeLineIndex = 1;
            }
            
            // 尝试匹配时间戳格式
            const timeLine = lines[timeLineIndex];
            const timeMatch = timeLine.match(/(\d+):(\d+):(\d+)[,.](\d+)\s*-->\s*(\d+):(\d+):(\d+)[,.](\d+)/);
            
            if (timeMatch) {
              startBlockIndex = i;
              break;
            }
          }
                    
          // 从第一个有效字幕块开始解析
          for (let i = startBlockIndex; i < blocks.length; i++) {
            const block = blocks[i];
            if (!block.trim()) continue;
            
            const lines = block.split('\n').filter(line => line.trim());
                        
            if (lines.length < 2) continue;
            
            let timeLineIndex = 0;
            // 跳过序号行
            if (!isNaN(parseInt(lines[0]))) {
              timeLineIndex = 1;
            }
            
            // 解析时间戳
            const timeLine = lines[timeLineIndex];
                        
            // 尝试匹配时间戳格式
            const timeMatch = timeLine.match(/(\d+):(\d+):(\d+)[,.](\d+)\s*-->\s*(\d+):(\d+):(\d+)[,.](\d+)/);
            
            if (timeMatch) {
                            const startHours = parseInt(timeMatch[1]);
              const startMinutes = parseInt(timeMatch[2]);
              const startSeconds = parseInt(timeMatch[3]);
              const startMs = parseInt(timeMatch[4]);
              
              const endHours = parseInt(timeMatch[5]);
              const endMinutes = parseInt(timeMatch[6]);
              const endSeconds = parseInt(timeMatch[7]);
              const endMs = parseInt(timeMatch[8]);
              
              const startTime = startHours * 3600 + startMinutes * 60 + startSeconds + startMs / 1000;
              const endTime = endHours * 3600 + endMinutes * 60 + endSeconds + endMs / 1000;
              
              // 解析字幕文本
              const textLines = lines.slice(timeLineIndex + 1);
              let text = textLines.join(' ').trim();
              
              // 处理 \N 换行符
              text = text.replace(/\\N/g, ' ').trim();
              // 清理广告内容和网址
              text = this.cleanSubtitleText(text);
              // 清理乱码：移除类似 "m 0 0 1 0" 这样的乱码
              text = text.replace(/\b[a-zA-Z]\s+[\d\s]+\b/g, '').trim();
              // 清理多余的空格
              text = text.replace(/\s+/g, ' ').trim();
                            
              if (text) {
                const subtitle = {
                  startTime,
                  endTime,
                  text
                };
                subtitles.push(subtitle);
                              }
            } else {
                          }
          }
        }
        
        console.log('解析完成，字幕数量:', subtitles.length);
                        
        // 确保赋值正确
        this.subtitles = [...subtitles];
                
        if (subtitles.length === 0) {
          uni.showToast({ title: '未解析到有效字幕', icon: 'none' });
        } else {
          uni.showToast({ title: `成功解析 ${subtitles.length} 条字幕`, icon: 'success' });
        }
      } catch (error) {
        console.error('字幕解析失败:', error);
                uni.showToast({ title: '字幕解析失败: ' + JSON.stringify(error), icon: 'none' });
        this.subtitles = [];
      }
    },
    
    /**
     * 跳转到指定时间点
     */
    jumpToTime(time) {
      console.log('跳转到时间点:', time, '视频URL:', this.videoUrl);
                  
      // 检查视频URL是否存在
      if (!this.videoUrl) {
        console.log('视频URL不存在');
                uni.showToast({ title: '请先选择视频文件', icon: 'none' });
        return;
      }
      
      // 检查是否存在document对象
      if (typeof document !== 'undefined') {
        console.log('H5环境');
        // 获取原生video元素
        const videoElement = document.querySelector('video');
        console.log('视频元素:', videoElement);
              
        if (videoElement) {
          console.log('视频元素存在，readyState:', videoElement.readyState);
                                     
          // 检查视频是否已加载
          if (videoElement.readyState === 0) {
            console.log('视频未加载，设置src并加载');
                      videoElement.src = this.videoUrl;
            videoElement.load();
            
            // 监听视频加载完成事件
            videoElement.addEventListener('loadedmetadata', () => {
              console.log('视频加载完成，跳转到时间:', time);
                          videoElement.currentTime = time;
              videoElement.play().catch(err => {
                console.error('自动播放失败:', err);
                              uni.showToast({ title: '视频播放失败，请检查视频文件', icon: 'none' });
              });
            });
            
            videoElement.addEventListener('error', (err) => {
              console.error('视频加载失败:', err);
                          uni.showToast({ title: '视频加载失败，请检查文件格式', icon: 'none' });
            });
          } else {
            // 视频已加载，直接播放
            console.log('视频已加载，直接跳转到时间:', time);
                      videoElement.currentTime = time;
            // 更新当前时间，触发字幕滚动
            this.currentTimeSec = time;
            this.updateCurrentSubtitle();
            videoElement.play().catch(err => {
              console.error('播放失败:', err);
                          uni.showToast({ title: '视频播放失败，请检查视频文件', icon: 'none' });
            });
          }
        } else {
          console.log('视频元素未找到');
                  uni.showToast({ title: '视频元素未找到', icon: 'none' });
        }
      } else {
        console.log('非H5环境');
        // 确保视频上下文已初始化
        if (!this.videoContext) {
          console.log('视频上下文未初始化，重新获取');
          this.getVideoContext();
        }
        
        if (this.videoContext) {
          console.log('使用存储的videoContext，跳转到时间:', time, 'isVideoReadyForSeek:', this.isVideoReadyForSeek);
          
          // 确保视频已准备好进行跳转
          if (this.isVideoReadyForSeek) {
            // 使用插件的API执行跳转
            console.log('执行toSeek操作，时间:', time);
            this.videoContext.toSeek(time, false); // 直接执行跳转
            // 更新当前时间，触发字幕滚动
            this.currentTimeSec = time;
            this.updateCurrentSubtitle();
            setTimeout(() => {
              console.log('执行play操作');
              this.videoContext.play();
              uni.showToast({ title: `跳转到 ${this.formatTime(time)}`, icon: 'success' });
            }, 500);
          } else {
            // 视频未准备好，等待一段时间后重试
            console.log('视频未准备好，等待后重试');
            setTimeout(() => {
              if (this.videoContext) {
                console.log('重试执行toSeek操作，时间:', time);
                this.videoContext.toSeek(time, false);
                // 更新当前时间，触发字幕滚动
                this.currentTimeSec = time;
                this.updateCurrentSubtitle();
                setTimeout(() => {
                  console.log('执行play操作');
                  this.videoContext.play();
                  uni.showToast({ title: `跳转到 ${this.formatTime(time)}`, icon: 'success' });
                }, 500);
              }
            }, 1000);
          }
        } else {
          console.log('视频播放器未初始化');
          uni.showToast({ title: '视频播放器未初始化', icon: 'none' });
        }
      }
    },
    
    /**
     * 格式化时间显示
     */
    formatTime(seconds) {
      const h = Math.floor(seconds / 3600);
      const m = Math.floor((seconds % 3600) / 60);
      const s = Math.floor(seconds % 60);
      return h.toString().padStart(2, '0') + ':' + m.toString().padStart(2, '0') + ':' + s.toString().padStart(2, '0');
    },
    
    /**
     * 清理字幕文本中的广告内容和网址
     */
    cleanSubtitleText(text) {
      let cleanText = text;
      
      // 清理网址链接（http、https、www开头）
      cleanText = cleanText.replace(/https?:\/\/[^\s]+/gi, '').trim();
      cleanText = cleanText.replace(/www\.[^\s]+/gi, '').trim();
      
      // 广告关键词
      const adKeywords = ['深影', 'ShinY', '字幕组', '论坛', '微博', '微信公众号', 'shinybbs', 'weibo.com'];
      
      // 使用广告关键词过滤
      adKeywords.forEach(keyword => {
        const regex = new RegExp(keyword, 'gi');
        cleanText = cleanText.replace(regex, '').trim();
      });
      
      // 清理常见的广告关键词模式
      const adPatterns = [
        /更多精彩.*论坛/g,
        /字幕组.*微博/g,
        /关注.*字幕组.*公众号/g,
        /微信公众号.*影视资讯/g,
        /论坛.*vip/g,
        /更多资源.*访问/g,
        /版权声明/g,
        /制作.*字幕组/g,
        /翻译.*校对/g,
        /压制.*后期/g,
        /海报.*设计/g,
        /请登录/g,
        /等你來发现/g
      ];
      
      adPatterns.forEach(pattern => {
        cleanText = cleanText.replace(pattern, '').trim();
      });
      
      // 清理多余的标点符号和空格
      cleanText = cleanText.replace(/[，。！？：；]/g, '').trim();
      cleanText = cleanText.replace(/\s+/g, ' ').trim();
      
      // 清理纯装饰符号（如 ■ 、 ★ 、 ● 等）单独成行的内容
      // 匹配仅包含装饰符号的行
      cleanText = cleanText.replace(/^[\s■★●◆▲△▼▏▎▍▌▋▊▉█▇▆▅▄▃▂▁]+$/gm, '').trim();
      
      return cleanText;
    },
    
    /**
     * 视频时间更新事件
     */
    onTimeUpdate() {
      // 在uni-app中，video组件的时间更新需要通过事件监听
      // 这里保留方法结构，实际更新由video组件的@timeupdate事件触发
      // 更新当前字幕索引
      this.updateCurrentSubtitle();
      
      // 处理循环播放
      this.handleLoop();
    },
    
    /**
     * 更新当前播放的字幕索引
     */
    updateCurrentSubtitle() {
      const currentTime = this.currentTimeSec;
      let foundIndex = -1;
      
      for (let i = 0; i < this.subtitles.length; i++) {
        const subtitle = this.subtitles[i];
        if (currentTime >= subtitle.startTime && currentTime <= subtitle.endTime) {
          foundIndex = i;
          break;
        }
      }
      
      // 只有当字幕索引发生变化时才更新并滚动
      if (this.currentSubtitleIndex !== foundIndex) {
        this.currentSubtitleIndex = foundIndex;
        // 滚动字幕到当前位置
        this.scrollToCurrentSubtitle();
      }
    },
    
    /**
     * 节流函数：限制函数执行频率
     */
    throttle(func, delay) {
      let timer = null;
      return function() {
        const context = this;
        const args = arguments;
        if (!timer) {
          timer = setTimeout(() => {
            func.apply(context, args);
            timer = null;
          }, delay);
        }
      };
    },

    /**
     * 滚动字幕到当前播放的字幕位置
     * 使用 scroll-into-view 原生方法，滚动到锚点元素使字幕项居中显示
     */
    scrollToCurrentSubtitle() {
      if (this.currentSubtitleIndex < 0 || this.subtitles.length === 0) return;

      // 节流处理，避免频繁滚动
      if (!this.throttledScroll) {
        this.throttledScroll = this.throttle((index) => {
          // 滚动到锚点元素，使字幕项居中显示
          const targetId = 'subtitle-anchor-' + index;
          
          // 重置 scrollIntoViewId 以确保滚动行为稳定
          this.scrollIntoViewId = '';
          
          // 使用 setTimeout 确保 DOM 更新后再设置新的目标
          setTimeout(() => {
            this.scrollIntoViewId = targetId;
          }, 0);
        }, 100); // 100ms 节流
      }

      this.throttledScroll(this.currentSubtitleIndex);
    },
    
    /**
     * 播放事件
     */
    onPlay() {
      this.isPlaying = true;
    },
    
    /**
     * 暂停事件
     */
    onPause() {
      this.isPlaying = false;
    },
    
    /**
     * 切换循环模式
     */
    toggleLoop() {
      this.loopMode = !this.loopMode;
      if (!this.loopMode) {
        this.loopStart = 0;
        this.loopEnd = 0;
      }
          },
    
    /**
     * 设置循环开始点
     * 在H5环境下使用原生video元素获取当前时间，在非H5环境下使用当前播放时间
     */
    setStartPoint() {
      console.log('设置起点，currentTimeSec:', this.currentTimeSec, 'isPlaying:', this.isPlaying);
      // 检查是否存在document对象（H5环境）
      if (typeof document !== 'undefined') {
        // 获取原生video元素
        const videoElement = document.querySelector('video');
        if (videoElement) {
          this.loopStart = videoElement.currentTime;
          console.log('H5环境，获取到时间:', this.loopStart);
          uni.showToast({ title: `起点 A: ${this.formatTime(this.loopStart)}`, icon: 'none' });
                }
      } else {
        // 在非H5环境，使用当前播放时间
        console.log('非H5环境，currentTimeSec:', this.currentTimeSec, 'isPlaying:', this.isPlaying);
        // 无论currentTimeSec是否为0，只要视频正在播放，就设置起点
        if (this.isPlaying) {
          // 使用当前时间或默认值
          this.loopStart = this.currentTimeSec > 0 ? this.currentTimeSec : 0.1;
          console.log('非H5环境，设置起点时间:', this.loopStart);
          uni.showToast({ title: `起点 A: ${this.formatTime(this.loopStart)}`, icon: 'none' });
        } else {
          uni.showToast({ title: '请先播放视频再设置起点', icon: 'none' });
        }
      }
    },
    
    /**
     * 设置循环结束点
     * 在H5环境下使用原生video元素获取当前时间，在非H5环境下使用当前播放时间
     */
    setEndPoint() {
      console.log('设置终点，currentTimeSec:', this.currentTimeSec, 'isPlaying:', this.isPlaying);
      // 检查是否存在document对象（H5环境）
      if (typeof document !== 'undefined') {
        // 获取原生video元素
        const videoElement = document.querySelector('video');
        if (videoElement) {
          this.loopEnd = videoElement.currentTime;
          console.log('H5环境，获取到时间:', this.loopEnd);
          uni.showToast({ title: `终点 B: ${this.formatTime(this.loopEnd)}`, icon: 'none' });
                }
      } else {
        // 在非H5环境，使用当前播放时间
        console.log('非H5环境，currentTimeSec:', this.currentTimeSec, 'isPlaying:', this.isPlaying);
        // 无论currentTimeSec是否为0，只要视频正在播放，就设置终点
        if (this.isPlaying) {
          // 使用当前时间或默认值
          this.loopEnd = this.currentTimeSec > 0 ? this.currentTimeSec : 1.0;
          console.log('非H5环境，设置终点时间:', this.loopEnd);
          uni.showToast({ title: `终点 B: ${this.formatTime(this.loopEnd)}`, icon: 'none' });
        } else {
          uni.showToast({ title: '请先播放视频再设置终点', icon: 'none' });
        }
      }
    },
    
    /**
     * 清除循环点
     */
    clearLoopPoints() {
      this.loopStart = 0;
      this.loopEnd = 0;
      uni.showToast({ title: '已清除循环点', icon: 'none' });
    },
    
    /**
     * 视频时间更新事件处理
     * 处理两种事件格式：
     * 1. 标准事件格式：包含 detail 属性的事件对象
     * 2. DomVideoPlayer 组件传递的格式：直接传递数据对象
     * 更新当前播放时间、总时长、当前字幕索引，并处理循环播放
     */
    onVideoTimeUpdate(e) {
      //console.log('onVideoTimeUpdate 事件触发，e:', e);
      // 检查 e 是否是一个对象
      if (e && typeof e === 'object') {
        // 检查 e 是否有 detail 属性（标准事件格式）
        if (e.detail) {
          this.currentTimeSec = e.detail.currentTime || 0;
          this.currentTime = this.formatTime(e.detail.currentTime || 0);
          
          if (e.detail.duration) {
            this.durationSec = e.detail.duration;
            this.duration = this.formatTime(e.detail.duration);
          }
        } else {
          // 直接使用 e 对象（DomVideoPlayer 组件传递的格式）
          this.currentTimeSec = e.currentTime || 0;
          this.currentTime = this.formatTime(e.currentTime || 0);
          
          if (e.duration) {
            this.durationSec = e.duration;
            this.duration = this.formatTime(e.duration);
          }
        }
      }
      
      // 更新当前字幕索引
      this.updateCurrentSubtitle();
      
      // 处理循环播放
      this.handleLoop();
    },
    
    /**
     * 视频加载完成事件处理
     * 处理两种事件格式：
     * 1. 标准事件格式：包含 detail 属性的事件对象
     * 2. DomVideoPlayer 组件传递的格式：直接传递数据对象
     * 更新视频总时长，并标记视频已准备好进行跳转
     */
    onVideoLoaded(e) {
      console.log('--- onVideoLoaded 事件被触发 ---', e);
      if (e && typeof e === 'object') {
        if (e.detail && e.detail.duration) {
          this.durationSec = e.detail.duration;
          this.duration = this.formatTime(e.detail.duration);
          console.log('视频加载成功，时长:', e.detail.duration, 'isVideoReadyForSeek:', this.isVideoReadyForSeek);
        } else if (e.duration) {
          // 直接使用 e 对象（DomVideoPlayer 组件传递的格式）
          this.durationSec = e.duration;
          this.duration = this.formatTime(e.duration);
          console.log('视频加载成功，时长:', e.duration, 'isVideoReadyForSeek:', this.isVideoReadyForSeek);
        }
      }
      this.isVideoReadyForSeek = true;
    },
    
    /**
     * 视频错误事件处理
     */
    onVideoError(e) {
      console.error('视频加载失败:', e);
      uni.showToast({ title: '视频加载失败，请检查文件格式', icon: 'none' });
    },
    
    /**
     * 处理循环播放
     * 在H5环境下使用原生video元素执行循环，在非H5环境下使用视频上下文执行循环
     * 支持两种循环模式：
     * 1. A-B 片段循环：在指定的时间区间内循环播放
     * 2. 整段循环：从视频开始到结束循环播放
     */
    handleLoop() {
      if (this.loopMode) {
        // 检查是否存在document对象（H5环境）
        if (typeof document !== 'undefined') {
          // 获取原生video元素
          const videoElement = document.querySelector('video');
          if (videoElement) {
            // 检查是否设置了循环点
            if (this.loopStart > 0 && this.loopEnd > this.loopStart) {
              // A-B 片段循环
              if (this.currentTimeSec >= this.loopEnd) {
                videoElement.currentTime = this.loopStart;
                videoElement.play().catch(err => {
                  console.error('自动播放失败:', err);
                });
              }
            } else {
              // 整段循环
              if (this.durationSec && this.currentTimeSec >= this.durationSec - 0.1) {
                videoElement.currentTime = 0;
                videoElement.play().catch(err => {
                  console.error('自动播放失败:', err);
                });
              }
            }
          }
        } else {
          // 非H5环境，使用视频上下文
          // 确保视频上下文已初始化
          if (!this.videoContext) {
            this.getVideoContext();
          }
          
          if (this.videoContext) {
            // 检查是否设置了循环点
            if (this.loopStart > 0 && this.loopEnd > this.loopStart) {
              // A-B 片段循环
              if (this.currentTimeSec >= this.loopEnd) {
                this.videoContext.toSeek(this.loopStart, false);
                this.videoContext.play();
              }
            } else {
              // 整段循环
              if (this.durationSec && this.currentTimeSec >= this.durationSec - 0.1) {
                this.videoContext.toSeek(0, false);
                this.videoContext.play();
              }
            }
          }
        }
      }
    }
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
.app-container {
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

/* 上传卡片样式 */
.upload-card {
  background-color: rgba(255, 255, 255, 0.05);
  border: 1rpx solid rgba(255, 255, 255, 0.1);
  border-radius: 12rpx;
  padding: 20rpx;
  margin-bottom: 20rpx;
}

.upload-label {
  display: flex;
  align-items: center;
  margin-bottom: 15rpx;
}

.upload-icon {
  font-size: 28rpx;
  margin-right: 10rpx;
}

.upload-title {
  font-size: 26rpx;
  color: #e2e8f0;
}

/* 上传区域样式 */
.upload-area {
  border: 2rpx dashed rgba(255, 255, 255, 0.2);
  border-radius: 8rpx;
  padding: 40rpx;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
}

.upload-area:hover {
  border-color: #63b3ed;
  background-color: rgba(99, 179, 237, 0.05);
}

.upload-content {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.upload-icon-large {
  font-size: 60rpx;
  margin-bottom: 15rpx;
}

.upload-text {
  font-size: 26rpx;
  color: #e2e8f0;
  margin-bottom: 10rpx;
}

.upload-hint {
  font-size: 22rpx;
  color: #718096;
}

/* 视频区域样式 */
.video-section {
  background-color: rgba(0, 0, 0, 0.3);
  border-radius: 12rpx;
  overflow: hidden;
  margin-bottom: 20rpx;
}

.video-container {
  width: 100%;
  height: 400rpx;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #000;
}

/* 视频空状态样式 */
.video-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #4a5568;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.video-placeholder-icon {
  font-size: 80rpx;
  margin-bottom: 15rpx;
}

.video-placeholder-text {
  font-size: 26rpx;
}

/* 控制区域样式 */
.control-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15rpx 20rpx;
  border-top: 1rpx solid rgba(255, 255, 255, 0.1);
}

.time-display {
  font-size: 24rpx;
  color: #a0aec0;
  font-family: monospace;
}

.control-buttons {
  display: flex;
  gap: 10rpx;
}

.control-btn {
  display: flex;
  align-items: center;
  gap: 5rpx;
  padding: 10rpx 20rpx;
  background-color: rgba(255, 255, 255, 0.1);
  border: none;
  border-radius: 6rpx;
  color: #e2e8f0;
  font-size: 22rpx;
}

.control-btn:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.btn-icon {
  font-size: 24rpx;
}

/* A-B循环区域样式 */
.ab-loop-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15rpx 20rpx;
  border-top: 1rpx solid rgba(255, 255, 255, 0.1);
  background-color: rgba(255, 255, 255, 0.02);
}

.ab-status {
  font-size: 22rpx;
  color: #718096;
}

.ab-buttons {
  display: flex;
  gap: 10rpx;
}

.ab-btn {
  padding: 8rpx 16rpx;
  background-color: rgba(255, 255, 255, 0.1);
  border: 1rpx solid rgba(255, 255, 255, 0.2);
  border-radius: 4rpx;
  color: #e2e8f0;
  font-size: 20rpx;
}

.ab-btn.active {
  background-color: #63b3ed;
  border-color: #63b3ed;
}

.clear-btn {
  padding: 8rpx 12rpx;
}

/* 字幕区域样式 */
.subtitle-section {
  background-color: rgba(255, 255, 255, 0.05);
  border: 1rpx solid rgba(255, 255, 255, 0.1);
  border-radius: 12rpx;
  overflow: hidden;
}

.subtitle-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20rpx;
  border-bottom: 1rpx solid rgba(255, 255, 255, 0.1);
}

.subtitle-title {
  font-size: 28rpx;
  font-weight: bold;
  color: #e2e8f0;
}

.subtitle-count {
  font-size: 22rpx;
  color: #718096;
  background-color: rgba(255, 255, 255, 0.1);
  padding: 4rpx 12rpx;
  border-radius: 4rpx;
}

.subtitle-list {
  max-height: 500rpx;
  overflow-y: auto;
  padding: 10rpx;
}

/* 空字幕提示样式 */
.empty-subtitle {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 80rpx 0;
  color: #4a5568;
}

.empty-icon {
  font-size: 60rpx;
  margin-bottom: 20rpx;
}

.empty-text {
  font-size: 26rpx;
  margin-bottom: 10rpx;
}

.empty-hint {
  font-size: 22rpx;
}

/* 字幕项样式 */
.subtitle-items {
  display: flex;
  flex-direction: column;
}

.subtitle-item {
  display: flex;
  padding: 15rpx;
  border-radius: 8rpx;
  cursor: pointer;
  transition: all 0.2s ease;
  margin-bottom: 5rpx;
  min-height: 60rpx;
  box-sizing: border-box;
}

.subtitle-item:hover {
  background-color: rgba(255, 255, 255, 0.05);
}

.subtitle-item.active {
  background-color: rgba(99, 179, 237, 0.2);
  border-left: 4rpx solid #63b3ed;
}

.subtitle-time {
  font-size: 22rpx;
  color: #63b3ed;
  font-weight: bold;
  margin-right: 20rpx;
  min-width: 120rpx;
  font-family: monospace;
}

.subtitle-text {
  font-size: 24rpx;
  line-height: 1.5;
  color: #e2e8f0;
  flex: 1;
}

/* 滚动条样式 */
::-webkit-scrollbar {
  width: 6rpx;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3rpx;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.3);
}

/* 锚点元素：用于 scroll-into-view 定位 */
.subtitle-anchor {
  position: absolute;
  top: -120rpx; /* 向上偏移，不占用实际空间 */
  height: 120rpx; /* 约为容器高度的一半，使字幕项居中显示 */
  width: 100%;
  pointer-events: none; /* 不影响点击事件 */
  opacity: 0; /* 隐藏锚点 */
}

/* 字幕项容器：用于定位锚点 */
.subtitle-items > view {
  position: relative;
}
</style>