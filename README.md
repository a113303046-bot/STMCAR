# NUCLEO-F446RE 四輪 encoder 手轉測試

這是加入既有 STM32CubeIDE/HAL 專案的整合包，不是可獨立匯入的完整 IDE 專案。
未取得實際 .ioc 與專案，不能替代 CubeMX 初始化、HAL 驅動、啟動檔、連結腳本或 IDE 建置設定。

## 檔案放置

|本包檔案|目的地／用途|
|---|---|
|Core/Inc/encoder_test.h|複製到專案 Core/Inc|
|Core/Src/encoder_test.c|複製到專案 Core/Src|
|main_integration.txt|照三個位置修改原本 main.c，不參與編譯|
|README.md|操作說明，不參與編譯|

若先前已貼過測試碼，先移除舊的 encoder 變數、函式、啟動及更新內容，再使用此模組。
不要再另外建立 htim2/3/4/5 的實體；本模組以 extern 使用 CubeMX 定義的 handles。

## CubeMX 設定與接線

依據使用者提供的《範例接線設定 (1).pdf》。

|輪子|Timer|藍線 A/CH1|綠線 B/CH2|Live Expressions|
|---|---|---|---|---|
|左前 FL|TIM2|PA15 / CN7-17|PB3 / CN10-31|encoder_FL|
|右前 FR|TIM5|PA0 / CN7-28|PA1 / CN7-30|encoder_FR|
|左後 RL|TIM4|PB6 / CN10-17|PB7 / CN7-21|encoder_RL|
|右後 RR|TIM3|PA6 / CN10-13|PA7 / CN10-11|encoder_RR|

黃線接 3.3V（CN7-16），白線接 GND（CN7-19）；CN7-20 是 NC，不是 GND。
SYS Debug 設 Serial Wire，不啟用占用 PB3 的 SWO Trace。

四個 Timer：Combined Channels = Encoder Mode；Encoder Mode = TI1 and TI2；
Prescaler = 0；Counter Mode = Up；Counter Period = 65535；
IC1/IC2 = Rising Edge、Direct TI、DIV1、Filter 0。
GPIO 上拉依編碼器輸出電路決定；已有板載上拉可用 No Pull。
GPIO 必須由 CubeMX 配成對應 Timer alternate function，不是一般 GPIO Input。
本測試不用開 Timer NVIC 中斷或 DMA，不需要 UART。

## 專案中必須保留的產生檔

- Core/Src/main.c、Core/Inc/main.h：主程式。
- 若選擇分開產生周邊檔：Core/Src/tim.c、Core/Inc/tim.h，及 gpio.c/gpio.h。
  未選分開產生時，Timer/GPIO 初始化可能放在 main.c；不必重複建立。
- Core/Src/stm32f4xx_hal_msp.c（及 tim.c 中的 MSP 初始化，視產生方式而定）：GPIO/Timer 時鐘與腳位。
- Core/Src/stm32f4xx_it.c、Core/Inc/stm32f4xx_it.h：保留 SysTick，HAL_GetTick 需要它。
- Core/Src/system_stm32f4xx.c、Core/Inc/stm32f4xx_hal_conf.h。
- CubeMX 產生的 Drivers/CMSIS、Drivers/STM32F4xx_HAL_Driver 與啟用的 HAL 原始碼。
- 專案產生的啟動組語、STM32F446RE 對應連結腳本、.ioc、IDE 專案與建置設定。

請用整個原專案 Build，不要只編譯 encoder_test.c，也不要刪除其他 CubeMX 產生檔。
確認 encoder_test.c 沒被 Exclude from Build，Core/Inc 在 include path。

## 測試

1. 關閉馬達動力電源，保留編碼器供電。若只以 ST-LINK USB 供電，依接線表將 JP5 設 U5V（1-2）；改跳帽前先斷電。
2. 複製兩個檔案、修改 main.c，Build 專案。
3. Debug 下載，Live Expressions 加入四個 encoder 變數、encoder_test_heartbeat、encoder_test_error。
4. Resume 後輕轉單一輪子；應只有對應計數連續變化，反轉時計數方向反轉，停下時大致不變。
5. delta_FL/FR/RL/RR 是最近 20 ms 的變化量，更新較快，初測優先觀察累積 encoder 值。

encoder_test_error：0 正常；1/2/3/4 分別代表 FL/FR/RL/RR 的 ARR 不是 65535；
11/12/13/14 分別代表該輪 HAL 啟動失敗。若進 Error_Handler，可暫停後檢查此變數。
心跳不變先檢查是否暫停、卡在 Error_Handler、SysTick 未執行；心跳有變但計數不變，再查供電、接地、訊號與 Timer 設定。

每次讀取間的實際位移必須少於 32768 counts；適用慢速手轉測試。
暫停除錯期間不要轉輪；若轉動過多，恢復後的位移可能無法正確還原。
累積計數為 int32_t，本程式用於短時間測試，不作無限期位置紀錄。
這份程式不會驅動馬達；若減速機無法輕鬆回轉，不要強扳。

## 驗證範圍

交付前以主機端 HAL 模擬檢查編譯與計數正反向、回繞和初始化錯誤處理。
尚未在使用者 STM32CubeIDE 專案完整建置或實機驗證。
