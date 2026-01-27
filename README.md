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
      - 呼叫 `PyModuleDef_Init(...)`
      - (...) 中的參數是 `&module_definition`  
      
      <img width="685" height="227" alt="image" src="https://github.com/user-attachments/assets/de88636f-d82f-4026-8310-e2c3c79d7b67" />

      > 代表這個 C 擴充模組的所有資訊（名字、方法表、等等）  
      > 都被包在一個 `PyModuleDef` 中的 `module_definition` 結構裡

## 方法表 (`PyMethodDef`) 在 `module_definition` (`PyModuleDef`) 的結構本體中
   1. 在右側 Decompile 視窗裡，用滑鼠左鍵雙擊 `module_definition` 後 Ghidra 就會跳到那個 symbol
   2. 現在看向中間的 Listing 視窗, 會跳到那個 data 結構的位置
      - module_definition 就是這個 binary 的模組定義  
      - 它告訴 Python：這個 .so 要被當成哪一個模組、叫什麼名字、有哪些函式可以給 Python 用
     
   3. 現在將滑鼠懸停在 `PyModuleDef` 上, 就能看到他的資料型別的結構
      - 內容大概是這樣：而方法表 (`PyMethodDef`) 也顯然在列
        ```C!
        struct PyModuleDef
         Length:104(0x68) Alignment:8
        {
            PyModuleDef_Base   m_base;     // 模組的基礎資訊（版本、size 等，共用 header）
            char *             m_name;     // 模組名稱字串，例如 "markupsafe._speedups"
            char *             m_doc;      // 模組的說明文字（__doc__），可以是 NULL
            Py_ssize_t         m_size;     // 模組「狀態大小」：-1 代表沒有 per-module state
            PyMethodDef *      m_methods;  // 指向方法表陣列（module_methods），定義所有 C 函式
            PyModuleDef_Slot * m_slots;    // PEP 489 用的 slots，一般簡單模組多半是 NULL
            traverseproc       m_traverse; // 給 GC 用的 traverse 函式（標記物件），常為 NULL
            inquiry            m_clear;    // 清理模組狀態用的 callback，常為 NULL
            freefunc           m_free;     // 卸載模組時釋放資源用的 free 函式，常為 NULL
        } pack()
        ```

      <img width="1150" height="585" alt="image" src="https://github.com/user-attachments/assets/3194d018-0945-47ad-a515-d3b17c464daf" />
   
   ## 跳到 `module_methods` (`PyMethodDef`) 尋找真正要 call 的函式
   1. 在 Listing 視窗中, `PyModuleDef` data 結構的位置往下, 就會看到 `module_methods`  
      雙擊它 即可跳到 `PyMethodDef` data 結構的位置

      <img width="1149" height="423" alt="image" src="https://github.com/user-attachments/assets/7d70650c-a49f-47d2-bef0-71fa8db59fed" />
   
   2. 方法表指標：module_methods ＝ 一個「C 函式列表」, 裡面會列出這個模組要給 Python 用的所有函式  
      函式表中：  
      - 名字叫 "escape" 的 Python 函式 → 真正要 call 的 C 函式是 escape_unicode

      <img width="1289" height="396" alt="image" src="https://github.com/user-attachments/assets/cd04ac9e-561f-4127-bb74-902628c907f7" />

   3. 將鼠標懸停在 `PyMethodDef` 查看他的資料型別的結構
      <img width="1032" height="372" alt="image" src="https://github.com/user-attachments/assets/2e8cba30-fb33-45ae-b164-93492165ee79" />

      - 會發現有可奇怪錯誤的地方 `byte`

        ```c!
        struct PyMethodDef {
            char *ml_name;   // 8 bytes
            byte  field_0x8; // 1 byte（這裡應該是 8 bytes 的指標）
            int   ml_flags;  // 4 bytes
            char *ml_doc;    // 8 bytes
        };                   // 透過 padding 對齊成 24 bytes
        ```

      - 正確的話要顯示 `void  *ml_meth`, 但這只是 Ghidra 分析錯誤 並不影響運行

      ```c!
      struct PyMethodDef {
          char  *ml_name;   // Python 這邊看到的函式名稱，例如 "escape"
          void  *ml_meth;   // 指向真正 C 實作的函式指標（PyCFunction）
          int    ml_flags;  // 呼叫方式／參數型態旗標（METH_O、METH_VARARGS 等）
          char  *ml_doc;    // 函式說明字串（__doc__），可以是 NULL
      };
      ```

   ## 分析 `escape_unicode` 後續用於 exploit 腳本時, 所需的參數  
   1. 確認 0x30c8 這格的值 = `escape_unicode` → 也就是 `ml_meth` 指向的 C 函式  
      <img width="1038" height="368" alt="image" src="https://github.com/user-attachments/assets/6b39e619-4469-4835-8c2e-9c2c63b330cf" />

      - 之後 Python 呼叫這個 method，就會透過 `ml_meth` 跳到 `escape_unicode`
      > 流程：只要我把 0x30c8 那個函式指標改成 shellcode 的位址，
      > Python 呼叫這個函式時，就會執行我的 shellcode

   2. 雙擊 `escape_unicode` 跳到函式的 data 位置  
      <img width="1148" height="577" alt="image" src="https://github.com/user-attachments/assets/18661f08-41f5-4129-959a-468f63f75656" />
      
      - 現在這張圖說明「escape_unicode 的實際 code 起點 = 0x1130」, 真實 code 位址 = addr + 0x1130
      > 只要 call escape_unicode，就一定是從 0x1130 開始跑。
      > 那我把 0x1130 開頭那一段換成 shellcode，我的 shellcode 就一定會被執行

   3. 側邊的 `escape_unicode` 反編譯內容  
      <img width="490" height="546" alt="image" src="https://github.com/user-attachments/assets/6954d4ed-42b9-45f3-a710-c8706a3e3878" />

      - 這整段都將被我覆蓋成 shellcode, 對這次漏洞利用沒有幫助



