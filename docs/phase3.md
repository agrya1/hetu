## 第三部分：功能实现

### 1. 导航系统

#### 1.1 导航控制器
```dart
// lib/app/controllers/navigation_controller.dart
class NavigationController extends GetxController {
  final RxBool _isExpanded = true.obs;
  final RxInt _currentIndex = 0.obs;
  
  bool get isExpanded => _isExpanded.value;
  int get currentIndex => _currentIndex.value;
  
  // 导航项配置
  final List<NavigationItem> items = [
    NavigationItem(
      icon: Icons.search,
      label: '搜索',
      route: Routes.SEARCH,
    ),
    NavigationItem(
      icon: Icons.explore,
      label: '发现',
      route: Routes.DISCOVER,
    ),
    // ... 其他导航项
  ];
  
  void toggleExpansion() => _isExpanded.toggle();
  
  void changePage(int index) {
    _currentIndex.value = index;
    Get.toNamed(items[index].route);
  }
  
  // 手势控制
  void handleDragUpdate(DragUpdateDetails details) {
    if (details.delta.dx < -20 && isExpanded) {
      toggleExpansion();
    }
  }
}
```

#### 1.2 导航视图
```dart
// lib/app/modules/navigation/views/navigation_view.dart
class NavigationView extends GetView<NavigationController> {
  @override
  Widget build(BuildContext context) {
    return Obx(() => AnimatedContainer(
      duration: Duration(milliseconds: 200),
      width: controller.isExpanded ? 320 : 60,
      child: GestureDetector(
        onHorizontalDragUpdate: controller.handleDragUpdate,
        child: Drawer(
          child: Column(
            children: [
              _buildHeader(),
              _buildNavigationItems(),
              _buildFooter(),
            ],
          ),
        ),
      ),
    ));
  }
  
  Widget _buildNavigationItems() {
    return Expanded(
      child: ListView.builder(
        itemCount: controller.items.length,
        itemBuilder: (context, index) {
          final item = controller.items[index];
          return NavigationItemWidget(
            item: item,
            isExpanded: controller.isExpanded,
            isSelected: controller.currentIndex == index,
            onTap: () => controller.changePage(index),
          );
        },
      ),
    );
  }
}

// lib/app/modules/navigation/widgets/navigation_item_widget.dart
class NavigationItemWidget extends StatelessWidget {
  final NavigationItem item;
  final bool isExpanded;
  final bool isSelected;
  final VoidCallback onTap;
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Icon(
        item.icon,
        color: isSelected ? Theme.of(context).primaryColor : null,
      ),
      title: isExpanded ? Text(item.label) : null,
      selected: isSelected,
      onTap: onTap,
    );
  }
}
```

### 2. 画板系统

#### 2.1 画板数据模型
```dart
// lib/app/data/models/board_model.dart
@HiveType(typeId: 0)
class Board extends HiveObject {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  String title;
  
  @HiveField(2)
  String? description;
  
  @HiveField(3)
  List<Item> items;
  
  @HiveField(4)
  DateTime createdAt;
  
  @HiveField(5)
  DateTime updatedAt;
  
  // JSON序列化
  factory Board.fromJson(Map<String, dynamic> json) => _$BoardFromJson(json);
  Map<String, dynamic> toJson() => _$BoardToJson(this);
}
```

#### 2.2 画板视图模式
```dart
// lib/app/modules/board/controllers/board_view_controller.dart
class BoardViewController extends GetxController {
  final Rx<ViewMode> _currentMode = ViewMode.folder.obs;
  final RxList<String> _expandedNodes = <String>[].obs;
  
  ViewMode get currentMode => _currentMode.value;
  List<String> get expandedNodes => _expandedNodes;
  
  // 视图切换
  void changeViewMode(ViewMode mode) {
    _currentMode.value = mode;
    // 保存用户偏好
    Get.find<PreferencesService>().saveViewMode(mode);
  }
  
  // 节点展开/收起控制
  void toggleNode(String nodeId) {
    if (_expandedNodes.contains(nodeId)) {
      _expandedNodes.remove(nodeId);
    } else {
      _expandedNodes.add(nodeId);
    }
  }
}
```

#### 2.3 画板视图实现
```dart
// lib/app/modules/board/views/board_view.dart
class BoardView extends GetView<BoardController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: _buildAppBar(),
      body: Column(
        children: [
          _buildViewModeSelector(),
          Expanded(
            child: Obx(() => _buildContent()),
          ),
        ],
      ),
      floatingActionButton: _buildFAB(),
    );
  }
  
  Widget _buildContent() {
    switch (controller.viewMode) {
      case ViewMode.folder:
        return FolderView();
      case ViewMode.outline:
        return OutlineView();
      case ViewMode.thumbnail:
        return ThumbnailView();
      default:
        return FolderView();
    }
  }
}

// 文件夹视图实现
class FolderView extends GetView<BoardController> {
  @override
  Widget build(BuildContext context) {
    return ReorderableListView.builder(
      itemCount: controller.boards.length,
      onReorder: controller.reorderBoards,
      itemBuilder: (context, index) {
        final board = controller.boards[index];
        return BoardFolderItem(
          key: ValueKey(board.id),
          board: board,
          isExpanded: controller.isNodeExpanded(board.id),
          onToggle: () => controller.toggleNode(board.id),
        );
      },
    );
  }
}
```

### 3. 发现页面

#### 3.1 瀑布流实现
```dart
// lib/app/modules/discover/views/discover_view.dart
class DiscoverView extends GetView<DiscoverController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: RefreshIndicator(
        onRefresh: controller.refresh,
        child: Obx(() => MasonryGridView.count(
          crossAxisCount: 2,
          mainAxisSpacing: 4,
          crossAxisSpacing: 4,
          itemCount: controller.items.length,
          itemBuilder: (context, index) {
            // 检查是否需要加载更多
            if (index == controller.items.length - 1) {
              controller.loadMore();
            }
            return DiscoverItemCard(
              item: controller.items[index],
              onTap: () => controller.openDetail(index),
            );
          },
        )),
      ),
    );
  }
}

// lib/app/modules/discover/widgets/discover_item_card.dart
class DiscoverItemCard extends StatelessWidget {
  final DiscoverItem item;
  final VoidCallback onTap;
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: onTap,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            CachedNetworkImage(
              imageUrl: item.imageUrl,
              placeholder: (context, url) => ShimmerPlaceholder(),
              errorWidget: (context, url, error) => ErrorPlaceholder(),
            ),
            Padding(
              padding: EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(item.title),
                  Text(item.description),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### 3.2 筛选系统
```dart
// lib/app/modules/discover/controllers/filter_controller.dart
class FilterController extends GetxController {
  final RxList<String> _selectedCategories = <String>[].obs;
  final RxList<String> _selectedTags = <String>[].obs;
  
  List<String> get selectedCategories => _selectedCategories;
  List<String> get selectedTags => _selectedTags;
  
  void toggleCategory(String category) {
    if (_selectedCategories.contains(category)) {
      _selectedCategories.remove(category);
    } else {
      _selectedCategories.add(category);
    }
    _refreshContent();
  }
  
  void toggleTag(String tag) {
    if (_selectedTags.contains(tag)) {
      _selectedTags.remove(tag);
    } else {
      _selectedTags.add(tag);
    }
    _refreshContent();
  }
  
  Future<void> _refreshContent() async {
    Get.find<DiscoverController>().applyFilters(
      categories: _selectedCategories,
      tags: _selectedTags,
    );
  }
}
```

### 4. 关注系统

#### 4.1 关注功能实现
```dart
// lib/app/modules/follow/controllers/follow_controller.dart
class FollowController extends GetxController {
  final FollowRepository _repository;
  final RxList<FollowItem> _followedItems = <FollowItem>[].obs;
  
  List<FollowItem> get followedItems => _followedItems;
  
  Future<void> toggleFollow(String itemId) async {
    try {
      final isFollowed = _followedItems.any((item) => item.id == itemId);
      if (isFollowed) {
        await _repository.unfollow(itemId);
        _followedItems.removeWhere((item) => item.id == itemId);
      } else {
        final item = await _repository.follow(itemId);
        _followedItems.add(item);
      }
      
      // 更新本地存储
      await _updateLocalStorage();
      
    } catch (e) {
      Get.snackbar('Error', 'Failed to update follow status');
    }
  }
  
  Future<void> _updateLocalStorage() async {
    await Get.find<StorageService>().saveFollowedItems(_followedItems);
  }
}
```

#### 4.2 关注内容同步
```dart
// lib/app/data/repositories/follow_repository.dart
class FollowRepository {
  final ApiProvider _apiProvider;
  final StorageService _storageService;
  
  Future<void> syncFollowedItems() async {
    try {
      // 获取本地数据
      final localItems = await _storageService.getFollowedItems();
      
      // 获取服务器数据
      final serverItems = await _apiProvider.getFollowedItems();
      
      // 合并数据
      final mergedItems = _mergeFollowedItems(localItems, serverItems);
      
      // 更新本地存储
      await _storageService.saveFollowedItems(mergedItems);
      
      // 更新服务器
      await _apiProvider.syncFollowedItems(mergedItems);
      
    } catch (e) {
      // 错误处理
      throw FollowSyncException(e.toString());
    }
  }
  
  List<FollowItem> _mergeFollowedItems(
    List<FollowItem> local,
    List<FollowItem> server,
  ) {
    // 合并逻辑实现
    // ...
  }
}
```

### 5. 个人中心

#### 5.1 用户信息管理
```dart
// lib/app/modules/profile/controllers/profile_controller.dart
class ProfileController extends GetxController {
  final UserRepository _repository;
  final Rx<User> _user = User.empty().obs;
  
  User get user => _user.value;
  
  @override
  void onInit() {
    super.onInit();
    _loadUserInfo();
  }
  
  Future<void> updateUserInfo({
    String? name,
    String? avatar,
    String? bio,
  }) async {
    try {
      final updatedUser = await _repository.updateUser(
        userId: user.id,
        name: name,
        avatar: avatar,
        bio: bio,
      );
      _user.value = updatedUser;
      
      Get.snackbar('Success', 'Profile updated successfully');
    } catch (e) {
      Get.snackbar('Error', 'Failed to update profile');
    }
  }
  
  Future<void> _loadUserInfo() async {
    try {
      final userData = await _repository.getCurrentUser();
      _user.value = userData;
    } catch (e) {
      Get.snackbar('Error', 'Failed to load user info');
    }
  }
}
```

#### 5.2 个人中心视图
```dart
// lib/app/modules/profile/views/profile_view.dart
class ProfileView extends GetView<ProfileController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('个人中心'),
        actions: [
          IconButton(
            icon: Icon(Icons.settings),
            onPressed: () => Get.toNamed(Routes.SETTINGS),
          ),
        ],
      ),
      body: Obx(() => SingleChildScrollView(
        child: Column(
          children: [
            _buildUserHeader(),
            _buildStatistics(),
            _buildActionList(),
          ],
        ),
      )),
    );
  }
  
  Widget _buildUserHeader() {
    return UserHeaderWidget(
      user: controller.user,
      onAvatarTap: () => controller.updateAvatar(),
      onNameTap: () => controller.editName(),
    );
  }
  
  Widget _buildStatistics() {
    return StatisticsWidget(
      boards: controller.user.boardsCount,
      followers: controller.user.followersCount,
      following: controller.user.followingCount,
    );
  }
  
  Widget _buildActionList() {
    return Column(
      children: [
        ActionItem(
          icon: Icons.collections,
          title: '我的画板',
          onTap: () => Get.toNamed(Routes.MY_BOARDS),
        ),
        ActionItem(
          icon: Icons.favorite,
          title: '我的收藏',
          onTap: () => Get.toNamed(Routes.FAVORITES),
        ),
        ActionItem(
          icon: Icons.history,
          title: '浏览历史',
          onTap: () => Get.toNamed(Routes.HISTORY),
        ),
      ],
    );
  }
}
```