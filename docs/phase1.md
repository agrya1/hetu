# 河图(Hetu) - 技术实现方案

## 第一部分：项目基础

### 1. 开发环境搭建

#### 1.1 基础环境要求
```shell
# 开发环境版本要求
Flutter: 3.16.0 或更高
Dart: 3.2.0 或更高
Android Studio: 2023.1.1 或更高
Xcode: 15.0 或更高
VS Code: 最新版本
Git: 2.34.1 或更高
```

#### 1.2 Flutter环境搭建
```shell
# 1. 下载 Flutter SDK
git clone https://github.com/flutter/flutter.git -b stable

# 2. 添加环境变量
export PATH="$PATH:`pwd`/flutter/bin"

# 3. 检查环境
flutter doctor

# 4. 安装所需组件
flutter doctor --android-licenses
```

#### 1.3 IDE配置
```shell
# VS Code 插件安装
- Flutter
- Dart
- Flutter Widget Snippets
- Flutter Tree
- Awesome Flutter Snippets

# Android Studio 插件安装
- Flutter
- Dart
- Flutter Enhancement Suite
- Flutter Snippets
```

#### 1.4 开发工具配置
```yaml
# VS Code settings.json 推荐配置
{
  "editor.formatOnSave": true,
  "editor.formatOnType": true,
  "dart.previewFlutterUiGuides": true,
  "dart.openDevTools": "flutter",
  "dart.debugExternalPackageLibraries": true,
  "dart.debugSdkLibraries": false,
  "dart.warnWhenEditingFilesOutsideWorkspace": true
}
```

### 2. 项目初始化

#### 2.1 创建项目
```shell
# 创建新项目
flutter create --org com.hetu --platforms ios,android hetu

# 项目结构初始化
cd hetu
mkdir -p lib/{app,core,global}
mkdir -p lib/app/{bindings,controllers,data,modules,routes}
mkdir -p lib/app/data/{models,providers,repositories}
mkdir -p lib/core/{theme,utils,values}
```

#### 2.2 基础配置文件
```yaml
# pubspec.yaml
name: hetu
description: A creative content collection and organization tool
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.2.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  get: ^4.6.5
  dio: ^5.3.2
  hive: ^2.2.3
  sqflite: ^2.3.0
  cached_network_image: ^3.3.0
  flutter_staggered_grid_view: ^0.7.0
  photo_view: ^0.14.0
  flutter_slidable: ^3.0.0
  path_provider: ^2.1.1
  shared_preferences: ^2.2.1
  connectivity_plus: ^5.0.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0
  build_runner: ^2.4.6
  json_serializable: ^6.7.1
  mockito: ^5.4.2
```

#### 2.3 项目配置
```dart
// lib/global/config.dart
class AppConfig {
  static const String appName = 'Hetu';
  static const String apiBaseUrl = 'https://api.hetu.com';
  
  // API版本
  static const String apiVersion = 'v1';
  
  // 缓存配置
  static const int maxCacheSize = 100 * 1024 * 1024; // 100MB
  static const Duration maxCacheAge = Duration(days: 7);
  
  // 图片配置
  static const int maxImageWidth = 1080;
  static const double imageQuality = 0.8;
  
  // 分页配置
  static const int defaultPageSize = 20;
  
  // 超时配置
  static const Duration connectionTimeout = Duration(seconds: 10);
  static const Duration receiveTimeout = Duration(seconds: 10);
}
```

### 3. 技术选型详解

#### 3.1 状态管理
```dart
// GetX状态管理示例
class BoardController extends GetxController {
  final RxList<Board> _boards = <Board>[].obs;
  final RxBool _isLoading = false.obs;
  
  List<Board> get boards => _boards;
  bool get isLoading => _isLoading.value;
  
  @override
  void onInit() {
    super.onInit();
    _loadBoards();
  }
  
  Future<void> _loadBoards() async {
    try {
      _isLoading.value = true;
      final boards = await _boardRepository.getBoards();
      _boards.assignAll(boards);
    } catch (e) {
      ErrorHandler.handleError(e);
    } finally {
      _isLoading.value = false;
    }
  }
}
```

#### 3.2 网络请求
```dart
// Dio配置示例
class ApiProvider {
  static Dio _dio;
  
  static Dio get dio {
    if (_dio == null) {
      _dio = Dio(BaseOptions(
        baseUrl: AppConfig.apiBaseUrl,
        connectTimeout: AppConfig.connectionTimeout,
        receiveTimeout: AppConfig.receiveTimeout,
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
        },
      ));
      
      // 拦截器配置
      _dio.interceptors.add(LogInterceptor(
        requestBody: true,
        responseBody: true,
      ));
      
      // 错误处理
      _dio.interceptors.add(ErrorInterceptor());
    }
    return _dio;
  }
}
```

#### 3.3 本地存储
```dart
// Hive配置示例
class StorageManager {
  static Future<void> init() async {
    final appDocumentDir = await getApplicationDocumentsDirectory();
    Hive.init(appDocumentDir.path);
    
    // 注册适配器
    Hive.registerAdapter(BoardAdapter());
    Hive.registerAdapter(ItemAdapter());
    
    // 打开盒子
    await Hive.openBox<Board>('boards');
    await Hive.openBox<Item>('items');
  }
}
```

### 4. 项目结构设计

#### 4.1 目录结构说明
```
lib/
├── app/                    # 应用层
│   ├── bindings/          # 依赖注入
│   │   ├── home_binding.dart
│   │   └── board_binding.dart
│   ├── controllers/       # 控制器
│   │   ├── home_controller.dart
│   │   └── board_controller.dart
│   ├── data/             # 数据层
│   │   ├── models/       # 数据模型
│   │   ├── providers/    # 数据提供者
│   │   └── repositories/ # 数据仓库
│   ├── modules/          # 功能模块
│   │   ├── home/
│   │   └── board/
│   └── routes/           # 路由配置
├── core/                 # 核心功能
│   ├── theme/           # 主题配置
│   ├── utils/           # 工具类
│   └── values/          # 常量值
├── global/              # 全局配置
└── main.dart            # 入口文件
```

#### 4.2 模块划分
```dart
// 示例：画板模块结构
board/
├── bindings/
│   └── board_binding.dart
├── controllers/
│   └── board_controller.dart
├── models/
│   └── board_model.dart
├── repositories/
│   └── board_repository.dart
└── views/
    ├── board_view.dart
    ├── components/
    │   ├── board_card.dart
    │   └── board_list.dart
    └── widgets/
        ├── board_header.dart
        └── board_footer.dart
```

### 5. 开发规范制定

#### 5.1 代码风格
```dart
// 1. 命名规范
// 类名使用大驼峰
class BoardController {}

// 变量和方法使用小驼峰
final String boardTitle;
void updateBoard() {}

// 私有属性使用下划线前缀
final String _privateField;

// 2. 文件命名
// 全小写，使用下划线分隔
board_controller.dart
board_repository.dart

// 3. 常量命名
// 使用大写字母和下划线
const int MAX_RETRY_COUNT = 3;
```

#### 5.2 文档规范
```dart
/// 画板控制器
/// 
/// 负责管理画板的状态和业务逻辑
/// 包括：
/// * 画板的CRUD操作
/// * 画板内容的排序
/// * 画板的同步功能
class BoardController extends GetxController {
  /// 创建新画板
  /// 
  /// [title] 画板标题
  /// [description] 画板描述
  /// 
  /// 返回创建的画板ID
  /// 如果创建失败，抛出 [BoardException]
  Future<String> createBoard({
    required String title,
    String? description,
  }) async {
    // 实现代码
  }
}
```

#### 5.3 Git提交规范
```shell
# Commit Message格式
<type>(<scope>): <subject>

# type类型
feat: 新功能
fix: 修复bug
docs: 文档更新
style: 代码格式化
refactor: 重构代码
test: 测试相关
chore: 构建过程或辅助工具的变动

# 示例
feat(board): add board creation feature
fix(auth): fix login validation
```

#### 5.4 测试规范
```dart
void main() {
  group('Board Controller Tests', () {
    late BoardController controller;
    late MockBoardRepository mockRepository;

    setUp(() {
      mockRepository = MockBoardRepository();
      controller = BoardController(repository: mockRepository);
    });

    test('should create board successfully', () async {
      // Arrange
      when(mockRepository.createBoard(any))
          .thenAnswer((_) async => 'board_id');

      // Act
      final result = await controller.createBoard(
        title: 'Test Board',
        description: 'Test Description',
      );

      // Assert
      expect(result, isNotEmpty);
      verify(mockRepository.createBoard(any)).called(1);
    });
  });
}
```