# Android Bundle 详解

## 1. Bundle 基础概念

### 1.1 什么是Bundle？
Bundle是Android中用于在组件之间传递数据的容器类，它继承自`BaseBundle`，实现了`Parcelable`接口。

```java
public final class Bundle extends BaseBundle implements Cloneable, Parcelable
```

### 1.2 Bundle的作用

- **Activity间数据传递**：通过Intent携带Bundle传递数据
- **Fragment参数传递**：通过setArguments()方法传递参数
- **状态保存**：在onSaveInstanceState()中保存Activity/Fragment状态
- **系统回调**：系统服务通过Bundle返回数据

## 2. Bundle 内部实现原理

### 2.1 数据存储结构
Bundle内部使用`ArrayMap<String, Object>`来存储键值对：

```java
public class BaseBundle {
    // Bundle内部的数据存储容器
    ArrayMap<String, Object> mMap = null;
    
    // Parcel对象，用于序列化
    Parcel mParcelledData = null;
    
    // 标记数据是否已经从Parcel中解析
    boolean mHasFds = false;
}
```

### 2.2 懒加载机制
Bundle采用懒加载策略，只有在真正访问数据时才会反序列化：

```java
void unparcel() {
    synchronized (this) {
        final Parcel source = mParcelledData;
        if (source != null) {
            initializeFromParcelLocked(source, /*recycleParcel=*/ true);
        } else {
            if (mMap == null) {
                mMap = new ArrayMap<>(1);
            }
        }
    }
}
```

## 3. Bundle 支持的数据类型

### 3.1 基本数据类型
```java
Bundle bundle = new Bundle();

// 基本类型
bundle.putBoolean("key_boolean", true);
bundle.putByte("key_byte", (byte) 1);
bundle.putChar("key_char", 'a');
bundle.putShort("key_short", (short) 2);
bundle.putInt("key_int", 100);
bundle.putLong("key_long", 1000L);
bundle.putFloat("key_float", 1.5f);
bundle.putDouble("key_double", 2.5d);
bundle.putString("key_string", "hello");
```

### 3.2 数组类型
```java
// 基本类型数组
bundle.putBooleanArray("key_boolean_array", new boolean[]{true, false});
bundle.putIntArray("key_int_array", new int[]{1, 2, 3});
bundle.putStringArray("key_string_array", new String[]{"a", "b", "c"});

// 集合类型
ArrayList<String> stringList = new ArrayList<>();
stringList.add("item1");
bundle.putStringArrayList("key_string_list", stringList);

ArrayList<Integer> intList = new ArrayList<>();
intList.add(1);
bundle.putIntegerArrayList("key_int_list", intList);
```

### 3.3 复杂对象类型
```java
// Parcelable对象
bundle.putParcelable("key_parcelable", user);
bundle.putParcelableArray("key_parcelable_array", users);
bundle.putParcelableArrayList("key_parcelable_list", userList);

// Serializable对象
bundle.putSerializable("key_serializable", person);

// 嵌套Bundle
Bundle nestedBundle = new Bundle();
bundle.putBundle("key_bundle", nestedBundle);
```

## 4. Bundle 序列化机制

### 4.1 Parcelable序列化过程
Bundle实现Parcelable接口，支持高效的序列化：

```java
@Override
public void writeToParcel(Parcel parcel, int flags) {
    final boolean oldAllowFds = parcel.pushAllowFds(mAllowFds);
    try {
        writeToParcelInner(parcel, flags);
    } finally {
        parcel.restoreAllowFds(oldAllowFds);
    }
}

void writeToParcelInner(Parcel parcel, int flags) {
    // 如果数据还在Parcel中未解析，直接追加
    if (mParcelledData != null) {
        if (isEmptyParcel()) {
            parcel.writeInt(0);
        } else {
            parcel.appendFrom(mParcelledData, 0, mParcelledData.dataSize());
        }
    } else {
        // 否则写入Map数据
        parcel.writeArrayMapInternal(mMap);
    }
}
```

### 4.2 反序列化过程
```java
void initializeFromParcelLocked(@NonNull Parcel parcelledData, boolean recycleParcel) {
    if (isEmptyParcel(parcelledData)) {
        if (mMap == null) {
            mMap = new ArrayMap<>(1);
        } else {
            mMap.erase();
        }
        mParcelledData = null;
        return;
    }
    
    // 从Parcel中读取数据到Map
    mMap = parcelledData.readArrayMapInternal(mClassLoader);
    mParcelledData = null;
    
    if (recycleParcel) {
        recycleParcel(parcelledData);
    }
}
```

## 5. Bundle 使用场景

### 5.1 Activity间传递数据
```java
// 发送Activity
Intent intent = new Intent(this, TargetActivity.class);
Bundle bundle = new Bundle();
bundle.putString("name", "张三");
bundle.putInt("age", 25);
bundle.putParcelable("user", user);
intent.putExtras(bundle);
startActivity(intent);

// 接收Activity
Bundle bundle = getIntent().getExtras();
if (bundle != null) {
    String name = bundle.getString("name");
    int age = bundle.getInt("age");
    User user = bundle.getParcelable("user");
}
```

### 5.2 Fragment参数传递
```java
// 创建Fragment时传递参数
public static UserFragment newInstance(String userId, boolean isVip) {
    UserFragment fragment = new UserFragment();
    Bundle args = new Bundle();
    args.putString("user_id", userId);
    args.putBoolean("is_vip", isVip);
    fragment.setArguments(args);
    return fragment;
}

// Fragment中获取参数
@Override
public void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Bundle args = getArguments();
    if (args != null) {
        String userId = args.getString("user_id");
        boolean isVip = args.getBoolean("is_vip");
    }
}
```

### 5.3 状态保存和恢复
```java
@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putString("current_text", editText.getText().toString());
    outState.putInt("current_position", currentPosition);
    outState.putParcelableArrayList("data_list", dataList);
}

@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    
    if (savedInstanceState != null) {
        String text = savedInstanceState.getString("current_text");
        int position = savedInstanceState.getInt("current_position");
        ArrayList<Data> dataList = savedInstanceState.getParcelableArrayList("data_list");
        
        // 恢复状态
        editText.setText(text);
        currentPosition = position;
        this.dataList = dataList;
    }
}
```

## 6. Bundle 性能优化

### 6.1 大小限制
Bundle通过Binder传输时有大小限制：
- **TransactionTooLargeException**：当Bundle超过1MB时会抛出此异常
- **实际限制**：约为1MB-8KB（Binder缓冲区开销）

```java
public class BundleSizeHelper {
    public static int getBundleSize(Bundle bundle) {
        Parcel parcel = Parcel.obtain();
        try {
            parcel.writeBundle(bundle);
            return parcel.dataSize();
        } finally {
            parcel.recycle();
        }
    }
    
    public static boolean isBundleSafe(Bundle bundle) {
        return getBundleSize(bundle) < 500 * 1024; // 500KB安全阈值
    }
}
```

### 6.2 性能优化建议

#### 避免大对象传递
```java
// ❌ 错误做法：传递大量数据
Bundle bundle = new Bundle();
bundle.putParcelableArrayList("large_list", largeList); // 可能导致ANR

// ✅ 正确做法：传递ID，在目标组件中查询
Bundle bundle = new Bundle();
bundle.putString("list_id", listId);
// 在目标Activity/Fragment中通过ID查询数据
```

#### 使用合适的数据类型
```java
// ❌ 避免使用Serializable
bundle.putSerializable("data", object); // 性能较差

// ✅ 优先使用Parcelable
bundle.putParcelable("data", parcelableObject); // 性能更好
```

#### 延迟加载数据
```java
public class LazyDataFragment extends Fragment {
    private String dataId;
    private Data data;
    
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Bundle args = getArguments();
        if (args != null) {
            // 只传递ID
            dataId = args.getString("data_id");
        }
    }
    
    @Override
    public void onResume() {
        super.onResume();
        // 在需要时才加载数据
        if (data == null && dataId != null) {
            data = DataManager.getInstance().getData(dataId);
        }
    }
}
```

## 7. Bundle 与其他传递方式对比

### 7.1 Bundle vs Intent直接传递
```java
// Intent直接传递（适合简单数据）
intent.putExtra("name", "张三");
intent.putExtra("age", 25);

// Bundle传递（适合复杂数据）
Bundle bundle = new Bundle();
bundle.putString("name", "张三");
bundle.putInt("age", 25);
bundle.putParcelable("user", user);
intent.putExtras(bundle);
```

### 7.2 Bundle vs 全局变量
```java
// ❌ 全局变量方式（不推荐）
public class GlobalData {
    public static User currentUser;
}

// ✅ Bundle方式（推荐）
Bundle bundle = new Bundle();
bundle.putParcelable("user", user);
```

### 7.3 Bundle vs EventBus
```java
// Bundle：用于组件间参数传递
Bundle args = new Bundle();
args.putString("message", "hello");
fragment.setArguments(args);

// EventBus：用于组件间事件通信
EventBus.getDefault().post(new MessageEvent("hello"));
```

## 8. 常见问题与解决方案

### 8.1 TransactionTooLargeException
```java
public class SafeBundleHelper {
    private static final int MAX_BUNDLE_SIZE = 500 * 1024; // 500KB
    
    public static boolean checkBundleSize(Bundle bundle) {
        Parcel parcel = Parcel.obtain();
        try {
            bundle.writeToParcel(parcel, 0);
            int size = parcel.dataSize();
            if (size > MAX_BUNDLE_SIZE) {
                Log.w("Bundle", "Bundle size is too large: " + size + " bytes");
                return false;
            }
            return true;
        } finally {
            parcel.recycle();
        }
    }
}
```

### 8.2 ClassCastException
```java
// ❌ 错误的类型转换
String value = (String) bundle.get("key"); // 可能抛出ClassCastException

// ✅ 安全的获取方式
String value = bundle.getString("key", "default");
Object obj = bundle.get("key");
if (obj instanceof String) {
    String value = (String) obj;
}
```

### 8.3 空指针异常
```java
// ✅ 安全的Bundle使用
public void handleBundle(Bundle bundle) {
    if (bundle != null) {
        String name = bundle.getString("name");
        int age = bundle.getInt("age", 0); // 提供默认值
        
        // 对于可能为null的对象
        User user = bundle.getParcelable("user");
        if (user != null) {
            // 处理user对象
        }
    }
}
```

## 9. Bundle 面试常考问题

### 9.1 Bundle的底层实现原理？
- Bundle内部使用ArrayMap存储键值对数据
- 实现了Parcelable接口，支持序列化传输
- 采用懒加载机制，只有在访问时才反序列化
- 通过Binder进行跨进程传输

### 9.2 Bundle传递数据的大小限制？
- Binder传输限制约1MB
- 实际可用空间约1MB-8KB
- 超过限制会抛出TransactionTooLargeException
- 建议单次传输不超过500KB

### 9.3 Bundle与Serializable、Parcelable的关系？
- Bundle本身实现了Parcelable接口
- Bundle可以存储Parcelable和Serializable对象
- Parcelable性能更好，Serializable使用更简单
- Bundle优先选择Parcelable进行序列化

### 9.4 Bundle在不同场景下的生命周期？
- **Activity传递**：跟随Intent生命周期
- **Fragment参数**：跟随Fragment生命周期
- **状态保存**：在系统回收和重建时使用
- **跨进程**：通过Binder传输，接收方获得副本

### 9.5 如何优化Bundle的性能？
- 避免传递大对象，优先传递ID
- 使用Parcelable替代Serializable
- 控制Bundle大小，避免超过限制
- 合理设计数据传递策略

## 10. 最佳实践

### 10.1 Bundle工具类封装
```java
public class BundleUtils {
    
    public static Bundle createBundle() {
        return new Bundle();
    }
    
    public static Bundle putString(Bundle bundle, String key, String value) {
        if (bundle != null && key != null) {
            bundle.putString(key, value);
        }
        return bundle;
    }
    
    public static String getString(Bundle bundle, String key) {
        return getString(bundle, key, "");
    }
    
    public static String getString(Bundle bundle, String key, String defaultValue) {
        if (bundle != null && bundle.containsKey(key)) {
            return bundle.getString(key, defaultValue);
        }
        return defaultValue;
    }
    
    public static boolean isSafe(Bundle bundle) {
        if (bundle == null) return true;
        
        Parcel parcel = Parcel.obtain();
        try {
            bundle.writeToParcel(parcel, 0);
            return parcel.dataSize() < 500 * 1024;
        } finally {
            parcel.recycle();
        }
    }
}
```

### 10.2 数据传递设计模式
```java
// 数据传递接口
public interface DataTransfer {
    void putData(Bundle bundle);
    void getData(Bundle bundle);
}

// Activity实现
public class UserActivity extends AppCompatActivity implements DataTransfer {
    
    public static void start(Context context, User user) {
        Intent intent = new Intent(context, UserActivity.class);
        Bundle bundle = new Bundle();
        DataTransferHelper.putUser(bundle, user);
        intent.putExtras(bundle);
        context.startActivity(intent);
    }
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Bundle bundle = getIntent().getExtras();
        if (bundle != null) {
            getData(bundle);
        }
    }
    
    @Override
    public void getData(Bundle bundle) {
        User user = DataTransferHelper.getUser(bundle);
        // 使用user数据
    }
}
```
