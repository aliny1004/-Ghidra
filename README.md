# 用 Ghidra 分析 binary (此為 uoftctf2026_fileupload 的 binary)

## 前置作業（下載本體＆安裝環境）
1. 安裝 JDK
   ```bash
   sudo apt install -y default-jdk
   ```

2. 驗證 JDK 有裝好
   ```bash
   javac -version
   java -version
   ```

3. 下載 Ghidra
   * 到官方網站下載 Ghidra 壓縮檔
   https://github.com/NationalSecurityAgency/ghidra/releases

4. 解壓縮 Ghidra
   ```bash
   unzip ghidra_**version_number**.zip
   ```

## 使用 Ghidra
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
C 擴充模組的入口通常是 `PyInit__模組名`，這是整個模組初始化流程的起點

所有這個 module 的「方法表 (`PyMethodDef`)、型別物件、初始化邏輯」
幾乎一定會在這裡被註冊給 Python

你想知道哪個函式會被 Python call，一定要來這裡找
   1. 左側 Symbol Tree 視窗
   2. 展開 Functions
   3. 找到並點 `PyInit__speedups`
   
      <img width="273" height="349" alt="image" src="https://github.com/user-attachments/assets/0df3f366-133c-40ef-8a3b-774e4b0494e9" />

## 在 `PyInit__speedups` 裡找到方法表 (`PyMethodDef`)
   1. 右側 Decompile 反編譯視窗
   2. 看到兩個關鍵資訊：
      - 呼叫 PyModuleDef_Init(...)
      - (...) 中的參數是 &module_definition  
      
      <img width="685" height="227" alt="image" src="https://github.com/user-attachments/assets/de88636f-d82f-4026-8310-e2c3c79d7b67" />

      > 代表這個 C 擴充模組的所有資訊（名字、方法表、等等）  
      > 都被包在一個 PyModuleDef 中的 module_definition 結構裡
      
