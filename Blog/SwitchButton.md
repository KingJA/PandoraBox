>版权声明：本文为博主原创文章，未经博主允许不得转载
<br>转载请标明出处： http://kingja.github.io/

------
动态图在结尾，大家可以拉下去看，想学习的同学看好了再回来看文章，高手直接看**结篇**然后进传送门 - -！
## 开篇
> 艺术来源于生活

**自定义View**来源于需求，来源于灵感。自定义View如果做的好，就是艺术，就是艺术，就是艺术。
<br>文章讲的SwitchButton是大家开发过程中经常遇到的切换按钮，提供两个或两个以上的选项，居家旅行必备。

## 正文
文章是以SwitchButton的实现步骤作为大纲，主要包含以下内容：
* 自定义属性
* 构造方法
* 绘制流程
* 接口回调
* 事件响应
* 方法设置
* 状态恢复

### 自定义属性

![attr](https://github.com/KingJA/SwitchButton/blob/master/img/mark.png)  

1.定义
```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="SwitchMultiButton">
        <attr name="strokeRadius" format="dimension" />
        <attr name="strokeWidth" format="dimension" />
        <attr name="textSize" format="dimension" />
        <attr name="selectedTab" format="integer" />
        <attr name="selectedColor" format="color|reference" />
    </declare-styleable>
</resources>
```
2.获取
```java
private void initAttrs(Context context, AttributeSet attrs) {
        TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.SwitchMultiButton);
        mStrokeRadius = typedArray.getDimension(R.styleable.SwitchMultiButton_strokeRadius, dp2px(STROKE_RADIUS));
        mStrokeWidth = typedArray.getDimension(R.styleable.SwitchMultiButton_strokeWidth, dp2px(STROKE_WIDTH));
        mTextSize = typedArray.getDimension(R.styleable.SwitchMultiButton_textSize, sp2px(TEXT_SIZE));
        mSelectedColor = typedArray.getColor(R.styleable.SwitchMultiButton_selectedColor, SELECTED_COLOR);
        mSelectedTab = typedArray.getInteger(R.styleable.SwitchMultiButton_selectedTab, SELECTED_TAB);
        typedArray.recycle();
    }
```

### 构造函数
```java
public SwitchMultiButton(Context context) {
        this(context, null);
    }

    public SwitchMultiButton(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SwitchMultiButton(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        initAttrs(context, attrs);
        initPaint();
    }
```

### 绘制流程

#### 1.测量 onMeasure()
**onMeasure()**的主要目的是解决wrap_content情况下尺寸设置，这里我们设置默认的尺寸来填充wrap_content情况

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int defaultWidth = dp2px(DEFAULT_WIDTH_DP);
        int defaultHeight = dp2px(DEFAULT_HEIGHT_DP);
        setMeasuredDimension(getExpectSize(defaultWidth, widthMeasureSpec), getExpectSize(defaultHeight, heightMeasureSpec));
    }

    private int getExpectSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
                result = size;
                break;
            case MeasureSpec.AT_MOST:
                result = Math.min(size, specSize);
                break;
        }
        return result;
    }
```

#### 2.绘制onDraw()
我们的交互是动态的，要根据点击进行重新绘制。基本的绘制步骤如下：

**第1步**：绘制外层边框
![stroke](https://github.com/KingJA/SwitchButton/blob/master/img/stroke.png) 

根据用户设置的strokeRadio(圆角半径)绘制外层的边框,默认是矩形，strokeRadio>0则是圆角矩形。**注意** 边框画笔是有width(粗度)的,笔刷的起点在中间的位置，因此我们需要画笔的落笔范围要往内收缩mStrokeWidth /2的距离(Line4-Line7)，这样才能确保画笔完整地出现在画布内。

**第2步**：绘制垂直分割线。
![line](https://github.com/KingJA/SwitchButton/blob/master/img/line.png) 
这步较为简单，根据传入的字符串集合的size()进行宽度等分，drawLine(Line15)。

**第3部**：绘制选中时候的填充矩形(圆角矩形)以及所有文字。
![selected](https://github.com/KingJA/SwitchButton/blob/master/img/selected.png) 
![leftPath](https://github.com/KingJA/SwitchButton/blob/master/img/leftPath.png) 
**注意** 绘制选中矩形要最左边(第一个)和最右边(最后一个)是矩形和半圆角矩形两种情况，因此用路径绘制比较合适(Line23，Line26 )。
文字绘制比较简单，选中的用上色画笔，else用白色画笔即可(Line33，Line38 )。需要注意的是它们的 **位置** 的摆放。

```java
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        float left = mStrokeWidth * 0.5f;
        float top = mStrokeWidth * 0.5f;
        float right = mWidth - mStrokeWidth * 0.5f;
        float bottom = mHeight - mStrokeWidth * 0.5f;

        //draw rounded rectangle
        canvas.drawRoundRect(new RectF(left, top, right, bottom), mStrokeRadius, mStrokeRadius, mStrokePaint);

        //draw line
        for (int i = 0; i < mTabNum - 1; i++) {
            canvas.drawLine(perWidth * (i + 1), top, perWidth * (i + 1), bottom, mStrokePaint);
        }
        //draw tab and line
        for (int i = 0; i < mTabNum; i++) {
            String tabText = mTabTextList.get(i);
            float tabTextWidth = mSelectedTextPaint.measureText(tabText);
            if (i == mSelectedTab) {
                //draw selected tab
                if (i == 0) {
                    drawLeftPath(canvas, left, top, bottom);

                } else if (i == mTabNum - 1) {
                    drawRightPath(canvas, top, right, bottom);

                } else {
                    canvas.drawRect(new RectF(perWidth * i, top, perWidth * (i + 1), bottom), mFillPaint);
                }
                // draw selected text
                canvas.drawText(tabText, 0.5f * perWidth * (2 * i + 1) - 0.5f * tabTextWidth, mHeight * 0.5f 
+ mTextHeightOffset, mUnselectedTextPaint);

            } else {
                //draw unselected text
                canvas.drawText(tabText, 0.5f * perWidth * (2 * i + 1) - 0.5f * tabTextWidth, mHeight * 0.5f 
+ mTextHeightOffset, mSelectedTextPaint);
            }
        }
    }
```





### 接口回调
会了能响应用户的交互，我们需要设置接口，在用户点击的时候进行回调
```java
public interface OnSwitchListener {
        void onSwitch(int position, String tabText);
    }

    public void setOnSwitchListener(@NonNull OnSwitchListener onSwitchListener) {
        this.onSwitchListener = onSwitchListener;
    }
```

### 事件响应
这里我们接收用户的点击事件，由点击的位置来决定SwitchButton的选择tab，进行重新绘制。

```java
@Override
    public boolean onTouchEvent(MotionEvent event) {
        if (event.getAction() == MotionEvent.ACTION_UP) {
            float x = event.getX();
            for (int i = 0; i < mTabNum; i++) {
                if (x > perWidth * i && x < perWidth * (i + 1)) {
                    if (mSelectedTab == i) {
                        return true;
                    }
                    mSelectedTab = i;
                    if (onSwitchListener != null) {
                        onSwitchListener.onSwitch(i, mTabTextList.get(i));
                    }
                }
            }
            invalidate();
        }
        return true;
    }  
```
### 设置方法
我们的SwitchButton的内容是由用户传入的，因此要对外提供数据设置方法
```java
public SwitchMultiButton setText(@NonNull List<String> list) {
        if (list.size() > 1) {
            this.mTabTextList = list;
            mTabNum = list.size();
            invalidate();
            return this;
        } else {
            throw new IllegalArgumentException("the size of list should greater then 1");
        }
    }
```

### 状态恢复
带自定义View的界面难免会有切到后台的时候，再回来的时候要恢复它原来的状态，就要这这里做保存和恢复的设置。
```java
@Override
    protected Parcelable onSaveInstanceState() {
        Bundle bundle = new Bundle();
        bundle.putParcelable("View", super.onSaveInstanceState());
        bundle.putFloat("StrokeRadius", mStrokeRadius);
        bundle.putFloat("StrokeWidth", mStrokeWidth);
        bundle.putFloat("TextSize", mTextSize);
        bundle.putInt("SelectedColor", mSelectedColor);
        bundle.putInt("SelectedTab", mSelectedTab);
        return bundle;
    }

    @Override
    protected void onRestoreInstanceState(Parcelable state) {
        if (state instanceof Bundle) {
            Bundle bundle = (Bundle) state;
            mStrokeRadius = bundle.getInt("StrokeRadius");
            mStrokeWidth = bundle.getInt("StrokeWidth");
            mTextSize = bundle.getInt("TextSize");
            mSelectedColor = bundle.getInt("SelectedColor");
            mSelectedTab = bundle.getInt("SelectedTab");
            super.onRestoreInstanceState(bundle.getParcelable("View"));
        } else {
            super.onRestoreInstanceState(state);
        }
    } 
```
## 结篇
每当我看到别人的项目，如果刹那间，它点悟了我，或者让我回想起曾经的皱眉一刻，或者恰好是我未实现的灵光一闪，我会倍感亲切，马上滚动鼠标到顶Star一下。同样的，我希望您也能给我一颗Star，您给的这颗Star是一只萤火虫，是我漆黑生活中的一盏明灯，是沙漠中的一片绿洲，是汪洋里的一座灯塔，给我动力，指引着我前进，让这世界充满爱,来吧，让这份爱一直传递下去吧。

[[GitHub传送门，点Star送回城]](https://github.com/KingJA/SwitchButton) 
<br>
![screenshot](https://github.com/KingJA/SwitchButton/blob/master/img/usage.gif)  




.

