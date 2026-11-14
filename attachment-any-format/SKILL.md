---
name: attachment-any-format
description: "當使用者把任意格式附件（影片、音訊、PDF、Office、壓縮檔、未知二進位）加入對話框或文本框，並要求閱讀、摘要、轉錄或分析時使用；負責把附件轉成模型實際能消費的內容。"
---

# 任意格式附件處理

DSH 的上傳管線不設格式白名單：任何檔案都能加入對話框與文本框。差別只在「模型怎麼看到它」——只有 PNG／JPEG／WebP／GIF 會被正規化成多模態影像送進模型，其餘格式以唯讀路徑交給你，必須由你主動開啟。

**本技能只做路由，不重造既有工具。** 下列既有技能已在本機可用，優先呼叫它們。

## Workflow

1. 確認附件的落地位置與類型
   - Desktop 上使用者附加的檔案會以 `@路徑` 參照晶片出現在草稿；用 `read` 或 `Get-ChildItem` 確認實際路徑。
   - 先量大小：`Get-Item <path> | Select-Object Name,Length`。
2. 依副檔名路由到既有技能（不要自己寫解析器）
   | 類型 | 副檔名 | 路由 |
   |---|---|---|
   | 影像 | `.png` `.jpg` `.jpeg` `.webp` `.gif` | 直接 `read_image`，不需要其他處理 |
   | 影片 | `.mp4` `.mov` `.mkv` `.avi` `.webm` `.m4v` | `video-frames` 技能抽幀後逐幀 `read_image`；剪輯／轉碼需求用 `remotion-video-director` |
   | 音訊 | `.mp3` `.wav` `.m4a` `.flac` `.ogg` `.aac` | `transcribe` 技能轉錄；配音需求用 `speech` 技能 |
   | 文件 | `.pdf` `.docx` `.xlsx` `.pptx` `.md` `.txt` `.csv` `.html` `.rtf` | `markitdown-skill` 抽文字；版面／渲染需求用 `pdf` 或 `office` 技能 |
   | 壓縮檔 | `.zip` `.7z` `.tar` `.gz` | 先解壓到工作區暫存目錄，再對每個成員重複步驟 2 |
   | 未知 | 其他 | 先看檔頭再歸類：`Get-Content -AsByteStream -TotalCount 16 <path>`，比對魔術數字（PDF `%PDF`、ZIP `PK`、PNG、GIF、RIFF／WebP、ID3／M4A 等） |
3. 影片抽幀（既有 `video-frames` 技能）
   - 首幀：`bash {baseDir}/../video-frames/scripts/frame.sh <path> --out <暫存>/frame.png`
   - 指定時間：`... --time 00:00:10 --out <暫存>/frame-10s.png`
   - 連續多幀：直接用 `ffmpeg -i <path> -vf fps=0.5 <暫存>/frame_%02d.png`，再逐幀 `read_image`。
4. 回報時說明：開啟了哪個檔案、呼叫了哪個既有技能、得到什麼結論；原始附件一律維持唯讀。

## Guardrails

- 模型不會自動看到非影像附件的內容；不要以「看不到影片」結束，先抽幀或轉錄。
- 不要自己重寫已有技能的功能（markitdown、transcribe、pdf、office、video-frames 都已存在）；缺能力時先查 GitHub 既有實作再決定是否新增。
- 不要把整段影片、音訊或二進位直接塞進 prompt 或命令列參數。
- 抽幀要節制：每 2–5 秒一幀、總量不超過約 24 幀；需要全片覆蓋時分段處理。
- 解壓與抽幀產物只寫到工作區暫存目錄（例如 `.tmp-attachments/`），完成後清理；不得改動使用者的原始檔案。
- 上傳本身沒有格式白名單；上傳失敗時讀錯誤訊息（磁碟、權限、大小），不要去改 picker 的 accept。
- 第三方服務（NVIDIA 等）只吃文字與影像；敏感附件先在本機處理，不要外傳原檔。

## Resources

- `video-frames`（本機既有，MIT，源自 [ArgentAIOS/agent-skills](https://github.com/ArgentAIOS/agent-skills)）— ffmpeg 抽幀。
- `markitdown-skill`、`transcribe`、`pdf`、`office`、`remotion-video-director` — 本機既有，各司其職。
- ffmpeg 位於 PATH（`C:\Users\cao41\AppData\Local\Microsoft\WinGet\Links\ffmpeg.exe`）。
