# **第九章 Unity 客户端结构设计（完整版）**

------

# **9.1 项目文件夹结构**

为方便开发和个人维护，推荐使用以下结构：

```
Assets/
├─ Art/                  # 美术资源
│   ├─ Sprites/          # 2D 精灵
│   ├─ UI/               # UI 图标、面板
│   └─ Effects/          # 特效
├─ Prefabs/              # 预制体
│   ├─ Monsters/         # 怪物预制体
│   ├─ NPCs/             # NPC 预制体
│   ├─ Items/            # 道具/装备预制体
│   └─ UI/               # UI 面板预制体
├─ Scripts/              # 脚本
│   ├─ Managers/         # 游戏管理类
│   ├─ Player/           # 玩家控制脚本
│   ├─ Monster/          # 怪物逻辑
│   ├─ Inventory/        # 背包逻辑
│   ├─ Equipment/        # 装备逻辑
│   ├─ Map/              # 地图逻辑
│   ├─ UI/               # UI 控制脚本
│   └─ Network/          # API/服务端通信
├─ Resources/            # JSON 配置文件
│   ├─ Equipments.json
│   ├─ Items.json
│   ├─ Monsters.json
│   └─ Maps.json
└─ Scenes/
    ├─ Main.unity        # 主场景
    ├─ Village.unity     # 新手村
    ├─ Field.unity       # 野外
    └─ Dungeon.unity     # 副本
```

> 使用此结构，可快速定位资源和逻辑，便于单人开发和后期扩展。

------

# **9.2 脚本结构设计**

### **9.2.1 Managers（管理类）**

- **GameManager.cs**
  - 全局控制游戏状态、初始化场景
  - 维护玩家、怪物、地图管理器引用
- **UIManager.cs**
  - 管理 UI 面板显示/隐藏
  - 快捷栏、背包、任务、战斗界面
- **InventoryManager.cs**
  - 管理背包数据
  - 装备穿戴、道具使用、分解、排序
- **DropManager.cs**
  - 管理怪物掉落计算
  - 调用掉落 JSON 表生成道具

------

### **9.2.2 玩家模块 (Player)**

- **PlayerController.cs**
  - 移动、攻击、技能释放
  - 接收输入（键盘/鼠标/虚拟摇杆）
- **PlayerStats.cs**
  - 玩家属性计算（基础 + 装备 + 强化 + Buff）
  - 提供方法获取战力
- **PlayerInventory.cs**
  - 玩家背包数据结构
  - JSON 数据加载、存储

------

### **9.2.3 怪物模块 (Monster)**

- **MonsterController.cs**
  - 怪物移动、攻击、死亡逻辑
  - AI 简单寻路与攻击
- **MonsterStats.cs**
  - 血量、攻击、防御计算
  - 经验与掉落触发
- **SpawnManager.cs**
  - 地图怪物刷新点
  - 控制普通怪、精英怪、Boss刷新

------

### **9.2.4 装备模块 (Equipment)**

- **EquipmentItem.cs**
  - 装备数据结构，含词条
  - 强化等级、附加属性计算
- **EquipmentManager.cs**
  - 穿戴/卸下装备
  - 自动更新玩家属性
  - 与背包系统对接

------

### **9.2.5 UI 模块 (UI)**

- **BackpackUI.cs**
  - 背包格子显示
  - 拖拽、右键、分解操作
- **EquipmentUI.cs**
  - 装备面板显示
  - 自动更新装备属性与词条
- **SkillUI.cs**
  - 技能快捷栏
  - 技能冷却显示、释放操作
- **DropUI.cs**
  - 掉落物显示面板
  - 掉落提示和拾取

------

### **9.2.6 网络模块 (Network)**

- **APIClient.cs**
  - HTTP / WebSocket 请求封装
  - 与服务端 API 对接
  - JSON 数据解析
- **DataSyncManager.cs**
  - 玩家数据本地与服务端同步
  - 背包、装备、任务状态更新

------

# **9.3 数据与逻辑绑定**

- JSON 配置（装备、怪物、地图） → Resources 文件夹 → 加载到对应 Manager
- 示例：

```
// Load Equipments.json
TextAsset equipData = Resources.Load<TextAsset>("Equipments");
EquipmentItem[] equipments = JsonUtility.FromJson<EquipmentItem[]>(equipData.text);
```

- 游戏逻辑调用：
  - 怪物死亡 → DropManager 计算掉落 → InventoryManager 添加道具
  - 玩家装备 → EquipmentManager 更新 PlayerStats
  - 背包 UI → InventoryManager 数据变化 → BackpackUI 更新显示

------

# **9.4 Unity MVP 开发注意事项**

1. **单人原型**
   - 使用 ScriptableObject 或 JSON 管理配置
   - 不必立即实现多人同步
2. **性能优化**
   - 怪物数量不宜过多
   - 使用对象池（Object Pool）管理怪物、掉落物
3. **UI 管理**
   - 所有 UI 面板通过 UIManager 控制显示隐藏
   - 背包、装备、技能栏可快速切换
4. **调试与日志**
   - GameManager 提供全局 Debug 输出
   - 掉落、战斗日志可在开发阶段打印

------

# **9.5 第九章总结**

1. 文件夹结构清晰，便于单人开发
2. Manager + 模块化脚本设计
3. 数据 JSON 与逻辑绑定，方便快速修改
4. UI 与逻辑分离，支持拖拽、穿戴、掉落操作
5. 可快速搭建 MVP 原型，并在此基础上迭代