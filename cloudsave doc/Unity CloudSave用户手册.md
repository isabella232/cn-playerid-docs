# CloudSave用户手册
## 简介
Unity CloudSave服务（简称CloudSave）旨在提供一种非常方便的方法来保存玩家的游戏状态数据，即某些角色的级别/项目或玩家的偏好，保存在云服务器上。使用此服务，玩家可以在不同设备之间同步他们的游戏状态。 例如，在新的平板电脑上启动游戏而不会丢失任何游戏进度。 CloudSave使用[Dataset](#dataset)作为保存游戏状态的存储单元，它支持自定义键值对存储格式。 此外，可以自动检测[冲突](#syncconflict)以帮助解决本地端和服务器端游戏状态数据之间的不一致问题。

## 目录
### <a name="dataset"></a>Dataset
Dataset用作保存游戏状态的存储单元。 每个Dataset由当前玩家identity id和您自定义Dataset名称进行唯一识别。一个Dataset可以存储多个键值对（称为“Record”）作为已保存的游戏状态详细信息。 您将自己定义key和value，请遵循[限制](#limits)并参考[提示](#tips)以使CloudSave正确工作。 同时，Dataset是[同步](#synchronization)的基本单元。
### <a name="syncconflict"></a>冲突
同步已保存的游戏状态数据时，如果本地端数据与服务器端数据存在一些不一致，则可能会出现冲突。 冲突发生在“Record”级别，并且冲突存储的是发生在同一键本地Record和远程Record。请注意，并非所有的不一致都会导致冲突。只有当服务器端记录的更新与本地记录不同时，才会导致冲突。 [解决冲突](#rc)提供了有关如何处理冲突的更多详细信息。
## APIs
### 本地存储
在同步游戏状态数据之前，您首先需要初始化一个本地Dataset：

**打开或者新建 Dataset:**

```IDataset dataset = CloudSave.OpenOrCreateDataset("dataset_name");```

#### **写入 Record:**
```dataset.Put("key", "value");```
####  **写入多个 Records:**
```Dictionary<string, string> records = new Dictionary<string, string>();
records.Add("key_1", "value_1");
records.Add("key_2", "value_2");
dataset.PutAll(records);
```

#### **读 Record:**
```Reord record = dataset.GetRecord("key");```
#### **读 Record 的value值:**
```string value = dataset.Get("key");```
#### **读所有Records:**
```List<Record> records = dataset.GetAllRecords();```
  <br> or:
  
<br>```Dictionary<string, string> records = dataset.GetAll();```
## <a name="synchronization"></a>同步
CloudSave提供了三种同步API，供您同步Dataset：

1. **即时同步:**

 ```dataset.SynchronizeAsync(SyncCallback);```

 这个方法需要在网络可行时候调用，如果网络有问题，则会抛出  ```SyncCanceledException```
 
2. **延时直到网络可达时同步**

 ```dataset.SynchronizeOnConnectivityAsync(SyncCallback);```

  这个方法会等待直到网络可行时会同步
 
3. **延时直到wifi可达时同步**


 ```dataset.SynchronizeOnWifiOnlyAsync(SyncCallback);```
  
   这个方法会等待直到wifi可行会进行同步

**注意**：所有同步方法都是异步的，并且接收ISyncCallback接口的实现，有关该实现，您可以在[处理回调](#hc)中找到更多详细信息。
调用同步方法时，将首先提取服务器端保存的Dataset，以与本地更新数据进行比较。 然后，如果发生任何冲突，则解决冲突，最后一次更新将会被推送到服务器。
如果多个同步方法同时调用，则将仅保留最后一个，而所有先前的方法将引发```SyncCanceledException```。
## <a name="hc"></a>处理回调
通过实现```ISyncCallback```接口，可以处理同步结果：
#### **同步成功时:**

 ```void OnSuccess(IDataset dataset);```
 
 **注意**:这个回调将会在Dataset更新同步成功的时候被触发.

#### **错误发生**

 ``` void OnError(IDataset dataset, DatasetSyncException syncEx);```
 
**注意**:这个```回调方法```将会在同步时抛出```exception``` 的时候被触发，比如之前提过的
```SyncCanceledException```,这意味着同步失败。

#### **冲突被检测到**

 ```bool OnConflict(IDataset dataset, IList<SyncConflict> conflicts);```
 
 **注意**：这个回调方法将会在同步操作时(```pulling```)产生冲突时被触发。你需要解决冲突并且更新 ```Dataset```正确的数据在这个方法。这个方法应该返回``` true ```如果解决完冲突，否则返回```false```。详情请参阅[解决冲突](#rc)。

## <a name="rc"></a>解决冲突

CloudSave默认提供两种策略来解决冲突：

```public Record ResolveWithRemoteRecord();```

这个方法采用服务端的```Record```作为正确的```data```, 从而放弃本地端的更新。

   或者:
   
 
 ```public Record ResolveWithLocalRecord();```

 这个方法采用本地端```Record```作为正确的data, 重写在服务端的```Record```。


通常，您可能希望实施自己的策略来解决冲突。例如，合并本地更新和服务器端的数据，或者让玩家选择他要保留的数据。如果这样做，您将在获取要保存的数据后调用```ResolveWithValue```方法，例如

```Record resolvedRecord = ResolveWithValue("the_value");```


下面给出了```ResolveWithRemoteRecord```解决冲突的简单实现：

```
bool OnConflict(IDataset dataset, IList<SyncConflict> conflicts)
{
	IList<Record> resolved = new List<Record>(conflicts.Count);
	try
	{
		foreach (SyncConflict conf in conflicts)
   		{
   			// resolve with remote Record
   			resolved.Add(conf.ResolveWithRemoteRecord());
   		}
   		// update dataset
    	dataset.ResolveConflicts(resolved);
    	return true;
    }
    catch (Exception e)
    {
    	// do something with e
    	return false;
    }
```

## <a name="limits"></a>限制
下表描述了CloudSave的限制：

| |  | 
| ------ | ------ |
|对每一个identity id最大数量Dataset|20 | 
| Dataset 名字允许最长字符的容量| 200 bytes|
|每个DataSet最大容量 |5 MB|
|一个DataSet允许的最大Record 数量 |1000|
|Record 中key的允许的最长字符容量|64 bytes|

## <a name="tips"></a>使用建议

+ **使Dataset易于管理：**

 对于保存在Dataset中的内容没有任何限制，因为我们希望开发人员在应用CloudSave时不会受到太多限制。 但是，我们仍然强烈建议您为您的保存策略制定周到的计划。

 例如，如果您的游戏支持一个玩家有十个不同的存档，则可以将这些存档的所有游戏状态数据简单地保存在一个Dataset中，例如：
```Dataset:
 	Records:
 		"archive_one_game_state": "...",
 		"archive_two_game_state": "...",
 		...
```

  <br>但是显然，这**不是**一个好主意，因为同步发生在Dataset级别，因此每次同步都会浪费网络资源在未使用的游戏状态数据上。此外，这样的策略将使Dataset很难满足[限制](#limits)。
  
   <br>更好的方式是每个游戏存档使用一个dataset，还可以为每个存档添加一个封面图像。如果有必要，可以将玩家的偏好保存在另一个Dataset里：
  ```
  Dataset_Archive_One:
 	Records:
 		"game_state": "...",
 		"cover_image": "...",
 		...
 Dataset_Archive_Two:
 	Records:
 		"game_state": "...",
 		"cover_image": "...",
 		...
 Dataset_Preference:
 	Records:
 		"locale": "...",
 	 ... 
 ```
 	
 	
+  **善于利用“Record”：**
 
   就像Dataset一样，您决定存储在一条Record中的内容完全取决于您自己的设计。 例如，如果一个玩家在他的游戏中获得了两种武器，**Excalibur**和**Avalon**，那么您可能会将它们另存为两条Record：
```Records:
 	"item_weapon_one": "Excalibur",
 	"item_weapon_two": "Avalon",
 ```
 
   这似乎易于使用，但是在进行同步时可能会发生潜在的错误。 每个记录可能由不同的设备更新，从而使数据变为非法状态。假设游戏里规定只能增强一种武器，如果分别使用两种设备分别增强Excalibur和Avalon的强度，则在同步后的某些情况下，这两种武器都将得到增强。
  
  <br>我们的建议是将游戏状态数据全部记录在一个Record中，以避免上述问题。 尽管您可能需要进行一些额外的数据编码/解码工作，但这将确保数据的正确性。
  
  <br>当然，您可以根据您的游戏设计更复杂的Records使用方法。 例如，为游戏状态数据使用多个Records是为了一定的便利，并使Records之间不相互依赖。


+ **在正确的时间进行同步：**

  最好在初始化Dataset后立即执行同步：
  
 ```dataset = CloudSave.OpenOrCreateDataset("dataset_name");
   dataset.SynchronizeAsync(SyncCallback);
 ```
 
 这将使得保存在服务器上的数据在一开始就获取更新，并尽早处理存在的冲突。

















