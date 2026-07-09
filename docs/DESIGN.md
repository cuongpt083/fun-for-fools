# fun-for-fools — Tài liệu Thiết kế & Tech Decision (Milestone 1)

> Trạng thái: **BRAINSTORM / chờ owner confirm**. Chưa viết code.

## 0. Recap phạm vi M1

- Nền tảng web đa-game học tập (khung tối thiểu: menu chọn môn/game/lesson).
- Game mẫu MVP: **Temple Run Vocabulary** (chạy & thu thập chữ cái đúng thứ tự thành từ hợp lệ).
- Ngữ cảnh đầu: **"At the Beach"** (bãi biển).
- Stack: TypeScript + Vite + Phaser 3.

---

## 1. Tech Stack & Tech Decisions

| Quyết định | Chốt đề xuất | Lý do | Phương án thay thế đã xét |
|---|---|---|---|
| Ngôn ngữ | **TypeScript** | Đa-game → cần type-safe cho game plugin contract & data model | JS thuần (bỏ qua do rủi ro refactor) |
| Bundler/dev | **Vite** | HMR nhanh, sinh static build dễ deploy, tích hợp TS sẵn | Webpack (chậm hơn, config rườm rà) |
| Game engine | **Phaser 3** | 2D, scene system mạnh, cộng đồng lớn, hợp runner 2D | PixiJS (chỉ render, thiếu input/physics), Kaboom (nhỏ nhưng ít tài liệu) |
| Định dạng runner | **2D 3-lane (vuông góc)** | Phaser tốt nhất cho 2D; 3 lane dễ điều khiển, đủ cảm giác Temple Run | Pseudo-3D (phức tạp, vượt MVP) |
| State chia sẻ scene | **Phaser Registry + Event Bus nhẹ** | Tích hợp sẵn, không cần thêm dep | Redux/Zustand (thừa cho MVP) |
| Phát âm từ | **Web Speech API (`speechSynthesis`)** | Không cần file audio cho từng từ; phát được mọi từ động; tái dùng cho game Nghe sau này | File mp3 (tốn công, không scale) |
| Nhận dạng giọng nói (game Nói sau) | **Web Speech API (`SpeechRecognition`)** | Cùng họ API, miễn phí, dùng cho milestone sau | Whisper API (cần backend + key) |
| Asset nghệ thuật | **Vector shapes (Phaser Graphics) cho MVP** + Kenney Toon Characters (CC0) làm nâng cấp | MVP tập trung chữ cái/không cần art; shape không lo bản quyền; Kenney Toon Characters đã verify CC0 để nâng cấp sau | Vẽ riêng (chưa cần ở MVP) |
| Lưu tiến độ | **localStorage** | Đủ cho MVP, không backend | Backend/DB (chưa cần M1) |
| Test | **Vitest** (logic thuần) | Test hàm validate từ/dữ liệu bài học; nhanh, cùng Vite | Jest (thừa dep) |
| Lint/format | **ESLint + Prettier** | Chuẩn cộng đồng TS | — |
| Deploy | **Static (GitHub Pages)** | Vite build ra static, khớp repo GitHub | — |

---

## 2. Kiến trúc Platform (đa-game)

### 2.1 Nguyên tắc: "Game Plugin Contract"

Mỗi game là một **plugin** cắm vào platform qua một interface chung. Platform không cần biết logic game — chỉ khởi tạo, truyền dữ liệu bài học, nhận sự kiện kết quả.

```ts
interface GamePlugin {
  id: string;                       // "temple-run-vocab"
  title: string;                    // hiển thị menu
  skill: Skill;                     // "reading" | "listening" | "writing" | "speaking" | "math" | "science"
  scene: Phaser.SceneClass;         // scene chính của game
  supportedContexts: ContextId[];   // ngữ cảnh tương thích
}
```

Platform cung cấp:
- **GameRegistry**: danh sách game đăng ký.
- **Router/SceneManager**: chọn scene để chạy theo `gameId + contextId`.
- **LessonLoader**: đọc JSON bài học theo `contextId`.
- **SharedUI**: HUD, nút pause, overlay Game Over (dùng cho mọi game).
- **ProgressStore**: lưu điểm/tiến độ (localStorage).

> Lợi thế: game sau (Alchemist STEM, game Toán...) chỉ việc implement `GamePlugin`, không đụng platform.

### 2.2 Cấu trúc Phaser Scene

```
BootScene        -> load asset tối thiểu, init config
PreloadScene     -> load asset + data bài học (theo context đang chọn)
MenuScene        -> chọn Môn > Game > Ngữ cảnh(bài học)
GameScene        -> scene riêng của từng game (VD: TempleRunVocabScene)
UIScene          -> overlay HUD (điểm, mạng, từ đích, nút pause) — chạy song song GameScene
GameOverScene    -> summary + restart/exit
```
### 2.3 Data Model (JSON bài học)

File: `src/data/lessons/beach.en.json`

```json
{
  "contextId": "beach",
  "contextName": "At the Beach",
  "contextNameVi": "Ở bãi biển",
  "skill": "reading",
  "hintLocale": "vi",
  "words": [
    {
      "word": "WATER",
      "meaningVi": "nước",
      "ipa": "/ˈwɔːtər/",
      "example": "The water is cold.",
      "exampleVi": "Nước rất lạnh."
    }
  ]
---

## 3. Thiết kế Game: Temple Run Vocabulary

### 3.1 Cơ chế cốt lõi

- Nhân vật **tự chạy** (endless runner). Camera/đường cuộn về phía nhân vật.
- **3 lane** (trái/giữa/phải). Người chơi đổi lane, nhảy (jump), trượt (slide) để né chướng ngại & hứng xu.
- Mỗi **đồng xu mang 1 chữ cái**. Có cả xu "đúng chữ tiếp theo" và xu "mủ/chữ nhiễu" (distractor).
- **Từ đích** hiện ở HUD kèm nghĩa (Vi) + IPA + nút phát âm, nhưng **chỉ flash ~2–3 giây rồi biến mất** (che chữ). Người chơi phải **nhớ chính tả** để hứng đúng thứ tự.
  - Ví dụ đích: `W A T E R` (flash 2–3s) → ẩn → hứng `W → A → T → E → R` theo thứ tự từ trí nhớ.
  - Nút 🔊 phát âm lại luôn sẵn (giúp gợi nhớ) kể cả khi từ đã ẩn.
- **Hứng đúng chữ tiếp theo**: điền vào ô chữ, +điểm, hiệu ứng, advance progress.
- **Hứng sai chữ** (không phải chữ tiếp theo): **chỉ trừ điểm** (KHÔNG mất mạng), reset combo; progress của từ **giữ nguyên** (default — vẫn cần chữ tiếp theo đúng, nhẹ tay dễ học).
- **Mạng chỉ mất khi đụng chướng ngại** (né không kịp): mất 1 mạng, hiệu ứng stagger.
- **Hoàn thành từ**: bonus điểm lớn, phát âm từ + câu mẫu, nạp từ tiếp theo (lấy từ lesson).
- **Hết mạng** → Game Over.

### 3.2 Điều khiển

| Nền tảng | Phím/cử chỉ |
|---|---|
| Desktop | ← / → đổi lane; ↑ / Space nhảy; ↓ trượt; P pause; R restart (game over) |
| Mobile | Vuốt trái/phải đổi lane; vuốt lên nhảy; vuốt xuống trượt; nút pause on-screen |

> Phaser hỗ trợ cả Keyboard + Pointer (swipe). Mình sẽ viết helper `InputController` gói 2 nguồn vào 1 stream sự kiện thống nhất.

### 3.3 Độ khó (mapping vào MVP)

| Độ khó | Độ dài từ | Tốc độ chạy | Tỷ lệ xu nhiễu | Chướng ngại |
|---|---|---|---|---|
| Easy | 3–4 chữ | chậm | thấp | ít |
| Medium | 5–6 chữ | trung bình | trung | vừa |
| Hard | 7+ chữ | nhanh | cao | nhiều |

- Số mạng: **3**.
- Số từ mỗi session: **~10 từ** hoặc **xong bài học** → hiện "Lesson Complete" → **sang bài mới** (không endless vô tận). Hết mạng trước → Game Over.

### 3.4 Scoring (đề xuất)

- Hứng đúng 1 chữ: +10 × combo multiplier.
- Hoàn thành 1 từ: +100 × hệ số độ khó.
- Hứng sai chữ: **trừ điểm** (vd −5), KHÔNG mất mạng, reset combo; progress giữ nguyên.
- Bonus né chướng ngại liên tục: +nhỏ.

### 3.5 Gợi ý / Học liệu (pedagogy)

- Khi bắt đầu mỗi từ: **flash từ đích ~2–3 giây** (kèm nghĩa Vi + IPA), phát âm 1 lần (Web Speech API, có thể tắt), rồi **ẩn chữ**.
- Nút 🔊 phát âm lại luôn sẵn (kể cả khi từ đã ẩn) — hỗ trợ gợi nhớ chính tả.
- Khi hoàn thành từ: hiện lại nghĩa + câu mẫu vài giây + phát câu mẫu.
- Game Over: liệt kê các từ đã hoàn thành + cho nghe lại từng từ.

### 3.6 Logic validate (thuần, testable)

Tách hẳn ra module thuần để unit test bằng Vitest:

```ts
// src/games/temple-run-vocab/logic/wordProgress.ts
type CollectResult =
  | { kind: "correct"; progress: string; complete: boolean }
  | { kind: "wrong"; reason: "out-of-order" | "not-in-word" };

function collectLetter(target: string, progress: string, letter: string): CollectResult
```

- `correct` khi `letter === target[progress.length]`.
- `complete` khi `progress.length === target.length` sau khi thêm.
- `wrong` khi `letter` thuộc từ nhưng sai vị trí (`out-of-order`) hoặc không thuộc từ (`not-in-word`) — phân biệt để có feedback khác nhau (chốt đề xuất).

---

## 4. Cấu trúc thư mục dự án (đề xuất)

```
fun-for-fools/
├── docs/
│   ├── PRODUCT_VISION.md
│   └── DESIGN.md                 # file này
├── public/
│   └── assets/                   { img, audio(nếu có), sprite }
├── src/
│   ├── platform/
│   │   ├── GameRegistry.ts
│   │   ├── LessonLoader.ts
│   │   ├── ProgressStore.ts
│   │   ├── scenes/               { BootScene, PreloadScene, MenuScene, UIScene, GameOverScene }
│   │   └── types.ts              # GamePlugin, Skill, ContextId...
│   ├── games/
│   │   └── temple-run-vocab/
│   │       ├── TempleRunVocabScene.ts
│   │       ├── TempleRunVocabPlugin.ts   # implement GamePlugin
│   │       ├── logic/
│   │       │   └── wordProgress.ts       # thuần, unit test
│   │       ├── systems/                   { RunnerSystem, CoinSpawner, ObstacleSpawner, InputController }
│   │       └── config.ts
│   ├── data/
│   │   ├── lessons/
│   │   │   └── beach.en.json
│   │   └── contexts.ts
│   ├── shared/
│   │   ├── ui/                            { HUD, Button }
│   │   └── audio/                         { Speech.ts }  # wrap Web Speech API
│   ├── main.ts                            # bootstrap Phaser game
│   └── config.ts
├── tests/
│   └── temple-run-vocab/wordProgress.test.ts
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── .eslintrc.cjs
└── .prettierrc
```
---

## 5. Testing strategy

- **Unit (Vitest)**: `wordProgress.ts` (validate thứ tự chữ), `LessonLoader` (parse JSON), `ProgressStore` (read/write localStorage).
- **Manual/visual**: gameplay runner (khó auto-test ở MVP).
- **Sau này**: Playwright e2e cho menu → vào game → game over.

---


## 6. Quyết định ĐÃ CHỐT (owner confirmed)

| # | Quyết định | Trạng thái |
|---|---|---|
| 1 | Runner **2D 3-lane** | ✅ Đã chốt |
| 2 | Từ đích **flash ~2–3 giây** (kèm nghĩa Vi + IPA + 🔊) rồi **ẩn**; người chơi nhớ chính tả để hứng đúng thứ tự; nút 🔊 luôn sẵn để gợi nhớ | ✅ Đã chốt |
| 3 | Hứng sai chữ → **chỉ trừ điểm** (KHÔNG mất mạng), reset combo, progress giữ nguyên (default) | ✅ Đã chốt |
| 4 | Số mạng **= 3** (chỉ mất khi đụng chướng ngại) | ✅ Đã chốt |
| 5 | Session **~10 từ / xong bài học** → "Lesson Complete" → sang bài mới; hết mạng trước → Game Over | ✅ Đã chốt |
| 6 | Phát âm **Web Speech API** (`speechSynthesis`) | ✅ Đã chốt |
| 7 | Asset: **vector shapes cho MVP** (Kenney Toon Characters CC0 nâng cấp sau) | ✅ Đã chốt (agent chọn tạm) |
| 8 | M1: **chỉ khung platform + Temple Run**; Toán/STEM = placeholder trong menu, làm milestone sau | ✅ Đã chốt |

**Điểm phụ (default của agent, có thể sửa):**
- Hứng sai chữ: progress của từ **giữ nguyên** (không reset) — nhẹ tay, dễ học. Nếu muốn reset cả từ, nói mình.
- Trừ điểm sai chữ: tạm **−5**/lần.

---

## 7. Rủi ro đã nhận diện (SP-flaw check)

- Không nhầm **Aim** (mục tiêu/DoD) với **Time** (deadline): các con số mạng/số từ thuộc **Aim**, KHÔNG phải Time.
- "Viết unit test cho wordProgress" là **A-atom** (tiêu chí hoàn thành), không phải T-atom.
- Rủi ro kỹ thuật: Web Speech API chất lượng giọng không đồng đều giữa browser → cần fallback (hiện IPA + nghĩa rõ ràng). Đã ghi.
- Rủi ro scope: đa-game platform dễ bị feature creep → giữ M1 tối thiểu, game Toán/STEM chỉ là placeholder trong menu.


---

## 5. Testing strategy

- **Unit (Vitest)**: `wordProgress.ts` (validate thứ tự chữ), `LessonLoader` (parse JSON), `ProgressStore` (read/write localStorage).
- **Manual/visual**: gameplay runner (khó auto-test ở MVP).
- **Sau này**: Playwright e2e cho menu → vào game → game over.


- `correct` khi `letter === target[progress.length]`.
- `complete` khi `progress.length === target.length` sau khi thêm.
- `wrong` khi `letter` thuộc từ nhưng sai vị trí (`out-of-order`) hoặc không thuộc từ (`not-in-word`) — phân biệt để có feedback khác nhau (chốt đề xuất).

- Hứng đúng 1 chữ: +10 × combo multiplier.
- Hoàn thành 1 từ: +100 × độ khó.
- Hứng sai chữ: mất mạng, reset combo.
- Bonus né chướng ngại liên tục: +small.

- **Đụng chướng ngại** (né không kịp): mất 1 mạng, có hiệu ứng stagger.
- **Hoàn thành từ**: bonus điểm lớn, phát âm từ + câu mẫu, nạp từ tiếp theo (lấy từ lesson).
- **Hết mạng** → Game Over.

}
```

- `word`: chữ HOA, dùng để validate thứ tự chữ cái.
- `meaningVi`: gợi ý nghĩa (tiếng Việt) cho người mới học.
- `ipa`: phiên âm hiển thị.
- `example` / `exampleVi`: câu mẫu theo ngữ cảnh → mở rộng cho game Đọc/Viết sau này.
- Không có `audio`/`image` bắt buộc ở MVP → phát âm dùng Web Speech API.

MenuScene        -> chọn Môn > Game > Ngữ cảnh(bài học)
GameScene        -> scene riêng của từng game (VD: TempleRunVocabScene)
UIScene          -> overlay HUD (điểm, mạng, từ đích, nút pause) — chạy song song GameScene
GameOverScene    -> summary + restart/exit
```

