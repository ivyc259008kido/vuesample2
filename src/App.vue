<template>
 <div id="app">
    <header>
      <div>
        <div class="logo">BIBLIO<span>FIND</span></div>
      </div>
      <div class="tagline">Google Books 書籍検索</div>
      <button class="header-fav-btn" @click="toggleFavoritesView" :class="{ active: showFavorites }">
        ★ お気に入り<span v-if="favorites.length" class="fav-count">{{ favorites.length }}</span>
      </button>
      <button class="header-ai-btn" @click="openAIPanel">
        <span class="ai-dot"></span>
        AI に相談する
      </button>
    </header>

    <div class="search-area">
      <select class="genre-select" v-model="searchType" title="検索タイプ">
        <option value="inauthor">著者名</option>
        <option value="intitle">タイトル</option>
      </select>
      <div class="search-wrap">
        <input type="text" v-model="query" placeholder="タイトル・著者名で検索…" @keydown.enter="search" />
        <button @click="search">SEARCH</button>
      </div>
    </div>

    <div id="status">{{ status }}</div>

    <!-- AI推薦バー -->
    <div id="ai-suggest-bar" :class="{ show: books.length > 0 }">
      <p>📚 AIに「この中でおすすめを教えて」と聞けます</p>
      <button @click="askAIAboutResults">AI に絞り込んでもらう</button>
    </div>

        <div id="results">
      <div v-if="loading" class="loading">
        <div class="spinner"></div>検索中…
      </div>
      <div v-else-if="displayedBooks.length === 0" class="empty">
        <div class="empty-title">{{ showFavorites ? '★' : '📚' }}</div>
        <div>{{ showFavorites ? 'お気に入りはまだありません' : '検索結果がここに表示されます' }}</div>
      </div>
      <div
        v-for="(book, i) in displayedBooks"
        :key="book.id"
        class="book-card"
        :style="{ animationDelay: i * 0.04 + 's' }"
        @click="openModal(book)"
      >
        <div class="cover-wrap">
          <button
            class="fav-star"
            :class="{ active: isFavorite(book.id) }"
            @click="toggleFavorite(book, $event)"
            title="お気に入りに追加"
          >★</button>
          <img v-if="book.thumbnail" :src="book.thumbnail" alt="表紙" loading="lazy">
          <div v-else class="no-cover">
            <span class="no-cover-icon">📖</span>
            <span>No Cover</span>
          </div>
        </div>
        <div class="card-body">
          <div class="book-title">{{ book.title }}</div>
          <div class="book-author">{{ book.authors }}</div>
          <div class="book-meta">
            <span class="meta-tag"><strong>{{ book.year }}</strong> 年</span>
            <span v-if="book.pages !== '—'" class="meta-tag"><strong>{{ book.pages }}</strong> ページ</span>
          </div>
          <div v-if="book.description" class="book-desc">{{ book.description }}</div>
          <div class="subject-tags">
            <span v-for="s in book.categories.slice(0,3)" :key="s" class="subject-tag">{{ s }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- モーダル -->
    <div class="modal-overlay" :class="{ open: modalOpen }" @click.self="closeModal">
      <div class="modal" v-if="selectedBook">
        <div class="modal-header">
          <span>Book Detail</span>
          <button class="close-btn" @click="closeModal">✕</button>
        </div>
        <div class="modal-body">
          <div class="modal-cover">
            <img v-if="selectedBook.largeCover" :src="selectedBook.largeCover" alt="表紙">
            <div v-else class="no-cover" style="width:140px;height:200px;background:var(--cream);border:1px solid #ddd;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:0.4rem;color:var(--muted);font-size:0.78rem;">
              <span style="font-size:2.5rem;opacity:0.3">📖</span>
              <span>No Cover</span>
            </div>
          </div>
          <div class="modal-info">
            <div class="modal-title-row">
              <div class="modal-title">{{ selectedBook.title }}</div>
              <button
                class="fav-star modal-fav-star"
                :class="{ active: isFavorite(selectedBook.id) }"
                @click="toggleFavorite(selectedBook, $event)"
                title="お気に入りに追加"
              >★</button>
            </div>
            <div class="modal-author">{{ selectedBook.authors }}</div>
            <div class="modal-meta-grid">
              <div class="modal-meta-item"><label>出版年</label><span>{{ selectedBook.year }}</span></div>
              <div class="modal-meta-item"><label>ページ数</label><span>{{ selectedBook.pages }}</span></div>
              <div class="modal-meta-item"><label>出版社</label><span>{{ selectedBook.publisher }}</span></div>
              <div class="modal-meta-item"><label>言語</label><span>{{ selectedBook.language }}</span></div>
            </div>
            <div class="modal-subjects">
              <span v-for="s in selectedBook.categories" :key="s" class="subject-tag">{{ s }}</span>
            </div>
            <div v-if="selectedBook.description" class="modal-desc">
              <h4>概要</h4>{{ selectedBook.description }}
            </div>
            <a v-if="selectedBook.infoLink" class="modal-link" :href="selectedBook.infoLink" target="_blank" rel="noopener">Google Books で見る →</a>
          </div>
        </div>
      </div>
    </div>

    <!-- AIパネル オーバーレイ -->
    <div id="ai-overlay" :class="{ show: aiPanelOpen }" @click="closeAIPanel"></div>

    <!-- AI チャットパネル -->
    <div id="ai-panel" :class="{ open: aiPanelOpen }">
      <div class="ai-panel-header">
        <div class="ai-panel-title">
          📖 Book Advisor
          <span class="ai-label">GEMINI AI</span>
        </div>
        <button class="ai-close-btn" @click="closeAIPanel">✕</button>
      </div>

      <!-- Geminiキー未設定の場合の警告（.envが空の時だけ表示） -->
      <div class="ai-key-warning" v-if="!geminiApiKey">
        ⚠️ <strong>Gemini APIキーを設定してください</strong><br>
        <code>.env</code> の <code>VITE_GEMINI_API_KEY</code> にキーを入力してください。
      </div>

      <!-- クイック質問タグ -->
      <div class="ai-quick-tags">
        <span>クイック質問</span>
        <button class="quick-tag" @click="quickAsk('泣けるおすすめ小説を教えてください')">感動できる本</button>
        <button class="quick-tag" @click="quickAsk('初心者でも読みやすいSF小説を教えてください')">SF入門</button>
        <button class="quick-tag" @click="quickAsk('2～3時間で読めるおすすめ小説を教えてください')">短編・短時間</button>
        <button class="quick-tag" @click="quickAsk('仕事や人生に役立つ本を教えてください')">学びになる本</button>
        <button class="quick-tag" @click="quickAsk('どんでん返しが面白いミステリーを教えてください')">ミステリー名作</button>
        <button class="quick-tag" @click="quickAsk('現代日本文学のおすすめを教えてください')">現代日本文学</button>
      </div>

      <div id="ai-messages" ref="aiMessages">
        <!-- 初期メッセージ -->
        <div class="msg ai">
          <div class="msg-role">Book Advisor</div>
          <div class="msg-bubble">
            こんにちは！読書のお手伝いをします📚<br><br>
            好みのジャンル、最近読んだ本、気分などを教えてください。あなたにぴったりの本をGoogle Booksから探してご提案します。<br><br>
            例：「ミステリーが好きで、次は歴史小説を読んでみたい」
          </div>
        </div>

        <div v-for="(msg, i) in chatMessages" :key="i" :class="['msg', msg.role]">
          <div class="msg-role">{{ msg.role === 'user' ? 'あなた' : 'Book Advisor' }}</div>
          <div class="msg-bubble">
            <div v-html="msg.html"></div>

            <!-- AIおすすめ本カード（本ごとに個別の理由をつける） -->
            <template v-if="msg.books">
              <div
                v-for="(b, bi) in msg.books"
                :key="b.id"
                class="ai-book-card"
                @click="openModalFromAI(b)"
              >
                <img v-if="b.thumbnail" class="ai-book-cover" :src="b.thumbnail" alt="表紙">
                <div v-else class="ai-book-cover-placeholder">📖</div>
                <div class="ai-book-info">
                  <div class="ai-book-title">{{ b.title }}</div>
                  <div class="ai-book-author">{{ b.authors }}</div>
                  <div class="ai-book-reason">{{ (msg.perReasons && msg.perReasons[bi]) || msg.reason || '' }}</div>
                </div>
              </div>
            </template>
          </div>
        </div>

        <div v-if="isAIThinking" class="msg ai">
          <div class="msg-role">Book Advisor</div>
          <div class="typing-indicator">
            <div class="typing-dot"></div>
            <div class="typing-dot"></div>
            <div class="typing-dot"></div>
          </div>
        </div>
      </div>

      <div class="ai-input-area">
        <textarea
          id="ai-input"
          v-model="aiInputText"
          placeholder="好みや気分を教えてください…"
          rows="1"
          @keydown="handleAIKey"
        ></textarea>
        <button id="ai-send" @click="sendAIMessage" :disabled="isAIThinking">➤</button>
      </div>
      <div class="ai-hint">Enter で送信　Shift+Enter で改行</div>
    </div>
  </div>
</template>

<script>
const BOOKS_API_KEY = import.meta.env.VITE_API_KEY;
const GEMINI_API_KEY = import.meta.env.VITE_GEMINI_API_KEY;

export default {
  data() {
    return {
      // 書籍検索
      query: '',
      searchType: 'inauthor',
      books: [],
      status: 'キーワードを入力して書籍を検索してください',
      loading: false,
      modalOpen: false,
      selectedBook: null,
      favorites: JSON.parse(localStorage.getItem('bibliofind_favorites') || '[]'),
      showFavorites: false,
      // AIパネル
      aiPanelOpen: false,
      aiInputText: '',
      chatMessages: [],
      chatHistory: [],
      isAIThinking: false,
      geminiApiKey: GEMINI_API_KEY,
    }
  },
  computed: {
    displayedBooks() {
      return this.showFavorites ? this.favorites : this.books;
    },
  },
  methods: {
    // ═══════════════════════════════
    //  書籍検索
    // ═══════════════════════════════
    async search() {
      const q = this.query.trim();
      const searchQuery = q
        ? (this.searchType === 'intitle' ? `intitle:${encodeURIComponent(q)}` : `inauthor:${encodeURIComponent(q)}`)
        : 'bestseller';

      this.loading = true;
      this.books = [];
      this.status = '検索中…';

      try {
        let url = `https://www.googleapis.com/books/v1/volumes?q=${searchQuery}&maxResults=20&langRestrict=ja`;
        if (BOOKS_API_KEY) url += `&key=${BOOKS_API_KEY}`;
        const res = await fetch(url);
        const data = await res.json();
        if (data.error) throw new Error(data.error.message);
        const items = data.items || [];
        this.books = items.map(this.extractBook);
        this.status = `約 ${(data.totalItems ?? 0).toLocaleString()} 件中 ${items.length} 件を表示`;
      } catch (e) {
        this.status = 'エラー: ' + e.message;
      } finally {
        this.loading = false;
      }
    },

    extractBook(item) {
      const info = item.volumeInfo || {};
      return {
        id: item.id,
        title: info.title || '(タイトル不明)',
        authors: info.authors ? info.authors.slice(0, 2).join(', ') : '著者不明',
        year: info.publishedDate ? info.publishedDate.slice(0, 4) : '—',
        pages: info.pageCount || '—',
        thumbnail: info.imageLinks?.thumbnail?.replace('http://', 'https://') || null,
        largeCover: info.imageLinks?.large?.replace('http://', 'https://')
          || info.imageLinks?.thumbnail?.replace('http://', 'https://').replace('zoom=1', 'zoom=2')
          || null,
        description: info.description || '',
        categories: info.categories || [],
        infoLink: info.infoLink || null,
        publisher: info.publisher || '—',
        language: info.language || '—',
      };
    },

    openModal(book) {
      this.selectedBook = book;
      this.modalOpen = true;
    },

    closeModal() {
      this.modalOpen = false;
    },
  isFavorite(id) {
      return this.favorites.some(b => b.id === id);
    },

    toggleFavorite(book, event) {
      if (event) event.stopPropagation();
      const idx = this.favorites.findIndex(b => b.id === book.id);
      if (idx >= 0) {
        this.favorites.splice(idx, 1);
      } else {
        this.favorites.push(book);
      }
      localStorage.setItem('bibliofind_favorites', JSON.stringify(this.favorites));
    },

    toggleFavoritesView() {
      this.showFavorites = !this.showFavorites;
      if (this.showFavorites) {
        this.status = `お気に入り ${this.favorites.length} 件`;
      } else {
        this.status = 'キーワードを入力して書籍を検索してください';
      }
    },

    // ═══════════════════════════════
    //  AI パネル
    // ═══════════════════════════════
    openAIPanel() {
      this.aiPanelOpen = true;
      this.$nextTick(() => {
        document.getElementById('ai-input')?.focus();
      });
    },

    closeAIPanel() {
      this.aiPanelOpen = false;
    },

    askAIAboutResults() {
      this.openAIPanel();
      const titles = this.books.slice(0, 10).map(b => `・${b.title}（${b.authors}）`).join('\n');
      this.aiInputText = `今の検索結果の中でおすすめを教えてください。\n\n現在の検索結果：\n${titles}`;
      setTimeout(() => this.sendAIMessage(), 400);
    },

    quickAsk(text) {
      this.aiInputText = text;
      this.sendAIMessage();
    },

    handleAIKey(e) {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        this.sendAIMessage();
      }
    },

    async sendAIMessage() {
      if (this.isAIThinking) return;
      const text = this.aiInputText.trim();
      if (!text) return;

      if (!GEMINI_API_KEY) {
        this.chatMessages.push({ role: 'ai', html: '⚠️ Gemini APIキーが設定されていません。.env の VITE_GEMINI_API_KEY にキーを入力してください。' });
        return;
      }

      this.aiInputText = '';
      this.chatMessages.push({ role: 'user', html: this.escHtml(text).replace(/\n/g, '<br>') });
      this.isAIThinking = true;
      this.scrollToBottom();

      try {
        this.chatHistory.push({ role: 'user', parts: [{ text }] });

        const systemContext = `あなたは書籍推薦の専門家です。ユーザーの好みや気分を聞いて、最適な本を提案します。返答は日本語で簡潔に。
本を推薦する場合は返答の最後に必ず以下を含めること：
SEARCH_BOOKS:{"keyword":"検索キーワード（英語または日本語）","reason":"なぜこの本を勧めるか一言"}
挨拶や雑談の場合はSEARCH_BOOKSは不要。`;

        const response = await fetch(
          `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              system_instruction: { parts: [{ text: systemContext }] },
              contents: this.chatHistory,
              generationConfig: { maxOutputTokens: 2048, temperature: 0.8,
              thinkingConfig: {
                thinkingBudget: 0
              }
            },
          }),
        }
      );

        const data = await response.json();
        if (data.error) throw new Error(JSON.stringify(data.error));

        const aiText = data.candidates?.[0]?.content?.parts?.[0]?.text || '返答を取得できませんでした。';
        this.chatHistory.push({ role: 'model', parts: [{ text: aiText }] });

        const searchMatch = aiText.match(/SEARCH_BOOKS:(\{.*?\})/s);
        const displayText = aiText.replace(/SEARCH_BOOKS:\{.*?\}/s, '').trim();

        this.chatMessages.push({ role: 'ai', html: this.escHtml(displayText).replace(/\n/g, '<br>') });
        this.isAIThinking = false;
        this.scrollToBottom();

        if (searchMatch) {
          try {
            const searchData = JSON.parse(searchMatch[1]);
            await this.searchAndShowAIBooks(searchData.keyword, searchData.reason);
          } catch (e) {
            console.warn('SEARCH_BOOKS parse error', e);
          }
        }

      } catch (e) {
        this.isAIThinking = false;
        this.chatMessages.push({ role: 'ai', html: `⚠️ エラーが発生しました: ${this.escHtml(e.message)}` });
        this.scrollToBottom();
      }
    },

    // ═══════════════════════════════
    //  AI推薦: Google Booksで検索 → 本ごとの理由を生成 → 表示
    // ═══════════════════════════════
    async searchAndShowAIBooks(keyword, reason) {
      this.isAIThinking = true;
      this.scrollToBottom();

      try {
        let url = `https://www.googleapis.com/books/v1/volumes?q=${encodeURIComponent(keyword)}&maxResults=4&langRestrict=ja`;
        if (BOOKS_API_KEY) url += `&key=${BOOKS_API_KEY}`;
        const res = await fetch(url);
        const data = await res.json();

        if (data.error || !data.items || data.items.length === 0) {
          this.isAIThinking = false;
          this.chatMessages.push({ role: 'ai', html: '関連する書籍が見つかりませんでした。別のキーワードで試してみてください。' });
          this.scrollToBottom();
          return;
        }

        const aiBooks = data.items.slice(0, 4).map(this.extractBook);

        // 本ごとの個別理由をGeminiに生成してもらう
        const perReasons = await this.getPerBookReasons(aiBooks, keyword, reason);

        this.isAIThinking = false;
        this.chatMessages.push({
          role: 'ai',
          html: '<strong>📚 おすすめの書籍</strong>',
          books: aiBooks,
          reason,
          perReasons,
        });
        this.scrollToBottom();

      } catch (e) {
        this.isAIThinking = false;
        console.error('AI book search error', e);
      }
    },

    // ═══════════════════════════════
    //  AI: 検索結果の本ごとに、個別の推薦理由を生成
    // ═══════════════════════════════
    async getPerBookReasons(books, keyword, overallReason) {
      const bookList = books.map((b, i) => ({
        index: i,
        title: b.title,
        authors: b.authors,
        description: (b.description || '').slice(0, 150),
      }));

            const prompt = `ユーザーは「${keyword}」に関連する本を探しています（全体の背景：${overallReason || 'なし'}）。

以下の書籍リストそれぞれについて、なぜこの本がおすすめか、日本語で20〜40文字程度の一言理由を書いてください。

【重要なルール】
・同じ、または似た内容の理由文を2冊以上で使い回すことは禁止です。
・シリーズ物や同じ著者の巻違いが並んでいる場合は、説明文(description)が似ていても、タイトルに含まれる巻数・サブタイトルの違いに注目して、必ず本ごとに異なる切り口の理由を考えてください（例：「シリーズ第1巻で世界観を掴むのに最適」「中盤の巻で物語が大きく動く」など）。
・「SFの面白さを堪能できます」のような一般論だけで終わらせず、その本固有の特徴に触れてください。

書籍リスト：
${JSON.stringify(bookList, null, 2)}

必ず以下のJSON配列形式のみで返答してください（説明文や前置きは一切不要）：
[{"index":0,"reason":"..."},{"index":1,"reason":"..."}, ...]`;

      try {
        const response = await fetch(
          `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              contents: [{ role: 'user', parts: [{ text: prompt }] }],
              generationConfig: { maxOutputTokens: 2048, temperature: 0.7 },
            }),
          }
        );

        const data = await response.json();
        const text = data.candidates?.[0]?.content?.parts?.[0]?.text || '';
        const cleaned = text.replace(/```json|```/g, '').trim();
        const parsed = JSON.parse(cleaned);

        const reasons = [];
        parsed.forEach(item => { reasons[item.index] = item.reason; });
        return reasons;

      } catch (e) {
        console.warn('getPerBookReasons failed, falling back to overall reason', e);
        return [];
      }
    },

    openModalFromAI(book) {
      this.closeAIPanel();
      setTimeout(() => {
        this.selectedBook = book;
        this.modalOpen = true;
      }, 300);
    },

    scrollToBottom() {
      this.$nextTick(() => {
        const el = this.$refs.aiMessages;
        if (el) el.scrollTop = el.scrollHeight;
      });
    },

    escHtml(str) {
      return String(str)
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;');
    },
  }
}
</script>

<style>
:root {
  --ink: #1a1008;
  --paper: #f5f0e8;
  --cream: #ede6d6;
  --accent: #c0392b;
  --gold: #b8860b;
  --muted: #8a7968;
  --card-bg: #fff9f0;
}
* { margin: 0; padding: 0; box-sizing: border-box; }
body { background: var(--paper); color: var(--ink); font-family: 'DM Sans', sans-serif; min-height: 100vh; }

/* ヘッダー */
header { background: var(--ink); color: var(--paper); padding: 2rem 2rem 1.5rem; display: flex; align-items: flex-end; gap: 1.5rem; flex-wrap: wrap; }
.logo { font-family: 'Playfair Display', serif; font-size: 2.8rem; font-weight: 900; letter-spacing: -1px; line-height: 1; color: var(--paper); }
.logo span { color: var(--accent); }
.tagline { font-size: 0.78rem; letter-spacing: 0.2em; text-transform: uppercase; color: var(--muted); margin-bottom: 0.3rem; }
.header-ai-btn { margin-left: auto; background: var(--accent); color: white; border: none; padding: 0.6rem 1.2rem; font-family: 'DM Sans', sans-serif; font-size: 0.82rem; letter-spacing: 0.1em; text-transform: uppercase; cursor: pointer; display: flex; align-items: center; gap: 0.5rem; transition: background 0.2s; align-self: center; }
.header-ai-btn:hover { background: #a93226; }
.ai-dot { width: 8px; height: 8px; background: #fff; border-radius: 50%; animation: pulse 1.5s ease-in-out infinite; }
@keyframes pulse { 0%, 100% { opacity: 1; transform: scale(1); } 50% { opacity: 0.4; transform: scale(0.7); } }

/* 検索エリア */
.search-area { background: var(--cream); border-bottom: 2px solid var(--ink); padding: 1.5rem 2rem; display: flex; flex-wrap: wrap; gap: 0.75rem; align-items: center; }
.search-wrap { display: flex; flex: 1; min-width: 220px; border: 2px solid var(--ink); background: white; }
.search-wrap input { flex: 1; padding: 0.7rem 1rem; font-family: 'DM Sans', sans-serif; font-size: 1rem; border: none; outline: none; background: transparent; color: var(--ink); }
.search-wrap button { background: var(--ink); color: var(--paper); border: none; padding: 0 1.2rem; font-family: 'Playfair Display', serif; font-size: 0.85rem; letter-spacing: 0.05em; cursor: pointer; transition: background 0.2s; }
.search-wrap button:hover { background: var(--accent); }
.genre-select { padding: 0.7rem 2.5rem 0.7rem 1rem; border: 2px solid var(--ink); background: white url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%231a1008' d='M6 8L1 3h10z'/%3E%3C/svg%3E") no-repeat right 0.8rem center; appearance: none; -webkit-appearance: none; font-family: 'DM Sans', sans-serif; font-size: 0.9rem; color: var(--ink); cursor: pointer; outline: none; min-width: 120px; }

/* ステータス */
#status { padding: 0.6rem 2rem; font-size: 0.82rem; color: var(--muted); background: var(--paper); border-bottom: 1px solid var(--cream); min-height: 2rem; letter-spacing: 0.03em; }

/* AI推薦バー */
#ai-suggest-bar { display: none; background: var(--ink); color: var(--paper); padding: 0.8rem 2rem; align-items: center; gap: 1rem; border-bottom: 1px solid #333; }
#ai-suggest-bar.show { display: flex; }
#ai-suggest-bar p { font-size: 0.82rem; color: var(--muted); flex: 1; }
#ai-suggest-bar button { background: var(--accent); color: white; border: none; padding: 0.5rem 1rem; font-size: 0.78rem; letter-spacing: 0.08em; text-transform: uppercase; cursor: pointer; white-space: nowrap; }
#ai-suggest-bar button:hover { background: #a93226; }

/* 結果グリッド */
#results { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 0; padding: 0; }
.book-card { background: var(--card-bg); border-right: 1px solid var(--cream); border-bottom: 1px solid var(--cream); display: flex; flex-direction: column; transition: box-shadow 0.2s, transform 0.2s; cursor: pointer; overflow: hidden; animation: fadeUp 0.4s ease both; }
@keyframes fadeUp { from { opacity: 0; transform: translateY(16px); } to { opacity: 1; transform: translateY(0); } }
.book-card:hover { box-shadow: 4px 4px 0 var(--ink); transform: translate(-2px, -2px); z-index: 1; position: relative; }
.cover-wrap { background: var(--cream); height: 220px; display: flex; align-items: center; justify-content: center; overflow: hidden; border-bottom: 1px solid #ddd; position: relative; }
.cover-wrap img { height: 100%; width: 100%; object-fit: contain; }
.no-cover { display: flex; flex-direction: column; align-items: center; gap: 0.4rem; color: var(--muted); font-size: 0.78rem; letter-spacing: 0.1em; text-transform: uppercase; }
.no-cover-icon { font-size: 2.5rem; opacity: 0.3; }
.card-body { padding: 1rem 1.2rem 1.2rem; flex: 1; display: flex; flex-direction: column; gap: 0.3rem; min-height: 140px; }
.book-title { font-family: 'Playfair Display', serif; font-size: 1.05rem; font-weight: 700; line-height: 1.3; color: var(--ink); }
.book-author { font-size: 0.82rem; color: var(--accent); font-weight: 500; letter-spacing: 0.03em; }
.book-meta { display: flex; gap: 1rem; margin-top: 0.3rem; }
.meta-tag { font-size: 0.72rem; color: var(--muted); letter-spacing: 0.05em; text-transform: uppercase; }
.meta-tag strong { color: var(--ink); font-weight: 500; }
.book-desc { font-size: 0.82rem; color: #555; line-height: 1.55; margin-top: 0.5rem; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }
.subject-tags { display: flex; flex-wrap: wrap; gap: 0.3rem; margin-top: 0.6rem; }
.subject-tag { background: var(--cream); border: 1px solid #ccc; font-size: 0.68rem; padding: 0.15rem 0.5rem; color: var(--muted); letter-spacing: 0.04em; }
.loading { grid-column: 1 / -1; text-align: center; padding: 4rem 2rem; color: var(--muted); }
.spinner { width: 40px; height: 40px; border: 3px solid var(--cream); border-top-color: var(--ink); border-radius: 50%; animation: spin 0.8s linear infinite; margin: 0 auto 1rem; }
@keyframes spin { to { transform: rotate(360deg); } }
.empty { grid-column: 1 / -1; text-align: center; padding: 6rem 2rem; color: var(--muted); }
.empty-title { font-family: 'Playfair Display', serif; font-size: 2rem; opacity: 0.2; margin-bottom: 0.5rem; }
.header-fav-btn { background: transparent; color: var(--paper); border: 1px solid #555; padding: 0.6rem 1.1rem; font-family: 'DM Sans', sans-serif; font-size: 0.82rem; letter-spacing: 0.05em; cursor: pointer; align-self: center; transition: all 0.2s; display: flex; align-items: center; gap: 0.4rem; }
.header-fav-btn:hover { border-color: var(--gold); color: var(--gold); }
.header-fav-btn.active { background: var(--gold); border-color: var(--gold); color: var(--ink); }
.fav-count { background: var(--accent); color: white; font-size: 0.68rem; padding: 0.05rem 0.4rem; border-radius: 10px; }

.fav-star { position: absolute; top: 0.5rem; right: 0.5rem; z-index: 2; background: rgba(26,16,8,0.55); border: none; color: rgba(255,255,255,0.7); font-size: 1.3rem; width: 2rem; height: 2rem; border-radius: 50%; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s; line-height: 1; }
.fav-star:hover { background: rgba(26,16,8,0.8); color: var(--gold); transform: scale(1.1); }
.fav-star.active { color: var(--gold); background: rgba(26,16,8,0.75); }

.modal-title-row { display: flex; align-items: flex-start; justify-content: space-between; gap: 0.8rem; }
.modal-fav-star { position: static; background: none; color: var(--muted); font-size: 1.6rem; width: auto; height: auto; flex-shrink: 0; }
.modal-fav-star:hover { color: var(--gold); transform: none; }
.modal-fav-star.active { color: var(--gold); }

/* モーダル */
.modal-overlay { display: none; position: fixed; inset: 0; background: rgba(26,16,8,0.75); z-index: 100; align-items: center; justify-content: center; padding: 1rem; }
.modal-overlay.open { display: flex; }
.modal { background: var(--card-bg); max-width: 640px; width: 100%; max-height: 90vh; overflow-y: auto; border: 2px solid var(--ink); display: flex; flex-direction: column; animation: slideUp 0.3s ease; }
@keyframes slideUp { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: translateY(0); } }
.modal-header { background: var(--ink); color: var(--paper); padding: 0.8rem 1.2rem; display: flex; justify-content: space-between; align-items: center; }
.modal-header span { font-size: 0.75rem; letter-spacing: 0.15em; text-transform: uppercase; }
.close-btn { background: none; border: none; color: var(--paper); font-size: 1.4rem; cursor: pointer; line-height: 1; padding: 0 0.2rem; }
.modal-body { display: flex; gap: 1.5rem; padding: 1.5rem; flex-wrap: wrap; }
.modal-cover { width: 140px; flex-shrink: 0; }
.modal-cover img { width: 100%; border: 1px solid #ddd; }
.modal-info { flex: 1; min-width: 180px; }
.modal-title { font-family: 'Playfair Display', serif; font-size: 1.4rem; font-weight: 700; line-height: 1.3; margin-bottom: 0.4rem; }
.modal-author { color: var(--accent); font-size: 0.9rem; font-weight: 500; margin-bottom: 1rem; }
.modal-meta-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 0.6rem 1rem; margin-bottom: 1rem; }
.modal-meta-item label { font-size: 0.68rem; letter-spacing: 0.12em; text-transform: uppercase; color: var(--muted); display: block; }
.modal-meta-item span { font-size: 0.9rem; font-weight: 500; }
.modal-desc { font-size: 0.85rem; line-height: 1.65; color: #444; border-top: 1px solid var(--cream); padding-top: 1rem; }
.modal-desc h4 { font-family: 'Playfair Display', serif; font-size: 0.85rem; letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 0.5rem; color: var(--ink); }
.modal-subjects { display: flex; flex-wrap: wrap; gap: 0.3rem; margin-top: 0.8rem; }
.modal-link { display: inline-block; margin-top: 1rem; padding: 0.5rem 1rem; background: var(--ink); color: var(--paper); font-size: 0.78rem; letter-spacing: 0.1em; text-transform: uppercase; text-decoration: none; transition: background 0.2s; }
.modal-link:hover { background: var(--accent); }

/* AI パネル */
#ai-overlay { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.4); z-index: 190; }
#ai-overlay.show { display: block; }
#ai-panel { position: fixed; top: 0; right: 0; width: 555pxpx; max-width: 90vw; height: 100vh; background: #1a1008; color: var(--paper); display: flex; flex-direction: column; z-index: 200; transform: translateX(100%); transition: transform 0.35s cubic-bezier(0.4,0,0.2,1); border-left: 2px solid #333; }
#ai-panel.open { transform: translateX(0); }
.ai-panel-header { padding: 1.2rem 1.5rem; border-bottom: 1px solid #333; display: flex; align-items: center; justify-content: space-between; flex-shrink: 0; }
.ai-panel-title { font-family: 'Playfair Display', serif; font-size: 1.1rem; display: flex; align-items: center; gap: 0.6rem; }
.ai-label { background: var(--accent); color: white; font-size: 0.6rem; padding: 0.15rem 0.4rem; letter-spacing: 0.1em; font-family: 'DM Sans', sans-serif; font-weight: 500; }
.ai-close-btn { background: none; border: none; color: var(--muted); font-size: 1.3rem; cursor: pointer; padding: 0.2rem; transition: color 0.2s; }
.ai-close-btn:hover { color: var(--paper); }
.ai-quick-tags { padding: 0.8rem 1.5rem; border-bottom: 1px solid #222; display: flex; flex-wrap: wrap; gap: 0.4rem; flex-shrink: 0; }
.ai-quick-tags span { font-size: 0.7rem; color: var(--muted); letter-spacing: 0.08em; text-transform: uppercase; width: 100%; margin-bottom: 0.2rem; }
.quick-tag { background: #2a2018; border: 1px solid #444; color: #ccc; font-size: 0.72rem; padding: 0.3rem 0.7rem; cursor: pointer; transition: all 0.15s; font-family: 'DM Sans', sans-serif; }
.quick-tag:hover { background: var(--accent); border-color: var(--accent); color: white; }
#ai-messages { flex: 1; overflow-y: auto; padding: 1rem 1.5rem; display: flex; flex-direction: column; gap: 1rem; }
#ai-messages::-webkit-scrollbar { width: 4px; }
#ai-messages::-webkit-scrollbar-track { background: #111; }
#ai-messages::-webkit-scrollbar-thumb { background: #444; }
.msg { display: flex; flex-direction: column; gap: 0.2rem; animation: fadeUp 0.3s ease; }
.msg-role { font-size: 0.65rem; letter-spacing: 0.12em; text-transform: uppercase; }
.msg.user .msg-role { color: var(--muted); text-align: right; }
.msg.ai .msg-role { color: var(--accent); }
.msg-bubble { padding: 0.8rem 1rem; font-size: 0.85rem; line-height: 1.6; max-width: 90%; }
.msg.user .msg-bubble { background: #2a2018; border: 1px solid #3a3028; align-self: flex-end; color: #e8e0d0; }
.msg.ai .msg-bubble { background: #231508; border: 1px solid #3a2510; border-left: 3px solid var(--accent); align-self: flex-start; color: #e8e0d0; }
.ai-book-card { background: #2a1e10; border: 1px solid #3a2e20; padding: 0.8rem; margin-top: 0.5rem; display: flex; gap: 0.8rem; cursor: pointer; transition: border-color 0.2s; }
.ai-book-card:hover { border-color: var(--accent); }
.ai-book-cover { width: 50px; height: 70px; object-fit: cover; flex-shrink: 0; border: 1px solid #444; }
.ai-book-cover-placeholder { width: 50px; height: 70px; background: #1a1008; border: 1px solid #444; display: flex; align-items: center; justify-content: center; font-size: 1.4rem; flex-shrink: 0; }
.ai-book-info { flex: 1; min-width: 0; }
.ai-book-title { font-family: 'Playfair Display', serif; font-size: 0.88rem; font-weight: 700; color: var(--paper); line-height: 1.3; margin-bottom: 0.2rem; }
.ai-book-author { font-size: 0.72rem; color: var(--accent); margin-bottom: 0.3rem; }
.ai-book-reason { font-size: 0.72rem; color: #888; line-height: 1.4; }
.typing-indicator { display: flex; gap: 4px; align-items: center; padding: 0.8rem 1rem; background: #231508; border: 1px solid #3a2510; border-left: 3px solid var(--accent); width: fit-content; }
.typing-dot { width: 6px; height: 6px; background: var(--accent); border-radius: 50%; animation: typingDot 1.2s ease-in-out infinite; }
.typing-dot:nth-child(2) { animation-delay: 0.2s; }
.typing-dot:nth-child(3) { animation-delay: 0.4s; }
@keyframes typingDot { 0%, 60%, 100% { opacity: 0.2; transform: translateY(0); } 30% { opacity: 1; transform: translateY(-4px); } }
.ai-input-area { padding: 1rem 1.5rem; border-top: 1px solid #333; display: flex; gap: 0.5rem; flex-shrink: 0; }
#ai-input { flex: 1; background: #2a2018; border: 1px solid #444; color: var(--paper); padding: 0.7rem 1rem; font-family: 'DM Sans', sans-serif; font-size: 0.88rem; outline: none; resize: none; height: 44px; transition: border-color 0.2s; }
#ai-input:focus { border-color: var(--accent); }
#ai-input::placeholder { color: #555; }
#ai-send { background: var(--accent); color: white; border: none; padding: 0 1rem; font-size: 1rem; cursor: pointer; transition: background 0.2s; flex-shrink: 0; }
#ai-send:hover { background: #a93226; }
#ai-send:disabled { background: #444; cursor: not-allowed; }
.ai-hint { padding: 0.5rem 1.5rem 1rem; font-size: 0.68rem; color: #444; letter-spacing: 0.03em; flex-shrink: 0; }
.ai-key-warning { background: #2a1a08; border: 1px solid #5a3a10; margin: 1rem 1.5rem; padding: 0.8rem 1rem; font-size: 0.78rem; color: #c8a060; line-height: 1.5; }
.ai-key-warning strong { color: #e8c080; }
</style>
