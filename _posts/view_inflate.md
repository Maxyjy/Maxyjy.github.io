---
title:  "Advanced examples"
mathjax: true
layout: post
categories: media
---

View 的测量、布局是一个从视图树根结点以递归的方式，从上层 ViewGroup 到最底层 View 的过程。

测量 Measure

根结点 ViewGroup 会通过 getChildCount() 方法来获取所有子 View 数量，并以递归的方式调用每个子View的 measure() 方法，子 View 最终会调用自己的 onMeasure() 方法来测量自身，如果子 View 是 ViewGroup，那么子ViewGroup 也需要测量其子 View。

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    // 获取子View数量
    int count = getChildCount();
    for (int i = 0; i < count; i++) {
        View child = getChildAt(i);
        //调度子View测量自己
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
measure()只是调度方法，实际测量在 onMeasure() 方法中

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    onMeasure(widthMeasureSpec, heightMeasureSpec);
}
onMeasure()方法传参是两个int值，分别代表宽度、高度的MeasureSpec。

什么是 MeasureSpec ？
MeasureSpec 提供了从父View节点传递到子View的宽度或高度布局需求，包含模式(Mode)和大小(Size)。

/**
 * MeasureSpec模式 - Unspecified
 * 父View没有对子View施加任何约束，可以是任何尺寸。
 * 0b00 << 30
 */
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
/**
 * MeasureSpec模式 - Exactly
 * 父View指定了子View具体尺寸
 * 0b01 << 30
 */
public static final int EXACTLY     = 1 << MODE_SHIFT;
/**
 * MeasureSpec模式 - at most
 * 父View给定最大尺寸，子View可能是最大尺寸范围内的任意尺寸
 * 0b11 << 30
 */
public static final int AT_MOST     = 2 << MODE_SHIFT;
MeasureSpec 巧妙的采用了用一个int来描述模式+大小，int值由32位二进制组成，前2位代表模式(Mode)，后30位代表大小(Size)，不过不用担心，MeasureSpec 类为我们提供了拆箱、装箱的静态方法。

通过getMode()、getSize()方法对 MeasureSpec 进行拆箱，原理是将不需要的二进制位与0做与运算。

@MeasureSpecMode
public static int getMode(int measureSpec) {
    //getMode()方法，将 measureSpec 低30位通过与运算改为0，只取高2位。
    return (measureSpec & MODE_MASK);
}

public static int getSize(int measureSpec) {
    // getSize()方法，将 measureSpec 高2位通过与运算改为0，只取低30位。
    return (measureSpec & ~MODE_MASK);
}
makeMeasureSpec()方法是装箱方法，用来构造MeasureSpec。

public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                  @MeasureSpecMode int mode) {
    //小于SDK17用旧的方法构造
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        //拼接高2位和低30位
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
子View的 MeasureSpec 由父 View 和子 View 自身的 LayoutParam 决定

子View 测量完成后，View 通过 setMeasuredDimension() 方法将测量完成的自身尺寸保存起来，提供给父 ViewGroup 获取

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
布局 Layout

布局过程同样也是从 View 根结点，以递归的形式，从上至下的调用子 View 的 layout() 方法，父节点 View 会将 top、left、right、bottom 四个边距传给子 View，让子 View 在layout()方法中调用setFrame()来保存父 View 传来的位置和大小。

ViewGroup 在 onLayout() 中会调用自己的所有子 View 的 layout() 方法

由于 View 没有子 View，所以 View 的 onLayout() 什么也不做

绘制 Draw
在经过 Measure、Layout 后 ，View 确定了自身的位置，父 View 会通过dispatchDraw()方法来通知子 View 绘制自己。

public void draw(Canvas canvas) {
    ...
    dispatchDraw(canvas);
}
