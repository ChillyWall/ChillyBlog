---
title: TensorRT部署YOLO V8模型
date: 2026-04-10
tags:
  - TensorRT
  - YOLO
  - OpenCV
categories:
  - [AI, Deployment]
index_img: /img/categories/TensorRT.webp
---

本文介绍我使用TensorRT部署Yolo模型并实现读取摄像头实时目标检测的过程。

<!-- more -->

本文源码仓库在 [这里](https://github.com/ChillyWall/TensorRT-Deployment-Practices.git)。

## 基于 TensorRT-YOLO 部署

### 安装 TensorRT-YOLO

使用开源框架 [TensorRT-YOLO](https://github.com/laugh12321/TensorRT-YOLO) 进行模型的部署。将源码下载下来，正确配置 `TensorRT` 的 安装路径之后可以直接编译成功。

不过这个项目的的导出写得不是很好，明明安装目标时有 `EXPORT` 却没有安装，最后是自己手动设置的变量。我把缺失的安装部分给它加上了，之后可以正常 `find_package` 然后链接一下库就完成所有配置了。

### 准备 YOLO 模型

使用 Python 安装 `ultralytics` 之后直接导出 `yolov8n` 模型到 ONNX 格式。之后使用上面 `TensorRT-YOLO` 项目配套的 [trtyolo-export](https://github.com/laugh12321/trtyolo-export) 工具可以将 ONNX 直接编译成 engine 格式。

同时会在Yolo模型的后面增加一个Efficient NMS插件的处理层，在GPU端进行低置信框的过滤和NMS处理。

### 使用 OpenCV 读取摄像头

这里使用的摄像头之前已经标定过了，根据参数从摄像头读取图像数据后进行畸变校正后传出。

参数保存在 yml 文件中，通过 `cv::FileStorage` 读取并解析。

读取并校正代码如下：

```cpp
cv::Mat YoloCamera::read_frame() {
    cv::Mat frame;
    cap >> frame;

    if (frame.empty()) {
        throw FrameCaptureException();
    }

    cv::Mat undist_frame;
    cv::undistort(frame, undist_frame, calib_res.camera_matrix,
                  calib_res.dist_coeffs);

    return undist_frame;
}
```

### YOLO 实时目标监测

将 `TensorRT-YOLO` 提供的推理参数选项保存在 yml 文件中，创建模型时读取文件再设置。

```cpp
void YoloCamera::read_infer_option(const std::string& option_file) {
    cv::FileStorage fs(option_file, cv::FileStorage::READ);
    if (!fs.isOpened()) {
        throw FileNotFoundException(CONFIG_FILE);
    }
    cv::FileNode node;
    if ((node = fs["device_id"]).isInt()) {
        infer_option.setDeviceId((int) node);
    }
    if ((node = fs["cuda_memory"]).isInt()) {
        if ((int) node)
            infer_option.enableCudaMem();
    }
    if ((node = fs["managed_memory"]).isInt()) {
        if ((int) node)
            infer_option.enableManagedMemory();
    }
    if ((node = fs["enable_swap_rb"]).isInt()) {
        if ((int) node)
            infer_option.enableSwapRB();
    }
    if ((node = fs["enable_performance_report"]).isInt()) {
        if ((int) node)
            infer_option.enablePerformanceReport();
    }
    if ((node = fs["input_dimensions"]).isSeq()) {
        std::vector<int> dims;
        node >> dims;
        if (dims.size() == 2) {
            infer_option.setInputDimensions(dims[0], dims[1]);
        }
    }
}
```

预训练的 yolov8 模型使用的是 coco 数据集，搜索其类型名称保存在 yml 文件中，用于后续可视化时的标注。

将 OpenCV 校正过的图像用 TensorRT-YOLO 提供的图片类封装一下，直接进行推理。

推理完成后根据结果使用 OpenCV 绘制简单的框并加上 label，再显示出来。

可视化代码如下：

```cpp
cv::Mat YoloCamera::visualize(const cv::Mat& frame,
                              const trtyolo::DetectRes& res) const {
    cv::Mat image = frame.clone();
    for (size_t i = 0; i < res.num; ++i) {
        const auto& box = res.boxes[i];
        int cls = res.classes[i];
        float score = res.scores[i];
        const auto& label = labels[cls];
        std::string label_text = label + " " + cv::format("%.3f", score);

        // 绘制矩形和标签
        int base_line;
        cv::Size label_size = cv::getTextSize(
            label_text, cv::FONT_HERSHEY_SIMPLEX, 0.6, 1, &base_line);
        cv::rectangle(image, cv::Point(box.left, box.top),
                      cv::Point(box.right, box.bottom),
                      cv::Scalar(251, 81, 163), 2, cv::LINE_AA);
        cv::rectangle(image, cv::Point(box.left, box.top - label_size.height),
                      cv::Point(box.left + label_size.width, box.top),
                      cv::Scalar(125, 40, 81), -1);
        cv::putText(image, label_text, cv::Point(box.left, box.top),
                    cv::FONT_HERSHEY_SIMPLEX, 0.6, cv::Scalar(253, 168, 208),
                    1);
    }
    return image;
}
```

## 手写 C++代码进行部署

基于之前写的 Cifar100CNN 的类修改，去掉动态批大小之后只需要关注预处理和后处理。

介于要适配两种模型，一种支持 TensorRT 的官方插件 Efficient NMS，所以要根据模型类型 来修改分配显存和输入输出绑定。加入了成员变量 `enable_efficient_nms` 来标记模型类型。

和使用 TensorRT-YOLO 的版本一样，也加入了读取 labels 的功能。

```cpp
class YoloV8n {
private:
    std::string onnx_path;
    std::string engine_path;

    TRTPtr<nvinfer1::ICudaEngine> engine;
    TRTPtr<TRTInference> inference;

    // 模型是否启用了EfficientNMS插件
    bool enable_efficient_nms;

    float* input_buffer;
    float* output_buffer;

    size_t input_size;
    size_t output_size;

    void* gpu_input;
    void* gpu_output;

    std::vector<std::string> labels;
    void read_labels(const std::string& file_path);

    void set_tensor_addresses();

    std::vector<YoloDetectResult> decode_output();

    void apply_nms(std::vector<YoloDetectResult>& results,
                   float iou_threshold = 0.5f);

    void apply_deletterbox(std::vector<YoloDetectResult>& results);

    std::vector<YoloDetectResult> decode_output_nms();

    void preprocess(const cv::Mat& input);
    std::vector<YoloDetectResult> postprocess();
    void infer();
    auto InputData();
    auto OutputData();

public:
    YoloV8n(std::string onnx_path, std::string engine_path, TRTLogger& logger,
            bool enable_efficient_nms_plugin, bool always_rebuild = false);
    ~YoloV8n() noexcept;
    std::vector<YoloDetectResult> infer(const cv::Mat& input);
    cv::Mat visualize(const cv::Mat& input,
                      const std::vector<YoloDetectResult>& results);
};
```

### Yolo 预处理

目前的摄像头分辨率是 1280x720，而 YoloV8 的最佳分辨率一般为 640x640，需要进行变换， 使用 `letterbox` 方法进行。即先进行缩放，比如 1280x720 缩放为 640x360，空白部分补 上黑边。

```cpp
template <TensorSpecType InputSpec, TensorSpecType OutputSpec>
struct LetterBox {
    static cv::Mat process(const cv::Mat& input) {
        constexpr int input_width = InputSpec::dims()[0];
        constexpr int input_height = InputSpec::dims()[1];
        constexpr int output_width = OutputSpec::dims()[0];
        constexpr int output_height = OutputSpec::dims()[1];

        constexpr float scale =
            std::min(static_cast<float>(output_width) / input_width,
                     static_cast<float>(output_height) / input_height);

        constexpr int resized_width = static_cast<int>(input_width * scale);
        constexpr int resized_height = static_cast<int>(input_height * scale);

        constexpr int x_offset = (output_width - resized_width) / 2;
        constexpr int y_offset = (output_height - resized_height) / 2;

        cv::Mat resized;
        cv::resize(input, resized, cv::Size(resized_width, resized_height));

        cv::Mat output =
            cv::Mat::zeros(output_height, output_width, input.type());

        cv::Rect roi(x_offset, y_offset, resized_width, resized_height);
        resized.copyTo(output(roi));

        return output;
    }
};
```

### Yolo 后处理

Yolo 的后处理相对要复杂一些。

原始的 Yolo 模型的输出结果为 `(1, 84, 8400)`，及总共 8400 个框，每个框 84 个数据， 其中 0～1 为中心点坐标，2～3 为宽和高，4～83 为各个类别的置信率。

Yolo 的后处理主要包括三个步骤，首先是从 80 个类型置信率中得到最大的值作为检测结 果，其置信率作为该框的置信率，并过滤到低置信率的框，一般阈值为 0.25；之后是过滤 掉重复框，因为 Yolo 检测中一个物体可能有很多个重复框，需要使用 NMS 算法进行去重； 之后还需要将框转换回原图片。

#### 解析推理结果并过滤

解析的部分比较简单，唯一需要注意的就是结果的形状为 (84, 8400) 而不是 (8400, 84)， 也就是说在内存中是先 8400 个框的第一个元素，然后 8400 个框的第二个元素这样排列的。也 就是说一个框的 84 个元素之间都隔着 8399 个元素。

这种跳跃式读取实际上是比较慢的，因为局部性较差，CPU 缓存不命中。一种方法是转置， 但是转置其实也很慢，因为需要新分配一块内存再复制一遍。之后还需要再进行过滤，所以 虽然写法上看似简洁但性能较差。

所以最终还是选择了跳跃式读取的同时进行过滤，仅将合格的部分记录下来统一读取框的位 置，再向外输出。

```cpp
std::vector<YoloDetectResult> YoloV8n::decode_output() {
    std::vector<YoloDetectResult> results;
    // 预留50个结果的空间，避免频繁扩容
    results.reserve(50);
    constexpr int COLS = OutputRes::dims()[1];
    constexpr int ROWS = OutputRes::dims()[0];

    // 过滤低置信度的检测结果，并提取边界框和类别信息
    // 采用跳跃式访问，而非先转置输出矩阵，避免不必要的内存复制
    // 使用OpenMP进行并行处理，提升性能
#pragma omp parallel
    {
        // 每个线程创建自己的局部 vector
        std::vector<YoloDetectResult> local_results;
        local_results.reserve(10);  // 预留少量空间

#pragma omp for nowait  // nowait 减少同步开销
        for (size_t i = 0; i < COLS; ++i) {
            float max_score = 0;
            int class_id = -1;
            for (size_t j = 4; j < ROWS; ++j) {
                float s = output_buffer[j * COLS + i];
                if (s > max_score) {
                    max_score = s;
                    class_id = j - 4;
                }
            }
            if (max_score > 0.25f) {
                float x_center = output_buffer[0 * COLS + i];
                float y_center = output_buffer[1 * COLS + i];
                float width = output_buffer[2 * COLS + i];
                float height = output_buffer[3 * COLS + i];
                cv::Rect box(x_center - width / 2, y_center - height / 2, width,
                             height);
                local_results.push_back({class_id, max_score, box});
            }
        }

// 关键区域：将局部结果安全地合并到主结果
#pragma omp critical
        {
            results.insert(results.end(), local_results.begin(),
                           local_results.end());
        }
    }

    return results;
}
```

后来实现了批次处理，因为跳跃访问的主要问题是缓存不明中，CPU 读取连续内存的能力是 最强的。所以后面尝试进行优化，将框再进行分批处理，一次同时处理 16 个框，读取时每一 次是读取连续的 16 个元素，再统一进行比较，最后同意过滤。

但是这个写法也有问题，一次读取 16 个元素，所以加入了局部数组作为存储，又加了一层循 环，代码相对更复杂一些。最终经过测试，在开启多线程优化的前提下两个版本速度相差不 多，但是当线程数为 2 和 16 时，原版性能都显著高于优化后的版本。可能是 84\*8400 这个数 据量仍然在 CPU 的预取承受范围内，所以 CPU 自动进行的优化便已经很快了，而我家的优化反 而因为多了一层循环或其他原因 CPU 更难进行优化所以更慢。

#### NMS 算法实现

NMS 算法本身不算复杂，这里实现为先进行排序，然后从前至后逐个检查 IoU 即重叠率， 重叠率高于一定阈值并且是相同类别，则将删除。为优化性能，使用懒删除，为方便将置信 率作为标记，被标记要删除的框则将其置信率设为负值，最后通过 erase<sub>if</sub> 统一删除。

```cpp
static inline float calculate_iou(const cv::Rect& a, const cv::Rect& b) {
    float inter_area = (a & b).area();
    if (inter_area <= 0)
        return 0.f;
    float res =
        inter_area / static_cast<float>(a.area() + b.area() - inter_area);
    return res;
}

void YoloV8n::apply_nms(std::vector<YoloDetectResult>& candidates,
                        float iou_threshold) {
    if (candidates.empty())
        return;

    // 按置信率排序
    std::sort(candidates.begin(), candidates.end(),
              [](const YoloDetectResult& a, const YoloDetectResult& b) -> bool {
                  return a.confidence > b.confidence;
              });

    // 将要被删除的框的置信率设为负值，避免额外的 bool 标记空间
    for (size_t i = 0; i < candidates.size(); ++i) {
        if (candidates[i].confidence < 0)
            continue;  // 已经被标记删除

        for (size_t j = i + 1; j < candidates.size(); ++j) {
            if (candidates[j].confidence < 0)
                continue;

            if (candidates[i].class_id == candidates[j].class_id) {
                if (calculate_iou(candidates[i].box, candidates[j].box) >
                    iou_threshold) {
                    // 利用置信率字段作为标记位，负值表示被删除
                    candidates[j].confidence = -1.0f;
                }
            }
        }
    }

    // 删除被标记的框，即置信率被设为负值的项
    std::erase_if(candidates, [](const auto& d) { return d.confidence < 0; });
}
```

#### 启用 EfficientNMS 插件后的结果解析

当启用了 TensorRT 官方的 Efficient NMS 插件之后，过滤低置信框和 NMS 操作可以在 GPU 直接完成，输出结果变为四个 tensor，`num_dets int32(1, 1)`，`det_boxes float32(1, 100, 4)`，`det_classes int32(1, 100)`，`det_scores float32(1, 100)`，分别 是检测到的目标，目标的框，目标的类别和目标的置信率。

设计了一个类用于存储这四个数据，便于管理和表示，至于解析便很简单了，只需要将指向 结果缓存区的指针转为该类型，便可直接读取数据，然后直接输出即可。

```cpp
struct YoloDetectResultNMS {
    int32_t num_dets;
    float det_boxes[100][4];
    float det_scores[100];
    int32_t det_classes[100];
};

std::vector<YoloDetectResult> YoloV8n::decode_output_nms() {
    auto output = reinterpret_cast<YoloDetectResultNMS*>(output_buffer);
    static_assert(sizeof(YoloDetectResultNMS) == (1 + 100 * 4 + 100 + 100) * 4);

    std::vector<YoloDetectResult> results;
    results.reserve(output->num_dets);
    for (int i = 0; i < output->num_dets; ++i) {
        int class_id = static_cast<int>(output->det_classes[i]);
        float confidence = output->det_scores[i];
        float x1 = output->det_boxes[i][0];
        float y1 = output->det_boxes[i][1];
        float x2 = output->det_boxes[i][2];
        float y2 = output->det_boxes[i][3];
        cv::Rect box(x1, y1, x2 - x1, y2 - y1);
        results.push_back({class_id, confidence, box});
    }

    return results;
}
```

#### DeLetterBox 实现

上面提到，由于预处理时将图片胫骨 LetterBox 算法进行了缩放，传回的结果中的坐标也是 按照缩放后的坐标系。要将框映射到原图像中，需要进行 DeLetterBox，算法很简单，只需 要简单的线性代数相关知识就很容易理解这就是一个仿射变换，当然哪怕没有也很容易理解。

```cpp
template <TensorSpecType InputSpec, TensorSpecType OutputSpec>
struct DeLetterBox {
    static cv::Rect process(const cv::Rect& input) {
        constexpr int input_width = InputSpec::dims()[0];
        constexpr int input_height = InputSpec::dims()[1];
        constexpr int output_width = OutputSpec::dims()[0];
        constexpr int output_height = OutputSpec::dims()[1];

        constexpr float scale =
            std::min(static_cast<float>(output_width) / input_width,
                     static_cast<float>(output_height) / input_height);

        constexpr int resized_width = static_cast<int>(input_width * scale);
        constexpr int resized_height = static_cast<int>(input_height * scale);

        constexpr int x_offset = (output_width - resized_width) / 2;
        constexpr int y_offset = (output_height - resized_height) / 2;

        return cv::Rect((input.x - x_offset) / scale,
                        (input.y - y_offset) / scale, input.width / scale,
                        input.height / scale);
    }
};
```

#### 最终的后处理

根据是否启用了 Efficient NMS 插件选择不同的解析函数，并进行 NMS。最后同意 DeLetterBox 后返回结果。

```cpp
std::vector<YoloDetectResult> YoloV8n::postprocess() {
    std::vector<YoloDetectResult> results;
    if (enable_efficient_nms) {
        results = decode_output_nms();
    } else {
        results = decode_output();
        apply_nms(results, 0.45f);
    }
    apply_deletterbox(results);
    return results;
}
```

### 绑定输入输出

由于显存如何分配是根据模型的结构决定的，和绑定输入输出耦合度高，因此将显存分配和 绑定放到一起。

输入没有区别都是 `(1, 3, 640, 640)` 的图片，tensor 名称为 `images`，输出要根据模型类 型进行绑定。

原始的 Yolo 输出格式为 `(1, 84, 8400)`，tensor 名称为 `output0`。

而使用了 Efficient NMS 的则变为四个 tensor 输出，上面已经介绍过。为了方便分配以 及内存拷贝，分配显存时相当于直接分配一个 `YoloDetectResultNMS` 对象，使用取地址符 而不是手动偏移保证地址计算没有错误，然后绑定即可。

```cpp
void YoloV8n::set_tensor_addresses() {
    // 分配显存和内存，并绑定输入缓冲区地址
    input_size = sizeof(float) * InputImg::total_size();
    cudaHostAlloc((void**) &input_buffer, input_size, cudaHostAllocDefault);
    cudaMalloc(&gpu_input, input_size);
    inference->set_tensor_address("images", gpu_input);

    // 根据是否启用EfficientNMS插件，分配不同的输出缓冲区，并绑定输出地址
    if (enable_efficient_nms) {
        output_size = sizeof(YoloDetectResultNMS);
    } else {
        output_size = sizeof(float) * OutputRes::total_size();
    }

    cudaHostAlloc((void**) &output_buffer, output_size, cudaHostAllocDefault);
    cudaMalloc(&gpu_output, output_size);

    if (enable_efficient_nms) {
        inference->set_tensor_address(
            "num_dets",
            &(reinterpret_cast<YoloDetectResultNMS*>(gpu_output)->num_dets));
        inference->set_tensor_address(
            "det_boxes",
            &(reinterpret_cast<YoloDetectResultNMS*>(gpu_output)->det_boxes));
        inference->set_tensor_address(
            "det_scores",
            &(reinterpret_cast<YoloDetectResultNMS*>(gpu_output)->det_scores));
        inference->set_tensor_address(
            "det_classes",
            &(reinterpret_cast<YoloDetectResultNMS*>(gpu_output)->det_classes));
    } else {
        inference->set_tensor_address("output0", gpu_output);
    }
}
```

### 构造函数输入输出推理

使用 `output_size` 和 `input_size` 两个成员变量存储输入输出的数据大小，避免每次都手 动计算，传输数据的部分直接这两个变量来指定传输的数据量。

```cpp
void YoloV8n::infer() {
    inference->infer();
}

auto YoloV8n::InputData() {
    return cudaMemcpyAsync(gpu_input, input_buffer, input_size,
                           cudaMemcpyHostToDevice, inference->get_stream());
}

auto YoloV8n::OutputData() {
    return cudaMemcpyAsync(output_buffer, gpu_output, output_size,
                           cudaMemcpyDeviceToHost, inference->get_stream());
}

YoloV8n::YoloV8n(std::string onnx_path, std::string engine_path,
                 TRTLogger& logger, bool enable_efficient_nms_plugin,
                 bool always_rebuild)
    : onnx_path(onnx_path),
      engine_path(engine_path),
      enable_efficient_nms(enable_efficient_nms_plugin) {
    auto builder = TRTModelBuilder(logger);
    if (always_rebuild || !(engine = builder.loadFromPlan(engine_path))) {
        engine = builder.buildFromOnnx(
            onnx_path, engine_path,
            [](nvinfer1::IBuilderConfig* config,
               nvinfer1::INetworkDefinition* network,
               nvinfer1::IBuilder* builder) {
                // 2. 精度设置：虽然 kFP16 弃用，但在 10.0 中作为 BuilderFlag
                // 依然是生效的（会有警告）
                if (builder->platformHasFastFp16()) {
                    config->setFlag(nvinfer1::BuilderFlag::kFP16);
                }
            });
    }
    inference = TRTPtr<TRTInference>(new TRTInference(*engine));

    set_tensor_addresses();

    read_labels(std::string(PACKAGE_ROOT_DIR) + "/config/labels.yml");

    std::cout << std::format(
                     "Model initialized:\nONNX: {}\nEngine: {}\nEfficient NMS "
                     "Plugin: {}\nAlways rebuild: {}",
                     onnx_path, engine_path,
                     enable_efficient_nms ? "Enabled" : "Disabled",
                     always_rebuild ? "True" : "False")
              << std::endl;
}
```
