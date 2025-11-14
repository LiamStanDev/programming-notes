### Exceptional Control Flow
相較於普通控制流，異常控制流會從用戶態轉為內核態，會執行在完全不同的環境，觸發的情況有幾種：
1. Trap：發生於使用系統調用，是有意發生的。
2. Device Interrupt：外設中斷與 CPU 執行的指令完全無關，是異步的中斷。
4. Exception
> 在這邊統一都稱為 Trap


### 