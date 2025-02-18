Android ViewModel与LiveData 浅解析
2022-7-20 13:44   Android
ViewModel
我们常用的 AppCompatActivity 其实是继承自 ComponentActivity，ComponentActivity 又继承自 Activity，Activity内部有 mChangingConfigurations 的boolean来判断是否是真的销毁，而不是屏幕旋转。

public ComponentActivity() {
        Lifecycle lifecycle = getLifecycle();
        ...
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
并且，old Activity在销毁时，会执行onRetainNonConfigurationInstance方法，保存当前的 mViewModelStore。

    @Override
    @Nullable
    public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();
 
        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }
 
        if (viewModelStore == null && custom == null) {
            return null;
        }
   
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        //保存了viewModelStore
        nci.viewModelStore = viewModelStore;
        return nci;
    }
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            //获取上次保存的viewModelStore
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
LiveData
虽然LiveData在2017年就说已经过时了，但是面试还会问，真是fuck啊。

LiveData在添加观察者时，会传一个viewLifecycleOwner，this代表的也是viewLifecycleOwner。

    viewModel.userInfo.observe(viewLifecycleOwner) { userInfo -> ....}
    //或者
    viewModel.userInfo.observe(this) { userInfo -> ....}
LiveData 会观察 viewLifecycleOwner 的生命周期。

    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
如果宿主被销毁了，自动移除 Observer ，其实就是我们在添加对数据的观察者时调用的 observe 方法中的第二个传参 Observer<? super T>。

    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;
        ...
 
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
            if (currentState == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            Lifecycle.State prevState = null;
            while (prevState != currentState) {
                prevState = currentState;
                activeStateChanged(shouldBeActive());
                currentState = mOwner.getLifecycle().getCurrentState();
            }
        }
postValue() 与 setValue()
LiveData 为我们提供了两个不同的方法来进行数据更新，setValue(T value) 和 postValue(T value)

As you can see，setValue()方法强制要求我们在主线程使用该方法进行 LiveData 的数据更新，dispatchingValue(null)方法中会遍历所有的观察者，将更新事件会将value分发给观察者。

    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
postValue()则是帮我们post到主线程来更新数据。

    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }
除了帮我们切主线程，还有一点不同，postValue() 方法中我们是对 mPendingData 进行修改，mPendingData 其实是postToMainThread方法参数的 Runnable 中有引用的，Runnable中 会将 PendingData 作为 newValue 再调用 setValue()方法。

这样就会导致，如果主线程还没有执行Runnable时再次postValue()，旧数据会被新数据覆盖掉。

   private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };
简单写个 Demo 进行检验，同时两次setValue()，新旧数据都不会丢弃，会顺序执行两次Log，但是同时postValue()两次，只会打印一次新数据。

class MainActivity : AppCompatActivity() {
    // 收到后 Log UserName
    viewModel.addUserNameObserver(this) { 
        userName -> Log.d("LiveDataTest", t.toString()) 
    }
    // 点击触发ViewModel的fetch方法
    binding.root.setOnClickListener {
        viewModel.fetchUserName()
    }
}
 
class DataViewModel : ViewModel() {
    private val userName = MutableLiveData<String>()
    //一次性 set 或 post 两个值
    fun fetchUserName() {
        userName.value = "YJY1"
        userName.value = "YJY2"
 
        userName.postValue("YJY1")
        userName.postValue("YJY2")
    }
}
