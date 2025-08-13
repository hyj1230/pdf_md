# **加速半空间三角形光栅化**

## Péter Mileff, Károly Nehéz, Judit Dudra

University of Miskolc, Department of Information Technology, Miskolc-Egyetemváros, 3515 Miskolc, Hungary, mileff@iit.uni-miskolc.hu

University of Miskolc, Department of Information Engineering, Miskolc-Egyetemváros, 3515 Miskolc, Hungary, aitnehez@uni-miskolc.hu

Bay Zoltán Nonprofit Ltd. for Applied Research, Engineering Division (BAY-ENG), Department of Structural Integrity and Production Technologies, Iglói út 2, 3519 Miskolc, Hungary, judit.dudra@bayzoltan.hu

> *摘要: 在过去的十年间，图形处理器（GPU）已主导了交互式计算机图形和可视化技术领域。多年来，成熟的工具与编程接口（如 OpenGL, DirectX API）极大地便利了开发者工作，因为这些框架提供了所有基础可视化算法。本研究的目标是开发一种基于传统 CPU 的纯软件光栅化渲染器。尽管当前其性能相比 GPU 存在诸多不足，但现代多核系统的普及和平台多样性可能需要此类实现方案。本文聚焦于三角形光栅化这一渲染流程的核心环节，通过建立新型模型优化方案，并结合现有技术的改进与创新，显著提升了经典半空间光栅化算法的性能。所提出的技术方案在图形编辑器乃至计算机游戏等领域均具备应用潜力。*
> 
> *关键词: 半空间光栅化；二分算法；软件渲染* 

#### 1 **介绍**

计算机可视化技术的历史可追溯至数十年前，但其重要性在近年尤为凸显。如今，从传统台式计算机到移动及嵌入式设备，先进图形特性几乎无处不在。这一发展成果源于漫长的技术演进，主要动力来自于图形处理器的出现。早期计算机可视化仅依赖低速 CPU 完成整个光栅化流程的所有环节，而当今该任务已由图形加速卡（目标硬件）全面接管。该领域的主要演进方向受到专业计算机游戏、媒体产业以及日益增长的 CAD/CAM 系统需求的共同推动与主导。

然而，我们不应忽视 CPU 技术的最新发展。得益于多核技术和扩展指令集（如 SSE, AVX），现代 CPU 已具备诸多先进特性。除指令集升级和核心数量增加外，2014年问世的 DDR4 内存同样意义重大——尽管其速度较 DDR3 提升了一个数量级，仍无法匹敌现代显卡配备的 DDR5 显存，但这已是提升内存操作速度的重要里程碑。CPU 技术的持续进步正促使开发者重新审视现有图形应用的结构与逻辑模型。多线程游戏引擎模型已成为当代 AAA 级游戏的必备架构。主流图形引擎开发商（虚幻引擎、寒霜引擎、CryEngine 等）已充分认识到这些新技术的潜力。当高性能 GPU 普及的今天，基于 CPU 的解决方案是否仍有价值？顶尖游戏开发商已给出答案：部分现代游戏通过 CPU 分担特定任务以减轻 GPU 负载。软件遮挡剔除技术即是这种混合方案的典范——由 CPU 将多边形渲染至遮挡缓冲区而非 GPU. 诸如《KillZone 3》《Battlefield 3》等知名游戏及 CryEngine 等引擎均采用类似技术，因其能规避 GPU 遮挡查询的延迟问题。此类光栅化通常在低分辨率下进行（如《Battlefield》系列采用 $256 \times 114$ 像素），并可通过软件层次 Z 缓冲实现遮挡检测 [^12] [^19].

基于上述研究背景，本文的核心命题在于如何改进基础渲染算法之一——即面向 CPU 平台的基于多边形填充的光栅化技术。本文旨在针对半空间光栅化模型提出创新算法及扩展方案，为构建新型图形引擎奠定基础。该引擎将采用更复杂的混合渲染管线，可能通过 CPU 执行特定任务实现效能优化。

#### 2 **相关工作**

基于软件的渲染技术在90年代末至21世纪初曾经历繁荣期。随着 GPU 的出现，近年来涉及在可视化流程中应用 CPU 的研究论文数量锐减（但数量逐步增长）。这些论文大多讨论通用显示算法或面向着色器的 GPU 专用方案，此类方案无法直接应用于 CPU 平台。

在早期渲染时代（1996-1998年），ID Software 和 Epic Games 在基于软件的现代计算机图形学领域取得了显著成就。这两家公司因其高性能且复杂的图形引擎而闻名，这些引擎提供了高质量的视觉解决方案（例如，彩色光照、阴影、体积光、雾效、像素级剔除等）。这些引擎针对英特尔奔腾 MMX 处理器家族进行了优化，其渲染系统基于扫描线多边形填充算法，并应用了多种附加技术（如 BSP 树），以提供高性能。随着基于 GPU 的渲染技术持续普及，软件渲染逐渐退居次要地位。尽管如此，仍有一些杰出的成果出现，例如 Rad Game Tools 开发的 Pixomatic Renderer 和 TransGaming 开发的 Swiftshader [^2]. 这两款产品都完全兼容 DirectX 9，非常复杂且高度优化，能够充分利用现代 CPU 的多核多线程能力。它们的像素管线能够持续自我调整，以适应实际的渲染任务。由于这些产品都是专有的，其架构细节并未向公众公开。微软通过开发 DirectX 为 GPU 技术的普及奠定了基础，同时也开发了一款名为 WARP [^3] 的软件光栅化器。该渲染器能够利用多线程的优势，在某些情况下其性能甚至能够超越低端集成显卡。

2008年，基于问题与需求调研，英特尔公司计划通过 Larrabee 项目[^5] 开发其基于x86架构的显卡。从技术层面看，该显卡是多核 CPU 与 GPU 的混合体，旨在提供基于 x86 内核的完全可编程流水线，并配备16字节宽的 SIMD 矢量单元。这种新架构使得图形计算能够通过 x86 指令集进行编程，相比传统 GPU 实现了更灵活的编程方式。

其他研究者提出了三角形遍历算法的优化方案，并引入了新型光栅化模型。作为当前研究成果的一部分，Hengyong Jiangl 等人[^13]提出了一种中点三角形光栅化遍历算法，该算法减少了遍历点数量并提升了图形加速效率，其方案通过 FPGA 进行了验证。Pablo 团队研究比较了三种不同三角形遍历算法（基于方框、Z字形、希尔伯特曲线）的性能，并利用 ModelSim 在 Matlab 中完成仿真[^22]。实验结果表明：在硬件层面实现关键图像处理算法时，可在处理效能与硬件面积间达成重要平衡。Chih-Hao Sun 等人[^15]展示了一种基于边缘方程的分片扫描线三角形遍历算法。该方案通过通用共享硬件执行参数插值与光栅化基础功能，从而降低渲染成本。借助硬件共享及流水线调度等架构设计技术，该算法能以合理硬件成本满足图形应用的实时需求。Olano 团队则提出全新方案——采用二维齐次坐标的简化三角形扫描转换方法，实现快速实时渲染[^17]。该方案规避了昂贵的裁剪测试，渲染真实齐次三角形的速度显著超越先前方案。作为新一代并行算法代表，Zack Bethel 概述了基于多线程分块的现代软件渲染技术。

采用基于分块半空间理论的渲染方法（仅使用CPU进行计算），实现了性能提升[^1]。而 FreePipe 软件光栅化器[^9]专注于多片段特效处理，其工作流程中每个线程处理一个输入三角形，确定其像素覆盖范围，随后对每个像素依次执行着色与混合操作。随着通用 GPU 计算（GPGPU）技术的发展，利用 GPGPU 实现软件渲染的理念已多次被提出。英伟达公司就曾提出并研究了一种基于 CUDA 的高效渲染模型[^6]，其性能可达硬件图形管线的2至8倍。

近年来，游戏行业的主要公司再次认识到了 CPU 的潜力[^11]。他们的游戏引擎可以将特定的可视化任务委派给 CPU。在游戏 Battlefield 3 中，开发了一种基于 SPU 的延迟着色模型[^12]，其目标是利用 SPU 来分配着色工作并减轻 GPU 的负担。2011年，游戏 Killzone 3 支持复杂的被遮挡环境。为了在帧的早期剔除不可见几何体，该游戏利用 PlayStation 3 的 SPU 来光栅化一个保守深度缓冲（conservative depth buffer），并针对它执行快速的同步遮挡查询[^16]。软件遮挡剔除的主题也已被 Intel Software 在文献 [^18] 中研究过。该文献提出了一种类似 Killzone 的解决方案，但它是建立在 x86 架构基础之上，并针对 SSE 流式扩展进行了优化。

因此，正如最新研究表明，基于 CPU 的方法重新受到重视，旨在提升灵活性与性能。本文重点研究三角形遍历与填充算法的优化与扩展方法。

#### 3 **光栅化基础**

计算机可视化的目标是在屏幕上显示像素集合（例如二维图像或投影的三维物体）。渲染算法的类型或可呈现元素的处理流程，很大程度上取决于所采用的基于硬件或软件的可视化模型。光栅化是计算密集型的过程，当视觉元素包含 alpha 通道时尤为显著[^7][^21]。

尽管已发展出多种内存中表示形状的方法，但当今最主流且广泛应用的物体表示形式是多边形网格。在建模过程中，物体通常被分解为凸多边形（如三角形）。然而，渲染性能在很大程度上依赖于所采用的光栅化算法。虽然已开发出多种不同解决方案（光线追踪、立体渲染等），当前 GPU 制造商在实时可视化中普遍采用基于三角形填充的模型——该方法相比基于光线的算法能实现显著更快的渲染速度。

#### 3.1 **基于三角形的填充**

在传统意义上，填充操作需逐像素执行，因此内部迭代及各类计算需反复运行。尽管顶点映射与遍历过程看似简单，但填充性能很大程度上取决于所实现的算法及其优化程度。可以说，开发一种既充分考虑现代 CPU 硬件特性又高度优化的算法面临巨大挑战。填充模型的各组件以复杂方式相互影响，迭代逻辑的微小改动甚至可导致超过 10% 的性能差异。

当前存在两种主流三角形填充算法：扫描线算法与基于半空间的算法。经典扫描线算法的核心思想是自上而下逐行（扫描线）遍历三角形。每一行代表一条线段，其起点和终点由三角形边与扫描线沿 x 轴的交点确定。利用边的斜率值可通过增量方式计算线段端点。

扫描线算法虽应用广泛且可优化（如 s-buffer 技术），但难以适配现代硬件特性。其逐行光栅化模式对软硬件实现均存在多重弊端。其中一个问题是算法在 x 和 y 方向上不对称。对于细长三角形，水平和垂直方向的性能可能差异显著。外部扫描线循环是串行的，不利于硬件实现。负责扫描线迭代的内部循环因行长度不同而SIMD友好性不足。这导致算法具有方向依赖性。若因某些原因（如 Mipmapping, Multisampling）需同时处理多行，计算会进一步复杂化。总之，该方案难以应用于行和像素的并行处理。更好的解决方案是以2x2块（四边形）处理像素，预期将显著提升性能。

下文将聚焦基于半空间的三角形遍历方案，阐述其基础算法及深度优化策略，这些优化可大幅提升性能。

#### 4 **半空间光栅化**

该模型的名称在文献中尚未统一：部分文献称其为三角形内点判定（point in a triangle）、包围盒（bounding box）或半平面算法（half-plane algorithm）。该模型的基本思想源于多边形凸性理论：由 $n$ 条边构成的凸多边形内部区域，可定义为 $n$ 个半空间（half spaces）的交集。

对于三角形而言，三个半空间明确定义了其内部区域。
![](_page_5_Figure_2.jpeg)  
![](_page_5_Figure_4.jpeg)

多种数学方法可描述三角形内部像素，这些方法的核心都在于如何定义三角形边界。下文将介绍基于"垂直点积"（Perp Dot product）的模型。该运算本质上是三维叉积（cross-products）在二维空间的对应概念，通过计算一个向量与另一向量的垂直向量的点积获得。此垂直向量基于原向量旋转90度得到，且保持模长不变。二维向量 $a$ 与 $b$ 的垂直点积公式定义如下：  

$$\begin{align}a^2+b^2=c^2\tag{1.1}\end{align}$$

$$perpDotP(a,b) = a^{\perp} \cdot b = a_x b_y - a_y b_x = \begin{vmatrix} a_x & a_y \\ b_x & b_y \end{vmatrix}.$$
 (1)

The result is a scalar, which has specific properties:

- $perpDotP(a, b) = 0 \rightarrow a, b$  are parallel
- $perpDotP(a, b) > 0 \rightarrow b$  is counterclockwise from a
- $perpDotP(a, b) < 0 \rightarrow a$  is counterclockwise from b  $\bullet$

These properties are useful to determine whether a pixel is inside the triangle or not. In the case of a P point, the product needs to be calculated with all the three edges and to check its sign,

$$c_1 = perpDotP(a, p), c_2 = perpDotP(b, p), c_3 = perpDotP(c, p).$$

The final point-triangle containment relation depends on the prior knowledge of the triangle:

- $\bullet$ if the vertices A, B, C are given in clockwise order, then P is inside the triangle if  $c_1 > 0$  &&  $c_2 > 0$  &&  $c_3 > 0$
- if the vertices A, B, C are given in counter-clockwise order, then P is inside the triangle if  $c_1 < 0$  &&  $c_2 < 0$  &&  $c_3 < 0$
- if the order of the vertices A, B, C is unknown, then P is inside the  $\bullet$ triangle if

 $(c_1 > 0 \&\& c_2 > 0 \&\& c_3 > 0 \mid c_1 < 0 \&\& c_2 < 0 \&\& c_3 < 0)$ 

#### 4.1 **The Simple Filling Approach**

Based on the equations above, a simple filling algorithm can be formulated. First of all a bounded area is needed to specify which set of pixels is required to travel. We can use the area of the render target (e.g. screen buffer), but it requires a lot of unnecessary iterations. A better solution is if we use the axis aligned bounding box of the triangle. The pseudo code of the algorithm:

```
<pre>Calculate triangle bounding box (minX,minY,maxX,maxY);</pre>
<pre>Clip box against render target bounds(minX,minY,maxX,maxY);</pre>
Loop i=minX to maxX
  Loop j=minY to maxY
        P = P (i, j);c1 = perpDotProduct (\overline{AC}, \overline{AP});
        c2 = perpDotProduct(\overline{BC}, \overline{BP});
        c3 = perpDotProduct (\overline{CB}, \overline{CP});
        if (c1 >= 0 and c2 >= 0 and c3 >= 0)
            renderPixel(P);
   end
end
```

As the first step, the algorithm calculates the axis-aligned bounding box of the triangle and performs the clipping according to the bounds of the render target. In the second part of the process, all the points of the bounding box are crawled. Applying the above Perp Dot product formula the pixel-triangle containment relation can be determined.

Firstly, it is important to emphasize that this model is simple and easy to understand. Due to the bounding box the lines have the same length. Therefore. this approach is more SIMD friendly than the classical scanline algorithm. Taking advantage of the symmetry of the lines the model is much more parallel-friendly, the block or tile-based approach can be used more effectively. The processing logic is highly customizable for the CPUs used currently.

Although the basic algorithm seems simple, the model is not recommended in practice for several reasons (slow performance, lack of sub-pixel precision and filling rules). The main disadvantage is that the solution also travels unnecessary pixels, which means a lot of superfluous iterations in case of large and elongated triangles. The performance at these types of triangles will be significantly lower.

In the following, new approaches and optimizations will be presented, which dramatically improve the performance of the basic algorithm making it suitable for real-time games and other graphics-intensive applications.

#### 5 **Optimization Considerations**

Developing a fast rasterizer is a complex task, high-level and multi-layered (code and logic level) optimization is required. In order to achieve really good results, full graphics pipeline optimization should be performed. However, the present paper focuses on improving the rasterization performance. The rasterization process constitutes the dominant part of the total performance requirement of the image synthesis [21]. Thus, any kind of performance improvement (even eliminating a division) affects the final result significantly.

Optimization should be performed on the basis of Michael Abrash's idea: 'Assume nothing' [4]; any logical considerations and their results can only be justified by measurements; the assumptions themselves are not acceptable. During the optimization process two main goals can be set for the filling algorithm:

- The key issue is the acceleration of the calculations and iterations  $\bullet$ because the basic algorithm is not efficient in CPU time.
- A more effective solution is required to traverse the pixels of the  $\bullet$ bounding box

#### $$ **Incremental Approach**

One of the main problems of the simple filling model is that the pixel by pixel filling performed in the double iteration loop is extremely expensive computationally. To optimize these loops, let us start from the calculation of the Perp Dot product determinant. Based on the formula (1), the relationship of a point  $P$  and edge  $AB$  can be described as follows:

$$\begin{vmatrix} B_x - A_y & P_x - A_x \ B_y - A_y & P_y - A_y \end{vmatrix} = (B_x - A_x)(P_y - A_y) - (B_y - A_y)(P_x - A_x).$$
 (2)

Performing the multiplications and by rearranging the factors (2) can be written  $\mathbf{a}$ 

$$F_{01}(P) := (A_y - B_y)P_x + (B_x - A_x)P_y + (A_xB_y - A_yB_x).$$
 (3)

The resulting equation is called edge function. Examining the factors, if the vertex positions are constants, the function is linear for  $P$ . The other two edge functions applying the same transformation are the following:

$$F_{12}(P) := (B_y - C_y)P_x + (C_x - B_x)P_y + (B_x C_y - B_y C_x),$$
(4)

$$F_{20}(P) := (C_y - A_y)P_x + (A_x - C_x)P_y + (C_x A_y - C_y A_x).$$
 (5)

If a triangle is counter-clockwise, then every edge function is positive for  $P$ , and the point is inside the triangle.  $F_{01}$  equation can be simplified by introducing additional constant:

$$I_{01} := A_y - B_y,\tag{6}$$

$$J_{01} := B_x - A_x,\tag{7}$$

$$K_{01} := A_x B_y - A_y B_x. \tag{8}$$

By substituting:

$$F_{01}(P_x, P_y) = I_{01}P_x + J_{01}P_y + K_{01}.$$
(9)

During the iteration we travel row by row: move one pixel to the right, then one pixel up or down. Because  $F_{01}$  is linear, axis-aligned unit steps can be calculated in both directions by using the introduced constants  $(6)$   $(7)$   $(8)$ :

$$F_{01}(P_x + 1, P_y) - F_{01}(P_x, P_y) = I_{01},$$
(10)

$$F_{01}(P_x, P_y + 1) - F_{01}(P_x, P_y) = J_{01}.$$
(11)

If we move one pixel to the right,  $I_{01}$  should be added to the edge function. If we move one pixel up or down then  $J_{01}$  is the additive tag. Following the above logic, the pseudo implementation of the iteration-based and modified filling algorithm is as follows:

```
Calculate triangle bounding box (minX, minY, maxX, maxY);
Clip box against render target bounds (minX, minY, maxX,
maxY);Const.: I_{01}, J_{01}, K_{01}, I_{02}, J_{02}, K_{02}, I_{03}, J_{03}, K_{03}, F_{01}, F_{02}, F_{03}Cy1=F01; Cy2=F02; Cy3=F03;
Loop j=minY to maxY step 1
      Cx1=Cy1; Cx2=Cy2; Cx3=Cy3;
      Loop i=minX to maxX step 1
              if (Cx1 > 0 and Cx2 > 0 and Cx3 > 0 )
                  RenderPixel(P);<pre>// Inner loop increments</pre>
              Cx1 -= I01; Cx2 -= I02; Cx3 -= I03;end
       <pre>// Outer loop increments</pre>
      Cy1 += J01; Cy2 += J02; Cy3 += J03;end
```

#### $5.2$ **A Block-oriented Half-Space Rasterization Model**

The performance requirement of rasterization is caused by travelling all the pixels of the bounding box because the box also contains pixels, which are outside the triangle. Therefore, it is expedient to propose an extension of the basic model, by which these pixels can be skipped. However, it is important to note that only those logical approaches can be applied which can be quickly evaluated and executed. Any operation executed in a double iteration loop and storing additional intermediate states has a negative impact on the performance. In such cases the cost of travelling unnecessary pixels will be less than the execution of control logic. In the following an efficient block-based traversal algorithm will be presented.

The unnecessary calculations can really be reduced, if the traversal process is performed on larger pixel sets. To achieve this, the triangle bounding box should be divided into squares which will be the basic units of the traversal logic. Figure 2 shows the essence of the block-based traversal method:

![](_page_9_Figure_4.jpeg)

Figure 2 Block-based triangle covering

The smallest unit of iteration is a block. The algorithm is performed from top to bottom, row by row and the top-left block can be chosen as the starting point of the traversal. This approach provides an opportunity for the pixel-triangle containment relation to be performed at a block level. It is sufficient to investigate only the four corners of the square. On this basis, there are three cases:

- every corner of the block is outside the triangle;  $1.$
- $2.$ the block overlaps the triangle;
- $\mathfrak{Z}$ the block is completely inside the triangle.

The first case is the most favourable because in this case there is no need for additional calculation on writing pixels into the framebuffer. The block can be skipped entirely. The second case is the worst, where the pixel-triangle containment calculation needs to be done for each pixel in the block, and according to the results the color of the pixel should be calculated. This part of the algorithm is essentially the same as the basic, pixel-level filling. Case 3 is also favourable for traversal. Since each pixel of the block falls within the triangle, there is no need to calculate the edge functions per pixel or do the containment verification. Only the color of the pixels should be determined. The iteration also allows an additional supplement. When the edge of a triangle is reached in a row,

we can move straight to the next row because only empty blocks can be found from this position.

This traversal logic is simple, the block-level iteration does not require complex calculations: only a state should be stored and an additional condition check is required. Practical experience shows that the extra computing capacity for executing the logic is slightly smaller than the cost of checking blocks to the end of the row. However, the same solution cannot be applied for improving performance of the pixel level processing inside a block type 2. Although it seems appropriate, it degrades the performance.

The following pseudo code shows the logic of the algorithm:

```
<pre>Calculate triangle bounding box (minX, minY, maxX, maxY);</pre>
<pre>Clip box against render target bounds (minX, minY, maxX, maxY);</pre>
Loop j=minY to maxY step=q
    Loop i=minX to maxX step=q
       <pre>// Block corners</pre>
       C1x=i; C1y=(i+q-1) C2y=j; C2x=(j+q-1);
       if C1x, C2x, C1v, C2v all outside the triangle then
          continue:
       \! // Fully covered blocks
       if C1x, C2x, C1y, C2y all inside the triangle then
           RenderBlock(i,j,q);
           continue;
       end
       // Partially covered blocks
       Loop k=j to q step=1
           Loop l=i to q step=1 \frac{1}{2}if pixel(k, 1) inside the triangle
                   RenderPixel(k,l):
           end
       endendend
```

The key element of the algorithm is the iteration block size. If the block size is too small, then the block-level calculations require more computing capacity than checking only simple pixels. If large block size is chosen, then there will be fewer triangles which fulfil cases 1 or 3. Therefore, the block-based traversal does not result in any performance improvement. Practical measurement experience shows that an 8x8 block size typically provides accurate results.

The efficiency of the process can significantly be affected by the orientation of the triangles. If the presented triangle mesh consists of large polygons, or the camera view is close to a polygon, then the vast majority of the blocks will be in the triangle (case 3). In this case, rasterization can be very fast, especially when applying a SIMD instruction set in performance critical parts. However, if narrow triangles are dominating, there will be no notable speed improvement because the number of overlapping blocks is increasing (case 2).

Investigating modern computer games, it can clearly be seen that mainly pixellevel effects are dominating (bump/normal/parallax mapping, etc.). Wherever possible, programmers apply screen space pixel transformations instead of increasing the number of polygons. Therefore, the block-based approach is well applicable.

#### $5.2.1$ **Benefits of the Block-based Model**

The advantage of this approach is that during the process a lot of pixels outside the triangle can be excluded from rasterization. The solution reduces the number of iterations in the bounding box, which can fundamentally be a performanceenhancing factor. Certain operations like perspective correct texture mapping or mipmapping can be performed in larger units applying linear interpolation. It is also favourable for visibility determination algorithms. In addition, a hidden advantage of the block-based approach is that it allows a more localized memory access and efficient CPU cache usage because data are located close each other. The reduction of cache misses represents a further performance improvement [10]. Finally, it should not be forgotten that the block-based model is much more parallel-friendly. Due to the distribution of operations to several threads, the block as a larger unit has important advantages.

#### $5.3$ **An Adaptive Half-Space Rasterization Model**

The presented pixel- and block-level models can be applied in any triangle orientation. However, their efficiency is unbalanced. The algorithm handling only pixels is less efficient in the case of large triangles and the performance of the block-based approach is not sufficient in the case of thin triangles. Another performance-enhancing factor could be, if an adaptive model was used in accordance with the characteristics of the triangle and changes were made in the applied rasterization method dynamically. The precondition for this is to determine the characteristic of the triangle.

It is expedient to define two groups: triangles, which are narrow and those which correspond to the block-oriented approach. The orientation can be defined by introducing a metric based on the bounding box of the triangle:

$$orient_{\Delta} = \frac{BB_{max} - BB_{minx}}{BB_{max} - BB_{miny}}$$
 (12)

The definition of the formula is intentionally simple because rasterization setup costs should be kept at a minimum level. The orientation is a positive number. Its value is 1 if the bounding box is a rectangle. If the width is dominant, the value of the ratio increases and if the height is dominant, the value will be reduced. We can state that triangles with a ratio close to one are ideal for the block-based model, if their size reaches at least one block along the  $x$  or  $y$  axis. The more we move away from this value the more advisable it is to use the pixel-based algorithm. On this basis, a range for switching between the two methods can be determined by experience. Experience shows that it is triangles with orient values of  $0.4 - 1.6$  that should be rasterized with the block-based approach, in all other cases the pixelbased algorithm is recommended.

The adaptive algorithm is especially advantageous since it is always the preferred model that is applied of the two. Therefore, the rasterization performance can be significantly increased.

#### $5.4$ **Problem of Empty Blocks**

Although the block-based approach significantly helps to travel the empty area of the bounding box faster, the number of traversed empty blocks is still significant and superfluous. The following example illustrates the problem well:

![](_page_12_Figure_6.jpeg)

Figure 3 Block-based covering with empty block coloring

In Figure 3, the orange color indicates the traversed empty blocks. Since the algorithm always starts the investigation of the rows from the left of the bounding box, it travels several empty blocks before reaching the edges. Although these calculations are computationally less intensive than reading or writing pixels, but in the case of a complex scene a significant amount of unnecessary calculations are involved. In the above example (Figure 3), there are 18 completely empty blocks of all the 84, which is 21% of all the blocks. To reduce the traversal of the empty area, several approaches have been developed, where the ZigZag [22] and Backtrack [7] are the most widely known algorithms. Both algorithms perform rasterization at pixel level and their main characteristic is that they exclude unnecessary traversal at only one side of the triangle. However, on the other side, when stepping to the next row, they are not able to skip every empty pixel in many cases. In addition, the traversal direction changes row by row, which requires storing more states and increasing the number of conditions (CPU branching can be slow if statements are unpredictable).

#### $5.5$ **A Block-based Bisector Half-Space Rasterization Model**

In the following a block-based algorithm is presented which aims to minimize the number of traversed empty blocks and thus to reduce unnecessary calculations. Figure 4 explains the essence of the traversal.

The basic idea is that the problem of empty-block minimization can effectively be solved if the traversal starts from a common point inside the triangle and moves in two directions. The filling is divided into two directions along the y axis of the bounding box and performed from top to bottom, bottom to top and to half of the hox.

![](_page_13_Figure_5.jpeg)

Figure 4 Triangle traversal logic of the Bisector algorithm

The bidirectional traversal at row level starts from the inside out until an empty block is reached. The starting block of the next row is calculated from the first two blocks found outside the edges at the previous row:

#### $Start$ block = Left bound + (Right bound - Left bound) / 2. $(14)$

It is important to note that, if the starting block does not fall on a block boundary, then rounding is also required. The division by two can significantly increase the load. Therefore, in implementations it is expedient to use fixed-point arithmetic because starting block can be calculated without division and with block boundary alignment:

$$Start block \leftarrow (Left + (Right - Left) \gg 1) \& \sim (q - 1). \tag{15}$$

The traversal to the right direction starts from this block and the left starts filling from the previous block. The first row is special because there is no information available from the previous row. In this case, the  $x$  coordinate of the topmost vertex aligned to block boundary can be used as a starting block because it is inside the triangle. The initial state cannot be affected if two vertices of the

triangle are located at the top of the bounding box. Selecting either of them provides accurate results because these points are located at the edges of the bounding box.

Because of the block nature of the algorithm, due to the nature of the triangle edges, the iteration to the left will not find any fillable block if only one block can be filled in the first row. In this case, the value of the 'Left bound' takes the boundary coordinates of the bounding box and based on  $(15)$  the next row will definitely start at an outside block until the traversal reaches the interior of the triangle along the y axis. This is a serious problem because it increases the traversed empty blocks. To remedy this, the algorithm requires a further extension: the value of the 'Left bound' should only be changed if any inside block is found during the left directed traversal. Otherwise, its position should not be updated, so the next row also starts from the block of the previous 'Left bound', from the x coordinate of the topmost vertex.

The next key element of the algorithm is the requirement of the bottom-up traversal. Figure 5 illustrates the basic problem performing only the top-bottom crawling.

![](_page_14_Figure_5.jpeg)

Figure 5 The inefficiency problem of the top-bottom only traversal

In cases when an edge of the triangle is very flat, the calculation of a new middle block at the next row will result in an outside block (last row in Figure 5). The best case for the traversal would be when the middle block arrives at the corner of the bottom edges, but it cannot be guaranteed because the middle point is defined by the previous row. Therefore, the number of traversed empty blocks can increase.

The solution is offered by the bottom-up traversal of the lower part of the triangle. As the starting block can exactly be determined from the x coordinate of the bottommost vertex at this time, the traversal of the empty blocks seen in Figure 5 is completely eliminable like in Figure 4.

Although the solution appears complex, it does not require any complicated calculations and the control logic is simple. The algorithm effectively minimizes

the traversal of the empty blocks. The pseudo-code of the full algorithm is as follows:

```
Calculate triangle bounding box (minX, minY, maxX, maxY);
Clip box against render target bounds (minX, minY, maxX, maxY);
Find Topmost vertex, calculate leftPoint and rightPoint
halfY = minY + ((maxY - minY) >> 1) & \sim (q - 1);BlockSize = q;Loop j=minY to (halfY + BlockSize) step=BlockSize
   midPoint = leftPoint + ((rightPoint-leftPoint) >> 1) & ~ (BlockSize - 1);x = midpoint;Loop k=0 to 2 step=1
      Loop x to (q > 0 ? x < maxX : x > minX - BlockSize) step=q
          <pre>// Block corners</pre>
          C1x=x; C1y=(x+BlockSize-1) C2y=j; C2x=(j+BlockSize-1);
          if C1x, C2x, C1y, C2y all outside the triangle then
             continue;
          \! // Fully covered blocks
          if C1x, C2x, C1y, C2y all inside the triangle then
            RenderBlock(x, j, BlockSize);
            continue;
          endRenderPartiallyCoveredBlock(j,x, BlockSize);
       end
       q = -q;if k==0 then rightPoint = x - BlockSize;
      else leftPoint = x + BlockSize;
      x = midpoint - BlockSize;\hspace{15pt}\texttt{end}end
Find Bottommost vertex, calculate leftPoint and rightPoint
Loop j=maxY &~(BlockSize-1) to halfY step=-BlockSize
Repeat the above part
end
```

#### **Practical Experience and Results** 6

To evaluate the above rasterization models from a practical point of view, a single-threaded pipeline architecture was developed. Although it is possible to apply distributed approaches in several parts of the pipeline, this paper only focuses on accelerating single-threaded pixel operations.

#### $6.1$ **The Test Environment**

The sample programs were written in  $C++$  applying the GCC 4.8.1 compiler and the measurements were performed by an Intel Core i7-870 2.93 GHz CPU in a 64 bit Windows 7 environment. The implemented pipeline does not contain any hand-optimized SIMD code parts, only the compiler-optimized code was applied. The chosen screen resolution and color depth were 1024x768x32 in windowed mode and the used hardware for the test was an ATI Radeon HD 5670 with 1 GB of RAM. The software framebuffer was defined as a 32 bit unsigned integer array aligned to 16 bytes. This made an effective pixel handling possible, which stored the four components of a pixel together  $[21]$  to make memory operations faster. The prototype application used a software z-buffer and backface culling to solve visible surface determination, but texture mapping had not been implemented yet. To illuminate objects, the Lambertian reflectance was applied and rasterization used a top-left convention for filling.

#### $6.2$ **Benchmark Results**

During benchmarking, several different test cases were prepared. Each of them represented a special group of tasks frequently occurring in practice. The measured results are cumulated values calculated from the average frame rates (FPS) during a 20 sec running period. The distance of the models from the camera affects the performance largely. When an object is farther away, usually many small triangles should be drawn. However, getting closer to the camera, the projection of the polygons will be larger and the number of fillable pixels is increasing. Test Cases for benchmark:

Case 1: low poly model located farther from the camera.

Case 2: low poly model located close to the camera. The polygons cover about the 80% of screen pixels.

Case 3: high poly model built from small triangles (head) located farther from the camera

Case 4: high poly model built from small triangles (head) located close to the camera. The polygons cover the entire screen.

Case 5: medium poly model (statue) cover about the 80% of screen pixels. The model contains both small and large triangles.

Table 1 summarizes the results achieved by different types of rasterization algorithms. The test results show that the basic half-space algorithm is proved to be the slowest in every case as expected. Through its simplicity, it does not contain any optimizations. The incremental based approach can be regarded as a transition, its performance converges to the other, better solutions. If we examine the block-based model, it can be seen that during the tests C3 and C4 its performance was not satisfactory. The main reason for this is the nature of the scenes.

![](_page_17_Picture_2.jpeg)

Figure 6 Part of the sample models: head (196.602 triangles), statue (95.346 triangles)

| Table 1           |  |
|-------------------|--|
| Benchmark results |  |

|                          | <b>Benchmarks (FPS)</b>              |                           |                                  |                   |                                              |  |
|--------------------------|--------------------------------------|---------------------------|----------------------------------|-------------------|----------------------------------------------|--|
| Test                     | <b>Half-space algorithm variants</b> |                           |                                  |                   |                                              |  |
| cases                    | Simple<br>rasterizer                 | Incremental<br>rasterizer | <b>Block-based</b><br>rasterizer | Adaptive<br>model | Adaptive and<br><b>Bisector</b><br>algorithm |  |
| $\rm C~1$                | 112                                  | 294                       | 456                              | $506$             | 517                                          |  |
| C2                       | 58                                   | 165                       | 552                              | 552               | 564                                          |  |
| C3                       | 60                                   | 67                        | 51                               | 67                | 69                                           |  |
| C4                       | $23$                                 | 37                        | 35                               | $40$              | $45$                                         |  |
| $\mathbf{C}\;\mathbf{5}$ | 69                                   | 97                        | 106                              | 125               | 132                                          |  |

These tests used high poly models built up from small triangles, which cannot be managed effectively by the block-based algorithm (the number of partially covered blocks increase). The adaptive model variant proved its efficiency in all cases. The logic of its triangle-orientation-based decision was able to maintain the high level performance. The fastest solution is achieved by the combination of adaptive and bisector algorithms. This made it also possible to take advantage of the reduction of empty blocks.

### **Conclusion**

Although present day rasterization is almost exclusively performed by GPUs, we cannot forget the opportunities offered by modern CPUs. It should be recognized that certain functions can and should also be shared between GPU and CPU in order to make a more effective and robust rasterization model. To bring the two sides more closely together, this paper highlighted the basic problems of visualization. We can see that developing a fast and effective rendering model is not trivial, there are many difficulties. The authors presented some new variants of the classic half-space triangle rasterization model, which fits much better with modern CPUs and can be a good basis for developing a more complex rasterizer, for example a hybrid pipeline between GPU and CPU. In the future, it is expected that many applications will be released using a similar technology.

## **Acknowledgements**

This research was carried out as part of the TAMOP-4.2.1.B-10/2/KONV-2010-0001 project with support by the European Union, co-financed by the European Social Fund.

## 参考资料

[^1]: Bethel, Z.: A Modern Approach to Software Rasterization. University Workshop, Taylor University, December 14, 2011
[^2]: TransGaming Inc: Why the Future of 3D Graphics is in Software, White Paper: Swiftshader technology, Jan 29, 2013
[^3]: Microsoft Corporation: Windows Advanced Rasterization Platform (WARP) guide. 2012
[^4]: Abrash, M.: Rasterization on larrabee. Dr. Dobbs Portal, 2009
[^5]: Seiler, L., Carmean, D., Sprangle, E., Forsyth, T., Abrash, M., Dubey, P., Junkins, S., Lake, A., Sugerman, J., Cavin, R., Espasa, R., Grochowski, E., Juan, T., Hanrahan, P.: Larrabee: a Many-Core x86 Architecture for Visual Computing. ACM Transactions on Graphics (TOG) - Proceedings of ACM SIGGRAPH 2008 Volume 27 Issue 3, August 2008
[^6]: Laine, S., Karras T.: High-Performance Software Rasterization on GPUs. High Performance Graphics, Vancouver, Canada, August 5, 2011
[^7]: Akenine-möller, T., haines, E.: Real-Time Rendering, A. K. Peters. 3rd Edition, 2008
[^8]: Sugerman, J., Fatahalian, K., Boulos, S., Akeley, K., and Hanrahan, P.: Gramps: A Programming Model for Graphics Pipelines. ACM Trans. Graph. 28, 4:1–4:11, 2009
[^9]: Fang, L., Mengcheng H., Xuehui L., Enhua W.: FreePipe: A Programmable, Parallel Rendering Architecture for Efficient Multi-Fragment Effects. In Proceedings of ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games, 2010
[^10]: Agner, F.: Optimizing Software in C++ An Optimization Guide for Windows, Linux and Mac platforms. Study at Copenhagen University College of Engineering, 2014
[^11]: Swenney, T.: The End of the GPU Roadmap. Proceedings of the Conference on High Performance Graphics, 2009, pp. 45-52
[^12]: Coffin, C.: SPU-based Deferred Shading for Battlefield 3 on Playstation 3. Game Developer Conference Presentation, March 8, 2011
[^13]: Xuzhi W., Feng G., Mengyao, Z.: A More Efficient Triangle Rasterization Algorithm Implemented in FPGA, Audio, Language and Image Processing (ICALIP), July 16-18, 2012
[^14]: McCormack J., McNamara R.: Tiled Polygon Traversal Using Half-Plane Edge Functions. HWWS '00 Proceedings of the  ACM SIGGRAPH/EUROGRAPHICS workshop on Graphics hardware, 2000, pp. 15-21
[^15]: Chih-Hao, S., You-Ming, T., Ka-Hang, L., Shao-Yi, C.: Universal Rasterizer with Edge Equations and Tile-Scan Triangle Traversal Algorithm for Graphics Processing Units, In proceeding of: Proceedings of the 2009 IEEE International Conference on Multimedia and Expo, 2009
[^16]: Valient, M.: Practical Occlusion Culling in Killzone 3, Siggraph 2011, Vancouver, Oct. 14, 2011
[^17]: Olano, M., Trey, G.: Triangle Scan Conversion Using 2D Homogeneous Coordinates, Proceedings of the 1997 SIGGRAPH/Eurographics Workshop on Graphics Hardware, ACM SIGGRAPH, New York, August 2-4, 1997
[^18]: Chandrasekaran, C.: Software Occlusion Culling, Intel Developer Zone, Jan 14, 2013
[^19]: Leone, M., Barbagallo, L.: Implementing Software Occlusion Culling for Real-Time Applications, XVIII Argentine Congress on Computer Sciences, Oct. 9., 2012
[^20]: Hill, F. S. Jr.: The Pleasures of 'Perp Dot' Products. Chapter II.5 in Graphics Gems IV (Ed. P. S. Heckbert) San Diego: Academic Press, 1994, pp. 138-148
[^21]: Mileff, P., Dudra, J.: Advanced 2D Rasterization on Modern CPUs, Applied Information Science, Engineering and Technology: Selected Topics from the Field of Production Information Engineering and IT for Manufacturing: Theory and Practice, Series: Topics in Intelligent Engineering and Informatics, Vol. 7, Chapter 5, Springer International publishing, 2014, pp. 63-79
[^22]: Royer, P., Ituero, P., Lopez-Vallejo, M., Barrio, Carlos A. L.: Implementation Tradeoffs of Triangle Traversal Algorithms for Graphics Processing, Design of Circuits and Integrated Systems (DCIS), Madrid, Spain; November 26-28, 2014










