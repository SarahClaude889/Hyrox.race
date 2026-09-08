# Firebase setup — learning platform backend

呢份係 step 1–3 嘅實際操作指令。**要喺你自己部機行**，唔可以喺 Claude Code 呢個雲端 session 行 —
原因寫喺下面「點解要你自己行」。Step 4（security rules）已經有個安全基線喺 `firestore.rules`。

---

## 點解要你自己行

呢個 session 跑喺一個受機構網絡政策管住嘅雲端 container。出面嘅 HTTPS 全部經一個 egress
gateway，而 gateway 對 `auth.firebase.tools:443` 回 **403（policy denial）**：

```
{ "kind": "connect_rejected",
  "detail": "gateway answered 403 to CONNECT (policy denial or upstream failure)",
  "host": "auth.firebase.tools:443" }
```

`auth.firebase.tools` 就係 Firebase CLI 登入流程一定要用嘅域名（attest + OAuth redirect），
所以 `firebase login` 喺呢個 container 一定失敗。`dl.google.com` 一樣被擋，連 Firestore
emulator 個 jar 都下載唔到。其他 Google API host（`firebase.googleapis.com`、
`identitytoolkit.googleapis.com`、`firestore.googleapis.com` 等）就通。

呢個係機構政策，唔應該繞過，所以以下步驟由你喺本機執行。

> ⚠️ 順帶一提：唔好將 `firebase login:ci` 攞到嘅 token 貼入任何雲端 session。嗰個係長期有效、
> 有你成個 Firebase/GCP 帳戶權限嘅憑證，而呢類 container 係即用即棄嘅。

---

## Step 1 — 安裝 CLI 同登入

```bash
npm install -g firebase-tools
firebase --version          # 應該係 15.x 或以上

firebase login              # 開瀏覽器，揀 sgckh889@gmail.com 授權
firebase login:list         # 確認登入咗邊個帳戶
```

如果你部機冇瀏覽器（例如 SSH 入去嘅 server），改用：

```bash
firebase login --no-localhost
```

---

## Step 2 — 開新 project（No organisation，你係 Owner）

```bash
firebase projects:create hyrox-learning-platform \
  --display-name "Hyrox Learning Platform"
```

- **Project ID 規則**：6–30 個字元、細楷英文字母／數字／連字號、要以字母開頭、**全球唯一**。
  如果 `hyrox-learning-platform` 已經俾人用咗，加個 suffix，例如 `hyrox-learning-platform-889`。
- **No organisation**：唔加 `-o/--organization` 同 `-f/--folder` 就會開喺冇 parent 之下，
  即係「No organisation」。
- **Owner**：用 CLI 開 project 嘅帳戶自動成為 Owner，唔使額外設定。
- 第一次開可能要你喺瀏覽器接受 Google Cloud 條款，跟住指示做就得。

確認：

```bash
firebase projects:list
```

之後所有指令入面嘅 `<PROJECT_ID>` 換成你實際攞到嗰個。

---

## Step 3a — 開 Firestore（Singapore, asia-southeast1）

```bash
firebase firestore:locations --project <PROJECT_ID>     # 確認 asia-southeast1 喺清單入面

firebase firestore:databases:create "(default)" \
  --project <PROJECT_ID> \
  --location asia-southeast1
```

- `"(default)"` 係預設 database 嘅名，個括號要保留，所以要用引號括住。
- **location 揀咗就改唔到**，要改就只可以刪咗成個 database 重開。`asia-southeast1` = Singapore，
  正確。

確認：

```bash
firebase firestore:databases:list --project <PROJECT_ID>
```

---

## Step 3b — 開 Email/Password Auth

Firebase CLI 冇指令做呢步，行 Console 最快：

1. 開 https://console.firebase.google.com/project/<PROJECT_ID>/authentication
2. 撳 **Get started**
3. **Sign-in method** → **Email/Password** → 開 **Enable**（第二個掣 "Email link" 唔使開）
4. **Save**

如果你想用指令（要裝咗 `gcloud` 而且登入同一個帳戶）：

```bash
gcloud services enable identitytoolkit.googleapis.com --project <PROJECT_ID>

curl -X PATCH \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/<PROJECT_ID>/config?updateMask=signIn.email.enabled,signIn.email.passwordRequired" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"signIn":{"email":{"enabled":true,"passwordRequired":true}}}'
```

---

## Step 4 — 接上呢個 repo，部署 rules

呢個 repo 已經有 `firebase.json`、`firestore.rules`、`firestore.indexes.json`。

```bash
cd Hyrox.race
firebase use --add            # 揀你頭先開嘅 project，會寫入 .firebaserc
```

部署之前**一定要**先喺本機驗證 rules（我喺 container 度驗證唔到，emulator 下載唔到）：

```bash
firebase emulators:start --only firestore
```

行得起、冇 syntax error 就 Ctrl-C，然後：

```bash
firebase deploy --only firestore:rules
```

---

## 而家 rules 係點

`firestore.rules` 而家係 **deny by default**：除咗 `users/{userId}`（每個 user 淨係讀寫得
自己嗰份 profile，而且 email／createdAt 改唔到），其他全部拒絕。呢個係未知你 collections
之前唯一安全嘅狀態 — 部署咗都唔會漏嘢出街，只係新 collection 未通。

**下一步**：話俾我知你 app 實際用緊邊啲 collections、每個 collection 嘅欄位、同埋邊個角色
（學生／導師／admin）應該讀到寫到乜，我就填返上去。講得越具體越好，例如：

- collection 名同 document 結構（邊啲欄位係 server 控制、邊啲 user 改得）
- 有冇 subcollection（例如 `courses/{courseId}/lessons/{lessonId}`）
- 有冇「公開」內容（例如課程目錄唔使登入都睇到）
- 進度／成績類資料邊個睇得（淨係自己？導師？）
