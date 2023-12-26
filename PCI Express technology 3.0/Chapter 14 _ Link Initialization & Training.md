本章描述物理层的链路训练和状态机（Link Training and Status State Machine, LTSSM）：
- 链路的初始化过程是链路从通电或复位到链路达到完全运行的 L0 状态，在此期间发生正常的数据流量。
- 链路电源管理状态 L0s、L1、L2 和 L3 以及状态转换。
- 阐述恢复状态，在此期间的位锁定、符号（Symbol）锁定或块锁定。
- 链路带宽管理的链路速度和位宽变化。

# 1. Overview
# 2. Ordered Sets in Link Training
## 2.1 General
## 2.2 TS1 and TS2 Ordered Sets
# 3. Link Training and Status State Machine(LTSSM)
## 3.1 General
## 3.2 Overview of LTSSM States
## 3.3 Introduction, Examples and State/Substates
# 4. Detect State
## 4.1 Introduction
## 4.2 Detailed Detect Substate
### 4.2.1 Detect.Quiet
### 4.2.2 Detect.Active
# 5. Polling State
## 5.1 Introductin
## 5.2 Detailed Polling Substates
### 5.2.1 Polling.Active
### 5.2.2 Polling.Configuration
### 5.2.3 Polling.Compliance
# 6. Configuration State
## 6.1 Configuration State - General
## 6.2 Designing Devices with Links that can be Merged
## 6.3 Configuration State --Training Examples
### 6.3.1 Introduction
### 6.3.2 Link Configuration Example 1
### 6.3.3 Link Configuration Example 2
### 6.3.4 Link Configuration Example 3: Failed Lane
## 6.4 Detailed Configuration Substates
### 6.4.1 Configuration.Linkwidth.Start
### 6.4.2 Configuration.Linkwidth.Accpt
### 6.4.3 Configuration.Lanenum.Wait
### 6.4.4 Configuration.Lanenum.Accept
### 6.4.5 Configuration.Complete
### 6.4.6 Configuration.Idle
# 7. L0 State
## 7.1 Speed Change
## 7.2 Link Width Change
## 7.3 Link Partner Initiated
# 8. Recovery State
## 8.1 Reasons for Entering Recovery State
## 8.2 Initiating the Recovery Process
## 8.3 Detailed Recovery Substates
## 8.4 Speed Change Example
## 8.5 Link Equalization Overview
## 8.6 Detailed Equalization Substates
### 8.6.1 Recovery.Equalization
### 8.6.2 Recovery.Speed
### 8.6.3 Recovery.RcvrCfg
### 8.6.4 Recovery.Idle
## 8.7 L0s State
## 8.8 L1 State
## 8.9 L2 State
## 8.10 Hot Reset State
## 8.11 Disable State
## 8.12 Loopback State
### 8.12.1 Loopback.Entry
### 8.12.2 Loopback.Active
### 8.12.3 Loopback.Exit
# 9. Dynamic Bandwidth Changes
## 9.1 Dynamic Link Speed Changes
## 9.2 Upstream Port Initiates Speed Change
## 9.3 Speed Change Example
## 9.4 Software Control of Speed Changes
## 9.5 Dynamic Link Width Changes
## 9.6 Link Width Change Example
# 10. Related Configuration Registers
## 10.1 Link Capabilities Register.
### 10.1.1 Max Link Speed [3:0]
### 10.1.2 Maximum Link Width[9:4]
## 10.2 Link Capabilities 2 Register
## 10.3 Link Status Register
### 10.3.1 Current Link Speed[3:0]
### 10.3.2 Negotiated Link Width[9:4]
### 10.3.3 Undefined[10]
### 10.3.4 Link Training[11]
## 10.4 Link Control Register
### 10.4.1 Link Disable
### 10.4.2 Retrain Link
### 10.4.3 Extended Synch