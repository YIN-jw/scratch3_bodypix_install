# scratch3_bodypix_install
在Scratch3.0上安装bodypix插件
### 一、Scratch3.0 GUI搭建插件
可参考scratch3.0_posenet_install中的过程，发散思维，任何一个插件在GUI上的搭建过程均为这样。<br>
1.1、在scratch-vm\src\extensions下新建scratch3_bodypix文件夹，放入写好的index.js。<br>
1.2、修改extensions-manager.js,如下：<br>
```
const dispatch = require('../dispatch/central-dispatch');
const log = require('../util/log');
const maybeFormatMessage = require('../util/maybe-format-message');
const BlockType = require('./block-type');
const Scratch3KnnBlocks = require('../extensions/scratch3_knn');
const Scratch3FaceapiBlocks = require('../extensions/scratch3_faceapi');
+const Scratch3BodypixBlocks = require('../extensions/scratch3_bodypix');
// These extensions are currently built into the VM repository but should not be loaded at startup.
// TODO: move these out into a separate repository?
// TODO: change extension spec so that library info, including extension ID, can be collected through static methods

const builtinExtensions = {
    // This is an example that isn't loaded with the other core blocks,
    // but serves as a reference for loading core blocks as extensions.
    coreExample: () => require('../blocks/scratch3_core_example'),
    // These are the non-core built-in extensions.
    pen: () => require('../extensions/scratch3_pen'),
    wedo2: () => require('../extensions/scratch3_wedo2'),
    music: () => require('../extensions/scratch3_music'),
    microbit: () => require('../extensions/scratch3_microbit'),
    text2speech: () => require('../extensions/scratch3_text2speech'),
    translate: () => require('../extensions/scratch3_translate'),
    videoSensing: () => require('../extensions/scratch3_video_sensing'),
    ev3: () => require('../extensions/scratch3_ev3'),
    makeymakey: () => require('../extensions/scratch3_makeymakey'),
    boost: () => require('../extensions/scratch3_boost'),
    gdxfor: () => require('../extensions/scratch3_gdx_for'),
    knnAlgorithm:() =>require('../extensions/scratch3_knn'),
    faceapi:()=>require('../extensions/scratch3_faceapi'),
   +bodypix:()=>require('../extensions/scratch3_bodypix')
};
......
```
1.3、 在scratch-gui\src\lib\libraries\extensions路径下新建文件夹bodypix。在其中放入bodypix.png和bodypix-small.svg。<br>
1.4、 设置index.jsx 文件，该文件位于scratch-gui\src\lib\libraries\extensions路径下。修改方式如下，+所指部部分为添加的代码： 首先调用刚才放置好的.svg和.png图片作为模块的封面：
```
......
import makeymakeyIconURL from './makeymakey/makeymakey.png';
import makeymakeyInsetIconURL from './makeymakey/makeymakey-small.svg';

import knnalgorithmImage from './knnAlgorithm/knnAlgorithm.png';
import knnalgorithmInsetImage from './knnAlgorithm/knnAlgorithm-small.svg';

+import bodypixImage from './bodypix/bodypix.png';
+import bodypixInsetImage from './bodypix/bodypix-small.svg';

import microbitIconURL from './microbit/microbit.png';
import microbitInsetIconURL from './microbit/microbit-small.svg';
import microbitConnectionIconURL from './microbit/microbit-illustration.svg';
import microbitConnectionSmallIconURL from './microbit/microbit-small.svg';
......
```
之后设置bodypix extension的封面：
```
export default [
......
},
+{
+       name: (
+           <FormattedMessage
+               defaultMessage="bodypix"
+               description="Name for the 'bodypix' extension"
+               id="gui.extension.bodypix.name"
+           />
+       ),
+       extensionId: 'bodypix',
+       iconURL: bodypixImage,
+       insetIconURL: bodypixInsetImage,
+       description: (
+           <FormattedMessage
+               defaultMessage="bodypix."
+               description="Description for the 'bodypix' extension"
+               id="gui.extension.bodypix.description"
+           />
+       ),
+       featured: true
+   },
......
```
注意extensionId部分的内容和步骤2中第一个代码框中冒号前内容相同，这是bodypix extension的id 属性，因此必须相同，之后index.js的编写中也会有相应部分的提示。
1.5、进行测试。此时在/scratch-gui/ 中运行webpack-dev-server –https，之后新建console dialog 在/scratch-vm/ 中运行yarn run watch，再新建console dialog 在/scratch-gui/ 中运行yarn link scratch-vm。后所得的网页，在https://127.0.0.1:8601/ 端口可以看到bodypix的封面已经完成，但此时点击不会有内容，我们此时需要对index.js的内容进行编辑。
### 二、bodypix模型的加载<br>
2.1、环境配置<br>
使用的是：@tensorflow-models/body-pix@2.0.5；peerDependencies："@tensorflow/tfjs-converter": "^1.3.1"，"@tensorflow/tfjs-core": "^1.3.1"<br>
当使用npm或者是yarn后，模型依赖的版本可以在package.json文件中进行查看。<br>
2.2、导入Tensorflow.js库<br>
在index的开头加入：<br>
```
require('babel-polyfill');
const Runtime = require('../../engine/runtime');

const ArgumentType = require('../../extension-support/argument-type');
const BlockType = require('../../extension-support/block-type');
const Clone = require('../../util/clone');
const Cast = require('../../util/cast');
const Video = require('../../io/video');
const formatMessage = require('format-message');
//
//import * as bodyPix from '@tensorflow-models/body-pix';
const bodyPix = require('@tensorflow-models/body-pix');
const tf = require('@tensorflow/tfjs-core');
//const tf = require('@tensorflow/tfjs');
const canvas = require('canvas')
......
```
然后保存后根据console的报错安装依赖即可，这里直接yarn add @tensorflow/tfjs-core,安装的为1.7.2版本，bodypix和posenet均可使用。<br>
### 三、加载网络<br>
参考官方给的事例：https://github.com/tensorflow/tfjs-models/tree/master/body-pix<br>
可以加载的网络有两个，mobilenet和resnet，在此选用mobilenet进行加载。由于加载网络耗时久，采用异步加载的方法：<br>
```
    async bodyPixInit () {
		//
        this.bodyPix = await bodyPix.load({//加载模型，还可以选用'ResNet50'
        architecture: 'MobileNetV1',
        outputStride: 16,
        multiplier: 0.75,
        quantBytes: 2
        });
		console.log(this.bodyPix)
    }
```
可以通过控制台看到模型的相关信息，给出的是一个数列，与posenet类似，很好理解。<br>
### 四、做出预测（各API的使用）<br>
4.1、Person segmentation<br>
该API输入为一个或多个人的图像，person segmentation可做出对所有人体(此处不用区分单人或多人)的分割。它返回PersonSegmentation对应于图像中人物分割的对象。它不会在不同的个人之间消除歧义（即不会将多人中每个人区分开）。如果您需要将个人细分，请使用segmentMultiPerson（警告是此方法比较慢）。<br>
```
......
    personSegmentation(args, util) {
        if (this.globalVideoState === VideoState.OFF) {
            alert('请先打开摄像头')
            return
        }
		return new Promise((resolve, reject) => {
        this.timer = setInterval(async () => {
				const imageElement = this.video;
				this.video.width = 640;
				this.video.height = 480;
				const flipHorizontal = false;
				const internalResolution = 'medium';
				const segmentationThreshold = 0.7;
				const maxDetections = 5;
				const scoreThreshold = 0.3;
				const nmsRadius = 20;
				//
				const net = this.bodyPix;
                const personsegmentation = await net.segmentPerson(imageElement,{flipHorizontal,internalResolution,segmentationThreshold,maxDetections,scoreThreshold,nmsRadius}); 
                resolve('is loaded')
                console.log(personsegmentation)
            },5000);
        })
    }
......
```
返回personsegmentation，包括画布中每个像素的值，背景为0，人物为1，并且包括每个人的pose内容。图像中的多个人合并为一个二进制图像，相当于输出的是一个单个数组。<br>
4.2、Person body part segmentation<br>
输入为一个或多个人的图像，BodyPix的segmentPersonParts方法可以对图像中所有人的24个身体部位进行分割。PartSegmentation对于所有人体，它为每个像素返回对应于身体部位的对象。如果您需要将个人细分，请使用segmentMultiPersonParts（警告是此方法比较慢）。
```
......
	personbodypartSegmentation(args, util) {
        if (this.globalVideoState === VideoState.OFF) {
            alert('请先打开摄像头')
            return
        }
		return new Promise((resolve, reject) => {
        this.timer = setInterval(async () => {
				const imageElement = this.video;
				this.video.width = 640;
				this.video.height = 480;
				const flipHorizontal = false;
				const internalResolution = 'medium';
				const segmentationThreshold = 0.7;
				const maxDetections = 5;
				const scoreThreshold = 0.3;
				const nmsRadius = 20;
				//
				const net = this.bodyPix;
                const personbodypartsegmentation = await net.segmentPersonParts(imageElement,{flipHorizontal,internalResolution,segmentationThreshold,maxDetections,scoreThreshold,nmsRadius}); 
				resolve('is loaded')
                
                console.log(personbodypartsegmentation)
            }, 5000);
        })
    }
......
```
返回该personbodypartsegmentation对象包含一个宽度，高度，Pose和一个Int32Array，其ID为对应的身体部位的一部分，其ID为0-24，否则为-1。即像素点为-1表示非人员部分，为0表示为人体的左脸。当图像中有多个人时，他们将合并为单个数组。<br>
4.3、Multi-person segmentation<br>
给定一个包含多人的图像，多人分割模型可以分别预测每个人的分割(能将多人中每个人再区分开)。它返回一个数组，PersonSegmentation每个数组对应一个人。每个元素都是一个人的二进制数组，其中一个人的像素为1，否则为0。阵列大小对应于图像中的像素数。如果您不需要将个人细分，则使用segmentPerson速度更快且不会细分个人。
