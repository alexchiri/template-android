# Debug Logs Skill

Creates a debug logs screen accessible from the app's settings screen where all application logs can be viewed.

## Instructions

When this skill is invoked:

1. Ask the user for:
   - Package name (e.g., `com.kroslabs.myapp`)
   - Whether they use Jetpack Compose or XML Views for UI

2. Create the following files based on the UI framework:

### For Jetpack Compose:

**Create `app/src/main/java/{{PACKAGE_PATH}}/logging/DebugLogger.kt`:**

```kotlin
package {{PACKAGE_NAME}}.logging

import android.util.Log
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

data class LogEntry(
    val timestamp: Long = System.currentTimeMillis(),
    val level: LogLevel,
    val tag: String,
    val message: String,
    val throwable: Throwable? = null
) {
    val formattedTime: String
        get() = SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault()).format(Date(timestamp))

    val formattedMessage: String
        get() = buildString {
            append("[$formattedTime] ${level.name}/$tag: $message")
            throwable?.let { append("\n${it.stackTraceToString()}") }
        }
}

enum class LogLevel { VERBOSE, DEBUG, INFO, WARN, ERROR }

object DebugLogger {
    private const val MAX_LOGS = 1000
    private const val MAX_AGE_MS = 60 * 60 * 1000L // 1 hour in milliseconds

    private val _logs = MutableStateFlow<List<LogEntry>>(emptyList())
    val logs: StateFlow<List<LogEntry>> = _logs.asStateFlow()

    private fun addLog(entry: LogEntry) {
        val now = System.currentTimeMillis()
        val cutoffTime = now - MAX_AGE_MS

        // Filter out logs older than 1 hour and add the new entry
        _logs.value = (_logs.value.filter { it.timestamp > cutoffTime } + entry).takeLast(MAX_LOGS)
    }

    private fun pruneOldLogs() {
        val cutoffTime = System.currentTimeMillis() - MAX_AGE_MS
        _logs.value = _logs.value.filter { it.timestamp > cutoffTime }
    }

    fun v(tag: String, message: String, throwable: Throwable? = null) {
        Log.v(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.VERBOSE, tag = tag, message = message, throwable = throwable))
    }

    fun d(tag: String, message: String, throwable: Throwable? = null) {
        Log.d(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.DEBUG, tag = tag, message = message, throwable = throwable))
    }

    fun i(tag: String, message: String, throwable: Throwable? = null) {
        Log.i(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.INFO, tag = tag, message = message, throwable = throwable))
    }

    fun w(tag: String, message: String, throwable: Throwable? = null) {
        Log.w(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.WARN, tag = tag, message = message, throwable = throwable))
    }

    fun e(tag: String, message: String, throwable: Throwable? = null) {
        Log.e(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.ERROR, tag = tag, message = message, throwable = throwable))
    }

    fun clearLogs() {
        _logs.value = emptyList()
    }

    fun exportLogs(): String {
        pruneOldLogs() // Prune before exporting
        return _logs.value.joinToString("\n") { it.formattedMessage }
    }
}
```

**Create `app/src/main/java/{{PACKAGE_PATH}}/ui/screens/DebugLogsScreen.kt`:**

```kotlin
package {{PACKAGE_NAME}}.ui.screens

import androidx.compose.foundation.background
import androidx.compose.foundation.horizontalScroll
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.foundation.rememberScrollState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material.icons.filled.Share
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalClipboardManager
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.AnnotatedString
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import {{PACKAGE_NAME}}.logging.DebugLogger
import {{PACKAGE_NAME}}.logging.LogLevel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DebugLogsScreen(
    onNavigateBack: () -> Unit
) {
    val logs by DebugLogger.logs.collectAsState()
    val listState = rememberLazyListState()
    val clipboardManager = LocalClipboardManager.current

    LaunchedEffect(logs.size) {
        if (logs.isNotEmpty()) {
            listState.animateScrollToItem(logs.size - 1)
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Debug Logs") },
                navigationIcon = {
                    IconButton(onClick = onNavigateBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                },
                actions = {
                    IconButton(onClick = {
                        clipboardManager.setText(AnnotatedString(DebugLogger.exportLogs()))
                    }) {
                        Icon(Icons.Default.Share, contentDescription = "Copy logs")
                    }
                    IconButton(onClick = { DebugLogger.clearLogs() }) {
                        Icon(Icons.Default.Delete, contentDescription = "Clear logs")
                    }
                }
            )
        }
    ) { padding ->
        if (logs.isEmpty()) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(padding),
                contentAlignment = androidx.compose.ui.Alignment.Center
            ) {
                Text(
                    text = "No logs yet",
                    style = MaterialTheme.typography.bodyLarge,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        } else {
            LazyColumn(
                state = listState,
                modifier = Modifier
                    .fillMaxSize()
                    .padding(padding)
                    .background(Color(0xFF1E1E1E))
            ) {
                items(logs, key = { it.timestamp }) { entry ->
                    Text(
                        text = entry.formattedMessage,
                        fontFamily = FontFamily.Monospace,
                        fontSize = 11.sp,
                        color = when (entry.level) {
                            LogLevel.VERBOSE -> Color.Gray
                            LogLevel.DEBUG -> Color.Cyan
                            LogLevel.INFO -> Color.Green
                            LogLevel.WARN -> Color.Yellow
                            LogLevel.ERROR -> Color.Red
                        },
                        modifier = Modifier
                            .fillMaxWidth()
                            .horizontalScroll(rememberScrollState())
                            .padding(horizontal = 8.dp, vertical = 2.dp)
                    )
                }
            }
        }
    }
}
```

### For XML Views:

**Create `app/src/main/java/{{PACKAGE_PATH}}/logging/DebugLogger.kt`:**

```kotlin
package {{PACKAGE_NAME}}.logging

import android.util.Log
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.concurrent.CopyOnWriteArrayList

data class LogEntry(
    val timestamp: Long = System.currentTimeMillis(),
    val level: LogLevel,
    val tag: String,
    val message: String,
    val throwable: Throwable? = null
) {
    val formattedTime: String
        get() = SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault()).format(Date(timestamp))

    val formattedMessage: String
        get() = buildString {
            append("[$formattedTime] ${level.name}/$tag: $message")
            throwable?.let { append("\n${it.stackTraceToString()}") }
        }
}

enum class LogLevel { VERBOSE, DEBUG, INFO, WARN, ERROR }

object DebugLogger {
    private const val MAX_LOGS = 1000
    private const val MAX_AGE_MS = 60 * 60 * 1000L // 1 hour in milliseconds

    private val _logs = CopyOnWriteArrayList<LogEntry>()
    val logs: List<LogEntry> get() {
        pruneOldLogs()
        return _logs.toList()
    }

    private val listeners = mutableListOf<() -> Unit>()

    fun addListener(listener: () -> Unit) {
        listeners.add(listener)
    }

    fun removeListener(listener: () -> Unit) {
        listeners.remove(listener)
    }

    private fun notifyListeners() {
        listeners.forEach { it() }
    }

    private fun pruneOldLogs() {
        val cutoffTime = System.currentTimeMillis() - MAX_AGE_MS
        val oldLogs = _logs.filter { it.timestamp <= cutoffTime }
        _logs.removeAll(oldLogs)
    }

    private fun addLog(entry: LogEntry) {
        // Prune logs older than 1 hour
        pruneOldLogs()

        _logs.add(entry)
        while (_logs.size > MAX_LOGS) {
            _logs.removeAt(0)
        }
        notifyListeners()
    }

    fun v(tag: String, message: String, throwable: Throwable? = null) {
        Log.v(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.VERBOSE, tag = tag, message = message, throwable = throwable))
    }

    fun d(tag: String, message: String, throwable: Throwable? = null) {
        Log.d(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.DEBUG, tag = tag, message = message, throwable = throwable))
    }

    fun i(tag: String, message: String, throwable: Throwable? = null) {
        Log.i(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.INFO, tag = tag, message = message, throwable = throwable))
    }

    fun w(tag: String, message: String, throwable: Throwable? = null) {
        Log.w(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.WARN, tag = tag, message = message, throwable = throwable))
    }

    fun e(tag: String, message: String, throwable: Throwable? = null) {
        Log.e(tag, message, throwable)
        addLog(LogEntry(level = LogLevel.ERROR, tag = tag, message = message, throwable = throwable))
    }

    fun clearLogs() {
        _logs.clear()
        notifyListeners()
    }

    fun exportLogs(): String {
        pruneOldLogs() // Prune before exporting
        return _logs.joinToString("\n") { it.formattedMessage }
    }
}
```

**Create `app/src/main/java/{{PACKAGE_PATH}}/ui/DebugLogsActivity.kt`:**

```kotlin
package {{PACKAGE_NAME}}.ui

import android.content.ClipData
import android.content.ClipboardManager
import android.content.Context
import android.graphics.Color
import android.os.Bundle
import android.view.LayoutInflater
import android.view.Menu
import android.view.MenuItem
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import {{PACKAGE_NAME}}.R
import {{PACKAGE_NAME}}.logging.DebugLogger
import {{PACKAGE_NAME}}.logging.LogEntry
import {{PACKAGE_NAME}}.logging.LogLevel

class DebugLogsActivity : AppCompatActivity() {

    private lateinit var recyclerView: RecyclerView
    private lateinit var emptyView: TextView
    private lateinit var adapter: LogsAdapter

    private val logsListener = {
        runOnUiThread { updateLogs() }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_debug_logs)

        supportActionBar?.setDisplayHomeAsUpEnabled(true)
        supportActionBar?.title = "Debug Logs"

        recyclerView = findViewById(R.id.recyclerView)
        emptyView = findViewById(R.id.emptyView)

        adapter = LogsAdapter()
        recyclerView.layoutManager = LinearLayoutManager(this)
        recyclerView.adapter = adapter

        DebugLogger.addListener(logsListener)
        updateLogs()
    }

    override fun onDestroy() {
        super.onDestroy()
        DebugLogger.removeListener(logsListener)
    }

    private fun updateLogs() {
        val logs = DebugLogger.logs
        adapter.submitList(logs)

        if (logs.isEmpty()) {
            emptyView.visibility = View.VISIBLE
            recyclerView.visibility = View.GONE
        } else {
            emptyView.visibility = View.GONE
            recyclerView.visibility = View.VISIBLE
            recyclerView.scrollToPosition(logs.size - 1)
        }
    }

    override fun onCreateOptionsMenu(menu: Menu): Boolean {
        menuInflater.inflate(R.menu.menu_debug_logs, menu)
        return true
    }

    override fun onOptionsItemSelected(item: MenuItem): Boolean {
        return when (item.itemId) {
            android.R.id.home -> {
                finish()
                true
            }
            R.id.action_copy -> {
                val clipboard = getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
                clipboard.setPrimaryClip(ClipData.newPlainText("Debug Logs", DebugLogger.exportLogs()))
                Toast.makeText(this, "Logs copied to clipboard", Toast.LENGTH_SHORT).show()
                true
            }
            R.id.action_clear -> {
                DebugLogger.clearLogs()
                true
            }
            else -> super.onOptionsItemSelected(item)
        }
    }

    private class LogsAdapter : RecyclerView.Adapter<LogsAdapter.ViewHolder>() {
        private var logs = listOf<LogEntry>()

        fun submitList(newLogs: List<LogEntry>) {
            logs = newLogs
            notifyDataSetChanged()
        }

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
            val view = LayoutInflater.from(parent.context)
                .inflate(R.layout.item_log_entry, parent, false)
            return ViewHolder(view)
        }

        override fun onBindViewHolder(holder: ViewHolder, position: Int) {
            holder.bind(logs[position])
        }

        override fun getItemCount() = logs.size

        class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
            private val textView: TextView = view.findViewById(R.id.logText)

            fun bind(entry: LogEntry) {
                textView.text = entry.formattedMessage
                textView.setTextColor(when (entry.level) {
                    LogLevel.VERBOSE -> Color.GRAY
                    LogLevel.DEBUG -> Color.CYAN
                    LogLevel.INFO -> Color.GREEN
                    LogLevel.WARN -> Color.YELLOW
                    LogLevel.ERROR -> Color.RED
                })
            }
        }
    }
}
```

**Create `app/src/main/res/layout/activity_debug_logs.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#1E1E1E">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <TextView
        android:id="@+id/emptyView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:text="No logs yet"
        android:textColor="#888888"
        android:textSize="16sp"
        android:visibility="gone" />

</FrameLayout>
```

**Create `app/src/main/res/layout/item_log_entry.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/logText"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:fontFamily="monospace"
    android:paddingHorizontal="8dp"
    android:paddingVertical="2dp"
    android:textSize="11sp"
    android:scrollHorizontally="true" />
```

**Create `app/src/main/res/menu/menu_debug_logs.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <item
        android:id="@+id/action_copy"
        android:icon="@android:drawable/ic_menu_share"
        android:title="Copy logs"
        app:showAsAction="ifRoom" />

    <item
        android:id="@+id/action_clear"
        android:icon="@android:drawable/ic_menu_delete"
        android:title="Clear logs"
        app:showAsAction="ifRoom" />

</menu>
```

**Add to `AndroidManifest.xml`:**

```xml
<activity
    android:name=".ui.DebugLogsActivity"
    android:exported="false"
    android:parentActivityName=".ui.SettingsActivity" />
```

3. **Add debug logging throughout the app:**

   After creating the DebugLogger infrastructure, scan the entire codebase and add comprehensive debug logging:

   **a. Replace existing Log.* calls:**
   - Search for all `android.util.Log.*` calls in the codebase
   - Replace them with equivalent `DebugLogger.*` calls
   - Update imports accordingly

   **b. Add logging to Activities/Fragments lifecycle:**
   ```kotlin
   class MyActivity : AppCompatActivity() {
       private val TAG = "MyActivity"

       override fun onCreate(savedInstanceState: Bundle?) {
           super.onCreate(savedInstanceState)
           DebugLogger.d(TAG, "onCreate called")
           // ...
       }

       override fun onResume() {
           super.onResume()
           DebugLogger.d(TAG, "onResume called")
       }

       override fun onPause() {
           super.onPause()
           DebugLogger.d(TAG, "onPause called")
       }

       override fun onDestroy() {
           super.onDestroy()
           DebugLogger.d(TAG, "onDestroy called")
       }
   }
   ```

   **c. Add logging to ViewModels:**
   ```kotlin
   class MyViewModel : ViewModel() {
       private val TAG = "MyViewModel"

       init {
           DebugLogger.d(TAG, "ViewModel initialized")
       }

       fun loadData() {
           DebugLogger.d(TAG, "loadData() called")
           // ... implementation
           DebugLogger.i(TAG, "Data loaded successfully: $result")
       }

       override fun onCleared() {
           super.onCleared()
           DebugLogger.d(TAG, "ViewModel cleared")
       }
   }
   ```

   **d. Add logging to network/API calls:**
   ```kotlin
   suspend fun fetchData(): Result<Data> {
       DebugLogger.d(TAG, "fetchData() - Starting API request")
       return try {
           val response = api.getData()
           DebugLogger.i(TAG, "fetchData() - Success: ${response.size} items received")
           Result.success(response)
       } catch (e: Exception) {
           DebugLogger.e(TAG, "fetchData() - Failed", e)
           Result.failure(e)
       }
   }
   ```

   **e. Add logging to user interactions:**
   ```kotlin
   Button(onClick = {
       DebugLogger.d(TAG, "Submit button clicked")
       viewModel.submitForm()
   }) {
       Text("Submit")
   }
   ```

   **f. Add logging to data operations:**
   ```kotlin
   fun saveToDatabase(item: Item) {
       DebugLogger.d(TAG, "saveToDatabase() - Saving item: ${item.id}")
       try {
           database.insert(item)
           DebugLogger.i(TAG, "saveToDatabase() - Item saved successfully")
       } catch (e: Exception) {
           DebugLogger.e(TAG, "saveToDatabase() - Failed to save item", e)
           throw e
       }
   }
   ```

   **g. Add logging to repositories:**
   ```kotlin
   class UserRepository(private val api: ApiService, private val db: UserDao) {
       private val TAG = "UserRepository"

       suspend fun getUser(id: String): User {
           DebugLogger.d(TAG, "getUser() - Fetching user: $id")

           // Try cache first
           val cached = db.getUser(id)
           if (cached != null) {
               DebugLogger.d(TAG, "getUser() - Found in cache")
               return cached
           }

           // Fetch from network
           DebugLogger.d(TAG, "getUser() - Cache miss, fetching from network")
           val user = api.getUser(id)
           db.insert(user)
           DebugLogger.i(TAG, "getUser() - Fetched and cached user: ${user.name}")
           return user
       }
   }
   ```

   **h. Add logging to navigation:**
   ```kotlin
   fun navigateToDetails(itemId: String) {
       DebugLogger.d(TAG, "Navigating to details screen for item: $itemId")
       navController.navigate("details/$itemId")
   }
   ```

4. **Logging best practices to follow:**
   - Use consistent TAG naming (class name)
   - Log method entry with parameters (excluding sensitive data)
   - Log method exit with results/status
   - Log errors with full exception context
   - Use appropriate log levels:
     - `VERBOSE`: Very detailed tracing
     - `DEBUG`: Development debugging info
     - `INFO`: Important state changes, success events
     - `WARN`: Recoverable issues, unexpected but handled situations
     - `ERROR`: Failures, exceptions, critical issues
   - **Never log sensitive data** (passwords, tokens, PII)

5. **After adding logging, instruct the user:**
   - Add a "Debug Logs" option to their settings screen that navigates to the debug logs screen
   - Note: In production builds, they may want to disable the DebugLogger or hide the menu option

6. Provide example usage:
   ```kotlin
   // Instead of:
   Log.d("MyTag", "Some debug message")

   // Use:
   DebugLogger.d("MyTag", "Some debug message")
   ```
