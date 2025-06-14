<template>
  <div class="element-popup-container" :class="{ 'newtab-mode': isNewTabMode }">
    <!-- 固定头部区域 -->
    <div class="fixed-header">
      <!-- 搜索头部 -->
      <el-card class="search-header-card" :body-style="{ padding: '16px' }">
        <!-- 搜索输入框 -->
        <div class="search-input-container" style="display: flex; gap: 8px; align-items: center">
          <el-input
            v-model="searchQuery"
            placeholder="搜索收藏夹、历史记录和下载文件..."
            size="large"
            clearable
            @input="handleSearchInput"
            @keydown.enter="handleSearchNow"
            ref="searchInput"
            style="flex: 1"
          >
            <template #prefix>
              <el-icon><Search /></el-icon>
            </template>
          </el-input>
          <el-button 
            size="large" 
            :icon="Setting" 
            circle
            @click="openSettings"
            title="打开设置"
          />
        </div>
        
        <!-- 搜索历史气泡 -->
        <div v-if="searchHistory.length > 0" class="search-history">
          <el-tag
            v-for="item in searchHistory" 
            :key="item.timestamp"
            type="info"
            effect="plain"
            size="small"
            class="history-tag"
            @click="selectHistoryItem(item.query)"
          >
            {{ item.query }}
          </el-tag>
        </div>
        
        <!-- 搜索选项 - 水平对齐 -->
        <div class="search-options">
          <el-row :gutter="4" align="middle" class="controls-row" justify="space-between">
            <!-- 数据源多选 -->
            <el-col :span="10" class="filter-control">
              <div class="control-item">
                <span class="control-label">搜索项:</span>
                <el-select
                  v-model="selectedDataSources"
                  multiple
                  collapse-tags
                  collapse-tags-tooltip
                  size="small"
                  class="control-select"
                  @change="updateSearchOptions"
                >
                  <el-option label="书签" value="bookmarks" />
                  <el-option label="历史记录" value="history" />
                  <el-option label="下载文件" value="downloads" />
                </el-select>
              </div>
            </el-col>
            <!-- 时间筛选 -->
            <el-col :span="10" class="filter-control">
              <div class="control-item">
                <span class="control-label">时间:</span>
                <el-select 
                  v-model="searchOptions.timeFilter" 
                  size="small" 
                  class="control-select"
                >
                  <el-option label="全部时间" value="all" />
                  <el-option label="今天" value="today" />
                  <el-option label="本周" value="week" />
                  <el-option label="本月" value="month" />
                </el-select>
              </div>
            </el-col>
            
            <!-- 排序选择 -->
            <el-col :span="10" class="filter-control">
              <div class="control-item">
                <span class="control-label">排序:</span>
                <el-select 
                  v-model="searchOptions.sortBy" 
                  size="small" 
                  class="control-select"
                >
                  <el-option label="相关性" value="relevance" />
                  <el-option label="最近访问" value="recent" />
                  <el-option label="访问频率" value="frequency" />
                </el-select>
              </div>
            </el-col>
          </el-row>
        </div>
      </el-card>

      <!-- 搜索统计 -->
      <div v-if="searchStats" class="search-stats">
        <el-space :size="8" wrap>
          <el-tag size="small" type="info" effect="plain">
            找到 {{ searchStats.totalResults }} 个结果
          </el-tag>
          <el-tag v-if="searchStats.bookmarkCount > 0" size="small" type="success" effect="plain">
            书签 {{ searchStats.bookmarkCount }}
          </el-tag>
          <el-tag v-if="searchStats.historyCount > 0" size="small" type="warning" effect="plain">
            历史 {{ searchStats.historyCount }}
          </el-tag>
          <el-tag v-if="searchStats.downloadCount > 0" size="small" type="info" effect="plain">
            下载 {{ searchStats.downloadCount }}
          </el-tag>
          <el-tag size="small" effect="plain">
            {{ searchStats.uniqueDomains }} 个域名
          </el-tag>
          <el-tag size="small" effect="plain">
            {{ searchStats.searchTime }}ms
          </el-tag>
        </el-space>
      </div>
    </div>

    <!-- 可滚动内容区域 -->
    <div class="scrollable-content">
      <!-- 加载状态 -->
      <div v-if="isLoading" v-loading="true" class="loading-container">
        <el-empty description="搜索中..." :image-size="60" />
      </div>

      <!-- 搜索结果 -->
      <div v-else-if="hasResults" class="results-container">
        <el-card
          v-for="(group, domain) in searchResults"
          :key="domain"
          class="domain-group-card"
          :body-style="{ padding: '12px' }"
          shadow="hover"
        >
          <template #header>
            <div class="domain-header">
              <img :src="getFaviconUrl(String(domain))" :alt="String(domain)" class="domain-favicon" />
              <span class="domain-name">{{ domain }}</span>
              <el-tag size="small" type="primary" effect="plain">{{ group.totalCount }}</el-tag>
            </div>
          </template>
          
          <div class="result-items">
            <el-card
              v-for="item in group.items"
              :key="item.id"
              class="result-item-card"
              :class="{ 'selected': selectedItem === item.id }"
              :data-id="item.id"
              :body-style="{ padding: '12px' }"
              shadow="hover"
              @click="selectAndOpenItem(item)"
            >
              <div class="result-item-content">
                <div class="item-icon">
                  {{ getItemIcon(item.type) }}
                </div>
                <div class="item-content">
                  <div class="item-title" :title="item.title">{{ item.title }}</div>
                  <div class="item-url" :title="item.url">{{ item.url }}</div>
                  <div class="item-meta">
                    <el-tag v-if="item.folderName" size="small" type="warning" effect="plain">
                      📁 {{ item.folderName }}
                    </el-tag>
                    <el-tag v-if="item.visitCount && item.type !== 'download'" size="small" type="info" effect="plain">
                      {{ item.visitCount }} 次访问
                    </el-tag>
                    <el-tag v-if="item.fileSize && item.type === 'download'" size="small" type="success" effect="plain">
                      {{ formatFileSize(item.fileSize) }}
                    </el-tag>
                    <span v-if="item.lastVisited" class="last-visited">
                      {{ formatDate(item.lastVisited) }}
                    </span>
                    <el-tag v-if="item.type === 'download' && !item.exists" size="small" type="danger" effect="dark">
                      ⚠️ 文件不存在
                    </el-tag>
                  </div>
                </div>
                <div class="item-actions">
                  <el-button 
                    v-if="item.type === 'history'"
                    size="small"
                    type="primary"
                    :icon="Star"
                    @click.stop="showBookmarkDialog(item)"
                  >
                    收藏
                  </el-button>
                  <el-button 
                    v-if="item.type === 'download'"
                    size="small"
                    type="success"
                    :icon="FolderOpened"
                    @click.stop="showDownloadFile(item.id)"
                  >
                    显示文件目录
                  </el-button>
                </div>
              </div>
            </el-card>
          </div>
        </el-card>
      </div>

      <!-- 空状态 -->
      <div v-else-if="searchQuery && !isLoading" class="empty-state">
        <el-empty description="未找到匹配的结果" :image-size="80">
          <template #description>
            <p>未找到匹配的结果</p>
            <p>尝试不同的关键词或调整搜索选项</p>
          </template>
        </el-empty>
      </div>

      <!-- 初始状态 -->
      <div v-else class="initial-state">
        <el-card class="welcome-card" :body-style="{ padding: '32px', textAlign: 'center' }">
          <div class="welcome-tips">
            <div class="tip-item">
              <el-icon class="tip-icon"><MagicStick /></el-icon>
              <span>支持模糊搜索</span>
            </div>
            <div class="tip-item">
              <el-icon class="tip-icon"><Collection /></el-icon>
              <span>结果按域名分组显示</span>
            </div>
            <div class="tip-item">
              <el-icon class="tip-icon"><Mouse /></el-icon>
              <span>单击直接打开链接</span>
            </div>
            <div class="tip-item">
              <el-icon class="tip-icon"><Star /></el-icon>
              <span>历史记录可添加到书签</span>
            </div>
            <div class="tip-item">
              <el-icon class="tip-icon"><Download /></el-icon>
              <span>支持搜索下载文件</span>
            </div>
            <div v-if="mainShortcut" class="tip-item">
              <el-icon class="tip-icon"><Tools /></el-icon>
              <span>快捷键: {{ mainShortcut }}</span>
            </div>
          </div>
        </el-card>
      </div>
    </div>

    <!-- 快捷键提示 -->
    <div class="shortcuts">
      <el-tag size="small" effect="plain">{{ navigationKeys.open }} 打开</el-tag>
      <el-tag size="small" effect="plain">{{ navigationKeys.up }}{{ navigationKeys.down }} 选择</el-tag>
      <el-tag size="small" effect="plain">Esc 关闭</el-tag>
    </div>
  </div>

    <!-- 书签对话框 -->
    <el-dialog
      v-model="bookmarkDialog.show"
      title="添加到书签"
      width="500px"
      :before-close="closeBookmarkDialog"
    >
      <el-form 
        :model="bookmarkDialog" 
        label-width="80px"
        :rules="bookmarkRules"
        ref="bookmarkForm"
      >
        <el-form-item label="标题" prop="title">
          <el-input v-model="bookmarkDialog.title" />
        </el-form-item>
        <el-form-item label="URL">
          <el-input v-model="bookmarkDialog.url" readonly />
        </el-form-item>
        <el-form-item label="文件夹">
          <el-tree-select
            v-model="bookmarkDialog.parentId"
            :data="bookmarkFoldersTree"
            :props="{ label: 'title', value: 'id', children: 'children' }"
            filterable
            placeholder="请选择文件夹"
            style="width: 100%"
            check-strictly
            clearable
          />
        </el-form-item>
      </el-form>
      
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="closeBookmarkDialog">取消</el-button>
          <el-button type="primary" @click="validateAndSaveBookmark">保存</el-button>
        </span>
      </template>
    </el-dialog>
</template>

<script setup lang="ts">
/// <reference types="chrome" />
import {
  Collection,
  Download,
  FolderOpened,
  MagicStick,
  Mouse,
  Search, Star,
  Tools,
  TopRight,
  Setting
} from '@element-plus/icons-vue';
import { computed, nextTick, onMounted, onUnmounted, reactive, ref, watch } from 'vue';
import {
  formatFileSize,
  getFaviconUrl,
  openDownloadFile,
  openUrl,
  searchBookmarksAndHistory,
  SearchHistoryManager,
  showDownloadFile as showDownloadFileInExplorer
} from '../utils/search';
import { formatShortcut, getNavigationKeys, getShortcut } from '../utils/shortcuts.ts';
import type {
  GroupedSearchResults,
  SearchHistoryItem,
  SearchOptions,
  SearchResultItem,
  SearchStats
} from '../utils/types';

// 检测是否为新标签页模式
const isNewTabMode = computed(() => {
  return window.location.pathname.includes('single_tab.html');
});

// 响应式数据
const searchQuery = ref('');
const searchResults = ref<GroupedSearchResults>({});
const searchStats = ref<SearchStats | null>(null);
const isLoading = ref(false);
const selectedItem = ref<string | null>(null);
const searchInput = ref<HTMLInputElement>();
const searchHistory = ref<SearchHistoryItem[]>([]);

// 防抖相关
const searchTimeout = ref<number | null>(null);
const DEBOUNCE_DELAY = 300; 

// 选中的数据源 - 默认全选
const selectedDataSources = ref<string[]>(['bookmarks', 'history', 'downloads']);

// 快捷键显示
const mainShortcut = ref('');
const navigationKeys = ref(getNavigationKeys());

// 键盘导航配置（从设置中加载）
const navigationConfig = reactive({
  up: 'ArrowUp',
  down: 'ArrowDown',
  open: 'Enter',
  close: 'Escape'
});

// 书签对话框状态
const bookmarkDialog = reactive({
  show: false,
  title: '',
  url: '',
  parentId: '',
  item: null as SearchResultItem | null
});

// 书签表单验证规则
const bookmarkRules = reactive({
  title: [
    { required: true, message: '请输入书签标题', trigger: 'blur' },
    { min: 1, max: 100, message: '长度在1到100个字符', trigger: 'blur' }
  ]
});

// 书签表单引用
const bookmarkForm = ref();

// 书签文件夹列表
const bookmarkFolders = ref<{id: string, title: string}[]>([]);

// 书签文件夹树形结构
const bookmarkFoldersTree = ref<any[]>([]);

// 验证并保存书签
const validateAndSaveBookmark = async () => {
  try {
    await bookmarkForm.value.validate();
    await saveBookmark();
  } catch (error) {
    console.error('表单验证失败:', error);
  }
};

// 搜索选项
const searchOptions = reactive<SearchOptions>({
  query: '',
  includeBookmarks: true,
  includeHistory: true,
  includeDownloads: true,
  maxResults: 50,
  sortBy: 'relevance',
  timeFilter: 'all'
});

// 计算属性
const hasResults = computed(() => {
  return Object.keys(searchResults.value).length > 0;
});

// 获取项目图标
const getItemIcon = (type: string): string => {
  switch (type) {
    case 'bookmark': return '🔖';
    case 'history': return '🕒';
    case 'download': return '📥';
    default: return '📄';
  }
};

// 获取选中数据源的显示文本
const getSelectedText = (): string => {
  const labels: Record<string, string> = {
    'bookmarks': '书签',
    'history': '历史记录',
    'downloads': '下载文件'
  };
  return selectedDataSources.value.map(key => labels[key]).join('、');
};

// 更新搜索选项
const updateSearchOptions = () => {
  searchOptions.includeBookmarks = selectedDataSources.value.includes('bookmarks');
  searchOptions.includeHistory = selectedDataSources.value.includes('history');
  searchOptions.includeDownloads = selectedDataSources.value.includes('downloads');
  
  // 如果当前有搜索查询，重新搜索
  if (searchQuery.value.trim()) {
    handleSearchNow();
  }
};

// 处理输入事件（带防抖）
const handleSearchInput = () => {
  // 清除之前的定时器
  if (searchTimeout.value !== null) {
    window.clearTimeout(searchTimeout.value);
    searchTimeout.value = null;
  }
  
  // 如果输入为空，立即清空结果
  if (!searchQuery.value.trim()) {
    searchResults.value = {};
    searchStats.value = null;
    return;
  }
  
  // 设置新的防抖定时器
  searchTimeout.value = window.setTimeout(() => {
    handleSearch();
  }, DEBOUNCE_DELAY);
};

// 立即搜索（回车或手动触发）
const handleSearchNow = () => {
  // 清除防抖定时器
  if (searchTimeout.value !== null) {
    window.clearTimeout(searchTimeout.value);
    searchTimeout.value = null;
  }
  
  handleSearch();
};

// 搜索处理函数
const handleSearch = async () => {
  if (!searchQuery.value.trim()) {
    searchResults.value = {};
    searchStats.value = null;
    return;
  }

  isLoading.value = true;
  
  try {
    const options = {
      ...searchOptions,
      query: searchQuery.value.trim()
    };
    
    const { results, stats } = await searchBookmarksAndHistory(options);
    searchResults.value = results;
    searchStats.value = stats;
    
    // 保存搜索历史
    await SearchHistoryManager.saveSearchHistory(searchQuery.value.trim());
    await loadSearchHistory();
  } catch (error) {
    console.error('搜索失败:', error);
  } finally {
    isLoading.value = false;
  }
};

// 选择历史记录项
const selectHistoryItem = (query: string) => {
  searchQuery.value = query;
  handleSearchNow();
};

// 加载搜索历史
const loadSearchHistory = async () => {
  try {
    searchHistory.value = await SearchHistoryManager.getSearchHistory();
  } catch (error) {
    console.error('加载搜索历史失败:', error);
  }
};

// 监听搜索选项变化（除了数据源选择，因为那个有单独的处理）
watch(() => [searchOptions.timeFilter, searchOptions.sortBy], () => {
  if (searchQuery.value.trim()) {
    handleSearchNow();
  }
}, { deep: true });

// 回车处理
const handleEnter = () => {
  if (selectedItem.value) {
    const item = findItemById(selectedItem.value);
    if (item) {
      openItem(item);
    }
  } else {
    // 打开第一个结果
    const firstGroup = Object.values(searchResults.value)[0];
    if (firstGroup && firstGroup.items.length > 0) {
      openItem(firstGroup.items[0]);
    }
  }
};

// 选择并打开项目（单击）
const selectAndOpenItem = async (item: SearchResultItem) => {
  selectedItem.value = item.id;
  await openItem(item);
};

// 打开项目
const openItem = async (item: SearchResultItem) => {
  if (item.type === 'download') {
    await openDownloadFile(item.id);
  } else {
    await openUrl(item.url);
    window.close(); // 关闭弹窗
  }
};

// 显示下载文件
const showDownloadFile = async (downloadId: string) => {
  await showDownloadFileInExplorer(downloadId);
};

// 根据ID查找项目
const findItemById = (itemId: string): SearchResultItem | null => {
  for (const group of Object.values(searchResults.value)) {
    const item = group.items.find(item => item.id === itemId);
    if (item) return item;
  }
  return null;
};

// 格式化日期
const formatDate = (timestamp: number): string => {
  const date = new Date(timestamp);
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffMins = Math.floor(diffMs / (1000 * 60));
  const diffHours = Math.floor(diffMs / (1000 * 60 * 60));
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));

  if (diffMins < 60) {
    return `${diffMins} 分钟前`;
  } else if (diffHours < 24) {
    return `${diffHours} 小时前`;
  } else if (diffDays < 7) {
    return `${diffDays} 天前`;
  } else {
    return date.toLocaleDateString('zh-CN');
  }
};

// 获取书签栏ID的辅助函数
const getBookmarkBarId = (bookmarks: chrome.bookmarks.BookmarkTreeNode[]): string | null => {
  // 书签栏通常是第一个根级别文件夹
  for (const node of bookmarks) {
    if (node.children) {
      for (const child of node.children) {
        if (child.title === '书签栏' || child.title === 'Bookmarks bar' || child.title === 'Bookmarks') {
          return child.id;
        }
      }
      // 如果没找到特定名称，返回第一个文件夹（通常是书签栏）
      if (node.children.length > 0 && !node.children[0].url) {
        return node.children[0].id;
      }
    }
  }
  return null;
};

// 递归查找文件夹ID
const findFolderById = (folder: any, targetId: string): boolean => {
  if (folder.id === targetId) return true;
  if (folder.children) {
    return folder.children.some((child: any) => findFolderById(child, targetId));
  }
  return false;
};

// 显示书签对话框
const showBookmarkDialog = async (item: SearchResultItem) => {
  bookmarkDialog.item = item;
  bookmarkDialog.title = item.title;
  bookmarkDialog.url = item.url;
  
  // 恢复上次选择的文件夹，如果没有则默认选择书签栏
  try {
    const result = await chrome.storage.local.get(['lastSelectedFolder']);
    const lastFolder = result.lastSelectedFolder;
    
    if (lastFolder && bookmarkFoldersTree.value.some(folder => findFolderById(folder, lastFolder))) {
      bookmarkDialog.parentId = lastFolder;
    } else {
      // 获取书签栏ID
      const bookmarks = await chrome.bookmarks.getTree();
      const bookmarkBarId = getBookmarkBarId(bookmarks);
      bookmarkDialog.parentId = bookmarkBarId || '';
    }
  } catch (error) {
    console.error('获取上次选择的文件夹失败:', error);
    // 默认选择书签栏
    try {
      const bookmarks = await chrome.bookmarks.getTree();
      const bookmarkBarId = getBookmarkBarId(bookmarks);
      bookmarkDialog.parentId = bookmarkBarId || '';
    } catch (err) {
      console.error('获取书签栏失败:', err);
      bookmarkDialog.parentId = '';
    }
  }
  
  bookmarkDialog.show = true;
};

// 关闭书签对话框
const closeBookmarkDialog = () => {
  bookmarkDialog.show = false;
  bookmarkDialog.title = '';
  bookmarkDialog.url = '';
  bookmarkDialog.parentId = '';
  bookmarkDialog.item = null;
};

// 保存书签
const saveBookmark = async () => {
  if (!bookmarkDialog.title || !bookmarkDialog.url) {
    alert('请填写标题和URL');
    return;
  }
  
  try {
    const bookmarkData: chrome.bookmarks.CreateDetails = {
      title: bookmarkDialog.title,
      url: bookmarkDialog.url
    };
    
    if (bookmarkDialog.parentId) {
      bookmarkData.parentId = bookmarkDialog.parentId;
      // 保存用户选择的文件夹
      await chrome.storage.local.set({ lastSelectedFolder: bookmarkDialog.parentId });
    }
    
    await chrome.bookmarks.create(bookmarkData);
    alert('书签添加成功！');
    closeBookmarkDialog();
  } catch (error) {
    console.error('添加书签失败:', error);
    alert('添加书签失败，请重试');
  }
};

// 获取书签文件夹
const loadBookmarkFolders = async () => {
  try {
    const bookmarks = await chrome.bookmarks.getTree();
    const folders: {id: string, title: string}[] = [];
    const foldersTree: any[] = [];
    
    // 跳过虚拟根节点，只处理实际的书签文件夹
    const actualFolders = bookmarks[0]?.children || [];
    
    const traverseBookmarks = (nodes: chrome.bookmarks.BookmarkTreeNode[], depth = 0, parentArray: any[] = foldersTree) => {
      for (const node of nodes) {
        if (!node.url) { // 文件夹
          const prefix = '  '.repeat(depth);
          // 为平铺结构添加
          folders.push({
            id: node.id,
            title: `${prefix}${node.title || '未命名文件夹'}`
          });
          
          // 为树形结构添加
          const treeNode = {
            id: node.id,
            title: node.title || '未命名文件夹',
            children: []
          };
          parentArray.push(treeNode);
          
          if (node.children) {
            traverseBookmarks(node.children, depth + 1, treeNode.children);
          }
        }
      }
    };
    
    traverseBookmarks(actualFolders);
    bookmarkFolders.value = folders;
    bookmarkFoldersTree.value = foldersTree;
  } catch (error) {
    console.error('获取书签文件夹失败:', error);
  }
};

// 加载快捷键配置
const loadShortcutConfig = async () => {
  try {
    // 加载主快捷键
    const shortcut = await getShortcut('_execute_action');
    mainShortcut.value = formatShortcut(shortcut);
    
    // 可选：加载备用快捷键
    const altShortcut = await getShortcut('open-search-alt');
    if (altShortcut && !mainShortcut.value) {
      mainShortcut.value = formatShortcut(altShortcut);
    }
  } catch (error) {
    console.error('加载快捷键配置失败:', error);
    mainShortcut.value = 'Ctrl+Shift+S'; // 默认值
  }
};

// 加载搜索设置
const loadSearchSettings = async () => {
  try {
    const result = await chrome.storage.local.get(['searchSettings']);
    if (result.searchSettings) {
      // 应用搜索设置到当前的搜索选项
      if (result.searchSettings.defaultMaxResults) {
        searchOptions.maxResults = Number(result.searchSettings.defaultMaxResults);
      }
      if (result.searchSettings.defaultSortBy) {
        searchOptions.sortBy = result.searchSettings.defaultSortBy;
      }
      
      console.log('已加载搜索设置:', result.searchSettings);
    }
  } catch (error) {
    console.error('加载搜索设置失败:', error);
  }
};

// 加载导航设置
const loadNavigationSettings = async () => {
  try {
    const result = await chrome.storage.local.get(['navigationSettings']);
    if (result.navigationSettings) {
      Object.assign(navigationConfig, result.navigationSettings);
      console.log('已加载导航设置:', result.navigationSettings);
    }
  } catch (error) {
    console.error('加载导航设置失败:', error);
  }
};

// 键盘导航
const handleKeyDown = (event: KeyboardEvent) => {
  if (!hasResults.value) return;
  
  const allItems = Object.values(searchResults.value)
    .flatMap(group => group.items);
  
  if (!allItems.length) return;
  
  const currentIndex = selectedItem.value ? 
    allItems.findIndex(item => item.id === selectedItem.value) : -1;
  
  switch (event.code) {
    case navigationConfig.down:
      event.preventDefault();
      const nextIndex = currentIndex < allItems.length - 1 ? currentIndex + 1 : 0;
      selectedItem.value = allItems[nextIndex].id;
      // 滚动到可见区域，但确保在可滚动容器内
      const nextElement = document.querySelector(`[data-id="${allItems[nextIndex].id}"]`);
      if (nextElement) {
        const scrollableContainer = document.querySelector('.scrollable-content');
        if (scrollableContainer) {
          const containerRect = scrollableContainer.getBoundingClientRect();
          const elementRect = nextElement.getBoundingClientRect();
          
          if (elementRect.bottom > containerRect.bottom) {
            nextElement.scrollIntoView({ block: 'end', behavior: 'smooth' });
          } else if (elementRect.top < containerRect.top) {
            nextElement.scrollIntoView({ block: 'start', behavior: 'smooth' });
          }
        }
      }
      break;
    case navigationConfig.up:
      event.preventDefault();
      const prevIndex = currentIndex > 0 ? currentIndex - 1 : allItems.length - 1;
      selectedItem.value = allItems[prevIndex].id;
      // 滚动到可见区域，但确保在可滚动容器内
      const prevElement = document.querySelector(`[data-id="${allItems[prevIndex].id}"]`);
      if (prevElement) {
        const scrollableContainer = document.querySelector('.scrollable-content');
        if (scrollableContainer) {
          const containerRect = scrollableContainer.getBoundingClientRect();
          const elementRect = prevElement.getBoundingClientRect();
          
          if (elementRect.top < containerRect.top) {
            prevElement.scrollIntoView({ block: 'start', behavior: 'smooth' });
          } else if (elementRect.bottom > containerRect.bottom) {
            prevElement.scrollIntoView({ block: 'end', behavior: 'smooth' });
          }
        }
      }
      break;
    case navigationConfig.open:
      if (selectedItem.value) {
        const item = findItemById(selectedItem.value);
        if (item) openItem(item);
      }
      break;
    case navigationConfig.close:
      window.close();
      break;
  }
};

// 监听storage变化
const handleStorageChange = (changes: Record<string, chrome.storage.StorageChange>) => {
  // 监听搜索设置变化
  if (changes.searchSettings) {
    const newSettings = changes.searchSettings.newValue;
    if (newSettings) {
      if (newSettings.defaultMaxResults) {
        searchOptions.maxResults = Number(newSettings.defaultMaxResults);
      }
      if (newSettings.defaultSortBy) {
        searchOptions.sortBy = newSettings.defaultSortBy;
      }
      console.log('搜索设置已更新:', newSettings);
      
      // 如果有搜索查询，重新搜索以应用新设置
      if (searchQuery.value.trim()) {
        handleSearchNow();
      }
    }
  }
  
  // 监听导航设置变化
  if (changes.navigationSettings) {
    const newSettings = changes.navigationSettings.newValue;
    if (newSettings) {
      Object.assign(navigationConfig, newSettings);
      console.log('导航设置已更新:', newSettings);
    }
  }
};

// 组件挂载
onMounted(async () => {
  // 加载快捷键配置
  await loadShortcutConfig();
  
  // 加载搜索设置
  await loadSearchSettings();
  
  // 加载导航设置
  await loadNavigationSettings();
  
  // 加载搜索历史
  await loadSearchHistory();
  
  // 聚焦搜索框
  await nextTick();
  searchInput.value?.focus();
  
  // 绑定键盘事件
  document.addEventListener('keydown', handleKeyDown);
  
  // 监听storage变化
  chrome.storage.onChanged.addListener(handleStorageChange);
  
  // 加载书签文件夹
  await loadBookmarkFolders();
});

// 组件卸载
onUnmounted(() => {
  // 清理防抖定时器
  if (searchTimeout.value !== null) {
    window.clearTimeout(searchTimeout.value);
  }
  
  document.removeEventListener('keydown', handleKeyDown);
  chrome.storage.onChanged.removeListener(handleStorageChange);
});

// 打开设置页面
const openSettings = () => {
  chrome.tabs.create({
    url: chrome.runtime.getURL('settings.html')
  });
};

// 在新标签页打开搜索界面
const openInNewTab = () => {
  const params = new URLSearchParams();
  params.set('q', searchQuery.value);
  params.set('bookmarks', searchOptions.includeBookmarks.toString());
  params.set('history', searchOptions.includeHistory.toString());
  params.set('downloads', searchOptions.includeDownloads.toString());
  params.set('sort', searchOptions.sortBy);
  params.set('time', searchOptions.timeFilter);
  
  chrome.tabs.create({
    url: chrome.runtime.getURL(`single_tab.html?${params.toString()}`)
  });
};

// 导出函数供模板使用
defineExpose({
  getFaviconUrl,
  formatFileSize,
  openInNewTab
});
</script>

<style lang="less" scoped>
@import '../entrypoints/styles/element-popup.less';
</style>
