# 发货机器人OpenAPI文档
版本 V1.0
### API调用说明
调用接口之前应当向希望使用第三方发货机器人的服主客户取得商城**OpenAPI鉴权字段**，请各位服主客户保护好自己的商城鉴权信息，更换机器人提供商或者发现有外泄嫌疑请向髓扬帝报告并更换字段信息，后续会在管理后台设置字段让服务器管理员自行管理OpenAPI鉴权字段

发货机器人OpenAPI遵循最低限度的restful API设计模式，以下是调用接口时必须使用的公共请求头
```javascript
req.header["scum-store-server-id"] = `${服务器ID}`
req.header["scum-store-secret-key"] = `${第三方鉴权字段}`
```
这两个字段在服主客户开通商城的时候会生成并交付给到服主，请妥善保管，注意必要的信息安全保护

### 机器人接入和工作流程最佳实践
1. 服主在机器人配置文件设置好**OpenAPI鉴权字段**
2. 调用`GET /serverConfig/robotConfigs`获取机器人配置信息
3. 调用`GET /queue/nextPending`(普通队列)或`GET /queue/nextPendingMonitor`(监控摄像头队列)获取下一条未处理消息队列
4. 调用`PUT /queue/updateQueueItem`把该条目的**status**置为**processed**
5. 发货
6. 视乎发货成功或者出错，调用`PUT /queue/updateQueueItem`把该条目的**status**置为**fullfilled**/**created**，如果发货失败可以根据需要设置条件进入冷却时间

**附加步骤**
1. 上面步骤6中的发货内容包含载具的，把载具ID回传
2. 上面步骤6中的发货内容包含特殊指令的(例如清空滴滴车，调用`GET /buy/listDidiVehicleIds`获取购买记录登记的滴滴车列表或者按照机器人配置信息中的滴滴车型号来清理滴滴车)，使用特殊逻辑，下面的接口列表会介绍所有特殊指令
3. 发货内容是购买玩家坐标的，调用`PUT /buy/buyUserLocationCallback`回调接口上传获取到的坐标
4. 开启了全图杀戮的，调用`PUT /buy/locationExposeCallback`定时上传不带玩家信息的地图坐标列表
5. 开启了定时上传全图载具列表的，调用`PUT /serverConfig/updateVehicleIdsList`上传载具列表
6. 开启了定时上传玩家信息和位置列表的，调用`PUT /serverConfig/allPlayersLocationsCallback`上传玩家信息和位置列表
7. 开启了安全区自定义设置/购买私人安全区的，调用`GET /serverConfig/safeZonePurchaseRobotConfigs`定时获取最新安全区配置以更新服务器安全区设置

### API返回结构体和返回码含义
调用OpenAPI返回的内容的`Content-Type`为`application/json`，一般格式为以下的json
```typescript
{
  "status": number, // 返回码，调用成功为200
  "data": any, // 返回内容，可能为任何类型，一般是object类型
  "msg": string, // 返回状态信息，status为200时一般为空
}
```
以下是返回码`status`常见的值和含义
1. `200`: 正常调用
2. `500`: 非法调用或者请求体畸形被阻拦
3. `59999`: 内部错误，也可能由不正确/非法调用的参数引起，详细请看返回状态信息msg
4. `60001`: OpenAPI鉴权不通过
5. `60005`: 参数不正确/缺失或重复导致调用失败

### API列表
#### GET /serverConfig/robotConfigs
##### 说明
获取机器人运行所需配置
##### 入参
无
##### 出参
```json
{
  "status": 200
  "data": {
    "DelayAfterNormalCommand": 50, // 输入一般指令后的停顿间隔(毫秒)
    "DelayAfterTeleportToPlayer": 45000, // 输入#teleportto指令传送到玩家后等待读图的停顿间隔(毫秒) 和运行机器人的云电脑性能有关系，参考数值: 750ti -> 60000 - 70000, 1060ti -> 45000 - 60000, 3070 -> 30000 - 45000, 1080ti -> 25000 - 30000, 2080ti -> 10000 - 25000
    "DelayAfterLocatePlayer": 50, // 输入#location指令定位玩家位置后的等待返回结果的停顿间隔(毫秒)
    "DelayAfterListPlayers": 50, // 输入#listplayers指令获取当前玩家列表后的等待返回结果的停顿间隔(毫秒)
    "DelayAfterListSpawnedVehicles": 50, // 输入#listspawnedvehicles指令获取当前地图载具列表后的等待返回结果的停顿间隔(毫秒)
    "DiDiVehicleType": "BPC", // 滴滴车型号
    "RobotRPYConfig": { "R": 0,"P": 0.220856,"Y": 73.575951 }, // 发货商品使用远程发货模式2的时候机器人发货所需要的球坐标设置
    "EnableAutoFetchVehicleIdList": true, // 是否开启定时上传全图载具列表
    "EnableAutoFetchPlayerLocationList": true, // 是否开启定时上传玩家信息和位置列表
    "EnableInnerMode": true, // 是否启用内置商城
    "EnableNativeInnerMode": true, // 为true的时候启用机器人识别，false的时候使用ftp日志
    "InnerModeConfigs": [{"function":"signup","enable":true,"keywordEn":"@signup","keywordCn":"@注册"}], // 内置商城配置 function->指令功能的key enable->是否启用 keywordEn->英文识别前缀 keywordCn->中文识别前缀
    "InnerModeShortCutConfigs": [{"shortcut":"@炼体区", "original":"@购买 传送到炼体区 1","enable":true}], // 内置商城快捷指令配置 shortcut->快捷指令 original->原指令 enable->是否启用
    "InnerModeDiyFaqConfigs": [{"ask":"指令大全", "answers":["@购买 等配置", "@购买 等配置"],"enable":true}], // 内置商城自定义问答配置 ask->问题 answers->答案，一个数组元素一行 enable->是否启用
    "RecycleConfigs": [{"title":"召唤空投", "needs":[{"item": "BP_C4", "count":3}, {"item": "BP_Weapon_MP5", "count":1}], "actions": [{"configType": "vip", ...其他参数}], "enable":true}], // 物品回收设置 title->选项标题 needs->需要物品和达到的数量 actions-> 满足条件后触发的动作 enable->是否启用
  },
  "msg": ""
}
```
#### GET /queue/nextPending
##### 说明
获取下一条未处理消息队列
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|filterMultiRobot|否|query|string|多机器人模式，默认不带这个参数，需要多机器人时传1|无|
|filterMultiRobotTotal|否|query|string|多机器人模式的机器人总数，仅filterMultiRobot=1的时候生效|无|
|filterMultiRobotNum|否|query|string|多机器人模式的机器人编号(1-n)，仅filterMultiRobot=1的时候生效|无|
|filterDoubleSingle|否|query|string|单双机器人模式，single是1号，double是2号，与前面的多机器人模式冲突|无|
|filterOnlyBuy|否|query|string|只接受订单消息|无|
|filterNotBuy|否|query|string|只接受非订单消息|无|
|filterOnlyBountyHunter|否|query|string|只接受赏金猎人消息|无|
|filterOnlyNormal|否|query|string|非订单和特殊功能之外的所有消息|无|

*需要注意的是，多机器人模式定义了n台机器人就要n台机器人都在线，否则每少一台机器人就会漏发多大约1/n的消息*
##### 出参
```json
{
  "status": 200
  "data": {
    "id": 1, // 队列ID，更新队列状态的时候需要用到
    "commands": "【玩家】， 您购买的商品将在60秒内到达，请在原地等候\n[玩家],  your order [commodity] will be proccessed in 60 seconds，please wait\n#teleportto 76566776677667776\n#SpawnItem BP_ImprovisedSmallPaddle\n 您购买的商品已发货完毕，请查收\n your order [commodity] has been delivered，please check\n#teleport 0 0 200", // 需要发送的指令，以\n分割换行，下面会介绍特殊指令
    "status": "created", // 队列状态 created: 新建, processed: 正在发货, fullfilled: 已完成
    "buyId": "2|4" // 关联的购买记录ID，可能是"2"或者"2|4"这种形式，以区别关联了一个或者多个订单，购买玩家位置的时候调用"/buy/buyUserLocationCallback"上传位置需要用到
    ...
  },
  "msg": ""
}
```
**commands里面的特殊指令**
* `<x>`: 按X键
* `<ctrl-c>`: 复制
* `<ctrl-v>`: 粘贴
* `<enter>`: 按回车键
* `<tab>`: 打开聊天框按Tab键
* `<tab-${number}>`: 多机器人模式使用，表示第n号机器人打开聊天框按Tab键
* `<clean-didi-vehicles>`: 根据`GET /serverConfig/robotConfigs`获取的滴滴车型号来清理滴滴车
* `<clean-didi-vehicles-by-id>`: 根据`GET /buy/listDidiVehicleIds`获取的滴滴车列表来清理滴滴车
* `<locate-steam-id>~${steamId}`: 发货之前获取这个玩家的坐标替换一下每一行指令的steam ID
* `<locate-steam-id-rpy>~${steamId}`: 发货之前获取这个玩家的坐标(SCUM新出的球坐标)替换一下每一行指令的steam ID
* `<buy-teleport-squad>~${steamId}~${toSteamId}`: 购买队友传送，发货之前要获取这传送/被传送玩家的坐标替换一下指令的两个steam ID
* `<callback-buy-user-location>~${steamId}~${buyId}`: 购买玩家位置，获取玩家位置之后调用`PUT /buy/buyUserLocationCallback`回调接口上传获取到的坐标
* `<callback-location-expose>`: 上传不带玩家信息的地图坐标列表，获取玩家位置之后调用`PUT /buy/locationExposeCallback`回调接口上传获取到的坐标列表
* `<rename-steam-id>~${steamId}~${newName}`: 重命名某Steam ID玩家的游戏名称
* `<relevel-steam-id>~${steamId}~${militaryLevel}`: 给某Steam ID玩家的游戏名称前缀塞入军衔等级(militaryLevel为空字符串的时候代表去除军衔)

**特殊策略触发**
1. 机器人离线策略触发 - nitrado服务器日志不实时，故不予处理
```json
{
  "status": 200
  "data": {
    "action": "offline"
  },
  "msg": ""
}
```
2. 服务器重启保护策略触发
```json
{
  "status": 200
  "data": {
    "action": "offline",
    "before": 10, // 重启前n分钟 - 没有传的话机器人默认两分钟
    "after": 10 // 重启后n分钟 - 没有传的话机器人默认两分钟
  },
  "msg": ""
}
```
#### GET /queue/nextPendingMonitor
##### 说明
监控探头功能专用机器人，获取下一条未处理消息队列
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|filterMultiRobot|否|query|string|多机器人模式，默认不带这个参数，需要多机器人时传1|无|
|filterMultiRobotTotal|否|query|string|多机器人模式的机器人总数，仅filterMultiRobot=1的时候生效|无|
|filterMultiRobotNum|否|query|string|多机器人模式的机器人编号(1-n)，仅filterMultiRobot=1的时候生效|无|
|filterDoubleSingle|否|query|string|单双机器人模式，single是1号，double是2号，与前面的多机器人模式冲突|无|

*需要注意的是，多机器人模式定义了n台机器人就要n台机器人都在线，否则每少一台机器人就会漏发多大约1/n的消息*
##### 出参
```json
{
  "status": 200
  "data": {
    "id": 1, // 队列ID，更新队列状态的时候需要用到
    "commands": "#teleport 0 0 200", // 需要发送的指令
    "status": "created" // 队列状态 created: 新建, processed: 正在发货, fullfilled: 已完成
    ...
  },
  "msg": ""
}
```
**特殊策略触发方针和`GET /queue/nextPending`一致**
#### GET /buy/listDidiVehicleIds
##### 说明
队列行解析识别到`<clean-didi-vehicles-by-id>`的时候获取最新的滴滴车列表进行清理，此方式比较精准，避免把一整个型号变成滴滴车
##### 入参
无
##### 出参
```json
{
  "status": 200
  "data": {
    "list": [23, 345, 566, 712] // 滴滴车编号集合
  },
  "msg": ""
}
```
#### GET /serverConfig/safeZonePurchaseRobotConfigs
##### 说明
开启了安全区自定义设置/购买私人安全区的，调用该接口定时获取最新安全区配置以更新服务器安全区设置
##### 入参
无
##### 出参
```json
{
  "status": 200
  "data": {
    "EnablePurchaseSafeZone": true, // 是否开启安全区自定义设置/购买私人安全区
    "ServerSafeZones": [] // 安全区列表
  },
  "msg": ""
}
```
**ServerSafeZones.item**
```json
{
  "zoneType": "circle", // 安全区形状 circle: 圆形, rectangle: 四边形
  "centerXY": { "x": 2000, "y": -2000 }, // 区域中心坐标，圆形安全区就是圆心，四边形就是中心
  "radius": 10000, // 圆形安全区半径
  "width": 8000, // 四边形安全区长边宽度
  "height": 5000, // 四边形安全区长短边宽度
  "fromXY": { "x": 2000, "y": -2000 }, // 四边形安全区左上角坐标
  "fromXY": { "x": 2000, "y": -2000 }, // 四边形安全区右下角坐标
}
```
#### PUT /queue/updateQueueItem
##### 说明
更新队列状态，发出一条队列之前先调这个接口把**status**置为**processed**，视乎发货成功或者出错，发送完之后再调一次这个接口把**status**置为**fullfilled**/**created**
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|id|是|json|number/string|队列ID 兼容新旧格式|无|
|status|是|json|number|状态(created/processed/fullfilled)|无|
|vehicleId|否|json|string|车辆ID(如有) 只有玩家发货了一辆车的时候才附带，一般为滴滴车|无|
##### 出参
```json
{
  "status": 200
  "data": {},
  "msg": ""
}
```
#### PUT /serverConfig/updateVehicleIdsList
##### 说明
定时上传全图载具列表
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|updateTimeStamp|是|json|string|获取时间的毫秒时间戳|无|
|list|是|json|array|车辆ID列表 元素类型为number|无|
##### 出参
```json
{
  "status": 200
  "data": {},
  "msg": ""
}
```
#### POST /buy/allPlayersLocationsCallback
##### 说明
定时上传玩家信息和位置列表
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|recordTimeStamp|是|json|string|获取时间的毫秒时间戳|无|
|locations|是|json|array|玩家位置列表 { "locations": "200 304.5 -2", "steamId": "76564443332345678" }|无|
##### 出参
```json
{
  "status": 200
  "data": { "success": 100, "fail": 0 },
  "msg": ""
}
```
#### POST /buy/buyUserLocationCallback
##### 说明
购买玩家坐标的回调
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|buyId|是|json|string|队列关联的订单ID|无|
|steamId|是|json|string|被购买坐标的玩家steam ID|无|
|rawLocation|是|json|string|`#location ${steamID}`返回的玩家位置信息，例如: "player1: X=200.22 Y=-455.45 Z=242.24"|无|
##### 出参
```json
{
  "status": 200
  "data": {},
  "msg": ""
}
```
#### POST /buy/locationExposeCallback
##### 说明
全图杀戮定时上传不带玩家信息的地图坐标列表的回调
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|locations|是|json|array|数组的每一项是`#location ${steamID}`返回的玩家位置信息，例如: "player1: X=200.22 Y=-455.45 Z=242.24"|无|
##### 出参
```json
{
  "status": 200
  "data": {},
  "msg": ""
}
```
#### POST /innerMode/userRequest
##### 说明
内置商城模式启用且机器人识别启用的时候，通过这个接口上传用户的指令
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|action|是|json|string|指令功能的key|无|
|steamId|是|json|string|发送者的Steam ID|无|
|scumId|是|json|string|发送者的游戏名|无|
|channel|是|json|string|消息发送的频道(Global->公屏 Squad->队内 Local->附近 Admin->管理)|无|
|message|是|json|string|消息原文|无|

**action支持的指令功能key和它们的含义**
|指令功能的key|含义
| :------------ | :------------ |
|signup|注册用户|
|setpassword|设置商城密码|
|welcomepack|领取新手礼包|
|welcomedidi|领取新手滴滴车|
|purchase|购买商品/套餐|
|sendset|发送套餐|
|teleportzone|地图分区域传送|
|teleporthot|热点区域传送|
|teleportprivate|私人传送|
|dollarloot|美金抽奖|
|itemloot|物品抽奖|
|getfame|声望兑换|
|transit|玩家转账|
|monthlyset|优惠月卡|
|checkin|每日签到|
|chekinpack|领取签到礼包|
|achievepack|领取成就礼包|
|lottery|购买公益彩票|
|squad|队伍管理|
|teleportsquad|队伍传送|
|squadgather|梁山好汉|
|bountyhunter|赏金猎人|
|joinhunter|领取猎人任务|
|exithunter|退出猎人任务|
|comparesize|比大小|
|recycleitem|物品回收(*需要机器人先检测妥当再执行回调)|
|militarylevel|购买军衔|
##### 出参
```json
{
  "status": 200
  "data": {},
  "msg": ""
}
```
#### POST /user/updateUserLogins
##### 说明
帝国神话游戏专用，机器人上传玩家登录退出记录，需要按照时间先后顺序升序排列
##### 入参
|参数名|必填|参数类型|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|records|是|json|array|登录退出记录|无|

**records数组的元素**
|参数名|必填|数据类型|描述|默认值|
| :------------ | :------------ | :------------ | :------------ | :------------ |
|steamId|是|string|玩家17位Steam ID|无|
|scumId|是|string|玩家服务器ID|无|
|status|是|string|登录还是退出(login/logout)|无|
|timeStamp|是|string|毫秒时间戳|无|
##### 出参
```json
{
  "status": 200
  "data": {
    "results": [ //解析出错的条目
      {
        //...records数组的元素
        "error": "报错详情"
      }
      ...
    ]
  },
  "msg": ""
}
```