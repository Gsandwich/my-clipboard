<script>
  import { convertFileSrc, invoke } from '@tauri-apps/api/core';
  import { afterUpdate, onMount } from 'svelte';
  import { fade, fly, scale } from 'svelte/transition';

  const DEFAULT_CATEGORY = '全部';
  const HISTORY_KEY = 'clip_v5_split';
  const LAST_DATA_KEY = 'clip_v5_last_data';
  const PASSWORDS_KEY = 'clip_v5_passwords';
  const CATEGORIES_KEY = 'clip_v5_categories';
  const LANG_KEY = 'clip_v6_lang';

  const I18N = {
    en: {
      clipboard: 'Clipboard',
      all: 'All',
      passwords: 'Passwords',
      searchClipboard: 'Search stored clipboard...',
      searchPasswords: 'Search stored passwords...',
      imageHistory: 'Image History',
      textHistory: 'Text History',
      myPasswords: 'My Passwords',
      clearAll: 'Clear All',
      copy: 'Copy',
      copied: 'Copied',
      imageCopied: 'Image Copied',
      imagePasted: 'Image Pasted',
      textPasted: 'Text Pasted',
      copyFailed: 'Failed to copy',
      saved: 'Saved',
      noImages: 'No images yet',
      noImagesSub: 'Images you copy will appear here for quick access.',
      noText: 'No text records in this category.',
      noPasswords: 'No password items yet.',
      newCategory: 'New category...',
      rename: 'Rename',
      deleteCategory: 'Delete Category',
      preview: 'Preview',
      addNew: 'Add New',
      forWhat: 'For what',
      purposePlaceholder: 'Enter purpose or website',
      username: 'Username',
      usernamePlaceholder: 'Enter username or email',
      password: 'Password',
      passwordPlaceholder: 'Enter password',
      save: 'Save',
      userLabel: 'User',
      passLabel: 'Pass',
      switchLanguage: 'Switch language'
    },
    zh: {
      clipboard: '剪贴板',
      all: '全部',
      passwords: '密码本',
      searchClipboard: '搜索剪贴板记录...',
      searchPasswords: '搜索密码记录...',
      imageHistory: '图片历史',
      textHistory: '文本历史',
      myPasswords: '我的密码',
      clearAll: '清空',
      copy: '复制',
      copied: '已复制',
      imageCopied: '图片已复制',
      imagePasted: '图片已粘贴',
      textPasted: '文本已粘贴',
      copyFailed: '复制失败',
      saved: '已保存',
      noImages: '还没有图片',
      noImagesSub: '你复制的图片会出现在这里，方便随时取用。',
      noText: '这个分类里还没有文本记录。',
      noPasswords: '还没有密码记录。',
      newCategory: '新分类...',
      rename: '重命名',
      deleteCategory: '删除分类',
      preview: '预览',
      addNew: '新增记录',
      forWhat: '用途',
      purposePlaceholder: '输入用途或网站名',
      username: '用户名',
      usernamePlaceholder: '输入用户名或邮箱',
      password: '密码',
      passwordPlaceholder: '输入密码',
      save: '保存',
      userLabel: '账号',
      passLabel: '密码',
      switchLanguage: '切换语言'
    }
  };

  /** @type {'en' | 'zh'} */
  let lang = 'zh';
  $: t = I18N[lang];

  /** @param {'en' | 'zh'} next */
  function setLang(next) {
    lang = next;
    localStorage.setItem(LANG_KEY, next);
    if (typeof document !== 'undefined') {
      document.documentElement.lang = next === 'zh' ? 'zh-CN' : 'en';
    }
  }

  /** @param {string} [msg] */
  function timestamp() {
    return new Date().toLocaleString(lang === 'zh' ? 'zh-CN' : 'en-US');
  }

  /**
   * @typedef {Object} HistoryItem
   * @property {'text' | 'image' | string} type
   * @property {string} content
   * @property {number} id
   * @property {string} timestamp
   * @property {string=} category
   */

  /**
   * @typedef {Object} PasswordItem
   * @property {number} id
   * @property {string} title
   * @property {string} username
   * @property {string} password
   * @property {boolean} showPass
   */

  let activeTab = 'clipboard';
  let showToast = false;
  let toastMsg = 'Copied!';

  let searchQuery = '';
  /** @type {HistoryItem[]} */
  let history = [];
  let lastData = '';

  let pwdSearchQuery = '';
  /** @type {PasswordItem[]} */
  let passwords = [];
  let newPwdTitle = '';
  let newPwdUser = '';
  let newPwdPass = '';

  let showNewPwd = false;
  let isAppCopying = false;

  let categories = [DEFAULT_CATEGORY];
  let activeCategory = DEFAULT_CATEGORY;
  let isAddingCat = false;
  let newCatName = '';

  let showContextMenu = false;
  let menuX = 0;
  let menuY = 0;
  let targetCategory = '';

  let showImageMenu = false;
  let imageMenuX = 0;
  let imageMenuY = 0;
  /** @type {HistoryItem | null} */
  let targetImage = null;
  /** @type {HistoryItem | null} */
  let previewImage = null;
  let previewScale = 1;

  let isRenamingCat = false;
  let renameCatName = '';

  /** @type {HTMLButtonElement | undefined} */
  let clipboardTabEl;
  /** @type {HTMLButtonElement | undefined} */
  let passwordTabEl;
  let tabLineX = 0;
  let tabLineW = 0;

  function syncTabIndicator() {
    const activeTabEl = activeTab === 'clipboard' ? clipboardTabEl : passwordTabEl;
    if (!activeTabEl) return;

    const nextX = activeTabEl.offsetLeft;
    const nextW = activeTabEl.offsetWidth;
    if (nextX !== tabLineX) tabLineX = nextX;
    if (nextW !== tabLineW) tabLineW = nextW;
  }

  /** @param {WheelEvent & { currentTarget: HTMLElement }} event */
  function handleCategoryWheel(event) {
    const delta = Math.abs(event.deltaY) > Math.abs(event.deltaX) ? event.deltaY : event.deltaX;
    event.currentTarget.scrollLeft += delta;
  }

  /** @param {number} nextScale */
  function clampPreviewScale(nextScale) {
    return Math.min(4, Math.max(0.2, Number(nextScale.toFixed(2))));
  }

  afterUpdate(syncTabIndicator);

  /** @param {string} content */
  function imageClipboardSignature(content) {
    if (!content) return '';
    const payload = content.includes('|') ? content.split('|')[1] : content;
    if (!payload) return '';
    const head = payload.slice(0, 32);
    const tail = payload.slice(-32);
    return `image:${payload.length}:${head}:${tail}`;
  }

  /**
   * @param {string} type
   * @param {string} content
   */
  function clipboardSignature(type, content) {
    if (type === 'image') return imageClipboardSignature(content);
    return content || '';
  }

  /** @param {unknown} content */
  function isFileImageContent(content) {
    return typeof content === 'string' && content.startsWith('file|');
  }

  /** @param {string} rawContent */
  async function persistImageContent(rawContent) {
    if (!rawContent) return '';
    const base64 = rawContent.includes('|') ? rawContent.split('|')[1] : rawContent;
    if (!base64) return '';
    const path = /** @type {string} */ (await invoke('persist_history_image', { base64 }));
    return `file|${path}`;
  }

  /** @param {HistoryItem[]} items */
  function collectImagePaths(items) {
    return items
      .filter((item) => item.type !== 'text' && isFileImageContent(item.content))
      .map((item) => item.content.slice(5));
  }

  /** @param {HistoryItem[]} items */
  async function cleanupImageFilesFromItems(items) {
    const paths = collectImagePaths(items);
    if (paths.length === 0) return;
    try {
      await invoke('delete_history_images', { paths });
    } catch {
      // ignore file cleanup failures
    }
  }

  async function migrateLegacyImageItems() {
    let changed = false;
    /** @type {HistoryItem[]} */
    const migrated = [];
    for (const item of history) {
      if (item.type !== 'text' && item.content && !isFileImageContent(item.content)) {
        try {
          const fileContent = await persistImageContent(item.content);
          migrated.push({ ...item, content: fileContent });
          changed = true;
          continue;
        } catch {
          // keep original item if migration fails
        }
      }
      migrated.push(item);
    }
    if (changed) {
      updateAndSaveHistory(migrated);
    }
  }

  /** @param {HistoryItem[]} newArray */
  function updateAndSaveHistory(newArray) {
    const previousHistory = history;
    const EXPIRATION_MS = 24 * 60 * 60 * 1000;
    const now = Date.now();

    const valid = newArray.filter((item) => {
      if (item.type === 'text' && (!item.category || item.category === DEFAULT_CATEGORY)) {
        return now - item.id < EXPIRATION_MS;
      }
      return true;
    });

    const imgs = valid.filter((item) => item.type !== 'text');
    const categorizedTexts = valid.filter(
      (item) => item.type === 'text' && item.category && item.category !== DEFAULT_CATEGORY
    );
    const uncategorizedTexts = valid.filter(
      (item) => item.type === 'text' && (!item.category || item.category === DEFAULT_CATEGORY)
    );

    const merged = [...imgs.slice(0, 10), ...categorizedTexts, ...uncategorizedTexts.slice(0, 50)];
    merged.sort((a, b) => b.id - a.id);

    const mergedIds = new Set(merged.map((item) => item.id));
    const removedImages = previousHistory.filter(
      (item) => item.type !== 'text' && !mergedIds.has(item.id)
    );

    history = merged;
    localStorage.setItem(HISTORY_KEY, JSON.stringify(history));
    localStorage.setItem(LAST_DATA_KEY, lastData);
    if (removedImages.length > 0) {
      void cleanupImageFilesFromItems(removedImages);
    }
  }

  function savePasswords() {
    localStorage.setItem(PASSWORDS_KEY, JSON.stringify(passwords));
  }

  function saveCategories() {
    localStorage.setItem(CATEGORIES_KEY, JSON.stringify(categories));
  }

  onMount(() => {
    const savedLang = localStorage.getItem(LANG_KEY);
    if (savedLang === 'en' || savedLang === 'zh') setLang(savedLang);

    const saved = localStorage.getItem(HISTORY_KEY);
    if (saved) {
      try {
        history = JSON.parse(saved);
      } catch {
        history = [];
      }
    }

    const savedLastData = localStorage.getItem(LAST_DATA_KEY);
    if (savedLastData) lastData = savedLastData;

    const savedPwds = localStorage.getItem(PASSWORDS_KEY);
    if (savedPwds) {
      try {
        passwords = JSON.parse(savedPwds);
      } catch {
        passwords = [];
      }
    }

    const savedCats = localStorage.getItem(CATEGORIES_KEY);
    if (savedCats) {
      try {
        categories = JSON.parse(savedCats);
      } catch {
        categories = [DEFAULT_CATEGORY];
      }
    }

    if (!categories.includes(DEFAULT_CATEGORY)) {
      categories = [DEFAULT_CATEGORY, ...categories];
      saveCategories();
    }

    updateAndSaveHistory(history);
    void migrateLegacyImageItems();

    const clipboardInterval = setInterval(async () => {
      const now = Date.now();
      const EXPIRATION_MS = 24 * 60 * 60 * 1000;
      if (
        history.some(
          (item) =>
            item.type === 'text' &&
            (!item.category || item.category === DEFAULT_CATEGORY) &&
            now - item.id > EXPIRATION_MS
        )
      ) {
        updateAndSaveHistory(history);
      }

      try {
        const data = await invoke('get_clipboard_data');
        if (data) {
          const [type, content] = data;
          const signature = clipboardSignature(type, content);
          if (signature !== lastData) {
            if (isAppCopying && type === 'image') {
              lastData = signature;
              isAppCopying = false;
              updateAndSaveHistory(history);
              return;
            }

            let storedContent = content;
            if (type === 'image') {
              storedContent = await persistImageContent(content);
            }

            const defaultCat =
              type === 'text' && activeCategory !== DEFAULT_CATEGORY ? activeCategory : DEFAULT_CATEGORY;
            const newItem = {
              type,
              content: storedContent,
              id: Date.now(),
              timestamp: timestamp(),
              category: defaultCat
            };

            lastData = signature;
            updateAndSaveHistory([newItem, ...history]);
          }
        }
      } catch {
        // ignore clipboard poll errors
      }
    }, 2000);

    /** @param {ClipboardEvent} e */
    const handlePaste = (e) => {
      const active = document.activeElement;
      if (active && (active.tagName === 'INPUT' || active.tagName === 'TEXTAREA')) return;
      if (!e.clipboardData) return;

      const items = e.clipboardData.items;
      for (let i = 0; i < items.length; i++) {
        if (items[i].type.includes('image')) {
          const blob = items[i].getAsFile();
          if (!blob) continue;
          const reader = new FileReader();
          reader.onload = async (event) => {
            const result = event?.target?.result;
            if (typeof result !== 'string') return;
            const base64 = result.split(',')[1];
            if (!base64) return;

            const content = `image|${base64}`;
            const signature = clipboardSignature('image', content);
            if (signature !== lastData) {
              let storedContent = content;
              try {
                storedContent = await persistImageContent(content);
              } catch {
                return;
              }
              const newItem = {
                type: 'image',
                content: storedContent,
                id: Date.now(),
                timestamp: timestamp(),
                category: DEFAULT_CATEGORY
              };
              lastData = signature;
              updateAndSaveHistory([newItem, ...history]);
              triggerToast(t.imagePasted);
            }
          };
          reader.readAsDataURL(blob);
        } else if (items[i].type === 'text/plain') {
          /** @param {string} text */
          items[i].getAsString((text) => {
            const signature = clipboardSignature('text', text);
            if (signature !== lastData) {
              const defaultCat = activeCategory !== DEFAULT_CATEGORY ? activeCategory : DEFAULT_CATEGORY;
              const newItem = {
                type: 'text',
                content: text,
                id: Date.now(),
                timestamp: timestamp(),
                category: defaultCat
              };
              lastData = signature;
              updateAndSaveHistory([newItem, ...history]);
              triggerToast(t.textPasted);
            }
          });
        }
      }
    };

    window.addEventListener('paste', handlePaste);

    return () => {
      clearInterval(clipboardInterval);
      window.removeEventListener('paste', handlePaste);
    };
  });

  /** @param {string} [msg] */
  function triggerToast(msg = t.copied) {
    toastMsg = msg;
    showToast = true;
    setTimeout(() => {
      showToast = false;
    }, 900);
  }

  /** @param {string} text */
  async function copyText(text) {
    if (!text) return;
    await invoke('set_clipboard_text', { text });
    lastData = clipboardSignature('text', text);
    updateAndSaveHistory(history);
    triggerToast(t.copied);
  }

  /** @param {string} content */
  async function copyImage(content) {
    if (!content) return;
    try {
      isAppCopying = true;
      if (isFileImageContent(content)) {
        await invoke('set_clipboard_image_from_path', { path: content.slice(5) });
      } else {
        const base64 = content.includes('|') ? content.split('|')[1] : content;
        await invoke('set_clipboard_image', { base64 });
      }
      triggerToast(t.imageCopied);
      setTimeout(() => {
        isAppCopying = false;
      }, 3000);
    } catch {
      triggerToast(t.copyFailed);
    }
  }

  /** @param {number} id */
  async function deleteItem(id) {
    const item = history.find((it) => it.id === id);
    const newHistory = history.filter((it) => it.id !== id);
    if (previewImage?.id === id) closeImagePreview();
    if (targetImage?.id === id) {
      targetImage = null;
      closeImageMenu();
    }
    if (item?.type === 'text' && item.content === lastData) {
      await invoke('set_clipboard_text', { text: '' });
      lastData = '';
    }
    if (item && item.type !== 'text') {
      await cleanupImageFilesFromItems([item]);
    }
    updateAndSaveHistory(newHistory);
  }

  async function clearImages() {
    const removedImages = history.filter((item) => item.type !== 'text');
    const newHistory = history.filter((item) => item.type === 'text');
    if (previewImage) closeImagePreview();
    targetImage = null;
    closeImageMenu();
    if (lastData.startsWith('image:')) {
      await invoke('set_clipboard_text', { text: '' });
      lastData = '';
    }
    await cleanupImageFilesFromItems(removedImages);
    updateAndSaveHistory(newHistory);
  }

  async function clearText() {
    const toDelete = history.filter(
      (item) =>
        item.type === 'text' &&
        (activeCategory === DEFAULT_CATEGORY || (item.category || DEFAULT_CATEGORY) === activeCategory)
    );
    const hasLastData = toDelete.some((it) => it.content === lastData);
    const newHistory = history.filter((item) => !toDelete.includes(item));

    if (hasLastData) {
      await invoke('set_clipboard_text', { text: '' });
      lastData = '';
    }
    updateAndSaveHistory(newHistory);
  }

  function addCategory() {
    const trimmed = newCatName.trim();
    if (trimmed && !categories.includes(trimmed)) {
      categories = [...categories, trimmed];
      saveCategories();
    }
    isAddingCat = false;
    newCatName = '';
  }

  /**
   * @param {number} id
   * @param {string} newCat
   */
  function changeCategory(id, newCat) {
    const newHistory = history.map((item) => (item.id === id ? { ...item, category: newCat } : item));
    updateAndSaveHistory(newHistory);
  }

  /**
   * @param {MouseEvent} e
   * @param {string} cat
   */
  function handleContextMenu(e, cat) {
    if (cat === DEFAULT_CATEGORY) return;
    targetCategory = cat;
    menuX = e.clientX;
    menuY = e.clientY;
    showImageMenu = false;
    showContextMenu = true;
  }

  function closeContextMenu() {
    showContextMenu = false;
  }

  /**
   * @param {MouseEvent} e
   * @param {HistoryItem} item
   */
  function handleImageContextMenu(e, item) {
    targetImage = item;
    imageMenuX = e.clientX;
    imageMenuY = e.clientY;
    showContextMenu = false;
    showImageMenu = true;
  }

  function closeImageMenu() {
    showImageMenu = false;
  }

  /** @param {HistoryItem | null} [item] */
  function openImagePreview(item = targetImage) {
    if (!item) return;
    previewImage = item;
    previewScale = 1;
    closeImageMenu();
  }

  function closeImagePreview() {
    previewImage = null;
    previewScale = 1;
  }

  /** @param {number} delta */
  function zoomPreview(delta) {
    previewScale = clampPreviewScale(previewScale + delta);
  }

  /** @param {WheelEvent} event */
  function handlePreviewWheel(event) {
    const delta = event.deltaY < 0 ? 0.15 : -0.15;
    zoomPreview(delta);
  }

  /** @param {KeyboardEvent} event */
  function handleWindowKeydown(event) {
    if (event.key === 'Escape') {
      closeImageMenu();
      closeContextMenu();
      if (previewImage) closeImagePreview();
    }
  }

  /** @param {MouseEvent} event */
  function handlePreviewBackdropMouseDown(event) {
    if (event.target === event.currentTarget) {
      closeImagePreview();
    }
  }

  function startRename() {
    renameCatName = targetCategory;
    isRenamingCat = true;
    closeContextMenu();
  }

  function confirmRename() {
    const trimmed = renameCatName.trim();
    if (trimmed && trimmed !== targetCategory && !categories.includes(trimmed)) {
      categories = categories.map((cat) => (cat === targetCategory ? trimmed : cat));
      const newHistory = history.map((item) =>
        item.category === targetCategory ? { ...item, category: trimmed } : item
      );
      if (activeCategory === targetCategory) activeCategory = trimmed;
      saveCategories();
      updateAndSaveHistory(newHistory);
    }
    isRenamingCat = false;
  }

  function deleteCategoryFromMenu() {
    const cat = targetCategory;
    categories = categories.filter((name) => name !== cat);
    const newHistory = history.map((item) =>
      item.category === cat ? { ...item, category: DEFAULT_CATEGORY } : item
    );
    saveCategories();
    if (activeCategory === cat) activeCategory = DEFAULT_CATEGORY;
    updateAndSaveHistory(newHistory);
    closeContextMenu();
  }

  function addPassword() {
    if (!newPwdTitle.trim() || !newPwdPass.trim()) return;
    const newItem = {
      id: Date.now(),
      title: newPwdTitle.trim(),
      username: newPwdUser.trim(),
      password: newPwdPass,
      showPass: false
    };
    passwords = [newItem, ...passwords];
    savePasswords();
    newPwdTitle = '';
    newPwdUser = '';
    newPwdPass = '';
    showNewPwd = false;
    triggerToast(t.saved);
  }

  /** @param {number} id */
  function deletePassword(id) {
    passwords = passwords.filter((pwd) => pwd.id !== id);
    savePasswords();
  }

  function clearAllPasswords() {
    passwords = [];
    savePasswords();
  }

  /** @param {number} id */
  function togglePassword(id) {
    passwords = passwords.map((pwd) => (pwd.id === id ? { ...pwd, showPass: !pwd.showPass } : pwd));
  }

  /** @param {string} content */
  function getImgSrc(content) {
    try {
      if (isFileImageContent(content)) {
        const path = content.slice(5);
        return convertFileSrc(path);
      }
      const base64 = content.includes('|') ? content.split('|')[1] : content;
      return `data:image/png;base64,${base64}`;
    } catch {
      return '';
    }
  }

  $: filteredImages = history.filter((item) => item.type !== 'text');
  $: displayedText = history
    .filter((item) => item.type === 'text')
    .filter((item) => activeCategory === DEFAULT_CATEGORY || (item.category || DEFAULT_CATEGORY) === activeCategory)
    .filter((item) => !searchQuery || item.content.toLowerCase().includes(searchQuery.toLowerCase()));
  $: filteredPasswords = pwdSearchQuery
    ? passwords.filter((pwd) => pwd.title.toLowerCase().includes(pwdSearchQuery.toLowerCase()))
    : passwords;
</script>

<svelte:window
  on:click={() => {
    closeContextMenu();
    closeImageMenu();
  }}
  on:keydown={handleWindowKeydown}
  on:resize={syncTabIndicator}
/>

<main class="app-shell">
  <div class="aurora" aria-hidden="true">
    <span class="blob b1"></span>
    <span class="blob b2"></span>
    <span class="blob b3"></span>
    <span class="blob b4"></span>
    <span class="grain"></span>
  </div>

  <nav class="top-tabs">
    <div class="segmented">
      <button
        class="tab-btn {activeTab === 'clipboard' ? 'active' : ''}"
        bind:this={clipboardTabEl}
        on:click={() => (activeTab = 'clipboard')}
      >
        {t.clipboard}
      </button>
      <button
        class="tab-btn {activeTab === 'passwords' ? 'active' : ''}"
        bind:this={passwordTabEl}
        on:click={() => (activeTab = 'passwords')}
      >
        {t.passwords}
      </button>
      <span
        class="tab-indicator"
        aria-hidden="true"
        style="width: {tabLineW}px; transform: translateX({tabLineX}px);"
      ></span>
    </div>

    <div class="lang-toggle" role="group" aria-label={t.switchLanguage}>
      <button class="lang-btn" class:active={lang === 'zh'} on:click={() => setLang('zh')} title="中文">
        中
      </button>
      <button class="lang-btn" class:active={lang === 'en'} on:click={() => setLang('en')} title="English">
        EN
      </button>
    </div>
  </nav>

  <div class="search-wrap">
    <div class="search-bar">
      <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="11" cy="11" r="8" />
        <line x1="21" y1="21" x2="16.65" y2="16.65" />
      </svg>
      {#if activeTab === 'clipboard'}
        <input type="text" placeholder={t.searchClipboard} bind:value={searchQuery} />
      {:else}
        <input type="text" placeholder={t.searchPasswords} bind:value={pwdSearchQuery} />
      {/if}
      <kbd>Ctrl K</kbd>
    </div>
  </div>

  {#if activeTab === 'clipboard'}
    <div class="workspace" in:fade={{ duration: 170 }}>
      <section class="panel panel-image">
        <header class="panel-head">
          <h3>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="18" height="18" rx="3" />
              <circle cx="8.6" cy="8.6" r="1.6" />
              <polyline points="21 15 15.7 9.8 5 21" />
            </svg>
            {t.imageHistory}
            {#if filteredImages.length}<span class="count-badge">{filteredImages.length}</span>{/if}
          </h3>
          <button class="clear-btn" on:click={clearImages}>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <polyline points="3 6 5 6 21 6" />
              <path d="M8 6v-2a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
              <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
            </svg>
            {t.clearAll}
          </button>
        </header>

        {#if filteredImages.length === 0}
          <div class="empty-state" in:fade={{ duration: 260 }}>
            <div class="empty-illustration">
              <svg viewBox="0 0 100 100" fill="none">
                <rect x="23" y="24" width="54" height="52" rx="11" stroke="#7fb4f3" stroke-width="3.4" />
                <path d="M34 60l11-16 11 12 9-9 11 13" stroke="#6da5ea" stroke-width="4" stroke-linecap="round" />
                <circle cx="63.5" cy="40.5" r="4.5" fill="#7fb4f3" />
              </svg>
            </div>
            <p class="empty-title">{t.noImages}</p>
            <p class="empty-sub">{t.noImagesSub}</p>
          </div>
        {:else}
          <div class="scroll-v image-grid">
            {#each filteredImages as item (item.id)}
              <div
                class="image-card card-pop"
                role="button"
                tabindex="0"
                on:click={() => copyImage(item.content)}
                on:contextmenu|preventDefault={(e) => handleImageContextMenu(e, item)}
                on:keydown={(e) => {
                  if (e.key === 'Enter' || e.key === ' ') {
                    e.preventDefault();
                    copyImage(item.content);
                  }
                }}
                in:fly={{ y: 12, duration: 260 }}
              >
                <img src={getImgSrc(item.content)} alt="clipboard" />
                <span class="image-card-overlay">{t.copy}</span>
                <button class="icon-btn danger image-del" on:click|stopPropagation={() => deleteItem(item.id)} aria-label="Delete image">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <line x1="18" y1="6" x2="6" y2="18" />
                    <line x1="6" y1="6" x2="18" y2="18" />
                  </svg>
                </button>
              </div>
            {/each}
          </div>
        {/if}
      </section>

      <section class="panel panel-text">
        <header class="panel-head">
          <h3>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="4" y1="6" x2="20" y2="6" />
              <line x1="4" y1="12" x2="20" y2="12" />
              <line x1="4" y1="18" x2="12" y2="18" />
            </svg>
            {t.textHistory}
            {#if displayedText.length}<span class="count-badge">{displayedText.length}</span>{/if}
          </h3>
          <button class="clear-btn" on:click={clearText}>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <polyline points="3 6 5 6 21 6" />
              <path d="M8 6v-2a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
              <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
            </svg>
            {t.clearAll}
          </button>
        </header>

        <div class="category-row">
          <div class="category-scroll" on:wheel|preventDefault={handleCategoryWheel}>
            {#each categories as cat}
              {#if isRenamingCat && targetCategory === cat}
                <input
                  class="cat-editor"
                  bind:value={renameCatName}
                  on:keydown={(e) => {
                    if (e.key === 'Enter') confirmRename();
                    else if (e.key === 'Escape') isRenamingCat = false;
                  }}
                  on:blur={confirmRename}
                />
              {:else}
                <button class="chip {activeCategory === cat ? 'active' : ''}" on:click={() => (activeCategory = cat)} on:contextmenu|preventDefault={(e) => handleContextMenu(e, cat)}>
                  {cat === DEFAULT_CATEGORY ? t.all : cat}
                </button>
              {/if}
            {/each}

            {#if isAddingCat}
              <input
                class="cat-editor"
                bind:value={newCatName}
                on:keydown={(e) => {
                  if (e.key === 'Enter') addCategory();
                  else if (e.key === 'Escape') isAddingCat = false;
                }}
                on:blur={addCategory}
                placeholder={t.newCategory}
              />
            {:else}
              <button class="chip add-chip" on:click={() => (isAddingCat = true)} aria-label="Add category">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <line x1="12" y1="5" x2="12" y2="19" />
                  <line x1="5" y1="12" x2="19" y2="12" />
                </svg>
              </button>
            {/if}
          </div>
        </div>

        <div class="scroll-v list-stack">
          {#if displayedText.length === 0}
            <div class="empty-list">{t.noText}</div>
          {/if}

          {#each displayedText as item (item.id)}
            <div
              class="text-item card-pop"
              role="button"
              tabindex="0"
              on:click={() => copyText(item.content)}
              on:keydown={(e) => {
                if (e.key === 'Enter' || e.key === ' ') {
                  e.preventDefault();
                  copyText(item.content);
                }
              }}
              in:fly={{ x: 14, duration: 260 }}
            >
              <div class="text-main">
                <div class="text-title" title={item.content}>{item.content}</div>
                <div class="text-meta">
                  <select
                    class="cat-select"
                    value={item.category || DEFAULT_CATEGORY}
                    on:change={(e) => changeCategory(item.id, e.currentTarget.value)}
                    on:click|stopPropagation
                  >
                    {#each categories as cat}
                      <option value={cat}>{cat === DEFAULT_CATEGORY ? t.all : cat}</option>
                    {/each}
                  </select>
                </div>
              </div>
              <div class="text-actions">
                <button class="icon-btn" on:click|stopPropagation={() => copyText(item.content)} aria-label="Copy text">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <rect x="9" y="9" width="13" height="13" rx="2" ry="2" />
                    <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                  </svg>
                </button>
                <button class="icon-btn danger" on:click|stopPropagation={() => deleteItem(item.id)} aria-label="Delete text">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <line x1="18" y1="6" x2="6" y2="18" />
                    <line x1="6" y1="6" x2="18" y2="18" />
                  </svg>
                </button>
              </div>
            </div>
          {/each}
        </div>
      </section>
    </div>
  {:else}
    <div class="workspace password-workspace" in:fade={{ duration: 170 }}>
      <section class="panel panel-form">
        <header class="panel-head single">
          <h3>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="11" width="18" height="10" rx="2" />
              <path d="M7 11V7a5 5 0 0 1 10 0v4" />
            </svg>
            {t.addNew}
          </h3>
        </header>

        <div class="form-body">
          <label for="pwd-title">{t.forWhat}</label>
          <div class="input-shell">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z" />
              <polyline points="14 2 14 8 20 8" />
              <line x1="16" y1="13" x2="8" y2="13" />
            </svg>
            <input id="pwd-title" bind:value={newPwdTitle} placeholder={t.purposePlaceholder} />
          </div>

          <label for="pwd-username">{t.username}</label>
          <div class="input-shell">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2" />
              <circle cx="12" cy="7" r="4" />
            </svg>
            <input id="pwd-username" bind:value={newPwdUser} placeholder={t.usernamePlaceholder} />
          </div>

          <label for="pwd-password">{t.password}</label>
          <div class="input-shell">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M20 12V8a4 4 0 0 0-8 0v4" />
              <rect x="4" y="12" width="16" height="9" rx="2" />
            </svg>
            <input id="pwd-password" bind:value={newPwdPass} placeholder={t.passwordPlaceholder} type={showNewPwd ? 'text' : 'password'} />
            <button class="inline-btn" on:click={() => (showNewPwd = !showNewPwd)} aria-label="Toggle password">
              {#if showNewPwd}
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path d="M17.94 17.94A10.94 10.94 0 0 1 12 20C5 20 1 12 1 12a18.4 18.4 0 0 1 5-6" />
                  <path d="M9.9 4.2A9.1 9.1 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.2 3.2" />
                  <line x1="1" y1="1" x2="23" y2="23" />
                </svg>
              {:else}
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z" />
                  <circle cx="12" cy="12" r="3" />
                </svg>
              {/if}
            </button>
          </div>

          <button class="save-btn" on:click={addPassword}>{t.save}</button>
        </div>
      </section>

      <section class="panel panel-passwords">
        <header class="panel-head">
          <h3>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M15 7h3a5 5 0 0 1 0 10h-3" />
              <path d="M9 17H6a5 5 0 1 1 0-10h3" />
              <line x1="8" y1="12" x2="16" y2="12" />
            </svg>
            {t.myPasswords}
            {#if filteredPasswords.length}<span class="count-badge">{filteredPasswords.length}</span>{/if}
          </h3>
          <button class="clear-btn" on:click={clearAllPasswords}>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <polyline points="3 6 5 6 21 6" />
              <path d="M8 6v-2a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
              <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
            </svg>
            {t.clearAll}
          </button>
        </header>

        <div class="scroll-v list-stack">
          {#if filteredPasswords.length === 0}
            <div class="empty-list">{t.noPasswords}</div>
          {/if}

          {#each filteredPasswords as item (item.id)}
            <div class="password-item card-pop" in:fly={{ x: 14, duration: 260 }}>
              <div class="password-head">
                <div class="password-title" title={item.title}>{item.title}</div>
                <button class="icon-btn danger" on:click={() => deletePassword(item.id)} aria-label="Delete password">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <line x1="18" y1="6" x2="6" y2="18" />
                    <line x1="6" y1="6" x2="18" y2="18" />
                  </svg>
                </button>
              </div>

              {#if item.username}
                <div class="password-line">
                  <span class="line-label">{t.userLabel}</span>
                  <span class="line-value">{item.username}</span>
                </div>
              {/if}

              <div class="password-line">
                <span class="line-label">{t.passLabel}</span>
                <span class="line-value">{item.showPass ? item.password : '••••'}</span>
              </div>

              <div class="password-actions">
                <button class="icon-btn" on:click={() => togglePassword(item.id)} aria-label="Toggle password">
                  {#if item.showPass}
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M17.94 17.94A10.94 10.94 0 0 1 12 20C5 20 1 12 1 12a18.4 18.4 0 0 1 5-6" />
                      <path d="M9.9 4.2A9.1 9.1 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.2 3.2" />
                      <line x1="1" y1="1" x2="23" y2="23" />
                    </svg>
                  {:else}
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z" />
                      <circle cx="12" cy="12" r="3" />
                    </svg>
                  {/if}
                </button>
                {#if item.username}
                  <button class="icon-btn" on:click={() => copyText(item.username)} aria-label="Copy username">
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2" />
                      <circle cx="12" cy="7" r="4" />
                    </svg>
                  </button>
                {/if}
                <button class="icon-btn" on:click={() => copyText(item.password)} aria-label="Copy password">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <rect x="9" y="9" width="13" height="13" rx="2" ry="2" />
                    <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                  </svg>
                </button>
                <button class="icon-btn danger" on:click={() => deletePassword(item.id)} aria-label="Delete password">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <polyline points="3 6 5 6 21 6" />
                    <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
                  </svg>
                </button>
              </div>
            </div>
          {/each}
        </div>
      </section>
    </div>
  {/if}

  {#if showToast}
    <div class="toast-tip" transition:scale={{ duration: 120 }}>{toastMsg}</div>
  {/if}

  {#if showContextMenu}
    <div
      class="context-menu"
      style="left: {menuX}px; top: {menuY}px;"
      role="menu"
      tabindex="-1"
      aria-label="Category menu"
      in:fade={{ duration: 100 }}
      on:mousedown|stopPropagation
    >
      <button class="menu-item" on:click={startRename}>
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M12 20h9" />
          <path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4Z" />
        </svg>
        {t.rename}
      </button>
      <div class="menu-divider"></div>
      <button class="menu-item danger" on:click={deleteCategoryFromMenu}>
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <polyline points="3 6 5 6 21 6" />
          <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
          <path d="M8 6v-2a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
        </svg>
        {t.deleteCategory}
      </button>
    </div>
  {/if}

  {#if showImageMenu && targetImage}
    <div
      class="context-menu image-menu"
      style="left: {imageMenuX}px; top: {imageMenuY}px;"
      role="menu"
      tabindex="-1"
      aria-label="Image menu"
      in:fade={{ duration: 100 }}
      on:mousedown|stopPropagation
    >
      <button class="menu-item" on:click={() => openImagePreview()}>
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="11" cy="11" r="7" />
          <line x1="16.5" y1="16.5" x2="21" y2="21" />
          <line x1="11" y1="8" x2="11" y2="14" />
          <line x1="8" y1="11" x2="14" y2="11" />
        </svg>
        {t.preview}
      </button>
    </div>
  {/if}

  {#if previewImage}
    <div
      class="preview-backdrop"
      role="dialog"
      tabindex="-1"
      aria-modal="true"
      aria-label="Image preview"
      transition:fade={{ duration: 120 }}
      on:mousedown={handlePreviewBackdropMouseDown}
    >
      <div class="preview-stage" role="document">
        <div class="preview-toolbar">
          <button class="preview-tool" on:click={() => zoomPreview(-0.2)} aria-label="Zoom out">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="11" cy="11" r="7" />
              <line x1="8" y1="11" x2="14" y2="11" />
              <line x1="16.5" y1="16.5" x2="21" y2="21" />
            </svg>
          </button>
          <button class="preview-tool reset" on:click={() => (previewScale = 1)} aria-label="Reset zoom">
            {Math.round(previewScale * 100)}%
          </button>
          <button class="preview-tool" on:click={() => zoomPreview(0.2)} aria-label="Zoom in">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="11" cy="11" r="7" />
              <line x1="11" y1="8" x2="11" y2="14" />
              <line x1="8" y1="11" x2="14" y2="11" />
              <line x1="16.5" y1="16.5" x2="21" y2="21" />
            </svg>
          </button>
          <button class="preview-tool" on:click={closeImagePreview} aria-label="Close preview">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="18" y1="6" x2="6" y2="18" />
              <line x1="6" y1="6" x2="18" y2="18" />
            </svg>
          </button>
        </div>
        <div class="preview-scroll" on:wheel|preventDefault={handlePreviewWheel}>
          <img
            src={getImgSrc(previewImage.content)}
            alt="Preview"
            style="transform: scale({previewScale});"
          />
        </div>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(:root) {
    --accent: #0a84ff;
    --accent-deep: #0060df;
    --label: #131722;
    --label-2: #565d70;
    --label-3: #868da0;
    --glass: rgba(255, 255, 255, 0.48);
    --glass-strong: rgba(255, 255, 255, 0.72);
    --hairline: rgba(255, 255, 255, 0.62);
    --radius-xl: 24px;
    --radius-lg: 20px;
    --radius-md: 16px;
    --radius-sm: 12px;
    --ease: cubic-bezier(0.32, 0.72, 0, 1);
    --spring: cubic-bezier(0.34, 1.35, 0.64, 1);
  }

  :global(html),
  :global(body) {
    height: 100%;
  }

  :global(body) {
    margin: 0;
    overflow: hidden;
    color: var(--label);
    background: #e9ecf7;
    font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Segoe UI Variable Text', 'Segoe UI',
      'PingFang SC', 'Microsoft YaHei', sans-serif;
    -webkit-font-smoothing: antialiased;
    text-rendering: optimizeLegibility;
    letter-spacing: -0.005em;
  }

  :global(*) { box-sizing: border-box; }
  :global(button), :global(input), :global(select) { font-family: inherit; }
  :global(button) { -webkit-tap-highlight-color: transparent; }

  /* ============ 背景：柔和光斑 + 细微颗粒，给玻璃提供“折射”对象 ============ */
  .aurora {
    position: fixed;
    inset: 0;
    z-index: 0;
    overflow: hidden;
    pointer-events: none;
    background: linear-gradient(158deg, #eef2ff 0%, #e6e9f7 42%, #f4eaf3 100%);
  }

  .blob {
    position: absolute;
    border-radius: 50%;
    filter: blur(56px);
    will-change: transform;
  }

  .b1 {
    width: 62%;
    aspect-ratio: 1;
    left: -15%;
    top: -20%;
    background: radial-gradient(circle at 40% 40%, rgba(147, 178, 255, 0.95), rgba(147, 178, 255, 0) 68%);
    animation: drift-a 28s ease-in-out infinite;
  }

  .b2 {
    width: 56%;
    aspect-ratio: 1;
    right: -18%;
    top: -12%;
    background: radial-gradient(circle at 50% 50%, rgba(255, 184, 211, 0.9), rgba(255, 184, 211, 0) 66%);
    animation: drift-b 34s ease-in-out infinite;
  }

  .b3 {
    width: 72%;
    aspect-ratio: 1;
    left: 4%;
    bottom: -38%;
    background: radial-gradient(circle at 50% 50%, rgba(150, 230, 245, 0.92), rgba(150, 230, 245, 0) 68%);
    animation: drift-c 40s ease-in-out infinite;
  }

  .b4 {
    width: 46%;
    aspect-ratio: 1;
    right: -8%;
    bottom: -20%;
    background: radial-gradient(circle at 50% 50%, rgba(203, 191, 255, 0.9), rgba(203, 191, 255, 0) 66%);
    animation: drift-a 32s ease-in-out infinite reverse;
  }

  .grain {
    position: absolute;
    inset: 0;
    opacity: 0.16;
    mix-blend-mode: overlay;
    background-image: radial-gradient(rgba(255, 255, 255, 0.9) 1px, transparent 1px);
    background-size: 3px 3px;
  }

  .app-shell {
    position: relative;
    z-index: 1;
    width: 100%;
    height: 100%;
    padding: 14px 14px 12px;
    display: flex;
    flex-direction: column;
    gap: 11px;
    animation: shell-reveal 0.45s var(--ease);
  }

  /* ============ 顶部分段控件（iOS/macOS 胶囊） ============ */
  .top-tabs {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
  }

  .segmented {
    position: relative;
    display: inline-flex;
    align-items: center;
    gap: 2px;
    padding: 4px;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.34);
    -webkit-backdrop-filter: blur(22px) saturate(180%);
    backdrop-filter: blur(22px) saturate(180%);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.55),
      inset 0 1px 0 rgba(255, 255, 255, 0.8),
      0 10px 24px -16px rgba(17, 23, 45, 0.6);
  }

  .tab-btn {
    position: relative;
    z-index: 1;
    border: 0;
    background: transparent;
    color: rgba(19, 23, 34, 0.5);
    font-size: 13.5px;
    font-weight: 600;
    letter-spacing: -0.01em;
    padding: 7px 20px;
    border-radius: 999px;
    cursor: pointer;
    transition: color 0.32s var(--ease), transform 0.32s var(--spring);
  }

  .tab-btn:hover { color: rgba(19, 23, 34, 0.78); }
  .tab-btn:active { transform: scale(0.97); }
  .tab-btn.active { color: var(--label); }

  .tab-indicator {
    position: absolute;
    top: 4px;
    bottom: 4px;
    left: 0;
    border-radius: 999px;
    background: linear-gradient(180deg, rgba(255, 255, 255, 0.98), rgba(255, 255, 255, 0.74));
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 1),
      0 6px 16px -6px rgba(19, 25, 48, 0.45),
      0 0 0 1px rgba(255, 255, 255, 0.65);
    transition: transform 0.44s var(--spring), width 0.44s var(--spring);
  }

  /* ============ 搜索框 ============ */
  .search-wrap { flex-shrink: 0; }

  .search-bar {
    position: relative;
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 13px;
    border-radius: 15px;
    background: var(--glass);
    -webkit-backdrop-filter: blur(26px) saturate(180%);
    backdrop-filter: blur(26px) saturate(180%);
    box-shadow:
      inset 0 0 0 1px var(--hairline),
      inset 0 1px 0 rgba(255, 255, 255, 0.85),
      0 12px 28px -22px rgba(17, 23, 45, 0.7);
    transition: background 0.3s var(--ease), box-shadow 0.3s var(--ease), transform 0.3s var(--ease);
  }

  .search-bar:focus-within {
    background: var(--glass-strong);
    transform: translateY(-1px);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.92),
      inset 0 1px 0 rgba(255, 255, 255, 1),
      0 18px 34px -22px rgba(17, 23, 45, 0.6),
      0 0 0 3.5px rgba(10, 132, 255, 0.16);
  }

  .search-icon {
    width: 17px;
    height: 17px;
    color: var(--label-3);
    flex-shrink: 0;
    transition: color 0.3s var(--ease);
  }

  .search-bar:focus-within .search-icon { color: var(--accent); }

  .search-bar input {
    border: none;
    outline: none;
    background: transparent;
    color: var(--label);
    font-size: 13.5px;
    font-weight: 500;
    flex: 1;
    min-width: 0;
  }

  .search-bar input::placeholder { color: var(--label-3); font-weight: 450; }

  kbd {
    flex-shrink: 0;
    font-family: inherit;
    font-size: 11px;
    font-weight: 600;
    color: var(--label-3);
    letter-spacing: 0.01em;
    padding: 3px 7px;
    min-width: 46px;
    text-align: center;
    border-radius: 7px;
    background: rgba(255, 255, 255, 0.6);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.9),
      0 1px 2px rgba(20, 26, 50, 0.08);
  }

  /* ============ 布局 ============ */
  .workspace {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1.06fr);
    gap: 11px;
  }

  .panel {
    position: relative;
    display: flex;
    flex-direction: column;
    min-height: 0;
    padding: 13px 13px 12px;
    border-radius: var(--radius-xl);
    overflow: hidden;
    background: linear-gradient(168deg, rgba(255, 255, 255, 0.62), rgba(255, 255, 255, 0.4));
    -webkit-backdrop-filter: blur(30px) saturate(185%);
    backdrop-filter: blur(30px) saturate(185%);
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 0.88),
      0 20px 44px -30px rgba(17, 23, 45, 0.7),
      0 3px 10px -6px rgba(17, 23, 45, 0.2);
    animation: panel-in 0.42s var(--ease);
  }

  /* 一圈渐变高光描边，模拟玻璃厚度 */
  .panel::before {
    content: '';
    position: absolute;
    inset: 0;
    border-radius: inherit;
    padding: 1px;
    background: linear-gradient(
      152deg,
      rgba(255, 255, 255, 0.95),
      rgba(255, 255, 255, 0.3) 36%,
      rgba(255, 255, 255, 0.14) 62%,
      rgba(255, 255, 255, 0.7)
    );
    -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0);
    -webkit-mask-composite: xor;
    mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0);
    mask-composite: exclude;
    pointer-events: none;
  }

  /* 左上角镜面反光 */
  .panel::after {
    content: '';
    position: absolute;
    inset: 0;
    border-radius: inherit;
    pointer-events: none;
    background: radial-gradient(115% 55% at 6% -32%, rgba(255, 255, 255, 0.55), rgba(255, 255, 255, 0) 62%);
    opacity: 0.6;
  }

  .panel-head {
    position: relative;
    z-index: 1;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 11px;
  }

  .panel-head h3 {
    display: flex;
    align-items: center;
    gap: 7px;
    margin: 0;
    font-size: clamp(14px, 1.5vw, 15.5px);
    font-weight: 650;
    letter-spacing: -0.02em;
    line-height: 1.1;
    color: var(--label);
  }

  .panel-head h3 svg {
    width: 17px;
    height: 17px;
    color: var(--accent);
    opacity: 0.9;
    flex-shrink: 0;
  }

  .count-badge {
    font-size: 11px;
    font-weight: 650;
    color: var(--label-2);
    padding: 1px 7px;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.62);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.85);
    font-variant-numeric: tabular-nums;
  }

  .clear-btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    border: none;
    padding: 5px 10px 5px 8px;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 560;
    color: var(--label-2);
    background: rgba(255, 255, 255, 0.42);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.6);
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.24s var(--ease), color 0.24s var(--ease), transform 0.24s var(--spring),
      box-shadow 0.24s var(--ease);
  }

  .clear-btn svg { width: 13px; height: 13px; }

  .clear-btn:hover {
    color: #d1445a;
    background: rgba(255, 255, 255, 0.78);
    transform: translateY(-1px);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.95), 0 8px 18px -12px rgba(209, 68, 90, 0.9);
  }

  .clear-btn:active { transform: scale(0.96); }

  /* ============ 滚动区域 ============ */
  .scroll-v,
  .form-body {
    position: relative;
    z-index: 1;
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding-right: 5px;
    scrollbar-width: thin;
  }

  .scroll-v::-webkit-scrollbar,
  .form-body::-webkit-scrollbar,
  .preview-scroll::-webkit-scrollbar { width: 7px; height: 7px; }

  .scroll-v::-webkit-scrollbar-thumb,
  .form-body::-webkit-scrollbar-thumb,
  .preview-scroll::-webkit-scrollbar-thumb {
    background: rgba(64, 72, 102, 0.22);
    border-radius: 999px;
  }

  .scroll-v::-webkit-scrollbar-thumb:hover,
  .form-body::-webkit-scrollbar-thumb:hover { background: rgba(64, 72, 102, 0.36); }

  .scroll-v::-webkit-scrollbar-track,
  .form-body::-webkit-scrollbar-track,
  .preview-scroll::-webkit-scrollbar-track { background: transparent; }

  /* ============ 空状态 ============ */
  .empty-state {
    position: relative;
    z-index: 1;
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 6px;
    text-align: center;
    padding-bottom: 14px;
  }

  .empty-illustration {
    position: relative;
    width: 96px;
    height: 96px;
    display: grid;
    place-items: center;
    border-radius: 50%;
    background: radial-gradient(circle at 32% 28%, rgba(255, 255, 255, 0.9), rgba(255, 255, 255, 0.28) 62%);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.8),
      inset 0 1px 0 rgba(255, 255, 255, 1),
      0 18px 34px -22px rgba(17, 23, 45, 0.65);
    animation: float-y 6s ease-in-out infinite;
  }

  .empty-illustration svg {
    width: 46px;
    height: 46px;
    filter: drop-shadow(0 8px 12px rgba(80, 130, 210, 0.22));
  }

  .empty-title {
    margin: 8px 0 0;
    font-size: 15px;
    font-weight: 640;
    letter-spacing: -0.02em;
    color: var(--label);
  }

  .empty-sub {
    margin: 0;
    max-width: 260px;
    font-size: 12.5px;
    font-weight: 450;
    color: var(--label-3);
    line-height: 1.45;
  }

  /* ============ 图片卡片 ============ */
  .image-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(84px, 1fr));
    gap: 9px;
    align-content: start;
  }

  .image-card {
    position: relative;
    aspect-ratio: 1;
    padding: 0;
    border: none;
    border-radius: 15px;
    overflow: hidden;
    cursor: pointer;
    background: rgba(255, 255, 255, 0.5);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.7),
      0 6px 16px -12px rgba(17, 23, 45, 0.65);
    transition: transform 0.3s var(--spring), box-shadow 0.3s var(--ease);
  }

  .image-card img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .image-card-overlay {
    position: absolute;
    left: 6px;
    right: 6px;
    bottom: 6px;
    padding: 4px 0;
    border-radius: 8px;
    font-size: 11.5px;
    font-weight: 600;
    text-align: center;
    color: #fff;
    background: rgba(20, 24, 36, 0.42);
    -webkit-backdrop-filter: blur(10px);
    backdrop-filter: blur(10px);
    opacity: 0;
    transform: translateY(8px);
    transition: opacity 0.26s var(--ease), transform 0.26s var(--spring);
  }

  .image-card:hover .image-card-overlay,
  .image-card:focus-visible .image-card-overlay { opacity: 1; transform: translateY(0); }

  .image-card:hover {
    transform: translateY(-2px) scale(1.012);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.95),
      0 18px 30px -18px rgba(17, 23, 45, 0.7);
  }

  .image-del {
    position: absolute;
    top: 5px;
    right: 5px;
    width: 22px;
    height: 22px;
    border-radius: 50%;
    color: #fff;
    background: rgba(20, 24, 36, 0.36);
    -webkit-backdrop-filter: blur(10px);
    backdrop-filter: blur(10px);
    opacity: 0;
    transition: opacity 0.24s var(--ease), background 0.24s var(--ease);
  }

  .image-del svg { width: 11px; height: 11px; }
  .image-card:hover .image-del { opacity: 1; }
  .image-del:hover { background: rgba(209, 68, 90, 0.9); color: #fff; }

  /* ============ 分类胶囊 ============ */
  .category-row { position: relative; z-index: 1; margin-bottom: 9px; }

  .category-scroll {
    display: flex;
    align-items: center;
    gap: 6px;
    overflow-x: auto;
    overflow-y: hidden;
    padding-bottom: 2px;
    scroll-behavior: smooth;
  }

  .category-scroll::-webkit-scrollbar { height: 0; }

  .chip {
    flex: 0 0 auto;
    border: none;
    padding: 5px 12px;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 560;
    letter-spacing: -0.005em;
    color: var(--label-2);
    cursor: pointer;
    background-color: rgba(255, 255, 255, 0.44);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.62);
    transform: translateZ(0);
    transition:
      background-color 0.36s var(--ease),
      color 0.3s var(--ease),
      box-shadow 0.36s var(--ease),
      transform 0.44s var(--spring),
      padding 0.36s var(--spring);
  }

  .chip:hover {
    background-color: rgba(255, 255, 255, 0.88);
    color: var(--label);
    transform: translateY(-1px);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.95),
      0 10px 18px -14px rgba(17, 23, 45, 0.7);
  }

  .chip.active {
    color: #fff;
    padding-left: 13px;
    padding-right: 13px;
    background-color: #0a84ff;
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 0.42),
      inset 0 -1px 0 rgba(0, 40, 90, 0.12),
      0 10px 22px -12px rgba(10, 132, 255, 0.95);
    animation: chip-pop 0.46s var(--spring);
  }

  .chip.active:hover { background-color: #1b8dff; }

  .add-chip {
    width: 26px;
    height: 26px;
    min-width: 26px;
    padding: 0;
    display: grid;
    place-items: center;
    border-radius: 50%;
    color: var(--label-2);
  }

  .add-chip svg { width: 12px; height: 12px; }

  .cat-editor {
    flex: 0 0 auto;
    width: 92px;
    border: none;
    outline: none;
    padding: 5px 10px;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 550;
    color: var(--label);
    background: rgba(255, 255, 255, 0.85);
    box-shadow: inset 0 0 0 1px rgba(10, 132, 255, 0.55), 0 0 0 3px rgba(10, 132, 255, 0.12);
  }

  /* ============ 列表条目 ============ */
  .list-stack { display: flex; flex-direction: column; gap: 7px; }

  .empty-list {
    padding: 16px;
    text-align: center;
    font-size: 12.5px;
    color: var(--label-3);
    border-radius: var(--radius-md);
    background: rgba(255, 255, 255, 0.26);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.5);
  }

  .card-pop { transition: transform 0.26s var(--spring), box-shadow 0.26s var(--ease), background 0.26s var(--ease); }

  .text-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 11px;
    border-radius: var(--radius-md);
    cursor: pointer;
    background: rgba(255, 255, 255, 0.44);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.6),
      0 1px 2px rgba(17, 23, 45, 0.04);
  }

  .text-item:hover,
  .text-item:focus-visible {
    background: rgba(255, 255, 255, 0.8);
    transform: translateY(-1px);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.95),
      0 14px 26px -18px rgba(17, 23, 45, 0.7);
  }

  .text-item:active { transform: scale(0.994); }

  .text-main { flex: 1; min-width: 0; }

  .text-title {
    font-size: 13px;
    font-weight: 550;
    line-height: 1.4;
    color: var(--label);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .text-meta { margin-top: 6px; }

  .cat-select {
    appearance: none;
    -webkit-appearance: none;
    max-width: 120px;
    padding: 3px 22px 3px 9px;
    border: none;
    outline: none;
    border-radius: 999px;
    font-size: 11px;
    font-weight: 560;
    color: var(--accent-deep);
    cursor: pointer;
    background-color: rgba(255, 255, 255, 0.68);
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%230a84ff' stroke-width='3' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 7px center;
    background-size: 10px;
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.85);
    transition: background-color 0.2s var(--ease);
  }

  .cat-select:hover { background-color: #fff; }

  .text-actions {
    display: flex;
    align-items: center;
    gap: 2px;
    flex-shrink: 0;
    opacity: 0.55;
    transition: opacity 0.24s var(--ease);
  }

  .text-item:hover .text-actions,
  .text-item:focus-within .text-actions { opacity: 1; }

  .icon-btn {
    width: 26px;
    height: 26px;
    padding: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: none;
    border-radius: 8px;
    color: var(--label-2);
    background: transparent;
    cursor: pointer;
    transition: background 0.2s var(--ease), color 0.2s var(--ease), transform 0.2s var(--spring);
  }

  .icon-btn svg { width: 14px; height: 14px; }
  .icon-btn:hover { color: var(--accent-deep); background: rgba(255, 255, 255, 0.9); }
  .icon-btn:active { transform: scale(0.92); }
  .icon-btn.danger:hover { color: #d1445a; background: rgba(255, 235, 238, 0.95); }

  /* ============ 密码页 ============ */
  .panel-form { max-width: 400px; }

  .form-body {
    display: flex;
    flex-direction: column;
    gap: 7px;
    padding-right: 2px;
  }

  .form-body label {
    margin: 5px 0 0 2px;
    font-size: 11.5px;
    font-weight: 600;
    letter-spacing: 0.01em;
    color: var(--label-3);
    text-transform: uppercase;
  }

  .input-shell {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 9px 11px;
    border-radius: 13px;
    background: rgba(255, 255, 255, 0.5);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.7),
      inset 0 1px 0 rgba(255, 255, 255, 0.9);
    transition: background 0.24s var(--ease), box-shadow 0.24s var(--ease), transform 0.24s var(--ease);
  }

  .input-shell:focus-within {
    background: #fff;
    transform: translateY(-1px);
    box-shadow:
      inset 0 0 0 1px rgba(10, 132, 255, 0.42),
      0 0 0 3.5px rgba(10, 132, 255, 0.13),
      0 10px 22px -18px rgba(17, 23, 45, 0.8);
  }

  .input-shell svg { width: 15px; height: 15px; color: var(--label-3); flex-shrink: 0; }

  .input-shell input {
    flex: 1;
    min-width: 0;
    border: none;
    outline: none;
    background: transparent;
    font-size: 13px;
    font-weight: 500;
    color: var(--label);
  }

  .input-shell input::placeholder { color: var(--label-3); font-weight: 450; }

  .inline-btn {
    width: 24px;
    height: 24px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: none;
    border-radius: 7px;
    background: transparent;
    color: var(--label-3);
    cursor: pointer;
    transition: background 0.2s var(--ease), color 0.2s var(--ease);
  }

  .inline-btn svg { width: 14px; height: 14px; }
  .inline-btn:hover { color: var(--accent-deep); background: rgba(255, 255, 255, 0.9); }

  .save-btn {
    margin-top: 13px;
    padding: 10px 12px;
    border: none;
    border-radius: 13px;
    font-size: 13.5px;
    font-weight: 640;
    letter-spacing: -0.01em;
    color: #fff;
    cursor: pointer;
    background: linear-gradient(180deg, #3a97ff, #0a7cf0);
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 0.45),
      0 12px 24px -14px rgba(10, 124, 240, 0.95);
    transition: transform 0.24s var(--spring), box-shadow 0.24s var(--ease), filter 0.24s var(--ease);
  }

  .save-btn:hover {
    transform: translateY(-1px);
    filter: brightness(1.04);
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 0.55),
      0 18px 30px -16px rgba(10, 124, 240, 0.95);
  }

  .save-btn:active { transform: scale(0.985); }

  .password-item {
    padding: 11px 12px;
    border-radius: var(--radius-md);
    background: rgba(255, 255, 255, 0.44);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.6),
      0 1px 2px rgba(17, 23, 45, 0.04);
  }

  .password-item:hover {
    background: rgba(255, 255, 255, 0.74);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.95),
      0 14px 26px -20px rgba(17, 23, 45, 0.7);
  }

  .password-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; }

  .password-title {
    font-size: 13.5px;
    font-weight: 620;
    letter-spacing: -0.01em;
    color: var(--label);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .password-line { display: flex; align-items: baseline; gap: 6px; margin-top: 6px; }

  .line-label {
    min-width: 34px;
    font-size: 10.5px;
    font-weight: 650;
    letter-spacing: 0.02em;
    text-transform: uppercase;
    color: var(--label-3);
  }

  .line-value {
    font-size: 12.5px;
    font-weight: 500;
    color: var(--label-2);
    line-height: 1.4;
    word-break: break-all;
  }

  .password-actions { display: flex; justify-content: flex-end; gap: 2px; margin-top: 8px; }

  /* ============ 浮动层 ============ */
  .toast-tip {
    position: fixed;
    left: 50%;
    bottom: 20px;
    z-index: 9999;
    transform: translateX(-50%);
    padding: 8px 15px;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 600;
    color: #fff;
    background: rgba(22, 26, 36, 0.66);
    -webkit-backdrop-filter: blur(24px) saturate(180%);
    backdrop-filter: blur(24px) saturate(180%);
    box-shadow:
      inset 0 1px 0 rgba(255, 255, 255, 0.22),
      0 16px 34px -16px rgba(10, 14, 25, 0.9);
  }

  .context-menu {
    position: fixed;
    z-index: 10000;
    min-width: 148px;
    padding: 5px;
    border-radius: 14px;
    background: rgba(248, 249, 253, 0.72);
    -webkit-backdrop-filter: blur(38px) saturate(190%);
    backdrop-filter: blur(38px) saturate(190%);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.85),
      inset 0 1px 0 rgba(255, 255, 255, 1),
      0 24px 48px -18px rgba(10, 14, 30, 0.6);
    animation: menu-in 0.2s var(--spring);
  }

  .image-menu { min-width: 140px; }

  .menu-item {
    width: 100%;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 7px 9px;
    border: none;
    border-radius: 9px;
    background: transparent;
    color: var(--label-2);
    font-size: 12.5px;
    font-weight: 540;
    cursor: pointer;
    transition: background 0.16s var(--ease), color 0.16s var(--ease);
  }

  .menu-item svg { width: 14px; height: 14px; }
  .menu-item:hover { background: rgba(10, 132, 255, 0.12); color: var(--accent-deep); }
  .menu-item.danger { color: #cf4859; }
  .menu-item.danger:hover { background: rgba(209, 68, 90, 0.13); color: #bf3144; }

  .menu-divider { height: 1px; margin: 4px 6px; background: rgba(60, 68, 96, 0.12); }

  /* ============ 图片预览 ============ */
  .preview-backdrop {
    position: fixed;
    inset: 0;
    z-index: 10001;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 18px;
    background: rgba(14, 18, 32, 0.5);
    -webkit-backdrop-filter: blur(22px) saturate(140%);
    backdrop-filter: blur(22px) saturate(140%);
  }

  .preview-stage {
    display: flex;
    flex-direction: column;
    gap: 10px;
    width: min(94vw, 1080px);
    height: min(90vh, 760px);
    min-width: 0;
    min-height: 0;
    animation: stage-in 0.3s var(--spring);
  }

  .preview-toolbar {
    align-self: center;
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px;
    border-radius: 16px;
    background: rgba(250, 251, 255, 0.6);
    -webkit-backdrop-filter: blur(34px) saturate(190%);
    backdrop-filter: blur(34px) saturate(190%);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.8),
      inset 0 1px 0 rgba(255, 255, 255, 1),
      0 22px 44px -20px rgba(8, 12, 24, 0.8);
  }

  .preview-tool {
    width: 30px;
    height: 30px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    border: none;
    border-radius: 10px;
    color: var(--label-2);
    background: transparent;
    cursor: pointer;
    transition: background 0.18s var(--ease), color 0.18s var(--ease), transform 0.18s var(--spring);
  }

  .preview-tool svg { width: 15px; height: 15px; }
  .preview-tool:hover { color: var(--accent-deep); background: rgba(255, 255, 255, 0.9); }
  .preview-tool:active { transform: scale(0.93); }

  .preview-tool.reset {
    width: 54px;
    font-size: 11.5px;
    font-weight: 650;
    color: var(--label);
    font-variant-numeric: tabular-nums;
    background: rgba(255, 255, 255, 0.55);
  }

  .preview-scroll {
    flex: 1;
    min-height: 0;
    display: grid;
    place-items: center;
    overflow: auto;
    border-radius: 20px;
    background:
      linear-gradient(45deg, rgba(255, 255, 255, 0.07) 25%, transparent 25%),
      linear-gradient(-45deg, rgba(255, 255, 255, 0.07) 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, rgba(255, 255, 255, 0.07) 75%),
      linear-gradient(-45deg, transparent 75%, rgba(255, 255, 255, 0.07) 75%),
      rgba(18, 22, 36, 0.42);
    background-size: 26px 26px;
    background-position: 0 0, 0 13px, 13px -13px, -13px 0;
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.14),
      0 30px 60px -30px rgba(0, 0, 0, 0.8);
  }

  .preview-scroll img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
    transform-origin: center center;
    transition: transform 0.14s var(--ease);
    border-radius: 6px;
    box-shadow: 0 24px 60px rgba(0, 0, 0, 0.4);
    user-select: none;
    -webkit-user-drag: none;
  }

  input::-ms-reveal,
  input::-ms-clear,
  input::-webkit-credentials-auto-fill-button { display: none !important; }

  :global(:focus-visible) { outline: 2px solid rgba(10, 132, 255, 0.55); outline-offset: 2px; }


  /* ============ 语言切换 ============ */
  .lang-toggle {
    position: absolute;
    right: 0;
    top: 50%;
    transform: translateY(-50%);
    display: inline-flex;
    gap: 2px;
    padding: 3px;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.32);
    -webkit-backdrop-filter: blur(18px) saturate(170%);
    backdrop-filter: blur(18px) saturate(170%);
    box-shadow:
      inset 0 0 0 1px rgba(255, 255, 255, 0.52),
      inset 0 1px 0 rgba(255, 255, 255, 0.8),
      0 8px 18px -14px rgba(17, 23, 45, 0.65);
  }

  .lang-btn {
    min-width: 27px;
    height: 21px;
    padding: 0 7px;
    border: none;
    border-radius: 999px;
    background: transparent;
    color: rgba(19, 23, 34, 0.48);
    font-size: 10.5px;
    font-weight: 680;
    letter-spacing: 0.01em;
    cursor: pointer;
    transition: background 0.26s var(--ease), color 0.26s var(--ease), transform 0.26s var(--spring),
      box-shadow 0.26s var(--ease);
  }

  .lang-btn:hover { color: var(--label); background: rgba(255, 255, 255, 0.55); }

  .lang-btn.active {
    color: #0a6fe0;
    background: #fff;
    box-shadow:
      inset 0 1px 0 #fff,
      0 4px 10px -6px rgba(19, 25, 48, 0.55),
      0 0 0 1px rgba(255, 255, 255, 0.9);
  }

  /* ============ 玻璃按钮：悬停高光 + 按压回弹 ============ */
  .tab-btn,
  .clear-btn,
  .chip,
  .add-chip,
  .icon-btn,
  .inline-btn,
  .save-btn,
  .preview-tool,
  .menu-item,
  .image-card,
  .lang-btn {
    position: relative;
    overflow: hidden;
    -webkit-tap-highlight-color: transparent;
  }

  /* 光斑掠过玻璃表面 */
  .tab-btn::after,
  .clear-btn::after,
  .chip::after,
  .add-chip::after,
  .save-btn::after,
  .preview-tool::after,
  .menu-item::after,
  .lang-btn::after {
    content: '';
    position: absolute;
    inset: 0;
    border-radius: inherit;
    pointer-events: none;
    background: linear-gradient(
      112deg,
      transparent 32%,
      rgba(255, 255, 255, 0.6) 46%,
      rgba(255, 255, 255, 0.16) 54%,
      transparent 68%
    );
    transform: translateX(-135%);
    opacity: 0;
    transition: transform 0.62s var(--ease), opacity 0.28s var(--ease);
  }

  .tab-btn:hover::after,
  .clear-btn:hover::after,
  .chip:hover::after,
  .add-chip:hover::after,
  .save-btn:hover::after,
  .preview-tool:hover::after,
  .menu-item:hover::after,
  .lang-btn:hover::after {
    transform: translateX(135%);
    opacity: 1;
  }

  /* 悬停：微微浮起、玻璃变亮 */
  .tab-btn:hover { transform: translateY(-0.5px); }

  .clear-btn:hover,
  .chip:hover,
  .add-chip:hover,
  .icon-btn:hover,
  .inline-btn:hover,
  .preview-tool:hover {
    transform: translateY(-1.5px);
  }

  .save-btn:hover { transform: translateY(-2px); }
  .menu-item:hover { transform: translateX(1px); }

  .image-card:hover { transform: translateY(-2px) scale(1.015); }

  /* 按下：内凹、轻微下沉，松手弹簧回弹 */
  .tab-btn:active,
  .clear-btn:active,
  .chip:active,
  .add-chip:active,
  .icon-btn:active,
  .inline-btn:active,
  .preview-tool:active,
  .menu-item:active,
  .lang-btn:active {
    transform: translateY(1px) scale(0.945);
    background: rgba(255, 255, 255, 0.95);
    box-shadow:
      inset 0 2px 7px rgba(17, 23, 45, 0.24),
      inset 0 0 0 1px rgba(255, 255, 255, 0.55);
    transition-duration: 0.11s;
  }

  .chip:active,
  .chip.active:active {
    transform: translateY(0.5px) scale(0.955);
    transition-duration: 0.12s;
  }

  .chip.active:active {
    background-color: #0a79e8;
    box-shadow: inset 0 2px 8px rgba(6, 60, 130, 0.38);
  }

  .save-btn:active {
    transform: translateY(1px) scale(0.982);
    background: linear-gradient(180deg, #2b87f0, #0770dd);
    box-shadow:
      inset 0 2px 9px rgba(3, 40, 90, 0.4),
      0 4px 10px -8px rgba(10, 124, 240, 0.9);
  }

  .image-card:active {
    transform: translateY(0) scale(0.975);
    box-shadow:
      inset 0 2px 10px rgba(17, 23, 45, 0.3),
      inset 0 0 0 1px rgba(255, 255, 255, 0.6);
  }

  .text-item:active {
    transform: translateY(0) scale(0.988);
    background: rgba(255, 255, 255, 0.66);
    box-shadow:
      inset 0 2px 8px rgba(17, 23, 45, 0.16),
      inset 0 0 0 1px rgba(255, 255, 255, 0.7);
  }

  /* ============ 动效 ============ */
  @keyframes shell-reveal {
    from { opacity: 0; transform: translateY(10px) scale(0.995); }
    to { opacity: 1; transform: translateY(0) scale(1); }
  }

  @keyframes panel-in {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }

  @keyframes chip-pop {
    0% { transform: scale(0.9); }
    58% { transform: scale(1.055); }
    100% { transform: scale(1); }
  }

  @keyframes menu-in {
    from { opacity: 0; transform: scale(0.94) translateY(-4px); }
    to { opacity: 1; transform: scale(1) translateY(0); }
  }

  @keyframes stage-in {
    from { opacity: 0; transform: scale(0.97); }
    to { opacity: 1; transform: scale(1); }
  }

  @keyframes float-y {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-7px); }
  }

  @keyframes drift-a {
    0%, 100% { transform: translate3d(0, 0, 0) scale(1); }
    50% { transform: translate3d(6%, 7%, 0) scale(1.09); }
  }

  @keyframes drift-b {
    0%, 100% { transform: translate3d(0, 0, 0) scale(1.05); }
    50% { transform: translate3d(-7%, 5%, 0) scale(0.95); }
  }

  @keyframes drift-c {
    0%, 100% { transform: translate3d(0, 0, 0) scale(1); }
    50% { transform: translate3d(5%, -6%, 0) scale(1.12); }
  }

  @media (prefers-reduced-motion: reduce) {
    :global(*),
    .blob,
    .empty-illustration {
      animation: none !important;
      transition-duration: 0.01ms !important;
    }
  }

  /* ============ 窄窗口 ============ */
  @media (max-width: 760px), (max-height: 700px) {
    .app-shell { padding: 11px; gap: 9px; }
    .tab-btn { font-size: 12.5px; padding: 6px 16px; }
    .search-bar { padding: 8px 11px; border-radius: 13px; }
    .search-bar input { font-size: 12.5px; }
    .panel { padding: 11px; border-radius: var(--radius-lg); }
    .panel-head { margin-bottom: 9px; }
    .panel-head h3 { font-size: 13.5px; gap: 6px; }
    .panel-head h3 svg { width: 15px; height: 15px; }
    .empty-illustration { width: 76px; height: 76px; }
    .empty-illustration svg { width: 36px; height: 36px; }
    .empty-title { font-size: 13.5px; }
    .empty-sub { font-size: 11.5px; max-width: 210px; }
    .workspace, .password-workspace { gap: 9px; }
    .image-grid { grid-template-columns: repeat(auto-fill, minmax(72px, 1fr)); gap: 7px; }
    .text-item { padding: 9px 10px; border-radius: 13px; }
    .text-title { font-size: 12.5px; }
    .password-title { font-size: 12.5px; }
    .line-value { font-size: 12px; }
  }
</style>
