

### **第一部分：平台總體願景與系統架構**

#### 1. 核心願景 (Core Vision)
「文樞」將不再只是一個翻譯器或遊戲，而是一個**「個人化的文言文智慧學伴」**。它會記錄您的學習軌跡，提供無限的內容挑戰，並透過成就和排行榜激勵您持續進步。

#### 2. 系統架構 (System Architecture)
我們將採用**前後端分離**的現代網頁應用架構。

*   **前端 (Frontend)**：您現有的 HTML/CSS/JavaScript。它將負責：
    *   所有使用者介面 (UI) 的渲染與互動。
    *   處理使用者輸入。
    *   呼叫 Firebase 服務。
    *   管理 Local Storage 中的非核心資料。

*   **後端 (Backend)**：**Firebase** 將是我們強大的後端，負責：
    *   **Authentication**：使用者註冊、登入與身份驗證。
    *   **Firestore Database**：作為核心的雲端資料庫，儲存所有使用者的關鍵資料。
    *   **(可選) Cloud Functions**：用於處理更複雜的後端邏輯，例如定期更新排行榜或觸發成就。

*   **混合儲存策略 (Hybrid Storage Strategy)**：
    *   **Firestore**：儲存所有**必須跨裝置同步**、**需要驗證**或**用於社交功能**的關鍵資料。例如：使用者帳號、閱讀歷史、成就、遊戲最高分、社群最新貼文。
    *   **Local Storage**：儲存**非關鍵性**或**使用者個人化**的快取資料，以提升效能和離線體驗。例如：「古人社群」的大量歷史貼文。

---

### **第二部分：Firestore 雲端資料庫結構設計**

這是整個計畫的基石，一個好的結構能讓開發事半功倍。

```json
// Firestore Database Structure

{
  "users": {
    "userID_1": {
      "email": "user1@example.com",
      "displayName": "孟德",
      "createdAt": "2025-08-06T10:00:00Z",
      "profileImage": "url_to_image",
      "achievements": { // 我的成就
        "read_10_articles": true,
        "first_game_win": true,
        "mastered_50_words": false
      }
    }
  },

  "articles": { // 儲存所有生成過的文章，確保不重複
    "articleID_1": { // ID 可以是內容的 hash 值，確保唯一性
      "title": "出師表",
      "author": "諸葛亮",
      "content": "臣本布衣，躬耕於南陽...",
      "introduction": {
        "authorIntro": "諸葛亮，字孔明...",
        "articleIntro": "《出師表》是三國時期..."
      },
      "translationData": { /* 完整的 JSON 翻譯資料 */ },
      "tags": ["三國", "奏疏", "忠誠"] // 便於未來分類搜尋
    }
  },

  "userProgress": { // 紀錄使用者的所有學習活動
    "progressID_1": {
      "userID": "userID_1",
      "articleID": "articleID_1",
      "type": "read", // 'read', 'test', 'game'
      "completedAt": "2025-08-06T11:00:00Z",
      "details": {
        "testScore": 90, // 如果是測驗
        "gameScore": 15000 // 如果是遊戲
      }
    }
  },

  "communityPosts": { // 古人社群 (只存最新的10筆)
    "postID_1": {
      "userID": "userID_1",
      "content": "今日讀《出師表》，深感孔明之忠。若為君主，當如何...",
      "timestamp": "2025-08-06T12:00:00Z",
      "articleID": "articleID_1" // 關聯的文章
    }
  },

  "leaderboards": { // 排行榜
    "articleID_1": { // 每個文章都有自己的排行榜
      "scores": [
        { "userID": "userID_1", "displayName": "孟德", "score": 15000 },
        { "userID": "userID_2", "displayName": "仲謀", "score": 12000 }
      ]
    }
  }
}
```

---

### **第三部分：功能模組詳細實施計畫**

我們將現有和新增的功能模組化，並規劃其實現方式。

#### **模組一：使用者認證與個人中心 (保留並強化)**
1.  **功能**：註冊、登入、登出、個人資料管理。
2.  **實現**：
    *   建立一個包含「登入/註冊」按鈕的導覽列。
    *   使用 Firebase Authentication SDK (`createUserWithEmailAndPassword`, `signInWithEmailAndPassword`, `onAuthStateChanged`) 處理所有認證流程。
    *   登入後，導覽列顯示使用者名稱和頭像，並出現「我的成就」、「古人社群」、「登出」等選項。
    *   所有後續的操作（如儲存歷史、發文）都必須檢查使用者是否已登入 (`auth.currentUser`)。

#### **模組二：核心學習引擎 (保留並強化)**
1.  **功能**：AI 尋找篇章、手動輸入、翻譯、逐句語譯、逐字解釋。
2.  **目標**：確保內容不重複，並將所有內容存檔。
3.  **實現**：
    *   **AI 尋找篇章 (`handleAiSearch`)**：
        1.  使用者點擊「尋找篇章」。
        2.  **[優化]** 後端邏輯：從 `userProgress` 查詢該使用者已讀過的 `articleID` 列表。
        3.  **[優化]** AI Prompt 修改：在給 AI 的提示中，加入一句：「**請勿推薦以下文章標題或內容：[已讀文章列表]**」。這能從源頭避免重複。
        4.  AI 回傳文章後，計算文章內容的 hash 值作為 `articleID`。
        5.  **[優化]** 檢查 `articles` 集合中是否已存在此 `articleID`。
        6.  若**存在**，直接從 Firestore 讀取文章資料（包含翻譯）。
        7.  若**不存在**，則呼叫 AI 進行翻譯、生成簡介，然後將完整的文章資料（原文、簡介、翻譯JSON）存入 `articles` 集合。
    *   **手動輸入**：流程類似，同樣在處理後將新文章存入 `articles` 集合。

#### **模組三：小測驗模式 (保留並強化)**
1.  **功能**：對文章的逐字解釋進行測驗。
2.  **目標**：確保題目不重複，並記錄測驗結果。
3.  **實現**：
    *   **題目生成**：
        *   測驗的題目（錯誤選項）是從文章中其他解釋隨機抽取的，這本身就具有**高度的隨機性**，很難重複。
        *   為確保每次測驗的體驗都新鮮，可以在生成測驗時，將錯誤選項的組合（或其 hash）與 `articleID` 一同儲存，下次為同一使用者生成同一文章的測驗時，確保生成不同的組合。
    *   **記錄結果**：
        *   測驗結束後，將結果（分數、是否成功）連同 `userID` 和 `articleID` 一同存入 `userProgress` 集合，`type` 設為 `test`。
        *   測驗成功可以觸發一個「成就」的解鎖。

#### **模組四：古人社群 (新增功能)**
1.  **功能**：使用者針對閱讀的文章發表個人心得，僅自己可見。
2.  **目標**：實現混合儲存策略（Firebase + Local Storage）。
3.  **實現**：
    *   **UI 設計**：一個獨立的頁面或彈出視窗，看起來像一個簡潔的個人日誌或社交媒體 Feed。包含一個文字輸入框和「發表」按鈕。
    *   **發表貼文 (`postReflection`)**：
        1.  使用者輸入心得並點擊「發表」。
        2.  獲取當前使用者 `userID` 和正在閱讀的 `articleID`。
        3.  **Local Storage**：從 Local Storage 讀取該使用者的所有貼文 `localPosts`（一個 JSON 陣列）。
        4.  將新貼文（包含內容、時間戳、articleID）加入到 `localPosts` 陣列的最前方。
        5.  將更新後的 `localPosts` 陣列完整存回 Local Storage。
        6.  **Firebase 同步**：從 `localPosts` 中選取**最新的 10 筆**貼文。
        7.  遍歷這 10 筆貼文，將它們上傳/更新到 Firestore 的 `communityPosts` 集合。可以使用 `set` 搭配 `merge: true` 來更新，或先刪除該使用者所有舊貼文再全部寫入。
    *   **瀏覽貼文 (`loadReflections`)**：
        1.  頁面載入時，優先從 Local Storage 讀取 `localPosts` 並渲染到畫面上，提供即時體驗。
        2.  同時，非同步地從 Firebase 的 `communityPosts` 查詢該使用者的貼文。
        3.  如果 Firebase 的資料比 Local Storage 新（例如在另一台裝置發過文），則用 Firebase 的資料更新 Local Storage 和畫面。

#### **模組五：我的成就 & 排行榜 (新增功能)**
1.  **功能**：展示個人成就和遊戲排行榜。
2.  **實現**：
    *   **我的成就 (My Achievements)**：
        1.  建立一個「我的成就」頁面。
        2.  定義一系列成就條件，例如：
            *   新手上路：完成第一次翻譯。
            *   學有所成：成功完成 10 次測驗。
            *   學富五車：閱讀 50 篇不同的文章。
            *   遊戲高手：在任意文章的遊戲中得分超過 20000 分。
            *   筆耕不輟：發表 20 篇社群心得。
        3.  在各個功能的完成點（例如翻譯成功後、測驗成功後）加入檢查函式 `checkAchievements(userID)`。
        4.  `checkAchievements` 函式會根據 `userProgress` 和 `communityPosts` 的資料來判斷是否達成條件。
        5.  若達成，則更新 `users/{userID}/achievements` 物件中對應的欄位為 `true`。
        6.  「我的成就」頁面讀取這個物件，並將 `true` 的成就以點亮圖示的方式顯示。
    *   **排行榜 (Leaderboard)**：
        1.  **遊戲結束後**：
            *   獲取 `userID`、`displayName`、`articleID` 和最終分數 `score`。
            *   將此紀錄存入 `userProgress` 集合。
            *   **更新排行榜**：讀取 `leaderboards/{articleID}` 文件。將新的分數加入 `scores` 陣列，並按分數從高到低排序。為節省空間，可以只保留前 20 名的紀錄。最後將更新後的陣列寫回 Firestore。
        2.  **顯示排行榜**：在遊戲介面或文章頁面，可以加入一個「查看排行」按鈕，點擊後從 `leaderboards/{articleID}` 讀取資料並顯示。

#### **模組六：可持續遊玩性 (確保不會「爆機」)**
這是貫穿所有設計的理念，透過以下方式實現：

1.  **無限的內容來源**：只要 AI 模型存在，就可以源源不斷地根據使用者輸入的主題或隨機推薦生成**幾乎無限**的古文篇章。
2.  **動態生成的挑戰**：測驗和遊戲的題目都是基於文章內容**動態生成**的，即使是同一篇文章，每次的體驗也可以不同。
3.  **社交驅動**：排行榜的存在，讓使用者有不斷挑戰高分、超越他人的動力。即使所有文章都玩過，也可以為了刷新紀錄而重玩。
4.  **個人化進程**：「我的成就」系統和「古人社群」提供了持續的個人目標和記錄空間，學習的重點從「通關」轉移到了「個人成長」。
5.  **定期更新**：您可以定期加入新的「官方成就」、新的「遊戲模式」或舉辦「特定主題閱讀週」活動，來保持平台的新鮮感。

---

### **第四部分：開發路線圖 (Development Roadmap)**

建議分階段進行，確保每一步都穩固。

*   **階段一：基礎建設 (Backend & Auth)**
    1.  建立 Firebase 專案，完成設定。
    2.  設計並建立 Firestore 資料庫結構。
    3.  完成使用者註冊、登入、登出功能。
    4.  建立基本的導覽列，能根據登入狀態顯示不同內容。

*   **階段二：核心功能雲端化 (Core Migration)**
    1.  重構「尋找篇章」功能，接入 Firestore `articles` 集合，實現內容不重複和雲端存檔。
    2.  將「閱讀歷史」從 Local Storage 遷移到 Firestore 的 `userProgress` 集合。
    3.  將「測驗結果」記錄到 `userProgress` 集合。

*   **階段三：新功能開發 (New Features)**
    1.  開發「古人社群」功能，實現混合儲存。
    2.  開發「我的成就」頁面和成就觸發邏輯。
    3.  開發「排行榜」功能，並與遊戲模式對接。

*   **階段四：打磨與優化 (Polish & Optimize)**
    1.  全面測試所有功能，特別是資料同步的邊界情況。
    2.  優化 UI/UX，讓使用者體驗更流暢。
    3.  考慮增加如「忘記密碼」、「修改個人資料」等輔助功能。

