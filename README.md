# 用 Ghidra 分析 binary (此為 uoftctf2026_fileupload 的 binary)

## 前置作業
1. 啟動 Ghidra `./ghidraRun`
2. 建立專案
    * File → New Project
    * 選 Non-Shared Project
    * 自訂名稱（例如：speedups_ctf）
      
3. 匯入 .so
    * 把 `_speedups.cpython-312-x86_64-linux-gnu.so` 拖進 Ghidra
    * 或 File → Import File... 選這顆 .so → OK
      
4. 用 CodeBrowser 打開
    * 左邊檔案 → 右鍵 → Open With → CodeBrowser
    * 跳出「Analyze?」→ 選 Yes → 用預設設定 → OK

## 在 Functions 找到 `PyInit__...`
1. 左側 Symbol Tree 視窗
2. 展開 Functions
3. 找到並點 `PyInit__speedups`

   <img width="273" height="349" alt="image" src="https://github.com/user-attachments/assets/0df3f366-133c-40ef-8a3b-774e4b0494e9" />
