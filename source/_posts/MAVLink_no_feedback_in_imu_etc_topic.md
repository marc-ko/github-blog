---
title: MAVLink「無回應」現象摘要：為何 MAVROS 有 `/mavros/state`，卻沒有高度／姿態／IMU
date: 2026-03-31
---

# MAVLink「無回應」現象摘要：為何 MAVROS 有 `/mavros/state`，卻沒有高度／姿態／IMU

本文整理在 **ArduPilot SITL + MAVROS2（ROS 2 Humble）** 環境下，出現 **`adaptive_precision_mission` 永遠讀不到高度**（`/mavros/local_position/pose`、`/mavros/global_position/rel_alt` 的 callback 計數為 0），但 **視覺節點／landing target 仍顯示約 9 m** 的完整成因、如何驗證、以及如何修復。

---

## 1. 表面現象（容易誤判的兩件事）

### 1.1 任務節點：高度永遠是 0、訊息計數為 0

日誌類似：

```text
Still climbing… altitude=0.00 m | pose_z=0.00 rel_alt=0.00 | msgs pose=0 rel_alt=0
```

程式裡的 `msgs pose=0`、`rel_alt=0` 指的是 **`local_pose_cb` / `rel_alt_cb` 從未被呼叫**，也就是 **ROS 上這兩個 topic 對該節點沒有送進任何一筆訊息**（不是「座標系解錯所以 z=0」而已）。

### 1.2 Landing target：body 座標出現約 -9 m

`landing_target_provider` 日誌可能出現：

```text
body=(0.004, -0.013, -9.153) | Dist: 9.15m
```

這來自 **AprilTag 偵測 + TF（相機／機體幾何）** 算出的**與標籤的相對幾何／距離**，**不是** FCU 經 MAVLink 回報的「離 home 高度」或 `GLOBAL_POSITION_INT.relative_alt`。

因此：**視覺有「約 9 m」並不能證明 MAVROS 有位置流**；兩條管線完全獨立。

---

## 2. 第一層排查（常被想到、多數不是主因）

| 假設 | 說明 |
|------|------|
| ROS 2 圖不一致（`ROS_DOMAIN_ID`、`ROS_LOCALHOST_ONLY`、未 `source install/setup.bash`） | **可能**導致某節點收不到任何 MAVROS topic；需與 launch 終端機比對環境。但若 **`/mavros/state` 有在更新**，通常表示至少與 MAVROS 在同一圖內。 |
| QoS 不符 | MAVROS 對多數感測器類 topic 使用 **BEST_EFFORT**（與 `qos_profile_sensor_data` 一致），與本專案任務節點訂閱方式一般 **相容**。 |
| 指令 `ros2 topic echo` 看不到資料 | 若預設訂閱 profile 與 publisher 不一致，可能誤以為「沒資料」；建議對感測器 topic 使用 **`--qos-profile sensor_data`** 或確認 `ros2 topic info -v` 的 QoS。 |

這些都值得檢查，但在本案例中 **還不是最精準的根**。

---

## 3. 根因（核心）：ArduPilot 預設 **不主動送** 位置／IMU 等「資料流」

### 3.1 ArduPilot 的 `SR0_*` 串流參數預設為 0

在 ArduCopter 的 `GCS_Mavlink.cpp` 裡，`GCS_MAVLINK_Parameters::var_info[]` 對各類 MAVLink **stream rate**（`SR0_RAW_SENS`、`SR0_POSITION`、`SR0_EXTRA1` 等）的預設值為 **0**（單位：Hz）。

語意上：**0 表示不依靠固定頻率推送該類訊息**，通常預期由 **GCS 發送 `REQUEST_DATA_STREAM`** 或後續透過 **`SET_MESSAGE_INTERVAL`** 等機制再決定要送什麼。

若 **沒有任何一側** 去「要資料」，則在該 MAVLink 連線上可能 **長期只有極少數訊息類型**（例如週期性 **HEARTBEAT**、**TIMESYNC**），而 **不會** 穩定出現：

- `GLOBAL_POSITION_INT`（關係到 `rel_alt` 等）
- `LOCAL_POSITION_NED`（關係到 `local_position/pose`）
- `RAW_IMU` / `ATTITUDE`（關係到 IMU／姿態外掛）

### 3.2 MAVROS 仍顯示 `connected: true` 的原因

`/mavros/state` 來自 **連線狀態／heartbeat 邏輯**，只要 **FCU 有在回心跳**，就容易看到 **`connected: true`**。  
這 **不代表** 高頻遙測（位置、IMU）已經在該連線上啟用。

因此會出現一種「很有趣」的組合：

- **有** `/mavros/state`（例如約 1 Hz）
- **沒有**（或長期收不到）`/mavros/local_position/pose`、`/mavros/global_position/rel_alt`、`/mavros/imu/data`

### 3.3 在 ROS 圖上直接驗證「線上到底有什麼 MAVLink」

若系統使用 **mavros_router**，可訂閱 **`/uas1/mavlink_source`**（型別 `mavros_msgs/msg/Mavlink`），在數秒內統計 **`msgid`**：

- **問題重現時**：可能只看到 **msgid 0（HEARTBEAT）**、**111（TIMESYNC）** 等極少類型。
- **修復後**：應能看到 **27（RAW_IMU）**、**30（ATTITUDE）**、**32（LOCAL_POSITION_NED）**、**33（GLOBAL_POSITION_INT）** 等。

這是區分「ROS／DDS 問題」與「FCU 根本沒送該類 MAVLink」的強證據。

---

## 4. 修復策略

### 4.1 推薦：`MAV_CMD_SET_MESSAGE_INTERVAL`（511）

對 ArduPilot 送出 **COMMAND_LONG**：

- **command** = `511`（`MAV_CMD_SET_MESSAGE_INTERVAL`）
- **param1** = MAVLink **message id**（例如 33、32、30、27）
- **param2** = 間隔（**微秒**），例如 `100000` → 約 10 Hz

**優點**：屬於 **連線上動態約定**，**不需要**為了開流先去改 EEPROM 參數並重開機（`SR0_*` 在參數元資料中常標 **RebootRequired**，實務上仍以現場測試為準）。

本倉庫已將此流程寫入 **`adaptive_precision_mission`**（在服務就緒後、設定 PLND 等參數前執行），並可用環境變數關閉或調整間隔：

- `APM_SET_MAVLINK_INTERVALS`：設為 `0` / `false` 可跳過
- `APM_MAVLINK_INTERVAL_US`：預設 `100000`

### 4.2 輔助腳本

- **`scripts/enable_mavlink_streams.sh`**：手動對 FCU 送同一組 `SET_MESSAGE_INTERVAL`（需在 MAVROS 已連線時執行）。
- **注意**：若腳本使用 **`set -u`**（nounset）又去 **`source /opt/ros/humble/setup.bash`**，可能觸發 `AMENT_TRACE_SETUP_FILES: unbound variable`；目前腳本已改為 **`set -eo pipefail`**（不使用 `-u`）以避免與 ROS 安裝腳本互斥。
```bash
#!/usr/bin/env bash
# One-shot: request ArduPilot to emit position/IMU/attitude (SR0_* default 0 in AP).
# Uses MAV_CMD_SET_MESSAGE_INTERVAL (511). Run after MAVROS is up:
#   source install/setup.bash && ./scripts/enable_mavlink_streams.sh
#
# Optional: MAVROS_PREFIX=/mavros (default)

set -eo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
WS_ROOT="${ARDU_WS:-$(cd "$SCRIPT_DIR/.." && pwd)}"

if [[ -f /opt/ros/humble/setup.bash ]]; then
  # shellcheck source=/dev/null
  source /opt/ros/humble/setup.bash
fi
if [[ -f "$WS_ROOT/install/setup.bash" ]]; then
  # shellcheck source=/dev/null
  source "$WS_ROOT/install/setup.bash"
fi

MP="${MAVROS_PREFIX:-/mavros}"
MP="${MP%/}"
INTERVAL_US="${APM_MAVLINK_INTERVAL_US:-100000}"

call_interval() {
  local id="$1"
  local name="$2"
  ros2 service call "${MP}/cmd/command" mavros_msgs/srv/CommandLong \
    "{broadcast: false, command: 511, confirmation: 0, param1: ${id}.0, param2: ${INTERVAL_US}.0, param3: 0.0, param4: 0.0, param5: 0.0, param6: 0.0, param7: 0.0}" \
    | tail -1
  echo "  SET_MESSAGE_INTERVAL id=${id} (${name}) interval_us=${INTERVAL_US}"
}

echo "Requesting MAVLink message intervals on ${MP}/cmd/command ..."
call_interval 33 GLOBAL_POSITION_INT
call_interval 32 LOCAL_POSITION_NED
call_interval 30 ATTITUDE
call_interval 27 RAW_IMU
echo "Done. Check: ros2 topic hz ${MP}/global_position/rel_alt"

```

### 4.3 替代／補強：調高 `SR0_*`（需理解參數與重啟行為）

亦可透過 MAVROS **參數寫入** 將 `SR0_POSITION`、`SR0_RAW_SENS`、`SR0_EXTRA1` 等設為非 0（Hz）。  
請以 ArduPilot 文件與實機行為為準，並注意頻寬與 **RebootRequired** 標記。

---

## 5. 驗證清單（修復前／後對照）

1. **`ros2 topic hz /mavros/global_position/rel_alt`**：修復後應有穩定 rate（例如約 10 Hz，視 interval 而定）。
2. **`ros2 topic hz /mavros/local_position/pose`**：同上。
3. **訂閱 `/uas1/mavlink_source` 統計 msgid**：應出現 27／30／32／33 等。
4. **任務日誌**：爬升階段 `msgs pose`、`rel_alt` 應 **> 0** 且高度隨起飛上升。

---

## 6. 與「純 ROS／DDS」問題的界線（簡表）

| 觀察 | 較像 DDS／環境 | 較像本摘要（MAVLink 流未開） |
|------|----------------|------------------------------|
| 多數 MAVROS 感測器 topic 都沒資料，但 launch 與 mission 的 `ROS_DOMAIN_ID` 不同 | ✓ | |
| `/mavros/state` 正常，但僅少數 msgid 出現在 `/uas1/mavlink_source` | | ✓ |
| 手動執行 `SET_MESSAGE_INTERVAL` 後，`hz` 立刻恢復 | | ✓ |

---

## 7. 結語

此問題本質上是 **MAVLink 層級「沒有向 FCU 約定要哪些訊息、多常送」**，在 ArduPilot 預設 **`SR0_* = 0`** 的情境下特別容易發生；MAVROS 仍可能顯示 **已連線**，但 **位置／高度／IMU 外掛沒有輸入資料**，進而讓上層任務以為「高度永遠 0」。  
與 **AprilTag／TF** 的視覺距離並無矛盾——它們本來就不是同一資料來源。

---

## 8. 相關檔案（本倉庫）

| 檔案 | 用途 |
|------|------|
| `src/adaptive_precision_mission/adaptive_precision_mission/mission.py` | `ensure_mavlink_message_intervals()`、爬升階段日誌、`mavros_ns` 參數 |
| `scripts/enable_mavlink_streams.sh` | 手動開啟 MAVLink 訊息間隔 |
| `scripts/check_mavros_streams.sh` | `topic info -v` 與 `hz` 抽樣 |
| `scripts/check_ros_env.sh` | 列印 `ROS_DOMAIN_ID` 等環境變數 |
| `src/ardupilot/ArduCopter/GCS_Mavlink.cpp` | `SR0_*` 預設與 stream 說明（原始碼依版本為準） |

---

*文件目的：保存一次完整除錯結論，方便日後遇到「MAVROS 連上了卻沒高度」時快速對照。*
