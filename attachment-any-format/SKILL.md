---
name: attachment-any-format
description: "當使用者把任意格式附件（影片、音訊、PDF、Office、壓縮檔、未知二進位）加入對話框或文本框，並要求閱讀、摘要、轉錄或分析時使用；負責把附件轉成模型實際能消費的內容。"
---

# 任意格式附件處理

DSH 的上傳管線不設格式白名單：任何檔案都能加入對話框與文本框。真正的差別只在「模型怎麼看到它」——只有 PNG／JPEG／WebP／GIF 會被正規化成多模態影像送進模型，其餘格式以唯讀路徑交給你，必須由你主動開啟。

## Workflow

1. 確認附件的落地位置與類型
   - Desktop 上使用者附加的檔案會以 `@路徑` 參照晶片出現在草稿；用 `read` 或 `Get-ChildItem` 確認實際路徑。
   - 先量大小：`Get-Item <path> | Select-Object Name,Length`。
2. 影像類（`.png` `.jpg` `.jpeg` `.webp` `.gif`）→ 直接 `read_image`，不需要其他處理。
3. 影片（`.mp4` `.mov` `.mkv` `.avi` `.webm` `.m4v`）→ 用 ffmpeg 抽幀，再把每幀當影像讀取
   - `powershell -File scripts/video-frames.ps1 -Input <path> -OutDir <工作區暫存>\frames -Fps 0.5`
   - 預設每 2 秒一幀；逐幀 `read_image` 後再分析。
4. 音訊（`.mp3` `.wav` `.m4a` `.flac` `.ogg` `.aac`）→ 用 transcribe 或 speech 技能轉錄，再依文字處理。
5. 文件（`.pdf` `.docx` `.xlsx` `.pptx` `.md` `.txt` `.csv` `.html` `.rtf`）→ 用 markitdown-skill 或 pdf、office 技能抽文字。
6. 壓縮檔（`.zip` `.7z` `.tar` `.gz`）→ 先解壓到工作區暫存目錄，再對每個成員重複步驟 3–5。
7. 未知副檔名 → 先看檔頭與長度再歸類：`Get-Content -AsByteStream -TotalCount 16 <path>`，比對魔術數字（PDF `%PDF`、ZIP `PK`、PNG、GIF、RIFF／WebP、ID3／M4A 等）。
8. 回報時說明：開啟了哪個檔案、用了什麼工具、得到什麼結論；原始附件一律維持唯讀。

## Guardrails

- 模型不會自動看到非影像附件的內容；不要以「看不到影片」結束，先抽幀或轉錄。
- 不要把整段影片、音訊或二進位直接塞進 prompt 或命令列參數。
- 抽幀要節制：每 2–5 秒一幀、總量不超過約 24 幀；需要全片覆蓋時分段處理。
- 解壓與抽幀產物只寫到工作區暫存目錄（例如 `.tmp-attachments/`），完成後清理；不得改動使用者的原始檔案。
- 上傳本身沒有格式白名單；上傳失敗時讀錯誤訊息（磁碟、權限、大小），不要去改 picker 的 accept。
- 第三方服務（NVIDIA 等）只吃文字與影像；敏感附件先在本機處理，不要外傳原檔。

## Resources

- `scripts/video-frames.ps1` — 以 ffmpeg 抽幀，輸出 PNG 序列與摘要。
- ffmpeg 位於 PATH（`C:\Users\cao41\AppData\Local\Microsoft\WinGet\Links\ffmpeg.exe`）。
