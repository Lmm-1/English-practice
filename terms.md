
## 专业术语表
### 通用术语
- 编解码器 (Codec)
- 比特率 (Bitrate)
- 转码 (Transcoding)
- 率失真 (Rate-Distortion, R-D)

### 图像压缩
- 有损压缩 (Lossy Compression)
- 无损压缩 (Lossless Compression)
- JPEG/PNG (图像格式)

### 视频压缩
- H.264/HEVC (视频编码标准)
- 量化参数 (Quantization Parameter, QP)
- 图像组 (Group of Pictures, GOP)
- 恒定速率因子 (Constant Rate Factor, CRF)
- libx264/libx265 (视频编码器)

### FFmpeg相关
- 预设系统 (Preset System)
- 多媒体框架 (Multimedia Framework)

### 元数据
- 元数据提取（Metadata Extraction）
- JSON 对象（JSON Objects）
- 检索能力（Querying Capabilities）

### 批量处理
- 并发机制（Concurrency Mechanisms）
- 并行处理（Parallel Processing）
- 工作流效率（Workflow Efficiency）

## 1. 系统架构组件
| 英文术语                  | 中文术语               | 原文中的句子示例                                                                 |
|---------------------------|------------------------|----------------------------------------------------------------------------------|
| `mmsEngineDBFacade`       | 多媒体服务引擎数据库门面 | `Validator validator(_mmsEngineDBFacade, _configurationRoot);`<br>`_mmsEngineDBFacade->modifyYouTubeConf(...);` |
| `MMSDeliveryAuthorization` | 内容分发授权管理器      | `shared_ptr<MMSDeliveryAuthorization> mmsDeliveryAuthorization`                  |
| `EncodersLoadBalancer`    | 编码器负载均衡器        | `_encodersLoadBalancer = make_shared<EncodersLoadBalancer>(_mmsEngineDBFacade, _configuration);` |

## 2. 任务调度与管理
| 英文术语                     | 中文术语         | 原文中的句子示例                                                                 |
|------------------------------|------------------|----------------------------------------------------------------------------------|
| `GroupOfTasks`              | 任务组容器       | `"type": "GroupOfTasks"`（JSON配置）                                            |
| `executionType: "parallel"` | 并行执行模式     | `"executionType": "parallel"`（JSON配置）                                       |
| `encodingPriority: "High"`  | 任务队列优先级   | `"encodingPriority": "High"`（JSON配置）                                        |
| `ulPeriodInMilliSecs`       | 定时任务周期     | `CheckIngestionTimes(unsigned long ulPeriodInMilliSecs, ...)`                   |

## 3. 视频/图像处理
| 英文术语                             | 中文术语           | 原文中的句子示例                                                                 |
|--------------------------------------|--------------------|----------------------------------------------------------------------------------|
| `cascadeName`                       | 人脸检测模型       | `"cascadeName": "haarcascade_frontalface_alt_tree"`                             |
| `initialFramesNumberToBeSkipped`    | 初始帧跳过数       | `"initialFramesNumberToBeSkipped": "${initialFramesNumberToBeSkipped}"`          |
| `processingStartingFrom`            | 任务执行起始时间   | `"processingStartingFrom": "${processingStartingFrom}"`（多次出现）              |

## 4. 资源与队列
| 英文术语                     | 中文术语           | 原文中的句子示例                                                                 |
|------------------------------|--------------------|----------------------------------------------------------------------------------|
| `lowPriorityEncodingJobs`    | 低优先级编码队列   | `for (EncodingJob &encodingJob : _lowPriorityEncodingJobs)`                      |
| `faceRecognitionNumber`      | 人脸识别计数器     | `shared_ptr<long> faceRecognitionNumber = make_shared<long>(0);`                 |

## 5. 存储与生命周期
| 英文术语              | 中文术语           | 原文中的句子示例                                                                 |
|-----------------------|--------------------|----------------------------------------------------------------------------------|
| `expirationInDays`   | 内容过期天数       | `"expirationInDays": 30`（配置默认值）                                          |
| `noFileSystemAccess` | 文件系统访问开关   | `bool noFileSystemAccess`（构造函数参数）                                       |

## 6. 网络与IO控制
| 英文术语                  | 中文术语           | 原文中的句子示例                                                                 |
|---------------------------|--------------------|----------------------------------------------------------------------------------|
| `fcgiAcceptMutex`        | FastCGI互斥锁      | `mutex *fcgiAcceptMutex`（构造函数参数）                                        |
| `FileUploadProgressData` | 文件上传跟踪器     | `FileUploadProgressData *fileUploadProgressData`（构造函数参数）                |

## 7. 线程安全机制
| 英文术语          | 中文术语       | 原文中的句子示例                                                                 |
|-------------------|----------------|----------------------------------------------------------------------------------|
| `_mtTimesMutex` | 定时任务互斥锁 | `lock_guard<mutex> locker(_mtTimesMutex);`                                      |




<!--by 罗娜-->

