# 安卓内存优化 -  ARTHook优雅检测bitmap图片大小

飞书文档链接，建议飞书打开：https://is0frj68ok.feishu.cn/wiki/DZ2FwDNUHiO7K2kRBvxcgG4Ynhf?from=from_copylink



我们先继承XC_MethodHook ，并override afterHookedMethod方法，在这个方法中，我们调用了*checkBitmap，*如果图标宽高都大于view的2倍以上，则警告

```Java
public class ImageHook extends XC_MethodHook {

    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        super.afterHookedMethod(param);
        // 实现我们的逻辑
        ImageView imageView = (ImageView) param.thisObject;
        checkBitmap(imageView,((ImageView) param.thisObject).getDrawable());
    }

    ...

    private static void warn(int bitmapWidth, int bitmapHeight, int viewWidth, int viewHeight, Throwable t) {
        String warnInfo = new StringBuilder("Bitmap size too large: ")
                .append("\n real size: (").append(bitmapWidth).append(',').append(bitmapHeight).append(')')
                .append("\n desired size: (").append(viewWidth).append(',').append(viewHeight).append(')')
                .append("\n call stack trace: \n").append(Log.getStackTraceString(t)).append('\n')
                .toString();

        LogUtils.i(warnInfo);
    }

}
```

checkBitmap的实现如下：

```Java
private static void checkBitmap(Object thiz, Drawable drawable) {
    if (drawable instanceof BitmapDrawable && thiz instanceof View) {
        final Bitmap bitmap = ((BitmapDrawable) drawable).getBitmap();
        if (bitmap != null) {
            final View view = (View) thiz;
            int width = view.getWidth();
            int height = view.getHeight();
            if (width > 0 && height > 0) {
                // 图标宽高都大于view带下的2倍以上，则警告
                if (bitmap.getWidth() >= (width << 1)
                        && bitmap.getHeight() >= (height << 1)) {
                    warn(bitmap.getWidth(), bitmap.getHeight(), width, height, new RuntimeException("Bitmap size too large"));
                }
            } else {
                final Throwable stackTrace = new RuntimeException();
                view.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                    @Override
                    public boolean onPreDraw() {
                        int w = view.getWidth();
                        int h = view.getHeight();
                        if (w > 0 && h > 0) {
                            if (bitmap.getWidth() >= (w << 1)
                                    && bitmap.getHeight() >= (h << 1)) {
                                warn(bitmap.getWidth(), bitmap.getHeight(), w, h, stackTrace);
                            }
                            view.getViewTreeObserver().removeOnPreDrawListener(this);
                        }
                        return true;
                    }
                });
            }
        }
    }
}
```

application类的onCreate方法中加入以下语句，来hook ImageView类的setImageBitmap 方法

```Java
// application类的onCreate方法中加入以下语句：

DexposedBridge.hookAllConstructors(ImageView.class, new XC_MethodHook() {
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        super.afterHookedMethod(param);
        DexposedBridge.findAndHookMethod(ImageView.class, "setImageBitmap", Bitmap.class, new ImageHook());
    }
});
```