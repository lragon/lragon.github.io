---
title: "译:如何制作牛逼的闪电特效2d"
date: 2014-04-13 22:33:23 +0800
comments: true
categories: 
---
这是我第一次翻译教程，翻译不好的地方请见谅，观看视频需备梯子。在此**声明**，此翻译原稿来自于互联网，供学习交流之用，请勿进行商业传播。同时，转载时不要移除本声明。如产生任何纠纷，与本博客所有人、发表该翻译稿之人无任何关系。谢谢合作！

翻译:ryan li

原文地址：[http://gamedevelopment.tutsplus.com/tutorials/how-to-generate-shockingly-good-2d-lightning-effects&mdash;gamedev-2681](http://gamedevelopment.tutsplus.com/tutorials/how-to-generate-shockingly-good-2d-lightning-effects--gamedev-2681)

闪电特效在游戏中有很大的用处，从烘托暴风雨的背景气氛到魔法师的闪电攻击。这篇教程里，作者会说明如何用程序生成很酷的2d闪电效果（2D lightning effects）:闪电链(bolts)，分支（branches）,甚至文字（text）

<!-- more -->

> **注意**：此教程虽是用c#和XNA写的，但是读者应该能把这里的技术和观念用到任何游戏开发环境中。

最终效果

<iframe width="560" height="315" src="//www.youtube.com/embed/FmLs8_-RHQA" frameborder="0" allowfullscreen></iframe>

### 1.画一个发光的线条

组成闪电的基本结构是一条线段。先打开你擅长的绘画软件，画一小段笔直的闪电。这是作者画的样子

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/24/Line%20Segment%20Example.png)

我们想要画不同的长度，所以要把这一小段闪电像下面这样截成三段。这样就能把中间的部分伸缩成任意想要的长度。因为中间的部分会被伸缩，所以截成1像素宽就行了。左右两边其实是一对镜像图片，保存一张就够了，我们可以用代码翻转它来表示另一张，这样一共就有中间部分和半圆两张图片。

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/24/Divided%20Line%20Segment%20Example.png)

现在我们声明一个类来处理线段：
	
    public class Line
    {
        public Vector2 A;
        public Vector2 B;
        public float Thickness;

        public Line() {}
        public Line(Vector2 a, Vector2 b, float thickness = 1)
        {
            A = a;
            B = b;
            Thickness = thickness;
        }
    }
    

A和B是线段的端点。通过缩放和旋转每条线段，我们能画出任何宽度，长度和旋转的线。然后给`Line`添加`Draw()`函数:

    public void Draw(SpriteBatch spriteBatch, Color color)
    {
        Vector2 tangent = B - A;
        float rotation = (float)Math.Atan2(tangent.Y, tangent.X);

        const float ImageThickness = 8;
        float thicknessScale = Thickness / ImageThickness;

        Vector2 capOrigin = new Vector2(Art.HalfCircle.Width, Art.HalfCircle.Height / 2f);
        Vector2 middleOrigin = new Vector2(0, Art.LightningSegment.Height / 2f);
        Vector2 middleScale = new Vector2(tangent.Length(), thicknessScale);

        spriteBatch.Draw(Art.LightningSegment, A, null, color, rotation, middleOrigin, middleScale, SpriteEffects.None, 0f);
        spriteBatch.Draw(Art.HalfCircle, A, null, color, rotation, capOrigin, thicknessScale, SpriteEffects.None, 0f);
        spriteBatch.Draw(Art.HalfCircle, B, null, color, rotation + MathHelper.Pi, capOrigin, thicknessScale, SpriteEffects.None, 0f);
    }
    

这里的`Art.LightningSegment`和`Art.HalfCircle`是静态`Texture2D`变量，分别是之前截出来那两张图片的中间部分和半圆。`ImageThickness`用来指定不发光部分的线段宽度，作者上面的图片是8像素。这里要把半圆和中间部分的位置计算好，好让它们无缝连起来。

XNA的`SpriteBatch`类的构造方法有一个`SpriteSortMode`参数，该参数表示绘制顺序为纹理模式(译者不懂XNA，这里是自己猜的)。画线段的时候，要确保传过来的`SpriteBatch`的`SpriteSortMode`是设置成`SpriteSortMode.Texture`的。这有助于提升程序的性能。

显卡很擅长重复绘制同一纹理，但是切换纹理有一定的开销，如果不设置绘制顺序，可能实际是这样绘制的：

LightningSegment, HalfCircle, HalfCircle, LightningSegment, HalfCircle, HalfCircle, &hellip;

这意味着绘制一条线段需要切换两次纹理，`SpriteSortMode.Texture`告诉`SpriteBatch`调用`Draw()`的时候要排序，这样所有的`LightningSegments`会被一起绘制，所有的`HalfCircles`也会被一起绘制。此外，我们用上面的代码来绘制闪电时，可以使用叠加混合模式让闪电重叠部分的光开起来更亮。

    SpriteBatch.Begin(SpriteSortMode.Texture, BlendState.Additive);
    // draw lines
    SpriteBatch.End();
    

### 2.锯齿型的线条

闪电一般会形成锯齿形的线条，所以我们需要算法来产生它。具体的方法是在一条线段上选出一些点，这些点将会被随机放置到一定距离内的另一个位置，然后将这些点连起来。我们可以控制点的数量和随机距离的范围，来调节产生的效果。

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/24/Making%20Jagged%20Lines.png)

这条线段做了平滑处理，使用上一个点稍微偏移的位置作为下一个点的位置，这能让线条整体有些起伏，看上去稍微平滑一些，不会那么的崎岖（找不到更好的词了）。代码如下所示：

    protected static List&lt;Line&gt; CreateBolt(Vector2 source, Vector2 dest, float thickness)
    {
        var results = new List&lt;Line&gt;();
        Vector2 tangent = dest - source;
        Vector2 normal = Vector2.Normalize(new Vector2(tangent.Y, -tangent.X));
        float length = tangent.Length();

        List&lt;float&gt; positions = new List&lt;float&gt;();
        positions.Add(0);

        for (int i = 0; i &lt; length / 4; i++)
            positions.Add(Rand(0, 1));

        positions.Sort();

        const float Sway = 80;
        const float Jaggedness = 1 / Sway;

        Vector2 prevPoint = source;
        float prevDisplacement = 0;
        for (int i = 1; i &lt; positions.Count; i++)
        {
            float pos = positions[i];

            // used to prevent sharp angles by ensuring very close positions also have small perpendicular variation.
            float scale = (length * Jaggedness) * (pos - positions[i - 1]);

            // defines an envelope. Points near the middle of the bolt can be further from the central line.
            float envelope = pos &gt; 0.95f ? 20 * (1 - pos) : 1;

            float displacement = Rand(-Sway, Sway);
            displacement -= (displacement - prevDisplacement) * (1 - scale);
            displacement *= envelope;

            Vector2 point = source + pos * tangent + displacement * normal;
            results.Add(new Line(prevPoint, point, thickness));
            prevPoint = point;
            prevDisplacement = displacement;
        }

        results.Add(new Line(prevPoint, dest, thickness));

        return results;
    }
    

这些代码可能有点吓人，但是当你明白它的逻辑之后会发现没那么糟糕。首先计算出线段的法向量(normal vector)和切向量(tangent vector)，以及长度。然后在这条线段上随机取一些点保存到`positions`中，这些`positions`的范围在`0`到`1`之间，`0`表示起点，`1`表示终点。接下来对`positions`排序方便下面使用。

循环上面随机出来点，随机重置它们的位置。`scale`是为了避免产生太尖锐的角度，`envelope`会让线段中间的点偏离更大，靠近两端的点偏离更小。

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/25/Lightning%20Bolt.jpg)

### 3.动画

闪电会放出耀眼的光然后淡出，为了实现这个，我们创建一个`LightningBolt`类

    class LightningBolt
    {
        public List&lt;Line&gt; Segments = new List&lt;Line&gt;();

        public float Alpha { get; set; }
        public float FadeOutRate { get; set; }
        public Color Tint { get; set; }

        public bool IsComplete { get { return Alpha &lt;= 0; } }

        public LightningBolt(Vector2 source, Vector2 dest) : this(source, dest, new Color(0.9f, 0.8f, 1f)) { }

        public LightningBolt(Vector2 source, Vector2 dest, Color color)
        {
            Segments = CreateBolt(source, dest, 2);

            Tint = color;
            Alpha = 1f;
            FadeOutRate = 0.03f;
        }

        public void Draw(SpriteBatch spriteBatch)
        {
            if (Alpha &lt;= 0)
                return;

            foreach (var segment in Segments)
                segment.Draw(spriteBatch, Tint * (Alpha * 0.6f));
        }

        public virtual void Update()
        {
            Alpha -= FadeOutRate;
        }

        protected static List&lt;Line&gt; CreateBolt(Vector2 source, Vector2 dest, float thickness)
        {
            // ...
        }

        // ...
    }
    

用法是先创建一个`LightningBolt`然后在每帧调用`Update()`和`Draw()`方法，`Update()`的作用是淡出，`IsComplete`会告诉你闪电是否完全淡出了。

现在你能在游戏中使用下面的代码来画一个闪电了！

    LightningBolt bolt;
    MouseState mouseState, lastMouseState;

    protected override void Update(GameTime gameTime)
    {
        lastMouseState = mouseState;
        mouseState = Mouse.GetState();

        var screenSize = new Vector2(GraphicsDevice.Viewport.Width, GraphicsDevice.Viewport.Height);
        var mousePosition = new Vector2(mouseState.X, mouseState.Y);

        if (MouseWasClicked())
            bolt = new LightningBolt(screenSize / 2, mousePosition);

        if (bolt != null)
            bolt.Update();
    }

    private bool MouseWasClicked()
    {
        return mouseState.LeftButton == ButtonState.Pressed &amp;&amp; lastMouseState.LeftButton == ButtonState.Released;
    }

    protected override void Draw(GameTime gameTime)
    {
        GraphicsDevice.Clear(Color.Black);

        spriteBatch.Begin(SpriteSortMode.Texture, BlendState.Additive);

        if (bolt != null)
            bolt.Draw(spriteBatch);

        spriteBatch.End();
    }
    

### 4.闪电分支

你可以用`LightningBolt`类为基础产生更多有趣的闪电效果，举个例子，可以给闪电加一些分支，如下图：

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/25/Branch%20Lightning.jpg)

为了生成分支，我们在原来的闪电链上随机选一些点在这些点上添加新的闪电链，下面的代码中，我们从主链上创建了三到六个30度的分支。

    class BranchLightning
    {
        List&lt;LightningBolt&gt; bolts = new List&lt;LightningBolt&gt;();

        public bool IsComplete { get { return bolts.Count == 0; } }
        public Vector2 End { get; private set; }
        private Vector2 direction;

        static Random rand = new Random();

        public BranchLightning(Vector2 start, Vector2 end)
        {
            End = end;
            direction = Vector2.Normalize(end - start);
            Create(start, end);
        }

        public void Update()
        {
            bolts = bolts.Where(x =&gt; !x.IsComplete).ToList();
            foreach (var bolt in bolts)
                bolt.Update();
        }

        public void Draw(SpriteBatch spriteBatch)
        {
            foreach (var bolt in bolts)
                bolt.Draw(spriteBatch);
        }

        private void Create(Vector2 start, Vector2 end)
        {
            var mainBolt = new LightningBolt(start, end);
            bolts.Add(mainBolt);

            int numBranches = rand.Next(3, 6);
            Vector2 diff = end - start;

            // pick a bunch of random points between 0 and 1 and sort them
            float[] branchPoints = Enumerable.Range(0, numBranches)
                .Select(x =&gt; Rand(0, 1f))
                .OrderBy(x =&gt; x).ToArray();

            for (int i = 0; i &lt; branchPoints.Length; i++)
            {
                // Bolt.GetPoint() gets the position of the lightning bolt at specified fraction (0 = start of bolt, 1 = end)
                Vector2 boltStart = mainBolt.GetPoint(branchPoints[i]);

                // rotate 30 degrees. Alternate between rotating left and right.
                Quaternion rot = Quaternion.CreateFromAxisAngle(Vector3.UnitZ, MathHelper.ToRadians(30 * ((i &amp; 1) == 0 ? 1 : -1)));
                Vector2 boltEnd = Vector2.Transform(diff * (1 - branchPoints[i]), rot) + boltStart;
                bolts.Add(new LightningBolt(boltStart, boltEnd));
            }
        }

        static float Rand(float min, float max)
        {
            return (float)rand.NextDouble() * (max - min) + min;
        }
    }
    

### 5.闪电文字

下面的视频是创造出的另一种效果

<iframe width="560" height="315" src="//www.youtube.com/embed/JcG1tVTRsBc" frameborder="0" allowfullscreen></iframe>

首先我们需要得到文字的像素数据，我们可以把文字绘制到`RenderTarget2D`，使用`RenderTarget2D.GetData&lt;T&gt;()`读取像素数据。如果你想了解更多的文字粒子特效，作者[这里有一篇](http://nullcandy.com/particle-text-in-xna-and-javascript/)更详细的教程。

我们把文字的像素坐标点存储成`List&lt;Vector2&gt;`，然后在每一帧里，在这些点中随机选一对点创建闪电链。我们想这样设计它，两个点越接近，越有机会产生一个闪电链。有一个小技巧来实现它：我们先随机选择一个点作为起点，然后在一些固定数量的随机点中选一个最接近的作为终点。

候选点的数量会影响最终的效果；使用较大的数更容易找到更近的点，这样文字会显得整齐清晰，但是字符之间的长闪电链会很少。使用小一点的数会使文字看上去很夸张而且不清晰。

    public void Update()
    {
        foreach (var particle in textParticles)
        {
            float x = particle.X / 500f;
            if (rand.Next(50) == 0)
            {
                Vector2 nearestParticle = Vector2.Zero;
                float nearestDist = float.MaxValue;
                for (int i = 0; i &lt; 50; i++)
                {
                    var other = textParticles[rand.Next(textParticles.Count)];
                    var dist = Vector2.DistanceSquared(particle, other);

                    if (dist &lt; nearestDist &amp;&amp; dist &gt; 10 * 10)
                    {
                        nearestDist = dist;
                        nearestParticle = other;
                    }
                }

                if (nearestDist &lt; 200 * 200 &amp;&amp; nearestDist &gt; 10 * 10)
                    bolts.Add(new LightningBolt(particle, nearestParticle, Color.White));
            }
        }

        for (int i = bolts.Count - 1; i &gt;= 0; i--)
        {
            bolts[i].Update();

            if (bolts[i].IsComplete)
                bolts.RemoveAt(i);
        }
    }
    

### 6.优化

上面的闪电文字可能会很平滑的运行，如果你有台配置较好的电脑的话，但是会相当吃力。每条闪电链持续30帧，每帧都会创建许多闪电链。每条闪电链可能有上百条小段，每条小段有三小块，绘制这么多精灵有点吃不消。我的例子中每帧绘制了25,000多个图片，这是没优化的情况，经过优化会好很多。

之前是一直绘制每一条闪电链直到它们消失，现在我们在每帧都创建新的闪电链并且这一帧结束时让它消失，这意味着不会再把一条闪电链绘制30或者更多帧了，现在只绘制一帧，这样就没有闪电链逐渐淡出或者持续更长时间的开销了。

首先，我们修改`LightningText`类让它把每条闪电链只绘制一帧，在你的游戏代码中，声明两个`RenderTarget2D`变量：`currentFrame`和`lastFrame`，在`LoadContent()`中，这样初始化它们：

    lastFrame    = new RenderTarget2D(GraphicsDevice, screenSize.X, screenSize.Y, false, SurfaceFormat.HdrBlendable, DepthFormat.None);
    currentFrame = new RenderTarget2D(GraphicsDevice, screenSize.X, screenSize.Y, false, SurfaceFormat.HdrBlendable, DepthFormat.None);
    

注意外观格式(surface format)设置成`HdrBlendable`，HDR代表[High Dynamic Range](http://en.wikipedia.org/wiki/High_dynamic_range_imaging)，它表示我们的HDR外观能表现更多范围的颜色。这是有必要的，因为它允许渲染目标有比白色更明亮的颜色。当多个闪电链重叠时我们需要渲染目标能存储它们颜色叠加后的值，这可能超过标准的颜色范围。当然这些比白色更亮的颜色在屏幕上仍然会显示成白色，但是存储他们的亮度才能正确的淡出它们。

> XNA提示：为了启用HDR混合模式，必须把XNA工程设置成Hi-Def，在解决方案资源管理器中右键点击工程，选择属性(properties)，然后在XNA工作室(XNA Game Studio)标签下选择hi-def。

每一帧，我们先把上一帧的内容绘制到当前帧，幷让它稍微暗一些，再把新的闪电链添加到当前帧，最后把当前帧渲染到屏幕上，交换两个渲染目标为下一帧做准备，`lastFrame`会指向刚刚渲染的那帧。

    void DrawLightningText()
    {
        GraphicsDevice.SetRenderTarget(currentFrame);
        GraphicsDevice.Clear(Color.Black);

        // draw the last frame at 96% brightness
        spriteBatch.Begin(0, BlendState.Opaque, SamplerState.PointClamp, null, null);
        spriteBatch.Draw(lastFrame, Vector2.Zero, Color.White * 0.96f);
        spriteBatch.End();

        // draw new bolts with additive blending
        spriteBatch.Begin(SpriteSortMode.Texture, BlendState.Additive);
        lightningText.Draw();
        spriteBatch.End();

        // draw the whole thing to the backbuffer
        GraphicsDevice.SetRenderTarget(null);
        spriteBatch.Begin(0, BlendState.Opaque, SamplerState.PointClamp, null, null);
        spriteBatch.Draw(currentFrame, Vector2.Zero, Color.White);
        spriteBatch.End();

        Swap(ref currentFrame, ref lastFrame);
    }

    void Swap&lt;T&gt;(ref T a, ref T b)
    {
        T temp = a;
        a = b;
        b = temp;
    }
    

### 7.其他变种

我们已经讨论了闪电分支和闪电文字，但是我们并不会止步于此，来看看一些你可能用到的其他变种。

#### 移动的闪电

有时你会想要闪电链移动，可以这样做，在上一帧的闪电链的末端创建一个新的闪电链

    Vector2 lightningEnd = new Vector2(100, 100);
    Vector2 lightningVelocity = new Vector2(50, 0);

    void Update(GameTime gameTime)
    {
        Bolts.Add(new LightningBolt(lightningEnd, lightningEnd + lightningVelocity));
        lightningEnd += lightningVelocity;

        // ...
    }
    

#### 平滑的闪电

你可能知道在闪电节点处的光更强一些，这取决于叠加混合模式，你可能想让你的闪电看上去再平滑一些。可以用下面的方法把混合状态函数改成源颜色和目标颜色都最大。

    private static readonly BlendState maxBlend = new BlendState()
    {
            AlphaBlendFunction = BlendFunction.Max,
            ColorBlendFunction = BlendFunction.Max,
            AlphaDestinationBlend = Blend.One,
            AlphaSourceBlend = Blend.One,
            ColorDestinationBlend = Blend.One,
            ColorSourceBlend = Blend.One
    };

然后在`Draw()`方法中调用`SpriteBatch.Begin()`时，把`BlendState`参数由原来的`BlendState.Additive`替换为`maxBlend`。下面两张图片显示了叠加混合模式(additive blending)和最大混合模式(max blending)的区别。

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/24/Additive%20blending.jpg)

![image](https://cdn.tutsplus.com/gamedev/authors/legacy/Michael%20Hoffman/2012/11/24/Max%20blending.jpg)

当然最大混合模式(max blending)不会让多个闪电链的光很好地叠加，如果你想让单个闪电链看上去更自然，并且更亮，你可以先用最大混合模式(max blending)把闪电链渲染到一个渲染目标，再把这个渲染目标使用叠加混合模式(additive blending)渲染到屏幕。注意不要使用太多像这样庞大的渲染目标，会有损性能。

有另一个选择，在闪电链数量多时很有用，去掉图片自己的发光效果，添加到后处理(post-processing)中。着色器(shader)的使用和创建发光效果超出了本教程的范围，但是你可以从[XNA Bloom Sample](http://xbox.create.msdn.com/en-US/education/catalog/sample/bloom)来起步，这种技术可以在添加更多的闪电链时不需要同时添加渲染目标。

### 总结

用闪电点缀游戏有特别棒的效果，这篇教程描述的效果是个不错的开端，但并不是说只能用闪电做这些，只要发挥一点想象力，就能做出各种各样令人称赞的闪电效果！赶快[下载源代码](http://cdn.tutsplus.com/gamedev/uploads/legacy/055_lightning/Shocking-2D-Lightning-Effects-Source.zip)亲自试试吧。

如果你觉得这篇教程不错，请看看作者的另一篇[关于2d水花效果的教程](http://gamedev.tutsplus.com/tutorials/implementation/make-a-splash-with-2d-water-effects/)。

原文链接：[http://gamedevelopment.tutsplus.com/tutorials/how-to-generate-shockingly-good-2d-lightning-effects&mdash;gamedev-2681](http://gamedevelopment.tutsplus.com/tutorials/how-to-generate-shockingly-good-2d-lightning-effects--gamedev-2681)