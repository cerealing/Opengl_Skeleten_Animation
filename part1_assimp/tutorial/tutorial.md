### 骨骼动画的原理

就像人体的骨骼与关节一样，程序中人物模型也是需要绑定一套骨骼来控制人物模型的动作

1. 将骨骼绑定到角色：建立骨骼对模型顶点的映射关系 即 一个顶点可以受到最多4块骨骼的影响，其中每块骨骼的影响大小由**权重**控制，每个顶点的控制骨骼的权重加起来等于1。这样在骨骼移动的时候模型也会跟着移动。

2. 设置骨骼动画：把你想要的角色动画拆解成几个关键帧，你只需要在每个关键帧上给骨骼摆造型。比如一个走路的动画，你只需要设置：1抬前腿,伸左手2抬后腿，伸右手（实际上要设置大概8个关键帧让动作更流畅）的关键帧。这样关键帧变动->骨骼造型变动->模型变动

3. 程序插值：在计算机程序中（建模软件或者opengl）实现：在每个关键帧中平滑地插入更多关键帧，让动画更流畅。

<table align="center">
  <tr>
    <td style="padding-right:100px;">
      <img src="tutorial.assets/poses.gif" style="height:200px;">
    </td>
    <td>
      <img src="tutorial.assets/interpolating.gif" style="height:200px;">
    </td>
  </tr>
</table>

插帧通过根据比例调整前后关键帧中相同骨骼的位移、旋转、缩放实现：

<div align="center">
    <img src="tutorial.assets/scale_factor.png" alt="img">
</div>

factor=current/diff就是当前要插值的帧在前后两个关键帧的比例位置，currentTime越靠近前一个关键帧factor就越小，越靠近后一个关键帧fctor就越大，我们使用线性插值函数作用前后的位移矩阵和缩放矩阵：

<p align="center"> newMat=lastMat*(1-factor)+nextMat*factor</p>

但是对于旋转矩阵应用这个公式就会出现奇怪的问题，我们采用四元数的球面插值来控制旋转

4. 渲染：我们拿到骨骼插值后的变换矩阵，对顶点变换使顶点随骨骼同步运动，最后把顶点丢到渲染管线当中渲染出动画

### Assimp的原理

在建模软件如blender中可以将创建的动画以各种格式如.fbx、.dae等，每种格式都有自己组织骨骼与模型顶点的方式，它们通常被记录在该格式的官方文档中，我们当然可以阅读该文档了解里面的数据是如何排列的，然后自己写一个readFile程序把数据读到内存当中以便渲染，但是如果采用这种方法，对于如果换了另一种模型的格式的情况，我们都要编写特定的代码然不太合适。所以我们需要一个可以读取任何合适的工具。

Assimp就是这种工具，它可以读取不同格式的模型并且以一种统一的组织方式存储到内存当中，我们就可以编写程序从这种格式当中获取骨骼和模型和动画的数据。

## 代码解析

先从assimp如何组织模型数据开始

我们用assimp读取文件后会返回一个aiScene这是最外层的结构

```cpp
const aiScene* scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_GenSmoothNormals | aiProcess_FlipUVs | aiProcess_CalcTangentSpace);
```

通常一个模型是由很多个网格mesh（每个网格有许多的面face，每个面有组成它的顶点vertex），就像一个角色有头，躯干和四肢这些mesh，头又是由三角形，四边形这些face组成，每个三角形又是由3个顶点组成
aiScene中包含一棵节点树，scene->mRootNode 是节点树的根节点，每个节点是一个aiNode*，每个节点中包含对于aiMesh的索引，aiMesh顶点，法线，纹理坐标这些数据，我们第归处理每个节点，拿到里面的数据，这棵树就反应了mesh的层级结构hierarchy

```cpp
void loadModel(string const& path)
{
	Assimp::Importer importer;
	const aiScene* scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_GenSmoothNormals | aiProcess_FlipUVs | aiProcess_CalcTangentSpace);

	if (!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode)
	{
		cout << "ERROR::ASSIMP::" << importer.GetErrorString() << endl;
		return;
	}

	directory = path.substr(0, path.find_last_of('/'));

	processNode(scene->mRootNode, scene);
}

	void processNode(aiNode* node, const aiScene* scene)
	{
		for (unsigned int i = 0; i < node->mNumMeshes; i++)
		{
			aiMesh* mesh = scene->mMeshes[node->mMeshes[i]];
			meshes.push_back(processMesh(mesh, scene));
		}
		for (unsigned int i = 0; i < node->mNumChildren; i++)
		{
			processNode(node->mChildren[i], scene);
		}
	}
```

`meshes.push_back(processMesh(mesh, scene));`处理mesh，把aiMesh的顶点，法线，纹理坐标这些数据拿到我们自己的mesh中，（aiMesh里面有了，我们还要再组织成我们程序能处理的格式。能不能直接渲染assimp里面的数据？当然可以哈哈，不过实现难度大，生命周期由assimp控制，结构不易掌控后期优化难度大）