### Memory Leak Demo App - Readme

---

#### 🚨 Memory Leak Scenario in Code
This app demonstrates a common memory leak pattern in Android development:

1. **MainActivity** sets a static reference to itself:
   ```kotlin
   companion object {
       lateinit var context: Context
   }
   ```

2. **SecondActivity** overrides this reference:
   ```kotlin
   MainActivity.context = this // Leak!
   ```

3. **Result**: When SecondActivity is destroyed (e.g., user presses back), it remains in memory because:
   - Static `MainActivity.context` holds a strong reference
   - Activity context (~10MB) can't be garbage collected
   - Repeated usage causes OutOfMemoryError

---

#### 🔍 How to Detect the Leak
1. Use **Android Studio Profiler**:
   - Run app → Open Profiler → Memory heap dump
   - Filter by destroyed activity name (`SecondActivity`)
   - If instances remain after destruction → Leak confirmed

2. Check Logcat for warnings:
   ```
   Activity com.example.memoryleckapp.SecondActivity has leaked
   ```

---

#### ✅ How to Fix the Memory Leak

**Option 1: Use Application Context**
```kotlin
// In any activity
MainActivity.context = applicationContext // Safe!
```

**Option 2: Weak References (Recommended)**
```kotlin
companion object {
    private var weakContext: WeakReference<Context>? = null
    
    // Set context
    fun setContext(context: Context) {
        weakContext = WeakReference(context)
    }
    
    // Get context (nullable)
    fun getContext() = weakContext?.get()
}
```

**Option 3: Clear References in onDestroy()**
```kotlin
// In SecondActivity
override fun onDestroy() {
    MainActivity.context = null // Clear reference
    super.onDestroy()
}
```

**Option 4: Use ViewModel/LiveData**
```kotlin
// SharedViewModel.kt
class SharedViewModel : ViewModel() {
    val contextData = MutableLiveData<Context>()
}

// In Activities
viewModel.contextData.value = requireContext()
```

---

#### 🛡️ Best Practices to Prevent Memory Leaks
1. **Never store Activity in static fields**
2. **Avoid non-static inner classes** (they hold outer class reference)
3. **Unregister listeners** in `onPause()`/`onDestroy()`:
   ```kotlin
   override fun onDestroy() {
       sensorManager.unregisterListener(this)
       _binding = null // Clear view binding
       super.onDestroy()
   }
   ```
4. **Use context wisely**:
   - `applicationContext` for long-lived operations
   - `activityContext` only for UI-related tasks
5. **LeakCanary integration** (automatic leak detection):
   ```gradle
   // build.gradle
   debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
   ```

---

#### 🧪 Testing the Fix
1. Implement WeakReference solution:
   ```kotlin
   // MainActivity.kt
   companion object {
       private var weakContext: WeakReference<Context>? = null
       
       fun setContext(context: Context) {
           weakContext = WeakReference(context)
       }
   }
   
   // SecondActivity.kt
   MainActivity.setContext(this) // Safe!
   ```
2. Verify with Profiler:
   - No SecondActivity instances after back press
   - Memory returns to baseline

---

#### 📚 Additional Resources
- [Android Memory Management Guide](https://developer.android.com/topic/performance/memory)
- [LeakCanary Documentation](https://square.github.io/leakcanary/)
- [Common Memory Leak Patterns](https://android.jlelse.eu/9-ways-to-avoid-memory-leaks-in-android-b6d81648e35e)

> **Always test with**:
> - Multiple configuration changes (rotation)
> - Back button navigation
> - Low-memory conditions (via Developer Options)
