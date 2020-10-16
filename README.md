# Analysis-of-workflow-of-NVDLA
Source code reading
===================
## NVDLA Compiler
### Default parameters setting 

  ```
  Usage: ./nvdla_compiler [options] --prototxt <prototxt_file> --caffemodel <caffemodel_file>
  where options include:
  -h                                              print this help message
  -P                                              project
  -i <inputpath>                                  inputPath
  -o <outputpath>                                 outputs wisdom files in 'outputpath' directory
  -t                                              testname
  --prototxt                                      prototxt file
  --caffemodel
  --cachemodel
  --profileName
  --profilecfg                                    profile from file
  --profile                                       computation profile Name (default: fast-math)
  --cprecision <fp16|int8>                        compute precision (default: int8)
  --configtarget <nv_full|nv_large|nv_small>      target platform (default: nv_full)
  --calibtable <int8 calibration table>           calibration table for INT8 networks (default: 0.00787)
  --quantizationMode <per-kernel|per-filter>      quantization mode for INT8 (default: NONE)
  --batch                                         batch size (default: 0)
  --informat <ncxhwx|nchw|nhwc>                   input data format (default: nhwc)
  ```

### Workflow of Compiler

1. important structures 

  ```c++
  struct TestInfo{
    // common
    nvdla::IWisdom* wisdom;
    std::string wisdomPath;
    // parse
    std::string modelsPath;
    std::string profilesPath;
    std::string calibTablesPath;
    // runtime
    nvdla::IRuntime* runtime;
    nvdla::ILoadable* compiledLoadable;
    NvU8 *pData;
    std::string inputImagesPath;
    std::string inputLoadablePath;
    std::map<std::string, NvDlaImage*> inputImages;
    std::map<std::string, void *> inputBuffers;
    std::map<std::string, NvDlaImage*> outputImages;
    std::map<std::string, void *> outputBuffers;
    std::vector<SubmitContext*> submits;
    NvU32 timeout;
    NvU16 numBatches; // runtime's point-of-view
    NvU32 numSubmits; };
    
  struct TestAppArgs{
    std::string project;
    std::string inputPath;
    std::string inputName;
    std::string outputPath;
    std::string testname;
    std::string testArgs;
    std::string prototxt; // This should be folded into testArgs
    std::string caffemodel; // This should be folded into testArgs
    std::string cachemodel; // This should be folded into testArgs
    std::string profileName; // ok here?
    std::string profileFile;
    std::string configtarget;
    std::string calibTable;
    nvdla::QuantizationMode quantizationMode;
    NvU16 numBatches;
    nvdla::DataFormat inDataFormat;
    nvdla::DataType computePrecision;
    std::map<std::string, NvF32> tensorScales; };
  ```
  
2. Set up parameters

3. launchTest（)
  * testSetup
    1. clear wisdom file if any exist 
    2. Initiaize TestInfor
  * parseAndCompile
    1. Create new wisdom
    2. Parse
      * important classes
        1. intermediate representation in memory of the inputting model
        ```c++      
        class Network : public INetwork{
        public: // externally facing
          virtual ITensor* addInput(const char* name, Dims4 dimensions);
          //	virtual void markChanged(const ILayer*);
          virtual bool markInput(ITensor * tensor);
          virtual void markOutput(ITensor* tensor);
          virtual IConvolutionLayer *    addConvolution(ITensor* input, int numOutputs, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasmode, int numGroups);
          virtual IFullyConnectedLayer * addFullyConnected(ITensor* input, int outputSize, Weights kernelWeights, Weights biasWeights, BiasMode biasMode);
          virtual IActivationLayer *     addActivation(ITensor* input, ActivationType type);
          virtual IPoolingLayer *        addPooling(ITensor* input, PoolingType type, Dims2 windowSize, Dims2 stride, Dims2 tlPadding, Dims2 brPadding);
          virtual ILRNLayer *            addLRN(ITensor* input, int window, float alpha, float beta, float k);
          virtual IScaleLayer *          addScale(ITensor* input, ScaleMode mode, Weights shift, Weights scale, Weights power);
          virtual IBatchNormLayer *      addBatchNorm(ITensor* input, BatchNormMode mode, Weights mean, Weights variance, float epsilon);
          virtual ISoftMaxLayer *        addSoftMax(ITensor* input);
          virtual IConcatenationLayer *  addConcatenation(ITensor * const * inputs, int numInputs);
          virtual ISliceLayer *          addSlice(ITensor* input, int numOutputs);
          virtual IDeconvolutionLayer *  addDeconvolution(ITensor* input, int numOutputs, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups);
          virtual IElementWiseLayer *    addElementWise(ITensor* input0, ITensor* input1, ElementWiseOperation op);   
          ...
        public: // internally facing
          Network();
          virtual ~Network();
          virtual bool serializeTo(WisdomContainerEntry *) const;
          virtual bool deserializeFrom(WisdomContainerEntry *);
          virtual bool assignSymbols(Wisdom *);
        protected:
          friend class Wisdom;
          friend class NetworkFactory;
          void destroy();
        private:
          std::string newLayerName() const;
          std::string newTensorName() const;
          ITensor* addTensor(const std::string & s);
          const ILayer* findLayer(const std::string& name) const;
          bool checkNames(const char* name);
        
          // intermediate analysis result before compiler
          std::vector<ITensor *> mTensors;    // recording all input tensors 
          std::vector<ILayer *>  mLayers;     // recording all layers
          std::vector<ITensor *> mInputs;     // recording all input tensors
          std::vector<ITensor *> mOutputs;    // recording the final output tensor 

          OutputDimensionsFormula* mConvDims, *mDeconvDims, *mPoolDims;
        };
        ```
        2. recording all tensors used in model inference into mMap
        ```c++
        //recording all tensors used in model inference into mMap
        class BlobNameToTensor : public IBlobNameToTensor{
        public:
          virtual void add(const std::string& name, ITensor* tensor);
          virtual ITensor* find(const char* name) const;
          virtual ITensor*& operator[](const std::string& name);
          virtual void setTensorNames();
          virtual ~BlobNameToTensor();
        private:
          std::map<std::string, ITensor*> mMap;
        };
        ```
        3. info of tensor     
        ```c++
        // info of tensor
        class Tensor  : public ITensor{
          ...
        protected:
          Dims4             mDimensions;
          INetwork*         mNetwork;
          std::string       mName;    // the user name if the user provided one, else
          DataFormat        mDataFormat;
          DataType          mDataType;
          TensorType        mTensorType; // the type of surface this tensor represents: image/i-o/kernel/bias
          std::vector<NvF32> mChnlScales;     // per-channel scaling factors
          std::vector<NvF32> mChnlOffsets;    // per-channel offsets
        };
        ```
        4. general info of layer  
        ```c++
        // info of layer
        class Layer : public virtual ILayer{
        public: // externally facing
          Layer(Network* network);
          ...
        public: // internally facing
          virtual NvU16 getFactoryType() const = 0;
          virtual bool serializeTo(WisdomContainerEntry *) const;
          virtual bool deserializeFrom(WisdomContainerEntry *);

          std::string getInputSymbol(int i) const;
          void setInput(int i, ITensor *);

          std::string getOutputSymbol(int o) const;
          void setOutput(int o, ITensor *);

          virtual bool assignSymbols(Wisdom *wisdom);

        protected:

          INetwork* mNetwork;

          Layer(INetwork *n, LayerType type, const std::string& name, ITensor * const * inputs, int numInputs, ITensor * const * outputs, int numOutputs);
          Layer(INetwork *n, LayerType type, const std::string& name, std::vector<std::string> &input_symbols, int numInputs, std::vector<std::string> &output_symbols, int numOutputs);
          Layer(INetwork *n, LayerType type, const std::string& name, ITensor* input, ITensor* output);
          virtual ~Layer();

          const LayerType mType;
          std::string mName;
          std::vector<ITensor *> mInputs, mOutputs;
          std::vector<std::string> mInputSymbols, mOutputSymbols;
        };
        ```
        5. info of convolutional layer     
        ```c++
        //info of convolutional layer
        class ConvolutionLayer : public virtual IConvolutionLayer, public priv::Layer{
        public:
          ConvolutionLayer(INetwork * network, const std::string & name, ITensor * input, ITensor * output, int numOutputMaps, Dims2 kernelSize, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups);
          ConvolutionLayer(INetwork * network, const std::string & name, ITensor * input, ITensor * output, int numOutputMaps, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups);
          virtual ~ConvolutionLayer();
          ...
        protected:
          friend class LayerFactory;
          ConvolutionLayer();
          Parameters mParams;
        };
        ```
      * workflow
        1. parsing Caffe Network
          * integrating whole information into network from caffemodel and prototxt
          ```c++
          const IBlobNameToTensor* CaffeParser::parse(const char* deployFile, const char* modelFile, INetwork * network){
            ...
            network->setPoolingOutputDimensionsFormula(new CaffeParserPoolingDimsCallback);   //network->mPoolDims = new CaffeParserPoolingDimsCallback;
            // reading information from caffemodel to mModel, which will be used for generating the variable weights
            mModel = new dc::NetParameter();
            readBinaryProto(mModel/*.get()*/, modelFile, mProtobufBufferSize);
            // reading information from prototxt to mDeploy
            mDeploy = new dc::NetParameter();
            readTextProto(mDeploy/*.get()*/, deployFile);
            // recording the weights info into variable weights
            CaffeWeightFactory weights(*mModel/**mModel.get()*/, false /*weightType == DataType::kHALF*/, mTmpAllocs);
            // integrating info into mMap, network->mTensors and network->mInputs
            for (int i = 0; i < mDeploy->input_size(); i++){
              Dims4 dims;
              ... // setting dims parameter
              ITensor* tensor = network->addInput(mDeploy->input().Get(0).c_str(), dims);   //adding the generated tensor object into network->mTensors; adding the generated tensor object into network->mInputs.
              mBlobNameToTensor->add(mDeploy->input().Get(0), tensor);   // recording tensor info into mBlobNameToTensor->mMap
            }
            // parsing each layer, integrating info into network->mlayers and recording output tensor info of each layer into mMap
            for (int i = 0; i < mDeploy->layer_size() && ok; i++){
              const dc::LayerParameter& layerMsg = mDeploy->layer(i);
              if (layerMsg.type() == "Dropout"){
                mBlobNameToTensor->add(layerMsg.top().Get(0), mBlobNameToTensor->find(layerMsg.bottom().Get(0).c_str()));
                continue;
              }
              if (layerMsg.type() == "Input"){
                const dc::InputParameter& p = layerMsg.input_param();
                for (int i = 0; i < layerMsg.top_size(); i++){
                  const dc::BlobShape& shape = p.shape().Get(i);
                  Dims4 dims(shape.dim().Get(0), shape.dim().Get(1), shape.dim().Get(2), shape.dim().Get(3));
                  ITensor* tensor = network->addInput(layerMsg.top(i).c_str(), dims);
                  mBlobNameToTensor->add(layerMsg.top().Get(i), tensor);
                }
                continue;
              }
              if (layerMsg.type() == "Flatten"){
                ITensor* tensor = (*mBlobNameToTensor)[layerMsg.bottom().Get(0)];
                (*mBlobNameToTensor)[layerMsg.top().Get(0)] = tensor;
                std::cout << "Warning: Flatten layer ignored." << std::endl;
                continue;
              }
              LayerParseFnMap::iterator v = gParseTable.find(layerMsg.type());
              ILayer* layer = (*v->second)(network, layerMsg, weights, mBlobNameToTensor); // parsing each layer and integrating corresponding layer informaion into network->mlayers, the detail of which is explained in the following section 
              layer->setName(layerMsg.name().c_str());
              mBlobNameToTensor->add(layerMsg.top(0), layer->getOutput(0));   //recording the output of each layer into mBlobNameToTensor->mMap
            }
          }
          ```
          * recording information for each layer into network（network->mlayers), taking convolutional layer as an example. 
          ```c++
          1. static ILayer* parseConvolution(INetwork *network, const dc::LayerParameter& msg, CaffeWeightFactory& weightFactory, IBlobNameToTensor* tensors){
                ...
                // TODO: cross-correlation vs convolution
                layer = network->addConvolution((*tensors)[msg.bottom(0)], numOutputs, 0, kernelSize, tlPadding, brPadding, stride, dilation, kernelWeights, biasWeights, biasMode, numGroups);
                return layer;
             }
          2. IConvolutionLayer* Network::addConvolution(ITensor* inputTensor, int numOutputChannels, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups){
                string name = newLayerName();
                ITensor* output = addTensor(newTensorName());
                Tensor*  output_priv = TensorFactory::priv(output);
                ConvolutionLayerDiamond d = LayerFactory::newConvolutionLayer(this, name, inputTensor, output, numOutputChannels, paddingValue, kernelSize, tlPadding, brPadding, stride, dilation, kernelWeights, biasWeights, biasMode, numGroups);
                output->setDimensions( d.derived().priv()->getOutputDimensions() );
                mLayers.push_back(d.base().i());
                return d.derived().i();
             }
          3. ConvolutionLayerDiamond LayerFactory::newConvolutionLayer(INetwork * network, const std::string & name, ITensor * input, ITensor * output, int numOutputMaps, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups){
                ...
                base_priv = derived_priv = new ConvolutionLayer(network, name, input, output, numOutputMaps, paddingValue, kernelSize, tlPadding, brPadding, stride, dilation, kernelWeights, biasWeights, biasMode, numGroups);
                ...
             }
          4. ConvolutionLayer::ConvolutionLayer(INetwork* network, const std::string& name, ITensor* input, ITensor* output, int numOutputMaps, int paddingValue, Dims2 kernelSize, Dims2 tlPadding, Dims2 brPadding, Dims2 stride, Dims2 dilation, Weights kernelWeights, Weights biasWeights, BiasMode biasMode, int numGroups): Layer(network, LayerType::kCONVOLUTION, name, input, output){
                mParams.kernelSize = kernelSize;   // each layer possesses a mParams displaying the parameters setting
                mParams.numOutputMaps = numOutputMaps;
                mParams.topLeftPadding = tlPadding;
                mParams.bottomRightPadding = brPadding;
                mParams.paddingValue = paddingValue;
                mParams.stride = stride;
                mParams.dilation = dilation;
                mParams.kernelWeights = kernelWeights;
                mParams.biasWeights = biasWeights;
                mParams.biasMode = biasMode;
                mParams.numGroups = numGroups;
             }
          ```
        2. marking the network's outputs
        3. parsing and setting tensor scales according to computation precision
        4. attaching parsed network to the wisdom
        ```c++
        wisdom->setNetworkTransient(network);
        ```
  3. Compile


NVDLA Virtual Platform
======================
## about NVDLA
  NVDLA is provided as a set of IP-core models based on open industry standards: the Verilog model is a synthesis and simulation model in RTL form, and the TLM SystemC simulation model can be used for software development, system integration and testing. 
### Hardware
  NVDLA introduces a modular architecture designed to simplify configuration, integration and portability; it exposes the building blocks used to accelerate core Deep Learning inference operations. NVDLA hardware is comprised of the following components:

    1. Convolution Core – optimized high-performance convolution engine.
    2. Single Data Processor – single-point lookup engine for activation functions.
    3. Planar Data Processor – planar averaging engine for pooling.
    4. Channel Data Processor – multi-channel averaging engine for advanced normalization functions.
    5. Dedicated Memory and Data Reshape Engines – memory-to-memory transformation acceleration for tensor reshape and copy operations.

   Each of these blocks are separate and independently configurable. A system that has no need for pooling, for instance, can remove the planar averaging engine entirely; or, a system that needs additional convolutional performance can scale up the performance of the convolution unit without modifying other units in the accelerator. Scheduling operations for each unit are delegated to a co-processor or CPU; they operate on extremely fine-grained scheduling boundaries with each unit operating independently. This requirement for closely-managed scheduling can be made part of the NVDLA sub-system with the addition of a dedicated management coprocessor (“headed” implementation), or this functionality can be fused with the higher-level driver implementation on the main system processor (“headless” implementation).
### Software
#### Compilation tools: model conversion
   Compiler is responsible for creating a sequence of hardware layers that are optimized for a given NVDLA configuration; having an optimized network of hardware layers increases performance by reducing model size, load and run times.
  1. Parser
   It can read a pre-trained Caffe model and create an “intermediate representation” of a network to pass to the next step of compilation.
  2. Compiler 
   The compiler takes the parsed intermediate representation and the hardware configuration of an NVDLA implementation as its inputs, and generates a network of hardware layers.
#### Runtime environment: run-time software to load and execute networks on NVDLA
   The runtime environment involves running a model on compatible NVDLA hardware. It is effectively divided into two layers:
   1. User Mode Driver
   The main interface with user-mode programs. After parsing the neural network, compiler compiles network layer by layer and converts it into a file format called NVDLA Loadable. User mode runtime driver loads this loadable and submits inference job to Kernel Mode Driver.
    2. Kernel Mode Driver
   Consists of drivers and firmware that do the work of scheduling layer operations on NVDLA and programming the NVDLA registers to configure each functional block. 
   
   Sample platforms are provided which allow users to observe, evaluate, and test NVDLA in a minimal SoC environment. A minimum SoC system configuration consists of a CPU, an NVDLA instance, an interconnect, and memories.
   
   Software
The initial NVDLA open-source release includes software for a “headless” implementation, compatible with Linux. Both a kernel-mode driver and a user-mode test utility are provided in source form, and can run on top of otherwise-unmodified Linux systems.

   The Verilog code included in this release is parameterized such that multiple configurations can be generated from a single source. A hardware tree build is needed to generate the final Verilog RTL code for a given configuration.
   
   For the configurable release, there are currently two spec files included: “nv_large” which has 2048 INT8 MAC’s, and “nv_small” which has 64 INT8 MAC’s plus some other reductions; the non-configurable release has a single spec file, “nv_full”, which has 2048 multi-precision MAC units
   
   If building the Virtual Platform, or another application that uses the NVDLA Cmodel, the following command will build it and install it into outdir/nv_full/cmod/release: ./tools/bin/tmake -build cmod_top
### hardware
  NVDLA is a fixed function accelerator engine which is targeted towards deep learning.
  NVDLA receives commands from the host processor via the CSB (configuration Bus) interface. The two independent memory interfaces provide access to storage for data feeding NVDLA and output data from NVDLA. The interrpt provides a notification to a controlling CPU that NVDLA has completed a task. 
### Simulation
  Virtual platforms reproduce system behavior, execution of target software, debug and development in the absence of "real" hardware platform. 
  The SystemC language allows hardware descriptions to be constructed in a C++ based language. However, as the complexity of the IPs increases, the SystemC simulation environment is not necessarily suitable to provide suitably fast models. It is theoreticaly possible to simulate complex IP's such as CPU's within SystemC simulation kernel. But performance can be an issue, especially when the processor is modelled at RTL level that is computationally intensive. A better solution for complex IPs like CPUs is to model it in a virtualizer or emulator and then to integrate the model into a SystemC simulation environment. Moreover, the TLM-2.0 (Transaction-Level Modeling) standard, which is an extension of SystemC, improves interoperability between memory mapped bus models. It also includes the notion of time quantum which was explicitly intended to assist with this sort of integration. 
#### QEMU
  QEMU is a generic and open source machine & userspace emulator and virtualizer. QEMU is capable of emulating a complete machine in software without any need for hardware virtualization support. 
  QBox is an industrial solution for virtual platform simulation using QEMU and SystemC TLM-2.0.
  QBox is an integration of QEMU virtualizer and emulator in a SystemC model. QBox or QEMU in a (SystemC)Box, treats QEMU as a standard SystemC module within a larger SystemC simulation context. SystemC simulation kernel remains the "master" of the simulation, while QEMU has to fulfi the SystemC API requirements. This solution is an open source QEMU implementation wrapped in a set of SystemC TLM-2.0 interfaces. 
  Depending of the host machine, QBox emulates or virtualizes the core part of the SoC. As QEMU is written in C (as opposed to SystemC which is standard C++ class binary), a wrapper called TLM2C is required to connect them. 
  
#### SystemC

