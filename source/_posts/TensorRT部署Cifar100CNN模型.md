---
title: TensorRT部署Cifar100CNN模型
date: 2026-04-10
tags:
  - TensorRT
categories:
  - [AI, Deployment]
index_img: /img/categories/TensorRT.webp
---

本文介绍如何使用TensorRT部署一个简单的用于Cifar100分类任务的CNN模型。

<!-- more -->

本文项目源码在 [这里](https://github.com/ChillyWall/TensorRT-Deployment-Practices.git)，所使用的模型为我自己训练的小型CNN模型，导出的ONNX格式模型和训练代码在 [这里](https://github.com/ChillyWall/CIFAR100-CNN-Classification-Practice.git)。

## PyTorch 导出 ONNX 格式

PyTorch 支持将模型导出到 ONNX 格式，便于后面使用 TensorRT 进行加速。支持动态 batch size，方便批处理。方法如下：

```python
def export_onnx(model, checkpoint):
    model = CIFAR100_VGG()
    load_model_params(model, checkpoint)
    model.eval()

    dummy_input = torch.randn(1, 3, 32, 32)
    torch.onnx.export(
        model,
        dummy_input,
        "cifar100_vgg.onnx",
        verbose=True,
        input_names=["input"],  # 输入节点名称
        output_names=["output"],  # 输出节点名称
        opset_version=13,  # 算子集版本
        dynamic_axes={
            "input": {0: "batch_size"},  # 第0维 = 动态 batch
            "output": {0: "batch_size"},  # 输出第0维也跟着动态
        },
    )


if __name__ == "__main__":
    model = CIFAR100_VGG()
    export_onnx(model, "./checkpoints/model_best.pth")
```

## TensorRT 编译 ONNX 模型

之后是使用 TensorRT 编译导出的 ONNX 模型。TensorRT 编译主要有三种方法，分别是通 过 trtexec 命令和通过 TensorRT 的 Python 和 C++的 API。

trtexec 命令最为简单快速，适应简单模型。而 Python API 相对要更方便一些，支持更多细致的配置。但是如果有自定义插件等更复杂的要求，还是需要 C++ API，其提供最多的特性和最高的性能。

这里进行的是简单的 CNN 模型，于是直接使用 trtexec 命令进行编译。命令大致如下：

```shell
trtexec --onnx=cifar100_vgg.onnx --saveEngine=cifar100_vgg.engine --fp16
```

在训练的过程中往往使用 32 位浮点型作为计算的数据结构，而在工程实践中，则常常使用 16 位，这在精度影响不大的情况下获得更高的速度。这便是量化。

量化的本质是：用更低比特（更少位数）的数据类型来近似表示高精度的浮点数据。除了 fp32 和 fp16，有时为了极致速度会使用 int8，这还会牵扯到“校准”的过程，精度也容易 出现明显下降。

这里我们使用常用的 fp16 来加速。

### 模型性能简单评估

可以使用 trtexec 提供的功能来简单评估模型的性能。

```shell
trtexec --loadEngine=cifar100_vgg.engine
```

输出的信息中有延迟和吞吐量等信息。吞吐量表示每秒可以处理多少个请求(queries per second/qps)。

## 使用 TensorRT C++ API 进行模型构建

目前计划写一个简单的 TensorRT C++ API 的封装工具，用于部署简单的模型，封装构建， 推理，预处理等过程，便于之后的学习。

目前已完成构建部分的代码，目前该模块实现了从 .engine 文件中直接加载模型和先将模 型从 onnx 格式转化再加载的功能。同时对 Nvidia 相关库中的指针进行了简单的 RAII 封 装，使用 unique<sub>ptr</sub> 保证保证资源被正确释放并自定了删除器。

后续会尝试通过 C++ API 进行推理，并使用 OpenCV 做基础的预处理。这些都完善之后就 是想办法将其集成到 ROS 中，在 Gazebo 仿真环境下进行基础的视觉任务。

### 使用 TensorRT C++ API 构建模型

封装了编译模型和加载模型的接口。在编译模型后将其保存为 `.engine` 文件，编译模型函数接 受一个回调函数，用于配置构建器等。也可以从 `.engine` 文件加载模型。

```cpp
// ModelBuilder.h
using TRTBuildConfigFun =
    std::function<void(nvinfer1::IBuilderConfig*, nvinfer1::INetworkDefinition*,
                       nvinfer1::IBuilder*)>;

class TRTModelBuilder {
public:
    TRTModelBuilder(nvinfer1::ILogger& logger) : m_logger(logger) {}

    // 从本地 .engine (Plan) 文件加载
    TRTPtr<nvinfer1::ICudaEngine> loadFromPlan(const std::string& enginePath);

    // 从 ONNX 编译并保存为 .engine
    TRTPtr<nvinfer1::ICudaEngine>
    buildFromOnnx(const std::string& onnxPath, const std::string& enginePath,
                  TRTBuildConfigFun configFun = nullptr);

private:
    nvinfer1::ILogger& m_logger;
};

// ModelBuilder.cpp
TRTPtr<nvinfer1::ICudaEngine>
TRTModelBuilder::loadFromPlan(const std::string& enginePath) {
    std::ifstream file(enginePath, std::ios::binary);
    if (!file.good())
        return nullptr;

    file.seekg(0, std::ios::end);
    size_t size = file.tellg();
    file.seekg(0, std::ios::beg);

    std::vector<char> data(size);
    file.read(data.data(), size);

    auto runtime =
        TRTPtr<nvinfer1::IRuntime>(nvinfer1::createInferRuntime(m_logger));
    return TRTPtr<nvinfer1::ICudaEngine>(
        runtime->deserializeCudaEngine(data.data(), size));
}

TRTPtr<nvinfer1::ICudaEngine>
TRTModelBuilder::buildFromOnnx(const std::string& onnxPath,
                               const std::string& enginePath,
                               TRTBuildConfigFun configFun) {
    auto builder =
        TRTPtr<nvinfer1::IBuilder>(nvinfer1::createInferBuilder(m_logger));
    auto network =
        TRTPtr<nvinfer1::INetworkDefinition>(builder->createNetworkV2(0U));
    auto config =
        TRTPtr<nvinfer1::IBuilderConfig>(builder->createBuilderConfig());
    auto parser = TRTPtr<nvonnxparser::IParser>(
        nvonnxparser::createParser(*network, m_logger));

    // 如果解析失败，说明 onnx 模型有问题
    if (!parser->parseFromFile(
            onnxPath.c_str(),
            static_cast<int>(nvinfer1::ILogger::Severity::kWARNING))) {
        return nullptr;
    }

    if (configFun) {
        // 调用配置函数
        configFun(config.get(), network.get(), builder.get());
    }

    // 编译模型
    auto plan = TRTPtr<nvinfer1::IHostMemory>(
        builder->buildSerializedNetwork(*network, *config));
    if (!plan)
        return nullptr;

    // 将编译好的 Engine 保存到磁盘，下次直接 load
    std::ofstream outfile(enginePath, std::ios::binary);
    outfile.write(reinterpret_cast<const char*>(plan->data()), plan->size());

    auto runtime =
        TRTPtr<nvinfer1::IRuntime>(nvinfer1::createInferRuntime(m_logger));
    return TRTPtr<nvinfer1::ICudaEngine>(
        runtime->deserializeCudaEngine(plan->data(), plan->size()));
}
```

### 使用 OpenCV 进行图片预处理

首先要获得用于推理的图片，当前是用 PyTorch 加载数据集之后使用 OpenCV 直接导出其 原始图片，并记录了元数据，记录了每张图片的类型 id 和名称，可以用于后续的推理验证。

要进行推理，需要将图片读取并完成预处理，包括转化成浮点数和归一化，以及将排列顺序 调整成 TensorRT 使用的顺序。

一般的图片使用 RGB 的格式，而 OpenCV 则使用 BGR 来存储图片，这使得要将 OpenCV 中 的图片喂给 TensorRT，必须先将 BGR 转为 RGB。

除此之外，OpenCV 默认使用 HWC 的顺序排列数据，即按照 RGBRGBRGB的形式排列，而 在 TensorRT 中，为了提升访存效率，使用 CHW 布局进行排列，即 RRRGGGBBB 的形 式。必须重新排布数据才能进行推理。

之后就是 PyTorch 中训练进行了张量化和归一化，因此要进行推理，我们也需要将原本的 uint8 类型转为 float，再归一化。

目前已实现了一个对 Cifar100 数据集进行预处理的类，并进行了一些性能优化上的尝试。

目前的实现中大量使用了模板元编程，使用模板参数来传递各种静态信息增强通用性，比如 通过可变模板参数实现编译期的数组等。

对于类似输入数据的维度，均值和方差等常量，以及不同颜色空间的映射声明模板类和对应 的 concept，声明 constexpr 成员函数和变量来传递。

将原本通过 opencv 完成的颜色空间重映射改为在改变排列顺序和归一化时一并进行，效率 比原本要再高一些，同时支持不同的颜色空间变换。

```cpp
// common.h
struct TRTDeleter {
    template <typename T>
    void operator()(T* obj) const {
        if (obj) {
#if NV_TENSORRT_MAJOR < 9
            obj->destroy();
#else
            delete obj;  // TensorRT 10.0+ 推荐做法
#endif
        }
    }
};

template <typename T>
using TRTPtr = std::unique_ptr<T, TRTDeleter>;

template <typename T>
concept Processor = requires(const cv::Mat& img, float* output) {
    { T::process(img, output) } -> std::same_as<void>;
};

// 张量规格，编译期维度信息
template <size_t... Sizes>
struct TensorSpec {
    static constexpr size_t total_size() {
        size_t size = 1;
        ((size *= Sizes), ...);
        return size;
    }
    static constexpr std::array<size_t, sizeof...(Sizes)> dims() {
        return {Sizes...};
    }
};

template <typename T>
concept TensorSpecType = requires {
    { T::total_size() } -> std::convertible_to<size_t>;
};

// 编译期数组
template <float... elems>
struct FloatArraySpec {
    static constexpr std::array<float, sizeof...(elems)> values() {
        return {elems...};
    }
};

/* 是是否编译期数组规格类型，即可通过values()方法获取std::array<float,
 * N>类型的数组，其中N为元素个数个数 */
template <typename T>
concept FloatArraySpecType = requires {
    {
        T::values()
    } -> std::convertible_to<std::array<float, T::values().size()>>;
};

// 颜色空间映射，编译器索引信息
// 注意：使用RGB表示第一个，第二个和第三个通道的索引位置，哪怕你不是要转成RGB
template <typename T>
concept ChannelMapType =
    requires {
        { T::r } -> std::convertible_to<int>;
        { T::g } -> std::convertible_to<int>;
        { T::b } -> std::convertible_to<int>;
        { T::index() } -> std::convertible_to<std::array<int, 3>>;
    } && (T::r >= 0 && T::r < 3) && (T::g >= 0 && T::g < 3) &&
    (T::b >= 0 && T::b < 3);

template <int R, int G, int B>
struct ChannelMapSpec {
    static constexpr int r = R;
    static constexpr int g = G;
    static constexpr int b = B;

    static constexpr std::array<int, 3> index() {
        return {R, G, B};
    }
};

using KeepChannelMap = ChannelMapSpec<0, 1, 2>;


// Processor.h
template <size_t... Is>
constexpr auto make_alphas_impl(const std::array<float, sizeof...(Is)>& stds,
                                std::index_sequence<Is...>) {
    return std::array<float, sizeof...(Is)> {(1.0f / (255.0f * stds[Is]))...};
}

template <size_t... Is>
constexpr auto make_betas_impl(const std::array<float, sizeof...(Is)>& means,
                               const std::array<float, sizeof...(Is)>& stds,
                               std::index_sequence<Is...>) {
    return std::array<float, sizeof...(Is)> {(-means[Is] / stds[Is])...};
}

template <TensorSpecType InputSpec, FloatArraySpecType Mean,
          FloatArraySpecType Std, ChannelMapType ChannelMap = KeepChannelMap>
class ConvertHWC2CHW {
private:
    constexpr static int input_height = InputSpec::dims()[0];
    constexpr static int input_width = InputSpec::dims()[1];
    constexpr static int channel_num = InputSpec::dims()[2];
    constexpr static std::array<int, channel_num> channel_map =
        ChannelMap::index();

    constexpr static std::array<float, channel_num> alphas = make_alphas_impl(
        Std::values(), std::make_index_sequence<channel_num> {});
    constexpr static std::array<float, channel_num> betas =
        make_betas_impl(Mean::values(), Std::values(),
                        std::make_index_sequence<channel_num> {});

public:
    static void process(const cv::Mat& input, float* output) {
        int channel_size = input_height * input_width;

        std::array<cv::Mat, channel_num> bgr_channels;
        cv::split(input, bgr_channels);

        for (int i = 0; i < channel_num; ++i) {
            cv::Mat target_slice(
                input_height, input_width, CV_32FC1,
                output + ChannelMap::index()[i] * channel_size);

            bgr_channels[i].convertTo(target_slice, CV_32FC1,
                                      alphas[channel_map[i]],
                                      betas[channel_map[i]]);
        }
    }
};
```

### 编写推理类

声明推理类，用于统一管理上下文和流的生命周期，并提供访问的 API。近用于执行推理， 数据的输入输出由外部管理，所需参数由外部提供。

推理类在构造时接受 ICudaEngine 对象，并创建其引用。之后通过它创建上下文 （IExecuteContext），并创建一个流（CudaStream）。通过对外提供的公共接口由外部分配内存并绑定模型输入输出的内存指针，对外提供的 `infer` 接口仅运行推理操作，由外部 负责及时读取和写入输入输出。`infer` 函数接受一个回调函数，用于对 `context` 进行一些 配置，如设置动态的 batch size。

```cpp
using TRTInferConfigFun = std::function<void(nvinfer1::IExecutionContext*)>;

class TRTInference {
protected:
    nvinfer1::ICudaEngine& engine;
    TRTPtr<nvinfer1::IExecutionContext> context;
    cudaStream_t stream;

public:
    TRTInference() = delete;

    TRTInference(nvinfer1::ICudaEngine& engine)
        : engine(engine), context(engine.createExecutionContext()) {
        cudaStreamCreate(&stream);
    }

    TRTInference(const TRTInference&) = delete;
    TRTInference& operator=(const TRTInference&) = delete;
    TRTInference(TRTInference&&) noexcept = delete;
    TRTInference& operator=(TRTInference&&) noexcept = delete;

    ~TRTInference() {
        cudaStreamDestroy(stream);
    }

    template <typename... Args>
    bool set_tensor_address(Args&&... args) {
        return context->setTensorAddress(std::forward<Args>(args)...);
    }

    template <typename... Args>
    const void* get_tensor_address(Args&&... args) {
        return context->getTensorAddress(std::forward<Args>(args)...);
    }

    cudaStream_t get_stream() {
        return stream;
    }

    bool infer(TRTInferConfigFun configFun = nullptr) {
        if (configFun) {
            configFun(context.get());
        }
        return context->enqueueV3(stream);
    }
};
```

### 编写 Cifar100 模型的运行类

为 Cifar100 编写运行类，接受模型的文件路径，声明各种模板类型，负责分配显存存储模 型的输入输出，以及从 CPU 向 GPU 传递数据的临时缓冲区。封装预处理，向 gpu 传入数 据，发送推理任务，从 gpu 读取结果，后处理等流程。推理接口接受图片数组并传回每张 图片的分类结果，即对应类别 ID。

支持动态 batch size，在 infer 函数中将输入数组分割成一个个 batch，再批量进行处理。 预处理和后处理部分使用 OpenMP 进行简单并行处理。模型构建时设置最小，最优，最大 batch size 分别为 1，64，256。分配显存和内存时按照最大来分配避免重复分配，传递数 据和推理时使用动态大小。

```cpp
// Cifar100CNN.h
class Cifar100CNN {
private:
    std::string onnx_path;
    std::string engine_path;

    TRTPtr<nvinfer1::ICudaEngine> engine;
    TRTPtr<TRTInference> inference;

    float* input_buffer;
    float* output_buffer;

    void* gpu_input;
    void* gpu_output;

    void set_tensor_addresses();

    void preprocess(std::vector<cv::Mat>::const_iterator input,
                    size_t batch_size);
    void postprocess(std::vector<int>::iterator output, size_t batch_size);
    void infer(size_t batch_size);
    auto InputData(size_t batch_size);
    auto OutputData(size_t batch_size);

public:
    Cifar100CNN(std::string onnx_path, std::string engine_path,
                TRTLogger& logger, bool always_rebuild = false);
    ~Cifar100CNN() noexcept;
    std::vector<int> infer(const std::vector<cv::Mat>& input,
                           size_t batch_size = 0);
};

// Cifar100CNN.cpp
using InputImg = TensorSpec<32, 32, 3>;
using OutputRes = TensorSpec<100>;
using Mean = FloatArraySpec<0.5071f, 0.4865f, 0.4409f>;
using Std = FloatArraySpec<0.2673f, 0.2564f, 0.2761f>;
using ChannelMap = ChannelMapSpec<2, 1, 0>;
using Cifar100Processor = ConvertHWC2CHW<InputImg, Mean, Std, ChannelMap>;

using BatchSize = TensorSpec<1, 64, 256>;
using Input = TensorSpec<BatchSize::dims()[2], 3, 32, 32>;
using Output = TensorSpec<BatchSize::dims()[2], 100>;

void Cifar100CNN::set_tensor_addresses() {
    inference->set_tensor_address("input", gpu_input);
    inference->set_tensor_address("output", gpu_output);
}

void Cifar100CNN::preprocess(std::vector<cv::Mat>::const_iterator input,
                             size_t batch_size) {
    size_t img_size = InputImg::total_size();

#pragma omp parallel for
    for (size_t i = 0; i < batch_size; ++i) {
        Cifar100Processor::process(*(input + i), input_buffer + i * img_size);
    }
}

void Cifar100CNN::postprocess(std::vector<int>::iterator output,
                              size_t batch_size) {
    size_t res_size = OutputRes::total_size();

#pragma omp parallel for
    for (size_t i = 0; i < batch_size; ++i) {
        float* output_buffer_idx = output_buffer + i * res_size;
        int class_id = std::distance(
            output_buffer_idx,
            std::max_element(output_buffer_idx, output_buffer_idx + res_size));
        *(output + i) = class_id;
    }
}

void Cifar100CNN::infer(size_t batch_size) {
    inference->infer([batch_size](nvinfer1::IExecutionContext* context) {
        context->setInputShape(
            "input", nvinfer1::Dims4 {(int64_t) batch_size, 3, 32, 32});
    });
}

auto Cifar100CNN::InputData(size_t batch_size) {
    return cudaMemcpyAsync(gpu_input, input_buffer,
                           sizeof(float) * InputImg::total_size() * batch_size,
                           cudaMemcpyHostToDevice, inference->get_stream());
}

auto Cifar100CNN::OutputData(size_t batch_size) {
    return cudaMemcpyAsync(output_buffer, gpu_output,
                           sizeof(float) * OutputRes::total_size() * batch_size,
                           cudaMemcpyDeviceToHost, inference->get_stream());
}

Cifar100CNN::Cifar100CNN(std::string onnx_path, std::string engine_path,
                         TRTLogger& logger, bool always_rebuild)
    : onnx_path(onnx_path), engine_path(engine_path) {
    auto builder = TRTModelBuilder(logger);
    if (always_rebuild || !(engine = builder.loadFromPlan(engine_path))) {
        engine = builder.buildFromOnnx(
            onnx_path, engine_path,
            [](nvinfer1::IBuilderConfig* config,
               nvinfer1::INetworkDefinition* network,
               nvinfer1::IBuilder* builder) {
                auto profile = builder->createOptimizationProfile();
                const char* inputName = network->getInput(0)->getName();
                auto batch_sizes = BatchSize::dims();
                // [Min, Opt, Max]
                profile->setDimensions(
                    inputName, nvinfer1::OptProfileSelector::kMIN,
                    nvinfer1::Dims4 {(int64_t) batch_sizes[0], 3, 32, 32});
                profile->setDimensions(
                    inputName, nvinfer1::OptProfileSelector::kOPT,
                    nvinfer1::Dims4 {(int64_t) batch_sizes[1], 3, 32, 32});
                profile->setDimensions(
                    inputName, nvinfer1::OptProfileSelector::kMAX,
                    nvinfer1::Dims4 {(int64_t) batch_sizes[2], 3, 32, 32});
                config->addOptimizationProfile(profile);

                // 2. 精度设置：虽然 kFP16 弃用，但在 10.0 中作为 BuilderFlag
                // 依然是生效的（会有警告）
                if (builder->platformHasFastFp16()) {
                    config->setFlag(nvinfer1::BuilderFlag::kFP16);
                }
            });
    }
    inference = TRTPtr<TRTInference>(new TRTInference(*engine));

    cudaHostAlloc((void**) &input_buffer, sizeof(float) * Input::total_size(),
                  cudaHostAllocDefault);
    cudaHostAlloc((void**) &output_buffer, sizeof(float) * Output::total_size(),
                  cudaHostAllocDefault);
    cudaMalloc(&gpu_input, sizeof(float) * Input::total_size());
    cudaMalloc(&gpu_output, sizeof(float) * Output::total_size());

    set_tensor_addresses();
}

Cifar100CNN::~Cifar100CNN() noexcept {
    cudaFree(gpu_input);
    cudaFree(gpu_output);
    cudaFreeHost(input_buffer);
    cudaFreeHost(output_buffer);
}

std::vector<int> Cifar100CNN::infer(const std::vector<cv::Mat>& input,
                                    size_t batch_size) {
    size_t input_size = input.size();
    std::vector<int> res(input_size);

    auto batch_sizes = BatchSize::dims();
    if (batch_size == 0) {
        // 使用默认的最优 batch size
        batch_size = batch_sizes[1];
    } else if (batch_size > batch_sizes[2]) {
        // 若超过则使用最大 batch size
        batch_size = batch_sizes[2];
    }

    size_t batches =
        input_size / batch_size + (((input_size % batch_size) == 0) ? 0 : 1);

    for (size_t i = 0; i < batches; ++i) {
        size_t cur_batch_size =
            std::min(batch_size, input_size - i * batch_size);
        preprocess(input.cbegin() + i * batch_size, cur_batch_size);
        InputData(cur_batch_size);
        infer(cur_batch_size);
        OutputData(cur_batch_size);
        cudaStreamSynchronize(inference->get_stream());
        postprocess(res.begin() + i * batch_size, cur_batch_size);

        std::cout << std::format("batch {} with size {} finished\n", i,
                                 cur_batch_size);
    }

    return res;
}
```

### 最终测试

使用之前 Pytorch 导出的数据库和元数据进行测试，用 C++读取 JSON 元数据并用 OpenCV 读取图片，通过上面写的 Cifar100 类进行推理，batch size 设置为 128，最终成功率为 69.32%，与训练时基本一致，说明以上各流程没有明显问题。
