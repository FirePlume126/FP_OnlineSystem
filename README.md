
# 联机系统

* **插件未开源**

## 作者信息

Copyright FirePlume, All Rights Reserved.

Email: fireplume@126.com<br>
GitHub: [FirePlume126](https://www.github.com/FirePlume126)<br>
Bilibili: [火羽FP](https://space.bilibili.com/395084718)<br>
YouTube: [FirePlume126](https://www.youtube.com/@FirePlume126)

**[返回目录](https://www.github.com/FirePlume126/FP_Readme#Directory)**

<a name="fponlinesystem"></a>
## FPOnlineSystem

管理服务器和会话，加载/保存服务器存档和玩家存档并生成玩家

* **此模块的主要类**

|类名|描述|
|:-:|:-:|
|FPOnlineManagerSubsystem|联机管理子系统：专用服务器创建会话和地图，客户端读取玩家ID和名称|
|FPOnlineServerSubsystem|服务器子系统：管理服务器数据，加载/保存服务器和玩家存档|
|FPOnlineSessionsSubsystem|会话子系统：管理会话|
|FPOnlinePlayerComponent|联机玩家组件：处理服务器玩家存档并生成玩家，仅支持添加给`APlayerController`|
|FPOnlinePlayerInterface|联机玩家接口，仅支持添加给`APlayerController`，重写接口函数`GetOnlinePlayerComponent()`来和系统内部进行交互|
|FPOnlineSaveServerArchive|保存服务器存档，继承此类添加要保存的服务器存档|
|FPOnlineDedicatedServerSettings|专用服务器设置，保存在ServerSettings.ini中|

* **存档和配置文件的读取保存时机**

|名称|功能|执行端|读取时机|保存时机|
|:-:|:-:|:-:|:-:|:-:|
|UserSettings.sav|保存玩家信息，主要包括玩家ID和名称，玩家ID不支持修改|客户端|游戏启动时|修改时|
|ServerArchive_地图名称.sav|保存服务器存档，主要包括玩家信息列表、服务器时间和自定义服务器数据|服务器|创建服务器|玩家退出游戏、退出服务器、周期性保存|
|Player_"玩家ID".sav|保存玩家游戏数据(类如：位置、属性等)，每个玩家对应一个存档|服务器|玩家加入游戏|更换`Pawn`(除了加入游戏更换`Pawn`)、玩家退出游戏、周期性保存|
|ServerSettings.ini|专用服务器配置文件|服务器|创建专用服务器|手动修改|

专用服务器配置文件：ServerSettings.ini
```ini
[ServerSettings]
bHasServerSettings=True
bServerCreateSession=True
SessionCreationDelay=2
ServerSaveCycle=600

[SessionsSettings]
ServerName=None
MapName=None
Password=
bDedicatedServer=True
bIsLANMatch=False
NumPublicConnections=10
NumPrivateConnections=0
BuildUniqueId=1
bAllowJoinInProgress=True
bAllowJoinViaPresence=True
bShouldAdvertise=True
bUsesPresence=False
bUseLobbiesIfAvailable=False
bUsesStats=False
bUseLobbiesVoiceChatIfAvailable=False
bAllowJoinViaPresenceFriendsOnly=False
bAllowInvites=True
bAntiCheatProtected=False
bStartAfterCreate=False
```

* **流程图**

1、专用服务器创建会话和地图，客户端读取玩家ID和名称
![FPOnlineSystem_StartGame](https://github.com/FirePlume126/FP_OnlineSystem/blob/main/Images/FPOnlineSystem_StartGame.png)

2、读取保存服务器存档和服务器玩家存档，服务器玩家存档类需要在项目设置中添加
![FPOnlineSystem_SaveGame](https://github.com/FirePlume126/FP_OnlineSystem/blob/main/Images/FPOnlineSystem_SaveGame.png)

* **使用指南**

1、**FPOnlineSystem**项目设置

|属性|数据类型|描述|
|:-:|:-:|:-:|
|MainMenu|`TSoftObjectPtr<UWorld>`|主菜单地图|
|Maps|`TArray<TSoftObjectPtr<UWorld>>`|可以切换的地图，专用服务器可以通过ServerSettings.ini设置要切换的地图名称|
|ServerSaveCycle|`float`|服务器保存周期，专用服务器在ServerSettings.ini中设置|
|SaveServerDataClass|`TSubclassOf<UFPOnlineSaveServerArchive>`|保存服务器数据的类，服务器用来保存服务器存档|
|SavePlayerDataClass|`TSubclassOf<USaveGame>`|保存玩家数据的类，服务器用来保存玩家存档|

![FPOnlineSystemSettings](https://github.com/FirePlume126/FP_OnlineSystem/blob/main/Images/FPOnlineSystemSettings.png)

2、在`AGameModeBase`绑定服务器存档更新委托并初始化服务器，在`APlayerController::OnUnPossess()`中调用`UFPOnlinePlayerComponent::WriteSavePlayerData()`

```c++
void AMyGameModeBase::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
	Super::InitGame(MapName, Options, ErrorMessage);
	if (UFPOnlineServerSubsystem* ServerSubsystem = UFPOnlineFunctionLibrary::GetServerSubsystem(this))
	{
		ServerSubsystem->OnServerDataOperation.AddDynamic(this, &AMyGameModeBase::HandleServerServerData);
		ServerSubsystem->InitServer();
	}
}

void AMyGameModeBase::HandleServerServerData(UFPOnlineSaveServerArchive* ServerArchiveObject, EFPOnlinePlayerDataOperation Operation)
{
	if (UMySaveServerData* SaveServerData = Cast<UMySaveServerData>(ServerArchiveObject))
	{
		if (Operation == EFPOnlinePlayerDataOperation::Load)
		{
			// 加载使用服务器存档
		}
		else if (Operation == EFPOnlinePlayerDataOperation::Save)
		{
			// 编写保存服务器存档
		}
	}
}

void AMyPlayerController::OnUnPossess()
{
	OnlinePlayerComp->WriteSavePlayerData();
	Super::OnUnPossess();
}
```

3、给需要联机的`APlayerController`添加`FPOnlinePlayerComponent`组件，并添加`FPOnlinePlayerInterface`接口，重写接口函数`GetOnlinePlayerComponent()`。
可以根据需要绑定组件的委托或调用组件的函数
```c++
// .h
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FFPOnlinePlayerSimpleDelegate);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FFPOnlinePlayerDataOperationDelegate, USaveGame*, NewSaveObject, EFPOnlinePlayerDataOperation, Operation);

// 玩家数据操作委托，把NewSaveObject转换成UFPOnlineProjectSettings::SavePlayerInfoClass后加载/保存玩家存档
UPROPERTY(BlueprintAssignable, BlueprintAuthorityOnly, Category = "FPOnline")
FFPOnlinePlayerDataOperationDelegate OnPlayerDataOperation;

// 禁止玩家加入游戏委托，不绑定此委托玩家会直接退出游戏，没有拉黑提示(仅在本地执行)
UPROPERTY(BlueprintAssignable, Category = "FPOnline")
FFPOnlinePlayerSimpleDelegate OnBanPlayerJoinGame;

// 加入游戏更换Pawn完成委托，绑定此委托执行让屏幕亮起来的逻辑(仅在本地执行)
UPROPERTY(BlueprintAssignable, Category = "FPOnline")
FFPOnlinePlayerSimpleDelegate OnJoinGameChangePawnCompleted;

// 检测世界分区流送，加载流送完成后生成角色
UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "FPOnline")
bool bCheckWorldPartitionStreaming = true;

// 加入游戏要更换的APawn类，在委托OnPlayerDataOperation加载存档时，修改此变量可以生成玩家职业对应的Pawn
UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "FPOnline")
TSubclassOf<APawn> PlayerPawnClass;

// 生成角色的变换，在委托OnPlayerDataOperation加载存档时，修改此变量可以在此位置生成Pawn
UPROPERTY(BlueprintReadWrite, Category = "FPOnline")
FTransform CharacterTransform;

// 更换角色
// @param NewPawnClass 不为空时修改PlayerPawnClass生成角色，否则直接使用PlayerPawnClass
// @param bIsPlayerStart 为true时，在开始游戏处生成角色
// @param InSpawnTransform bIsPlayerStart = false 时有效，InSpawnTransform位置不为零时，在此位置生成，否则直接使用CharacterTransform
UFUNCTION(BlueprintCallable, Category = "FPOnline")
void ChangeCharacter(TSubclassOf<APawn> NewPawnClass = nullptr, bool bIsPlayerStart = false, const FTransform& InSpawnTransform = FTransform());

// 玩家Pawn死亡时调用
// @param InRespawnDelay 复活时间，等于0时立即复活
UFUNCTION(BlueprintCallable, Category = "FPOnline")
void PlayerPawnDied(float InRespawnDelay = 0.0f);
```

4、调用UFPOnlineFunctionLibrary的函数切换地图或调用子系统功能，这些函数可以根据情况使用
```c++
// 获取联机玩家组件
// @param InPlayerController 玩家控制器
// @param bLookForComponent 可以通过FindComponentByClass()查找组件
UFUNCTION(BlueprintPure, Category = "FPOnline")
static UFPOnlinePlayerComponent* GetOnlinePlayerComponent(APlayerController* InPlayerController, bool bLookForComponent = true);

//====================================切换地图====================================>>

// 获取地图URL
// @param InMapName 需要在UFPOnlineProjectSettings::Maps设置才可以获取URL
UFUNCTION(BlueprintPure, Category = "FPOnline")
static FString GetMapURL(const FString& InMapName);

// 获取所有地图名称，需要在UFPOnlineProjectSettings::Maps设置
UFUNCTION(BlueprintPure, Category = "FPOnline")
static TArray<FString> GetAllMapName();

// 获取地图名称
UFUNCTION(BlueprintPure, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static FString GetMapName(const UObject* WorldContextObject);

// 设置地图名称，保存在硬盘，重启游戏默认选择的地图
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void SetMapName(const UObject* WorldContextObject, const FString& NewMapName);

// 打开服务器地图
// @param InMapName 需要在UFPOnlineProjectSettings::Maps设置才可以打开地图
// @param bInOnline 多人联机游戏输入true
// @return 成功打开服务器地图返回true
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static bool OpenServerLevel(const UObject* WorldContextObject, const FString& InMapName, bool bInOnline = false);

// 打开地图
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void CallOpenLevel(const UObject* WorldContextObject, const FString& InAddress);

// 客户端旅行
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void CallClientTravel(const UObject* WorldContextObject, const FString& InAddress);

// 打开主菜单地图，联机游戏会先删除会话
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void OpenMainMenu(const UObject* WorldContextObject);

//====================================子系统功能====================================>>

// 玩家升级时更新
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void UpdateWhenUpLevelPlayer(const UObject* WorldContextObject, int32 InPlayerId, int32 InLevel);

// 拉黑或取消拉黑玩家时更新
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void UpdateWhenBannedPlayer(const UObject* WorldContextObject, int32 InPlayerId, bool bInHasBan);

// 删除玩家存档时更新
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void UpdateWhenRemovePlayer(const UObject* WorldContextObject, int32 InPlayerId);

// 获取玩家信息列表
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static TArray<FFPOnlinePlayersInfo> GetPlayersInfoList(const UObject* WorldContextObject);

// 获取在线玩家控制器列表
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static TArray<APlayerController*> GetPlayerControllerList(const UObject* WorldContextObject);

// 设置服务器时间，仅用来作弊
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void SetServerTimeSeconds(const UObject* WorldContextObject, int32 InTime);

// 获取服务器时间
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static int32 GetServerTimeSeconds(const UObject* WorldContextObject);

// 设置玩家ID：本地调用，可以通过APlayerState获取玩家ID
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void SetPlayerId(const UObject* WorldContextObject, int32 NewPlayerId);

// 设置玩家名称：本地调用，可以通过APlayerState获取玩家名称
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "FPOnline")
static void SetPlayerName(const UObject* WorldContextObject, const FString& NewPlayerName);

// 获取网络模式
UFUNCTION(BlueprintPure, meta = (WorldContext = "WorldContextObject", DisplayName = "GetNetMode"), Category = "FPOnline")
static EFPOnlineNetMode K2_GetNetMode(const UObject* WorldContextObject);
```

5、会话子系统可以创建会话、搜索会话、加入会话、删除会话、开始会话和更新会话。调用这些函数需要绑定委托

```c++
// 创建会话
// @param NewCreateSessionData 创建会话的数据
void CreateSession(const FFPOnlineCreateSessionData& NewCreateSessionData);

// 搜索会话
// @param InMaxSearchResults 最大搜索结果
// @param bInDedicatedServer 仅搜索专用服务器的时候输入true
void FindSessions(int32 InMaxSearchResults, bool bInDedicatedServer);

// 加入会话
// @param InSessionResult 搜索会话返回的结果
void JoinSession(const FOnlineSessionSearchResult& InSessionResult);

// 删除会话
void DestroySession();

// 开始会话
void StartSession();

// 更新会话
// @param NewCreateSessionData 更新会话的数据
void UpdateSession(const FFPOnlineCreateSessionData& NewCreateSessionData);

// 完成以上事件调用的委托
FFPOnlineCreateSessionCompleteDelegate OnCreateSessionComplete;
FFPOnlineFindSessionsCompleteDelegate OnFindSessionsComplete;
FFPOnlineJoinSessionCompleteDelegate OnJoinSessionComplete;
FFPOnlineDestroySessionCompleteDelegate OnDestroySessionComplete;
FFPOnlineStartSessionCompleteDelegate OnStartSessionComplete;
FFPOnlineUpdateSessionCompleteDelegate OnUpdateSessionComplete;
```

6、没联机的时候(开始游戏界面)，`APlayerController`不需要继承`FPOnlinePlayerInterface`和添加`FPOnlinePlayerComponent`，这时候如果要获取ID和名称，可以通过`FPOnlineManagerSubsystem`直接获取。也可以设置给玩家状态后，再通过玩家状态获取

```c++
void AMyPlayerState::BeginPlay()
{
	Super::BeginPlay();
	if (UFPOnlineManagerSubsystem* OnlineManager = GetGameInstance()->GetSubsystem<UFPOnlineManagerSubsystem>())
	{
		SetPlayerId(OnlineManager->GetPlayerId());
		SetPlayerName(OnlineManager->GetPlayerName());
	}	
}
```

7、调用`FPOnlinePlayerComponent`的函数添加聊天消息功能

```c++
// 接收聊天消息委托，绑定委托后把消息添加到UI(仅在本地执行)
UPROPERTY(BlueprintAssignable, Category = "FPOnline")
FFPOnlinePlayerReceiveMessageDelegate OnReceiveMessage;

// 发送聊天信息，本地或服务器调用
UFUNCTION(BlueprintCallable, Category = "FPOnline")
void SendChatMessage(const FFPOnlineMessageData& InMessageData);
```
