## 第二部分：核心架构

### 1. 状态管理方案

#### 1.1 GetX状态管理架构
```dart
// lib/core/base/base_controller.dart
abstract class BaseController extends GetxController {
  final RxBool _isLoading = false.obs;
  final RxString _errorMessage = ''.obs;
  
  bool get isLoading => _isLoading.value;
  String get errorMessage => _errorMessage.value;
  
  // 统一的加载状态处理
  Future<T> runWithLoading<T>(Future<T> Function() task) async {
    try {
      _isLoading.value = true;
      return await task();
    } catch (e) {
      _errorMessage.value = e.toString();
      rethrow;
    } finally {
      _isLoading.value = false;
    }
  }
}

// lib/app/controllers/board_controller.dart
class BoardController extends BaseController {
  final BoardRepository _repository;
  final RxList<Board> boards = <Board>[].obs;
  final Rx<ViewMode> currentViewMode = ViewMode.folder.obs;
  
  // 响应式状态
  final RxBool isExpanded = true.obs;
  final RxString selectedBoardId = ''.obs;
  
  // 计算属性
  bool get hasBoards => boards.isNotEmpty;
  Board? get selectedBoard => boards.firstWhere(
    (b) => b.id == selectedBoardId.value,
    orElse: () => null,
  );
  
  // 状态更新方法
  void toggleExpanded() => isExpanded.toggle();
  void setViewMode(ViewMode mode) => currentViewMode.value = mode;
  
  // 业务逻辑
  Future<void> loadBoards() => runWithLoading(() async {
    final result = await _repository.getBoards();
    boards.assignAll(result);
  });
}
```

#### 1.2 状态绑定
```dart
// lib/app/bindings/board_binding.dart
class BoardBinding extends Bindings {
  @override
  void dependencies() {
    // 依赖注入
    Get.lazyPut<BoardRepository>(() => BoardRepository(
      apiProvider: Get.find(),
      storageProvider: Get.find(),
    ));
    
    Get.lazyPut<BoardController>(() => BoardController(
      repository: Get.find(),
    ));
  }
}
```

### 2. 路由管理方案

#### 2.1 路由配置
```dart
// lib/app/routes/app_pages.dart
abstract class AppPages {
  static final pages = [
    GetPage(
      name: Routes.HOME,
      page: () => HomeView(),
      binding: HomeBinding(),
      transition: Transition.fadeIn,
      middlewares: [AuthMiddleware()],
    ),
    GetPage(
      name: Routes.BOARD,
      page: () => BoardView(),
      binding: BoardBinding(),
      children: [
        GetPage(
          name: Routes.BOARD_DETAIL,
          page: () => BoardDetailView(),
        ),
      ],
    ),
  ];
}

// lib/app/routes/app_routes.dart
abstract class Routes {
  static const HOME = '/';
  static const BOARD = '/board';
  static const BOARD_DETAIL = '/detail';
  
  // 嵌套路由
  static String boardDetail(String id) => '$BOARD/$id$BOARD_DETAIL';
}
```

#### 2.2 路由中间件
```dart
// lib/app/middlewares/auth_middleware.dart
class AuthMiddleware extends GetMiddleware {
  @override
  RouteSettings? redirect(String? route) {
    final authService = Get.find<AuthService>();
    return authService.isAuthenticated 
      ? null 
      : RouteSettings(name: Routes.LOGIN);
  }
  
  @override
  GetPage? onPageCalled(GetPage? page) {
    // 页面调用前的处理
    return page;
  }
}
```

### 3. 网络层设计

#### 3.1 网络请求基础配置
```dart
// lib/core/network/api_provider.dart
class ApiProvider {
  late final Dio _dio;
  
  ApiProvider() {
    _dio = Dio(BaseOptions(
      baseUrl: AppConfig.apiBaseUrl,
      connectTimeout: AppConfig.connectionTimeout,
      receiveTimeout: AppConfig.receiveTimeout,
      headers: _defaultHeaders,
    ));
    
    _setupInterceptors();
  }
  
  Map<String, String> get _defaultHeaders => {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ${Get.find<AuthService>().token}',
  };
  
  void _setupInterceptors() {
    // 请求拦截
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        // 请求预处理
        return handler.next(options);
      },
      onResponse: (response, handler) {
        // 响应预处理
        return handler.next(response);
      },
      onError: (error, handler) {
        // 错误处理
        return handler.next(_handleError(error));
      },
    ));
    
    // 缓存拦截器
    _dio.interceptors.add(CacheInterceptor());
    
    // 重试拦截器
    _dio.interceptors.add(RetryInterceptor(
      dio: _dio,
      retries: 3,
      retryDelays: const [
        Duration(seconds: 1),
        Duration(seconds: 2),
        Duration(seconds: 3),
      ],
    ));
  }
}
```

#### 3.2 请求封装
```dart
// lib/core/network/api_client.dart
class ApiClient {
  final ApiProvider _provider;
  
  Future<T> get<T>(
    String path, {
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      final response = await _provider.dio.get(
        path,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );
      return _handleResponse<T>(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
  
  Future<T> post<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      final response = await _provider.dio.post(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );
      return _handleResponse<T>(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
}
```

### 4. 数据持久化方案

#### 4.1 本地存储管理
```dart
// lib/core/storage/storage_manager.dart
class StorageManager {
  static late final Box<dynamic> _box;
  
  static Future<void> init() async {
    await Hive.initFlutter();
    _box = await Hive.openBox('app_box');
    
    // 注册适配器
    Hive.registerAdapter(BoardAdapter());
    Hive.registerAdapter(ItemAdapter());
  }
  
  static Future<void> saveData<T>(String key, T value) async {
    await _box.put(key, value);
  }
  
  static T? getData<T>(String key) {
    return _box.get(key) as T?;
  }
  
  static Future<void> removeData(String key) async {
    await _box.delete(key);
  }
  
  static Future<void> clearAll() async {
    await _box.clear();
  }
}
```

#### 4.2 数据库管理
```dart
// lib/core/database/database_manager.dart
class DatabaseManager {
  static Database? _database;
  
  static Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  static Future<Database> _initDatabase() async {
    final path = await getDatabasesPath();
    final dbPath = join(path, 'hetu.db');
    
    return await openDatabase(
      dbPath,
      version: 1,
      onCreate: (db, version) async {
        // 创建表
        await db.execute('''
          CREATE TABLE boards (
            id TEXT PRIMARY KEY,
            title TEXT NOT NULL,
            description TEXT,
            cover_url TEXT,
            created_at INTEGER,
            updated_at INTEGER
          )
        ''');
        
        await db.execute('''
          CREATE TABLE items (
            id TEXT PRIMARY KEY,
            board_id TEXT,
            type TEXT,
            content TEXT,
            sort_order INTEGER,
            created_at INTEGER,
            FOREIGN KEY (board_id) REFERENCES boards (id)
          )
        ''');
      },
      onUpgrade: (db, oldVersion, newVersion) async {
        // 数据库升级逻辑
      },
    );
  }
}
```

### 5. 主题系统设计

#### 5.1 主题配置
```dart
// lib/core/theme/app_theme.dart
class AppTheme {
  // 颜色主题
  static final ThemeData lightTheme = ThemeData(
    primaryColor: AppColors.primary,
    scaffoldBackgroundColor: AppColors.background,
    textTheme: AppTextTheme.lightTextTheme,
    appBarTheme: AppBarTheme(
      backgroundColor: AppColors.primary,
      elevation: 0,
      iconTheme: IconThemeData(color: AppColors.white),
    ),
    // 其他主题配置...
  );
  
  static final ThemeData darkTheme = ThemeData(
    primaryColor: AppColors.darkPrimary,
    scaffoldBackgroundColor: AppColors.darkBackground,
    textTheme: AppTextTheme.darkTextTheme,
    // 其他暗色主题配置...
  );
}

// lib/core/theme/app_colors.dart
class AppColors {
  static const primary = Color(0xFF2196F3);
  static const secondary = Color(0xFF03DAC6);
  static const background = Color(0xFFF5F5F5);
  static const surface = Color(0xFFFFFFFF);
  static const error = Color(0xFFB00020);
  
  // 文本颜色
  static const textPrimary = Color(0xFF212121);
  static const textSecondary = Color(0xFF757575);
  
  // 暗色主题颜色
  static const darkPrimary = Color(0xFF1976D2);
  static const darkBackground = Color(0xFF121212);
}
```

#### 5.2 主题管理
```dart
// lib/core/theme/theme_service.dart
class ThemeService extends GetxService {
  final _box = GetStorage();
  final _key = 'isDarkMode';
  
  bool get isDarkMode => _box.read(_key) ?? false;
  ThemeMode get theme => isDarkMode ? ThemeMode.dark : ThemeMode.light;
  
  void changeTheme() {
    Get.changeThemeMode(isDarkMode ? ThemeMode.light : ThemeMode.dark);
    _box.write(_key, !isDarkMode);
  }
  
  // 响应系统主题变化
  void followSystem() {
    Get.changeThemeMode(ThemeMode.system);
    _box.remove(_key);
  }
}

// lib/core/theme/theme_binding.dart
class ThemeBinding extends Bindings {
  @override
  void dependencies() {
    Get.put(ThemeService());
  }
}
```

#### 5.3 主题使用
```dart
// lib/main.dart
void main() async {
  await initServices();
  
  runApp(
    GetMaterialApp(
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: Get.find<ThemeService>().theme,
      initialBinding: ThemeBinding(),
      getPages: AppPages.pages,
      initialRoute: Routes.HOME,
    ),
  );
}

// 在组件中使用主题
class CustomWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      color: Theme.of(context).primaryColor,
      child: Text(
        'Hello',
        style: Theme.of(context).textTheme.headline6,
      ),
    );
  }
}
```